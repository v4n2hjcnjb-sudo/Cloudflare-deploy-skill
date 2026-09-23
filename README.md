[cloudflare-deploy-guide.html](https://github.com/user-attachments/files/32545911/cloudflare-deploy-guide.html)




<div class="hero">
  <div class="badge">实战经验</div>
  <h1>Cloudflare Pages 部署实战指南</h1>
  <p class="subtitle">Next.js + Prisma + Neon PostgreSQL 全栈应用部署与避坑全攻略</p>
</div>

<div class="container">

<!-- TOC -->
<div class="toc">
  <h3>目录</h3>
  <ol>
    <li><a href="#intro">为什么选择 Cloudflare Pages</a></li>
    <li><a href="#overview">部署架构总览</a></li>
    <li><a href="#prereq">前置准备</a></li>
    <li><a href="#step1">Step 1: 创建 Cloudflare Pages 项目</a></li>
    <li><a href="#step2">Step 2: next.config.ts 环境检测</a></li>
    <li><a href="#step3">Step 3: Prisma 边缘运行时配置</a></li>
    <li><a href="#step4">Step 4: Neon HTTP 事务限制</a></li>
    <li><a href="#step5">Step 5: 构建配置</a></li>
    <li><a href="#step6">Step 6: 动态页面防预渲染</a></li>
    <li><a href="#step7">Step 7: 环境变量配置</a></li>
    <li><a href="#step8">Step 8: 自定义域名与路由</a></li>
    <li><a href="#step9">Step 9: 部署后验证</a></li>
    <li><a href="#pitfalls">十大避坑速查表</a></li>
    <li><a href="#checklist">上线检查清单</a></li>
  </ol>
</div>

<!-- Intro -->
<h2 id="intro">为什么选择 Cloudflare Pages</h2>
<p>AI 时代的应用开发已经变得极其简单 — 用 AI 写一个全栈网站可能只需要几小时。但真正的挑战在于<strong>如何把它部署到线上让用户访问</strong>。这一步会卡住绝大多数人。</p>
<p>Cloudflare Pages 是目前最佳的全栈应用托管平台之一：</p>
<ul>
  <li><strong>全球 CDN</strong> — 300+ 边缘节点，用户自动就近访问</li>
  <li><strong>免费额度充足</strong> — 每月 500 次构建、无限请求、100K 次/天 Functions 调用</li>
  <li><strong>Git 自动部署</strong> — 推送代码即自动构建上线</li>
  <li><strong>边缘运行时</strong> — 服务端代码直接跑在离用户最近的节点上</li>
</ul>
<p>但 Cloudflare Pages 的边缘运行时与传统 Node.js 服务器有本质区别，直接部署会踩很多坑。本文档基于真实生产项目的部署经验，帮你绕过所有已知陷阱。</p>

<!-- Overview -->
<h2 id="overview">部署架构总览</h2>
<div class="card">
<pre style="margin:0; background:none; border:none; padding:0;">
┌─────────────────────────────────────────────────┐
│              用户浏览器                          │
│   https://yourdomain.com/YourProject/...          │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────┐
│         Cloudflare Pages (边缘运行时)             │
│                                                   │
│  ┌─────────────┐  ┌──────────────────────────┐  │
│  │ 静态资源 CDN  │  │ Edge Functions (API)     │  │
│  │ _next/static │  │ /api/* → 服务端逻辑       │  │
│  └─────────────┘  └──────────┬───────────────┘  │
│                              │                   │
│  basePath: /YourProject     │                   │
│  next.config.ts 自动检测      │                   │
└──────────────────────────────┼──────────────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                 ▼
┌──────────────────┐ ┌───────────────┐ ┌──────────────┐
│  Neon Postgres    │ │  Resend (邮件) │ │  Stripe (支付) │
│  (HTTP 适配器)     │ │  REST API     │ │  Webhook      │
│  无 TCP 连接       │ │               │ │               │
└──────────────────┘ └───────────────┘ └──────────────┘
</pre>
</div>

<!-- Prereq -->
<h2 id="prereq">前置准备</h2>
<ul>
  <li><strong>GitHub 仓库</strong> — 代码已推送，Cloudflare Pages 支持 Git 自动部署</li>
  <li><strong>Neon 数据库</strong> — 在 https://neon.tech 注册免费账号，创建 PostgreSQL 数据库</li>
  <li><strong>Cloudflare 账号</strong> — 在 https://dash.cloudflare.com 注册</li>
  <li><strong>Node.js 20+</strong> — 本地开发环境</li>
</ul>

<!-- Step 1 -->
<h2 id="step1"><span class="step-num">1</span>创建 Cloudflare Pages 项目</h2>
<p>在 Cloudflare Dashboard 中创建 Pages 项目：</p>
<ol>
  <li>进入 <strong>Workers & Pages → Create application → Pages</strong></li>
  <li>选择 <strong>Connect to Git</strong>，授权并选择你的 GitHub 仓库</li>
  <li>配置构建参数：
    <ul>
      <li><strong>Framework preset</strong>: Next.js</li>
      <li><strong>Build command</strong>: <code>npx prisma generate && npm run build</code></li>
      <li><strong>Build output directory</strong>: <code>.next</code></li>
      <li><strong>Root directory</strong>: 你的项目子目录（如 <code>roost-and-co</code>）</li>
    </ul>
  </li>
  <li>先不急着部署 — 还需要配置环境变量（Step 7）</li>
</ol>
<div class="tip">
  <div class="tip-title">关键点</div>
  <p>Build command 中的 <code>npx prisma generate</code> 必须放在 <code>npm run build</code> 前面。Cloudflare 构建环境是全新的，不会使用你本地的 <code>node_modules/.prisma</code> 缓存。如果跳过这一步，构建时会报 <code>Cannot find module '@prisma/client/edge'</code>。</p>
</div>

<!-- Step 2 -->
<h2 id="step2"><span class="step-num">2</span>next.config.ts 环境检测</h2>
<p>Cloudflare Pages 构建时会自动注入 <code>CF_PAGES=1</code> 环境变量。利用这一点实现本地和云端的自动切换：</p>
<pre><code><span class="keyword">const</span> isCloudflare = process.env.<span class="constant">CF_PAGES</span> === <span class="string">"1"</span> || process.env.<span class="constant">CLOUDFLARE</span> === <span class="string">"1"</span>;
<span class="keyword">const</span> basePath = isCloudflare ? <span class="string">"/YourProject"</span> : <span class="string">""</span>;

<span class="keyword">const</span> nextConfig: NextConfig = {
  basePath: basePath || <span class="keyword">undefined</span>,
  env: { <span class="constant">NEXT_PUBLIC_BASE_PATH</span>: basePath },
  outputFileTracingIncludes: {
    <span class="string">"**/*"</span>: [
      <span class="string">"./node_modules/@neondatabase/serverless/**"</span>,
      <span class="string">"./node_modules/@prisma/adapter-neon/**"</span>,
      <span class="string">"./node_modules/.prisma/client/**"</span>,
    ],
  },
};</code></pre>
<div class="pitfall">
  <div class="pitfall-title">basePath 不匹配 → 全站 404</div>
  <p>如果你在 Cloudflare 上用了路径路由（如 <code>teainn.me/Roostco</code>），basePath 必须与 Pages 项目的访问路径一致。如果项目名是 <code>roost-and-co</code>，但你想通过 <code>/Roostco</code> 访问，需要在 Cloudflare Dashboard 中设置自定义路由，同时 <code>next.config.ts</code> 的 basePath 也要设为 <code>"/Roostco"</code>。</p>
