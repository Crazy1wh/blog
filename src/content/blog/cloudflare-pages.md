# 从 VPS 到 Cloudflare Pages：我的静态网站自动部署流程记录

这篇文章记录一下我把一个 React/Vite 静态工具站，从传统 VPS + 1Panel 部署，迁移到 GitHub + Cloudflare Pages 自动部署的完整过程。

最终实现的效果是：

```text
本地修改代码
↓
推送到 GitHub
↓
Cloudflare Pages 自动构建
↓
自动发布到 51qdd.com
```

以后更新网站，只需要执行：

```bash
git add .
git commit -m "更新内容"
git push
```

Cloudflare 就会自动完成构建和发布，不再需要手动上传 `dist` 文件，也不需要登录 1Panel 修改 Nginx 或重载 OpenResty。

---

## 一、原来的部署方式

最开始，我的网站是部署在香港 VPS 上的，使用的是 1Panel 面板。

整体结构大概是：

```text
用户
↓
Cloudflare
↓
香港 VPS
↓
1Panel
↓
OpenResty / Nginx
↓
静态网站目录
```

域名最开始是：

```text
finance.51qdd.com
```

对应的服务器目录类似：

```text
/www/sites/finance.51qdd.com/index
```

每次网站更新时，需要手动执行：

```bash
npm run build
```

然后把生成出来的静态文件上传到 1Panel 的网站目录。

例如 Vite 项目构建后会生成：

```text
dist/
├── assets/
├── favicon.svg
└── index.html
```

上传到服务器后，目录应该类似：

```text
/www/sites/finance.51qdd.com/index/
├── assets/
├── favicon.svg
└── index.html
```

这种方式能用，但是维护起来比较麻烦。

每次改代码都要：

```text
本地构建
↓
找到 dist 文件
↓
上传到 1Panel
↓
覆盖旧文件
↓
如果路由有问题，还要改 Nginx
```

对于一个纯静态网站来说，这种方式有点重。

---

## 二、项目技术栈

我的项目是一个理财计算工具站，主要用于存款收益、贷款月供、复利收益、日期规划等计算。

技术栈是：

```text
React
Vite
Tailwind CSS
Cloudflare Pages
GitHub
```

项目目录大致如下：

```text
fiance/
├── apps/
│   └── web/
│       ├── public/
│       ├── src/
│       ├── package.json
│       ├── vite.config.js
│       └── index.html
├── package.json
├── package-lock.json
└── README.md
```

其中真正的网站代码在：

```text
apps/web
```

---

## 三、本地运行项目

进入项目目录：

```bash
cd apps/web
```

安装依赖：

```bash
npm install
```

启动开发服务：

```bash
npm run dev
```

Vite 默认会启动一个本地开发地址，通常是：

```text
http://localhost:5173
```

如果想停止本地开发服务，在终端里按：

```text
Ctrl + C
```

即可结束。

---

## 四、本地构建静态文件

在项目目录执行：

```bash
npm run build
```

构建成功后，会生成：

```text
dist/
├── assets/
├── favicon.svg
└── index.html
```

这个 `dist` 目录就是最终可以部署到静态托管平台的文件。

如果是部署到 1Panel，只需要上传 `dist` 里面的内容即可。

注意，不是上传整个 `dist` 文件夹，而是上传里面的内容。

正确结构：

```text
网站根目录/
├── assets/
├── favicon.svg
└── index.html
```

错误结构：

```text
网站根目录/
└── dist/
    ├── assets/
    ├── favicon.svg
    └── index.html
```

---

## 五、VPS + Nginx 部署时遇到的问题

因为 React/Vite 项目通常是单页应用，直接访问首页没有问题，但是刷新子页面时可能会出现 404。

例如：

```text
https://finance.51qdd.com/tools/deposit-yield
```

第一次从首页点击进入正常，但如果直接刷新页面，就会显示：

```text
404 Not Found
openresty
```

原因是服务器会去查找真实路径：

```text
/tools/deposit-yield
```

但是静态目录里并没有这个文件。

真实存在的只有：

```text
index.html
assets/
```

所以需要让所有路由都回退到 `index.html`。

Nginx 配置需要加入：

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

完整示例：

