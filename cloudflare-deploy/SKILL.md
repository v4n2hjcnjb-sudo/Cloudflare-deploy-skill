---
name: "cloudflare-deploy"
description: "Deploy Next.js/full-stack apps to Cloudflare Pages with Prisma+Neon. Invoke when deploying to Cloudflare, configuring basePath, fixing Prisma edge runtime errors, or troubleshooting Cloudflare build failures."
---

# Cloudflare Pages Deployment Guide

Complete deployment playbook for Next.js + Prisma + Neon PostgreSQL on Cloudflare Pages. Based on real-world production deployment experience.

## When to Invoke

- Deploying a Next.js app to Cloudflare Pages
- Configuring Prisma for Cloudflare edge runtime
- Fixing Cloudflare build or runtime errors
- Setting up environment variables and custom domains
- Troubleshooting "Transactions are not supported in HTTP mode"
- Resolving basePath / routing issues on Cloudflare

---

## Step 1: Cloudflare Pages Project Setup

1. Go to https://dash.cloudflare.com → Workers & Pages → Create application → Pages → Upload assets (or connect Git)
2. **Framework Preset**: Next.js
3. **Build command**: `npx prisma generate && npm run build`
4. **Build output directory**: `.next`
5. **Root directory**: your project root (e.g., `roost-and-co`)
6. **Environment variables** (Production):
   ```
   DATABASE_URL=postgresql://...  (Neon connection string)
   NEXTAUTH_SECRET=<openssl-rand-base64-32>
   NEXTAUTH_URL=https://yourdomain.com
   NEXT_PUBLIC_SITE_URL=https://yourdomain.com
   CLOUDFLARE=1
   ```
   > `CF_PAGES=1` is auto-injected by Cloudflare at build time. Set `CLOUDFLARE=1` as backup.

---

## Step 2: next.config.ts Cloudflare Detection

```typescript
const isCloudflare = process.env.CF_PAGES === "1" || process.env.CLOUDFLARE === "1";
const basePath = isCloudflare ? "/YourProject" : "";

const nextConfig: NextConfig = {
  basePath: basePath || undefined,
  env: { NEXT_PUBLIC_BASE_PATH: basePath },
  outputFileTracingIncludes: {
    "**/*": [
      "./node_modules/@neondatabase/serverless/**",
      "./node_modules/@prisma/adapter-neon/**",
      "./node_modules/.prisma/client/**",
    ],
  },
};
```

**Pitfall**: If using a custom domain with path routing (e.g., `teainn.me/Roostco`), basePath must match the Cloudflare Pages project name. Mismatch causes 404 on all routes.

---

## Step 3: Prisma Edge Runtime Configuration

### schema.prisma

```prisma
generator client {
  provider = "prisma-client-js"   // Prisma v7+ with edge support
}

datasource db {
  provider = "postgresql"
}
```

### src/lib/prisma.ts (Lazy Proxy Pattern)

```typescript
import { PrismaClient } from "@prisma/client/edge";
import { PrismaNeonHttp } from "@prisma/adapter-neon";

function createPrismaClient(): PrismaClient {
  const dbUrl = process.env.DATABASE_URL!;
  // Remove channel_binding param — HTTP adapter doesn't support it
  const cleanedUrl = dbUrl.replace(/[?&]channel_binding=[^&]+/, "").replace(/&$/, "");
  const adapter = new PrismaNeonHttp(cleanedUrl, { arrayMode: false, fullResults: false });
  return new PrismaClient({ adapter, log: ["error"] });
}

// Lazy init via Proxy — avoids build-time DB connection errors
export const prisma = new Proxy({} as PrismaClient, {
  get(_, prop) {
    const client = (globalThis as any).prisma ?? createPrismaClient();
    (globalThis as any).prisma = client;
    return Reflect.get(client, prop);
  },
});
```

**Critical pitfalls (see Pitfall Guide below for details):**
- Must use `@prisma/client/edge`, NOT `@prisma/client`
- Must use `PrismaNeonHttp` (HTTP fetch), NOT `PrismaNeon` (WebSocket)
- Must remove `channel_binding` param from connection string
- Must use lazy Proxy — direct init causes build-time `DATABASE_URL not found`

---

## Step 4: Neon HTTP Adapter Transaction Limitations

**The #1 runtime error**: `"Transactions are not supported in HTTP mode"`

Neon's HTTP adapter cannot do database transactions. The following Prisma operations trigger implicit transactions and WILL FAIL at runtime:

| Operation | Problem | Fix |
|-----------|---------|-----|
| `prisma.$transaction([...])` | Explicit transaction | Run queries sequentially |
| `updateMany({ where, data })` | Implicit transaction | `findMany` + loop `update` |
| `deleteMany({ where })` | Implicit transaction | `findMany` + loop `delete` |
| Nested `create` (e.g., `create: { author: { create: {...} } })` | Transaction | Split into sequential `create` calls |
| `include` in `update`/`create` | Sometimes triggers transaction | Separate `findUnique` after write |

**Safe pattern** — replace `updateMany`:
```typescript
// BAD — triggers transaction
await prisma.order.updateMany({
  where: { stripePaymentIntentId: sessionId },
  data: { status: "PAID" },
});

// GOOD — individual updates
const orders = await prisma.order.findMany({
  where: { stripePaymentIntentId: sessionId },
  select: { id: true },
});
for (const order of orders) {
  await prisma.order.update({
    where: { id: order.id },
    data: { status: "PAID" },
  });
}
```

---

## Step 5: Build Configuration

### package.json build script

```json
{
  "scripts": {
    "build": "next build --webpack"
  }
}
```

**Pitfall**: Next.js 16 defaults to Turbopack. Prisma's edge Wasm modules are **incompatible with Turbopack**. The `--webpack` flag is mandatory. Without it, build fails with Wasm-related errors.

### Dependencies (package.json)

```json
{
  "dependencies": {
    "@neondatabase/serverless": "^1.1.0",
    "@prisma/adapter-neon": "^7.10.0",
    "pg-cloudflare": "^1.4.0"
  }
}
```

> Do NOT install `@prisma/adapter-pg` — it uses TCP sockets, which Cloudflare Workers/Pages runtime blocks. Only `@prisma/adapter-neon` (HTTP-based) works.

---

## Step 6: Locale & Dynamic Pages

Pages using server-side locale resolution or database queries must prevent static pre-rendering:

```typescript
// Add to ANY page that uses dynamic locale params or DB queries
export const dynamic = "force-dynamic";
```

**Pitfall**: Without this, `next build` tries to statically pre-render pages like `/[locale]/lead`, causing build-time DB access errors on Cloudflare.

---

## Step 7: Post-Deploy Verification

1. **Health check**: Visit `https://yourdomain.com/api/diagnostics` — should return JSON with DB status
2. **Database connectivity**: Check `db.ok === true` in diagnostics response
3. **Auth flow**: Test login → verify session cookie set → verify 30-min idle timeout works
4. **Routing**: Verify basePath is correctly applied — all internal links should work
5. **Static assets**: Images, CSS, JS should load (check for basePath in asset URLs)

---

## Pitfall Quick Reference

### 1. Build fails: "Cannot find module @prisma/client/edge"
- **Cause**: Prisma client not regenerated after schema changes
- **Fix**: Run `npx prisma generate` before `npm run build`

### 2. Runtime: "Transactions are not supported in HTTP mode"
- **Cause**: Used `updateMany`, `deleteMany`, `$transaction`, or nested `create`
- **Fix**: See Step 4 — replace with sequential individual operations

### 3. Build fails: Wasm/asyncify related error
- **Cause**: Using Turbopack (Next.js 16 default) with Prisma edge modules
- **Fix**: Use `next build --webpack` (add `--webpack` flag)

### 4. Build fails: "DATABASE_URL is not set"
- **Cause**: Prisma client initialized at build time, not runtime
- **Fix**: Use lazy Proxy pattern (Step 3). Ensure env var is set in Cloudflare Dashboard

### 5. Runtime: "channel_binding is not supported"
- **Cause**: Neon connection string contains `channel_binding=require`
- **Fix**: Strip it: `url.replace(/[?&]channel_binding=[^&]+/, "")`

### 6. 404 on all routes after deploy
- **Cause**: basePath mismatch between `next.config.ts` and Cloudflare Pages project name
- **Fix**: Ensure `basePath` matches your Cloudflare Pages project route path

### 7. Auth redirects to wrong URL
- **Cause**: `NEXTAUTH_URL` not updated to production domain
- **Fix**: Set `NEXTAUTH_URL=https://yourdomain.com` in Cloudflare env vars

### 8. Images not loading on deployed site
- **Cause**: Remote image hostnames not in `next.config.ts` `remotePatterns`
- **Fix**: Add all external image domains to `images.remotePatterns`

### 9. Webhook route returns 405/404
- **Cause**: Webhook URL doesn't include basePath
- **Fix**: Stripe webhook endpoint URL must include basePath (e.g., `https://domain.com/Roostco/api/webhooks/stripe`)

### 10. Locale pages return 500 during build
- **Cause**: Static pre-rendering tries to access DB at build time
- **Fix**: Add `export const dynamic = "force-dynamic"` to all locale-specific pages

---

## Environment Variable Checklist

| Variable | Where | Purpose |
|----------|-------|---------|
| `DATABASE_URL` | Cloudflare Dashboard | Neon PostgreSQL connection string |
| `NEXTAUTH_SECRET` | Cloudflare Dashboard | JWT signing secret |
| `NEXTAUTH_URL` | Cloudflare Dashboard | Production URL (https://...) |
| `NEXT_PUBLIC_SITE_URL` | Cloudflare Dashboard | Public site URL for client-side |
| `CF_PAGES` | Auto-injected | Cloudflare build detection |
| `CLOUDFLARE` | Set manually | Backup detection flag |
| `NEXT_PUBLIC_BASE_PATH` | Auto via next.config.ts | basePath for client components |