</div>

<!-- Step 3 -->
<h2 id="step3"><span class="step-num">3</span>Prisma 边缘运行时配置</h2>
<p>这是部署中最容易出问题的环节。Cloudflare Pages 的边缘运行时<strong>不支持 TCP 连接</strong>，传统的 Prisma PostgreSQL 驱动（基于 TCP）无法工作。必须切换到 HTTP 适配器。</p>

<h3>schema.prisma</h3>
<pre><code>generator client {
  provider = <span class="string">"prisma-client-js"</span>  <span class="comment">// Prisma v7+ 边缘兼容</span>
}

datasource db {
  provider = <span class="string">"postgresql"</span>
}</code></pre>

<h3>src/lib/prisma.ts — 懒加载 Proxy 模式</h3>
<pre><code><span class="keyword">import</span> { PrismaClient } <span class="keyword">from</span> <span class="string">"@prisma/client/edge"</span>;
<span class="keyword">import</span> { PrismaNeonHttp } <span class="keyword">from</span> <span class="string">"@prisma/adapter-neon"</span>;

<span class="keyword">function</span> createPrismaClient(): PrismaClient {
  <span class="keyword">const</span> dbUrl = process.env.<span class="constant">DATABASE_URL</span>!;
  <span class="comment">// 移除 channel_binding 参数 — HTTP 适配器不支持</span>
  <span class="keyword">const</span> cleanedUrl = dbUrl
    .replace(<span class="string">/[?&]channel_binding=[^&]+/</span>, <span class="string">""</span>)
    .replace(<span class="string">/&$/</span>, <span class="string">""</span>);
  <span class="keyword">const</span> adapter = <span class="keyword">new</span> PrismaNeonHttp(cleanedUrl);
  <span class="keyword">return</span> <span class="keyword">new</span> PrismaClient({ adapter, log: [<span class="string">"error"</span>] });
}