```nginx
server {
    listen 80;
    listen 443 ssl http2;
    server_name finance.51qdd.com;

    root /www/sites/finance.51qdd.com/index;
    index index.html index.htm;

    access_log /www/sites/finance.51qdd.com/log/access.log main;
    error_log /www/sites/finance.51qdd.com/log/error.log;

    if ($scheme = http) {
        return 301 https://$host$request_uri;
    }

    location ^~ /.well-known/acme-challenge {
        allow all;
        root /usr/share/nginx/html;
    }

    location / {
        try_files $uri $uri/ /index.html;
    }

    error_page 404 /index.html;

    ssl_certificate /www/sites/finance.51qdd.com/ssl/fullchain.pem;
    ssl_certificate_key /www/sites/finance.51qdd.com/ssl/privkey.pem;
    ssl_protocols TLSv1.3 TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    error_page 497 https://$host$request_uri;

    add_header Strict-Transport-Security "max-age=31536000";
}
```

这个配置可以解决 React 子路由刷新 404 的问题。

但是后来我发现，对于纯静态网站来说，其实完全没有必要继续依赖 VPS。

---

## 六、为什么迁移到 Cloudflare Pages

我的网站是纯静态网站：

```text
无数据库
无登录系统
无后端接口
无文件上传
所有计算都在浏览器本地完成
```

这种网站非常适合托管到 Cloudflare Pages。

迁移后结构变成：

```text
用户
↓
Cloudflare 边缘节点
↓
Cloudflare Pages
↓
静态文件
```

相比 VPS 部署，Cloudflare Pages 有几个优势：

```text
不需要服务器
不需要维护 Nginx
不需要手动申请 SSL
不需要手动上传文件
支持 GitHub 自动部署
全球 CDN 加速
免费额度足够个人网站使用
```

对于个人博客、工具站、文档站来说，非常合适。

---

## 七、把代码上传到 GitHub

如果项目还没有初始化 Git，可以先执行：

```bash
git init
```

添加所有文件：

```bash
git add .
```

提交：

```bash
git commit -m "Initial commit"
```

关联 GitHub 仓库：

```bash
git remote add origin https://github.com/Crazy1wh/fiance.git
```

把分支改成 `main`：

```bash
git branch -M main
```

推送到 GitHub：

```bash
git push -u origin main
```

后续每次更新，只需要：

```bash
git add .
git commit -m "更新网站内容"
git push
```

---

## 八、创建 Cloudflare Pages 项目

进入 Cloudflare 后台：

```text
Workers & Pages
↓
Create application
↓
Pages
↓
Connect to Git
```

然后选择 GitHub 账号和仓库。

例如：

```text
GitHub account: Crazy1wh
Repository: fiance
```

点击：

```text
Begin setup
```

---

## 九、Cloudflare Pages 构建配置

因为我的项目是 React + Vite，所以 Framework Preset 选择：

```text
React static
```

如果你的 Cloudflare 后台有 Vite，也可以选择 Vite。

我的项目代码在：

```text
apps/web
```

所以配置如下：

```text
Framework preset: React static
Root directory: apps/web
Build command: npm run build
Build output directory: dist
```

也就是：

```text
Root directory:
apps/web

Build command:
npm run build

Build output directory:
dist
```

这样 Cloudflare 会进入 `apps/web` 目录，然后执行：

```bash
npm install
npm run build
```

构建完成后，把 `apps/web/dist` 作为网站根目录发布。

---

## 十、第一次部署

配置完成后，点击：

```text
Save and Deploy
```

Cloudflare 会开始构建。

如果成功，会看到类似：

```text
Building application
Deploying to Cloudflare's global network
Success
```

Cloudflare 会生成一个默认域名：

```text
fiance.pages.dev
```

可以先访问这个地址测试：

```text
https://fiance.pages.dev
```

还要测试子页面：

```text
https://fiance.pages.dev/tools/deposit-yield
```

如果首页和子页面都能正常打开，说明部署成功。

---

## 十一、绑定自定义域名

部署成功后，进入项目：

```text
Workers & Pages
↓
fiance
↓
Custom domains
↓
Set up a custom domain
```

输入自己的域名：

```text
51qdd.com
```

Cloudflare 会提示创建 DNS 记录。

如果根域名之前已经有 A 记录指向 VPS，例如：

```text
Type: A
Name: 51qdd.com
Content: 38.76.168.151
```

Cloudflare 会提示将其替换为 Pages 对应的记录。

最终会变成类似：

```text
Type: CNAME
Name: 51qdd.com
Content: fiance.pages.dev
TTL: Auto
```

如果绑定的是根域名，Cloudflare 可能会提示：