<span class="comment">// 懒初始化 — 避免构建时连接数据库</span>
<span class="keyword">export const</span> prisma = <span class="keyword">new</span> Proxy({} <span class="keyword">as</span> PrismaClient, {
  get(_, prop) {
    <span class="keyword">const</span> client = (globalThis <span class="keyword">as</span> any).prisma ?? createPrismaClient();
    (globalThis <span class="keyword">as</span> any).prisma = client;
    <span class="keyword">return</span> Reflect.get(client, prop);
  },
});</code></pre>

<div class="card">
  <h3 style="margin-top:0;">为什么需要 Proxy 懒加载？</h3>
  <p>Cloudflare 构建时会执行 <code>next build</code>，此时 <code>DATABASE_URL</code> 环境变量可能还未注入构建环境。如果 Prisma 客户端在模块加载时直接初始化，会导致构建报错 <code>"DATABASE_URL is not set"</code>。</p>
  <p>Proxy 模式延迟初始化到第一次实际调用数据库时才创建连接，此时运行时环境已完整注入所有环境变量。</p>
</div>

<div class="pitfall">
  <div class="pitfall-title">三个必须遵守的规则</div>
  <ol>
    <li>用 <code>@prisma/client/edge</code> <span class="tag tag-good">正确</span>，不要用 <code>@prisma/client</code> <span class="tag tag-bad">错误</span></li>
    <li>用 <code>PrismaNeonHttp</code>（HTTP fetch）<span class="tag tag-good">正确</span>，不要用 <code>PrismaNeon</code>（WebSocket）<span class="tag tag-bad">错误</span></li>
    <li>不要安装 <code>@prisma/adapter-pg</code>（TCP 驱动，Cloudflare 封锁）<span class="tag tag-bad">错误</span></li>
  </ol>
</div>

<!-- Step 4 -->
<h2 id="step4"><span class="step-num">4</span>Neon HTTP 适配器事务限制</h2>
<p>这是生产环境中最常见的运行时错误。Neon 的 HTTP 适配器<strong>不支持数据库事务</strong>。以下 Prisma 操作会触发隐式事务并在运行时报错：</p>

<table>
  <thead>
    <tr>
      <th>操作</th>
      <th>问题</th>
      <th>替代方案</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>$transaction([...])</code></td>
      <td>显式事务</td>
      <td>按顺序执行查询</td>
    </tr>
    <tr>
      <td><code>updateMany({...})</code></td>
      <td>隐式事务</td>
      <td><code>findMany</code> + 循环 <code>update</code></td>
    </tr>
    <tr>
      <td><code>deleteMany({...})</code></td>
      <td>隐式事务</td>
      <td><code>findMany</code> + 循环 <code>delete</code></td>
    </tr>
    <tr>
      <td>嵌套 <code>create</code></td>
      <td>隐式事务</td>
      <td>拆分为顺序 <code>create</code> 调用</td>
    </tr>
    <tr>
      <td><code>include</code> in update/create</td>
      <td>有时触发事务</td>
      <td>写操作后单独 <code>findUnique</code></td>
    </tr>
  </tbody>
</table>

<h3>正确模式 — 替换 updateMany</h3>
<pre><code><span class="comment">// 错误 — 触发事务，运行时报错</span>
<span class="keyword">await</span> prisma.order.updateMany({
  where: { stripePaymentIntentId: sessionId },
  data: { status: <span class="string">"PAID"</span> },
});