```text
CNAME records normally can not be on the zone apex. We use CNAME flattening to make it possible.
```

这个不是错误。

意思是普通 DNS 不允许根域名直接使用 CNAME，但 Cloudflare 通过 CNAME Flattening 技术自动处理。

点击：

```text
Activate domain
```

等待生效。

---

## 十二、DNS 切换前要注意什么

如果原来的域名还在 VPS 上运行，例如：

```text
finance.51qdd.com
```

不要一开始就删除 VPS 上的网站。

建议顺序是：

```text
先部署 Cloudflare Pages
↓
测试 pages.dev 默认域名
↓
绑定自定义域名
↓
确认 51qdd.com 正常访问
↓
再考虑删除 VPS 上的旧站点
```

这样可以避免网站中断。

---

## 十三、旧域名 301 跳转到新域名

我之前还使用过：

```text
js.51qdd.com
finance.51qdd.com
```

后来主站统一到了：

```text
51qdd.com
```

为了避免多个域名重复内容影响 SEO，需要把旧域名 301 到主域名。

进入 Cloudflare：

```text
Rules
↓
Redirect Rules
↓
Create Rule
```

创建规则。

规则名称：

```text
js-to-root
```

匹配条件使用 Custom filter expression：

```text
http.host eq "js.51qdd.com"
```

目标跳转到：

```text
https://51qdd.com
```

状态码选择：

```text
301 Permanent Redirect
```

如果需要保留路径，效果应该是：

```text
https://js.51qdd.com/about
↓
https://51qdd.com/about
```

```text
https://js.51qdd.com/tools/deposit-yield
↓
https://51qdd.com/tools/deposit-yield
```

如果界面支持保留路径和查询参数，勾选：

```text
Preserve path
Preserve query string
```

如果需要使用表达式，可以使用类似：

```text
concat("https://51qdd.com", http.request.uri.path)
```

如果还要保留查询参数，则需要根据 Cloudflare 当前界面选项选择 Preserve query string。

---

## 十四、为什么要统一域名

如果同一个网站同时存在多个入口：

```text
51qdd.com
js.51qdd.com
finance.51qdd.com
```

搜索引擎可能会认为这是重复内容。

比较好的做法是：

```text
51qdd.com          主站
www.51qdd.com      301 到主站
js.51qdd.com       301 到主站
finance.51qdd.com  301 到主站
```

这样可以把 SEO 权重集中到一个域名。

---

## 十五、现在的最终架构

迁移完成后的架构如下：

```text
GitHub
↓
Cloudflare Pages
↓
51qdd.com
```

静态网站不再依赖 VPS。

香港 VPS 只保留给这些服务使用：

```text
1Panel
OpenClaw
Telegram Bot
需要后端的服务
数据库服务
```

前端静态站点完全交给 Cloudflare Pages。

---

## 十六、日常更新流程

以后开发网站，只需要本地修改代码。

启动本地开发：

```bash
npm run dev
```

浏览器访问：

```text
http://localhost:5173
```

修改完成后提交：

```bash
git add .
git commit -m "优化工具页 SEO 内容"
git push
```

Cloudflare Pages 会自动检测 GitHub 的更新，然后自动执行：

```bash
npm install
npm run build
```

发布成功后，访问：

```text
https://51qdd.com
```

即可看到最新版本。

---

## 十七、如果构建失败怎么办

Cloudflare Pages 构建失败时，不会影响线上旧版本。

也就是说：

```text
本次构建失败
↓
线上网站仍然保持上一个成功版本
```

可以进入：

```text
Workers & Pages
↓
项目
↓
Deployments
↓
查看 Build logs
```

常见错误有：

### 1. 找不到 package.json

可能是 Root directory 配错了。

如果项目结构是：

```text
fiance/
└── apps/
    └── web/
        └── package.json
```

Root directory 应该填：

```text
apps/web
```

### 2. 找不到 dist

可能是 Build output directory 配错了。

如果构建后生成：

```text
apps/web/dist
```

并且 Root directory 是：

```text
apps/web
```

那么 Build output directory 应该填：

```text
dist
```

### 3. npm run build 失败

需要本地先测试：

```bash
npm run build
```

如果本地都失败，Cloudflare 也会失败。

先修复本地构建问题，再提交。

---

## 十八、React 路由刷新 404 的问题

在 VPS + Nginx 部署时，需要配置：

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