<span class="comment">// 正确 — 逐条更新，避免事务</span>
<span class="keyword">const</span> orders = <span class="keyword">await</span> prisma.order.findMany({
  where: { stripePaymentIntentId: sessionId },
  select: { id: <span class="keyword">true</span> },
});
<span class="keyword">for</span> (<span class="keyword">const</span> order <span class="keyword">of</span> orders) {
  <span class="keyword">await</span> prisma.order.update({
    where: { id: order.id },
    data: { status: <span class="string">"PAID"</span> },
  });
}</code></pre>

<div class="tip">
  <div class="tip-title">正确模式 — 替换嵌套 create</div>
  <pre style="margin-top:8px;"><code><span class="comment">// 错误 — 嵌套 create 触发事务</span>
<span class="keyword">await</span> prisma.user.create({
  data: { email, host: { create: { name } } },
  include: { host: <span class="keyword">true</span> },
});

<span class="comment">// 正确 — 分步创建</span>
<span class="keyword">const</span> user = <span class="keyword">await</span> prisma.user.create({ data: { email } });
<span class="keyword">const</span> host = <span class="keyword">await</span> prisma.host.create({ data: { userId: user.id, name } });</code></pre>
</div>

<!-- Step 5 -->
<h2 id="step5"><span class="step-num">5</span>构建配置</h2>
<h3>package.json</h3>
<pre><code>{
  <span class="string">"scripts"</span>: {
    <span class="string">"build"</span>: <span class="string">"next build --webpack"</span>
  },
  <span class="string">"dependencies"</span>: {
    <span class="string">"@neondatabase/serverless"</span>: <span class="string">"^1.1.0"</span>,
    <span class="string">"@prisma/adapter-neon"</span>: <span class="string">"^7.10.0"</span>,
    <span class="string">"pg-cloudflare"</span>: <span class="string">"^1.4.0"</span>
  }
}</code></pre>

<div class="pitfall">
  <div class="pitfall-title">--webpack 标志是必须的</div>
  <p>Next.js 16 默认使用 Turbopack 构建器。Prisma 的边缘 Wasm 模块与 Turbopack 不兼容，构建时会报 Wasm/asyncify 相关错误。必须在 <code>npm run build</code> 命令中添加 <code>--webpack</code> 标志切换回 Webpack 构建器。</p>
</div>