Cloudflare Pages 对常见静态站点支持更好，但如果依然遇到子页面刷新 404，可以在项目 `public` 目录下新增 `_redirects` 文件。

路径：

```text
apps/web/public/_redirects
```

内容：

```text
/* /index.html 200
```

这样构建后 `_redirects` 会进入 `dist` 目录，Cloudflare Pages 会按照这个规则处理路由。

适用于这种场景：

```text
访问 /tools/deposit-yield
↓
Cloudflare 找不到真实文件
↓
回退到 /index.html
↓
React Router 接管路由
```

---

## 十九、robots.txt 配置

为了方便搜索引擎抓取，可以在 `public` 目录创建：

```text
apps/web/public/robots.txt
```

内容：

```txt
User-agent: *
Allow: /

Sitemap: https://51qdd.com/sitemap.xml
```

---

## 二十、sitemap.xml 配置

可以在 `public` 目录创建：

```text
apps/web/public/sitemap.xml
```

示例：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset
  xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">

  <url>
    <loc>https://51qdd.com/</loc>
    <priority>1.0</priority>
  </url>

  <url>
    <loc>https://51qdd.com/about</loc>
    <priority>0.8</priority>
  </url>

  <url>
    <loc>https://51qdd.com/privacy</loc>
    <priority>0.8</priority>
  </url>

  <url>
    <loc>https://51qdd.com/disclaimer</loc>
    <priority>0.8</priority>
  </url>

  <url>
    <loc>https://51qdd.com/contact</loc>
    <priority>0.8</priority>
  </url>

  <url>
    <loc>https://51qdd.com/tools/deposit-yield</loc>
    <priority>0.9</priority>
  </url>

</urlset>
```

如果工具页面很多，建议用脚本自动生成 sitemap。

---

## 二十一、Google Search Console

部署完成后，可以把站点添加到 Google Search Console。

地址：

```text
https://search.google.com/search-console
```

添加属性：

```text
51qdd.com
```

如果域名 DNS 在 Cloudflare，推荐使用 DNS 验证。

验证完成后，提交：

```text
https://51qdd.com/sitemap.xml
```

然后等待 Google 抓取和收录。

可以用以下命令检查是否收录：

```text
site:51qdd.com
```

---

## 二十二、Google AdSense 前的准备

如果后续要申请 Google AdSense，建议网站至少具备：

```text
关于我们页面
隐私政策页面
免责声明页面
联系我们页面
15 个以上有实际功能的工具页面
每个工具页面有说明文字
每个工具页面有 FAQ
robots.txt
sitemap.xml
HTTPS
移动端适配
```

不要刚上线就马上放广告。

更稳妥的做法是：

```text
先完善内容
↓
提交 Google Search Console
↓
等待部分页面收录
↓
再申请 AdSense
```

---

## 二十三、Cloudflare Pages 和 VPS 的区别

VPS 部署：

```text
需要自己维护服务器
需要自己管理 Nginx
需要自己处理证书
需要手动上传文件
服务器宕机会影响网站
```

Cloudflare Pages：

```text
不需要服务器
自动 HTTPS
自动 CDN
自动构建
自动发布
GitHub 推送后自动上线
```

对于纯静态网站，Cloudflare Pages 更省心。

---

## 二十四、适合 Cloudflare Pages 的网站

适合：

```text
个人博客
工具站
文档站
导航站
产品官网
作品集
React/Vue/Vite 静态站
Astro 博客
Hugo 博客
Hexo 博客
```

不适合：

```text
WordPress
论坛
需要数据库的网站
需要用户登录的网站
需要支付系统的网站
需要文件上传的网站
```

如果后续需要后端，可以考虑：

```text
Cloudflare Workers
Supabase
自建 VPS API
```

---

## 二十五、最终总结

这次迁移后，我的网站从传统的 VPS 静态部署，变成了更现代的自动化部署流程。

最终流程：

```text
本地开发
↓
GitHub 提交
↓
Cloudflare Pages 自动构建
↓
自动发布到 51qdd.com
```

日常更新只需要：

```bash
git add .
git commit -m "更新内容"
git push
```

Cloudflare Pages 会自动完成后面的事情。

对于个人工具站和博客来说，这种方式非常适合：

```text
成本低
维护少
速度快
自动化程度高
```

目前我的网站主站已经迁移到：

```text
https://51qdd.com
```

后续重点就不再是服务器运维，而是：

```text
持续增加工具
优化 SEO
完善内容
提交搜索引擎
申请 AdSense
```