<!-- Step 6 -->
<h2 id="step6"><span class="step-num">6</span>动态页面防预渲染</h2>
<p>使用国际化路由（如 next-intl）的页面在构建时会尝试静态预渲染，但此时数据库不可用，导致构建失败。</p>
<pre><code><span class="comment">// 在所有使用 locale 参数或数据库查询的页面中添加</span>
<span class="keyword">export const</span> dynamic = <span class="string">"force-dynamic"</span>;</code></pre>
<p>需要添加此行的典型页面：</p>
<ul>
  <li><code>/[locale]/page.tsx</code> — 首页</li>
  <li><code>/[locale]/stays/[slug]/page.tsx</code> — 动态详情页</li>
  <li><code>/[locale]/artworks/[slug]/page.tsx</code> — 动态详情页</li>
  <li><code>/[locale]/lead/page.tsx</code> — 表单提交页</li>
  <li>所有 <code>/admin/*</code> 页面</li>
</ul>

<!-- Step 7 -->
<h2 id="step7"><span class="step-num">7</span>环境变量配置</h2>
<p>在 Cloudflare Dashboard 中为 Production 环境配置以下变量：</p>
<table>
  <thead>
    <tr><th>变量名</th><th>用途</th><th>示例值</th></tr>
  </thead>
  <tbody>
    <tr><td><code>DATABASE_URL</code></td><td>Neon 数据库连接串</td><td><code>postgresql://...neon.tech/neondb</code></td></tr>
    <tr><td><code>NEXTAUTH_SECRET</code></td><td>JWT 签名密钥</td><td><code>openssl rand -base64 32</code></td></tr>
    <tr><td><code>NEXTAUTH_URL</code></td><td>生产域名</td><td><code>https://yourdomain.com</code></td></tr>
    <tr><td><code>NEXT_PUBLIC_SITE_URL</code></td><td>客户端公开 URL</td><td><code>https://yourdomain.com</code></td></tr>
    <tr><td><code>CLOUDFLARE</code></td><td>手动检测标志</td><td><code>1</code></td></tr>
    <tr><td><code>RESEND_API_KEY</code></td><td>邮件发送密钥</td><td><code>re_...</code></td></tr>
    <tr><td><code>STRIPE_SECRET_KEY</code></td><td>Stripe 支付密钥</td><td><code>sk_live_...</code></td></tr>
    <tr><td><code>STRIPE_WEBHOOK_SECRET</code></td><td>Webhook 签名密钥</td><td><code>whsec_...</code></td></tr>
  </tbody>
</table>
<div class="tip">
  <div class="tip-title">关于 NEXTAUTH_URL</div>
  <p>这个变量必须改为你的生产域名。如果仍然是 <code>http://localhost:3000</code>，登录后的回调会跳转到 localhost，导致认证失败。如果你的站点有 basePath（如 <code>/Roostco</code>），不需要在 NEXTAUTH_URL 中包含 basePath，NextAuth 会自动处理。</p>
</div>

<!-- Step 8 -->
<h2 id="step8"><span class="step-num">8</span>自定义域名与路由</h2>
<ol>
  <li>在 Cloudflare Pages 项目设置中添加自定义域名</li>
  <li>如果使用子路径路由（如 <code>domain.com/YourProject/</code>），确保 basePath 一致</li>
  <li>如果使用根域名，basePath 设为空</li>
  <li>DNS 配置：Cloudflare 会自动添加 CNAME 记录</li>
</ol>
<div class="pitfall">
  <div class="pitfall-title">第三方 Webhook URL 需要包含 basePath</div>
  <p>Stripe、Resend 等第三方服务的 Webhook 回调 URL 必须包含 basePath。例如，Stripe Webhook endpoint 应配置为 <code>https://yourdomain.com/YourProject/api/webhooks/stripe</code>，否则会返回 404。</p>
</div>

<!-- Step 9 -->
<h2 id="step9"><span class="step-num">9</span>部署后验证</h2>
<ol>
  <li><strong>健康检查</strong>：访问 <code>https://yourdomain.com/api/diagnostics</code>，确认 <code>ok: true</code></li>
  <li><strong>数据库连通性</strong>：检查诊断接口返回的 <code>database.ok</code> 是否为 <code>true</code></li>
  <li><strong>认证流程</strong>：测试登录 → 验证 session cookie → 验证 30 分钟空闲超时</li>
  <li><strong>路由完整性</strong>：所有内部链接正常跳转，不出现 404</li>
  <li><strong>静态资源</strong>：图片、CSS、JS 正常加载</li>
  <li><strong>API 接口</strong>：所有 <code>/api/*</code> 端点正常响应</li>
</ol>

<!-- Pitfalls -->
<h2 id="pitfalls">十大避坑速查表</h2>

<div class="card">
<div class="pitfall">
  <div class="pitfall-title">1. 构建报错: Cannot find module @prisma/client/edge</div>
  <p><strong>原因</strong>：Prisma 客户端未在构建环境中重新生成</p>
  <p><strong>解决</strong>：确保 build command 包含 <code>npx prisma generate</code>，放在 <code>npm run build</code> 前面</p>
</div>

<div class="pitfall">
  <div class="pitfall-title">2. 运行时报错: Transactions are not supported in HTTP mode</div>
  <p><strong>原因</strong>：使用了 <code>updateMany</code>、<code>deleteMany</code>、<code>$transaction</code> 或嵌套 <code>create</code></p>
  <p><strong>解决</strong>：用 <code>findMany</code> + 循环 <code>update</code> 替代，详见 Step 4</p>
</div>

<div class="pitfall">
  <div class="pitfall-title">3. 构建报错: Wasm/asyncify 相关错误</div>
  <p><strong>原因</strong>：Next.js 16 默认用 Turbopack，与 Prisma 边缘 Wasm 不兼容</p>
  <p><strong>解决</strong>：使用 <code>next build --webpack</code>（加 <code>--webpack</code> 标志）</p>
</div>

<div class="pitfall">
  <div class="pitfall-title">4. 构建报错: DATABASE_URL is not set</div>
  <p><strong>原因</strong>：Prisma 客户端在模块加载时直接初始化</p>
  <p><strong>解决</strong>：使用 Proxy 懒加载模式，延迟到运行时才创建连接</p>
</div>

<div class="pitfall">
  <div class="pitfall-title">5. 运行时报错: channel_binding is not supported</div>
  <p><strong>原因</strong>：Neon 连接串包含 <code>channel_binding=require</code></p>
  <p><strong>解决</strong>：在创建 Prisma 客户端前移除该参数</p>
</div>

<div class="pitfall">
  <div class="pitfall-title">6. 部署后全站 404</div>
  <p><strong>原因</strong>：basePath 与 Cloudflare Pages 项目路由路径不匹配</p>
  <p><strong>解决</strong>：确保 <code>next.config.ts</code> 中的 basePath 与 Cloudflare 访问路径一致</p>
</div>

<div class="pitfall">
  <div class="pitfall-title">7. 登录后跳转到 localhost</div>
  <p><strong>原因</strong>：<code>NEXTAUTH_URL</code> 未更新为生产域名</p>
  <p><strong>解决</strong>：在 Cloudflare 环境变量中设为 <code>https://yourdomain.com</code></p>
</div>

<div class="pitfall">
  <div class="pitfall-title">8. 图片不显示</div>
  <p><strong>原因</strong>：外部图片域名未在 next.config.ts 的 remotePatterns 中声明</p>
  <p><strong>解决</strong>：将所有外部图片来源域名添加到 <code>images.remotePatterns</code></p>
</div>

<div class="pitfall">
  <div class="pitfall-title">9. Stripe Webhook 返回 404</div>
  <p><strong>原因</strong>：Webhook URL 未包含 basePath</p>
  <p><strong>解决</strong>：Stripe Dashboard 中的 endpoint URL 需包含完整路径（含 basePath）</p>
</div>

<div class="pitfall">
  <div class="pitfall-title">10. 构建时 locale 页面返回 500</div>
  <p><strong>原因</strong>：静态预渲染尝试在构建时访问数据库</p>
  <p><strong>解决</strong>：在所有 locale 相关页面添加 <code>export const dynamic = "force-dynamic"</code></p>
</div>
</div>

<!-- Checklist -->
<h2 id="checklist">上线检查清单</h2>
<div class="card">
  <h3 style="margin-top:0;">部署前</h3>
  <ul>
    <li>☐ <code>next.config.ts</code> 包含 Cloudflare 环境检测和 basePath</li>
    <li>☐ <code>src/lib/prisma.ts</code> 使用 Proxy 懒加载 + <code>PrismaNeonHttp</code></li>
    <li>☐ <code>package.json</code> 的 build 脚本包含 <code>--webpack</code> 标志</li>
    <li>☐ 所有 locale 页面添加 <code>export const dynamic = "force-dynamic"</code></li>
    <li>☐ 没有使用 <code>updateMany</code>、<code>deleteMany</code>、<code>$transaction</code>、嵌套 <code>create</code></li>
    <li>☐ <code>outputFileTracingIncludes</code> 包含 Neon 和 Prisma 适配器路径</li>
    <li>☐ <code>.env</code> 文件不在 git 中提交（生产密钥不入仓库）</li>
  </ul>
  <h3>部署后</h3>
  <ul>
    <li>☐ <code>/api/diagnostics</code> 返回 <code>ok: true</code></li>
    <li>☐ 数据库连通性正常</li>
    <li>☐ 登录/登出流程正常</li>
    <li>☐ 所有页面路由可访问</li>
    <li>☐ 静态资源（图片、CSS、JS）正常加载</li>
    <li>☐ 第三方 Webhook URL 配置正确（含 basePath）</li>
    <li>☐ <code>NEXTAUTH_URL</code> 指向生产域名</li>
    <li>☐ 自定义域名 DNS 解析正常</li>
  </ul>
</div>

<div style="text-align:center; margin-top:48px; padding-top:24px; border-top:1px solid var(--border); color:var(--text-muted); font-size:0.875rem;">
  <p>本文档基于 Roost & Co. 项目真实生产部署经验整理</p>
  <p>适用于 Next.js 16 + Prisma v7 + Neon PostgreSQL + Cloudflare Pages 技术栈</p>
</div>

</div>
</body>
</html>
