# PersonalWeb - 个人技术博客

基于 Hexo + Butterfly 主题搭建的个人技术博客，通过 GitHub 自动部署到 Cloudflare（Workers）。

- 线上地址：https://blog.stackoverflowing.com/ （自定义域名，大陆可直连）
- 备用地址：https://personal-blog.2505157114.workers.dev/ （需代理访问）

## 日常写作流程

```bash
# 1. 新建文章（在 source/_posts/ 下生成 Markdown 文件）
hexo new post "文章标题"

# 2. 本地预览（http://localhost:4000）
npm run server

# 3. 写完后提交并推送，Cloudflare 自动构建发布（约 2 分钟上线）
git add .
git commit -m "post: 文章标题"
git push
```

## 常用命令

| 命令 | 说明 |
| ---- | ---- |
| `hexo new post "标题"` | 新建文章 |
| `hexo new draft "标题"` | 新建草稿（`hexo publish 标题` 转为正式文章） |
| `npm run server` | 本地预览 |
| `npm run clean` | 清理缓存和 public 目录 |
| `npm run build` | 生成静态页面到 public |

> Windows Git Bash 下 `npm` 直接调用有兼容问题，请用 `cmd //c "npm run server"` 方式；`hexo`、`git` 命令不受影响。

## 目录结构

```
├── _config.yml            # 站点配置（标题、作者、语言、url、永久链接规则）
├── _config.butterfly.yml  # Butterfly 主题配置（覆盖式，独立于主题包，升级主题不丢配置）
├── .nvmrc                 # 声明 Node 版本（供云端构建环境读取）
├── scaffolds/             # 文章模板（hexo new 时套用）
│   ├── post.md / draft.md / page.md
├── source/
│   └── _posts/            # 文章目录（Markdown + front-matter）
├── package.json           # 依赖清单（hexo 核心 + 渲染器 + 主题）
└── public/                # 构建产物（.gitignore 排除，不入库）
```

---

# 搭建全过程详解

以下记录本博客从零到上线的完整过程，并解释每一步背后的原理。换电脑重建、或想用同样方案搭建新站点时，可按顺序照做。

## 0. 技术选型：为什么是「静态生成 + 免费托管」

**静态博客 vs 动态博客的本质区别**：

- 动态博客（如 WordPress）：服务器上跑程序，每次有人访问都实时查数据库、渲染页面返回。功能强（评论、后台），但需要一直开着服务器，有安全面和运维成本。
- 静态博客（Hexo/Hugo 等）：**写文章时就在本地把 Markdown 编译成纯 HTML/CSS/JS 文件**，托管服务只需要"把文件发出去"，没有任何运行时逻辑。免费托管平台遍地都是，安全性极高（没有可被攻击的动态面），访问速度也快（纯静态文件 + CDN 缓存）。

本站架构与数据流：

```
[写作端]                    [版本管理]         [构建部署]                [分发]
Markdown 文章 --git push--> GitHub 仓库 --webhook--> Cloudflare 构建
                                                npm install
                                                hexo generate
                                                public/ 发布到全球边缘节点
读者 --https--> Cloudflare CDN（就近返回缓存页面）
```

关键角色分工：

| 组件 | 职责 | 为什么选它 |
| ---- | ---- | ---- |
| Hexo | 把 Markdown 编译成网站 | Node.js 生态、中文社区大、插件全 |
| Butterfly | 主题（网站的"皮"） | 功能全（暗色模式/本地搜索/响应式）、文档完善 |
| GitHub | 存源码、触发部署 | 事实标准，push 即发布；网站源码天然有版本历史 |
| Cloudflare Workers | 构建与全球托管 | 免费额度充足、带宽不限量、全球 CDN、可绑自定义域名 |

## 1. 本地环境搭建（Hexo 初始化）

### 1.1 前置条件

- **Node.js**（≥18）：Hexo 本身是一个 Node 程序，`hexo` 命令是 Node 脚本的入口
- **npm 镜像**：大陆访问官方源 `registry.npmjs.org` 慢，本机配置了 npmmirror（淘宝镜像）：

  ```bash
  npm config set registry https://registry.npmmirror.com
  ```

  副作用是 `package-lock.json` 里的依赖下载地址（`resolved` 字段）都指向 npmmirror——该镜像全球可达，不影响 Cloudflare 海外节点构建。

### 1.2 初始化站点：正常路径与替代路径

**正常路径**是一行命令：

```bash
npm install -g hexo-cli
hexo init .
npm install
```

**原理**：`hexo init` 做的事只是从 GitHub 克隆官方模板仓库 [hexo-starter](https://github.com/hexojs/hexo-starter)（内容 = 一个 package.json + 一份默认 _config.yml + scaffolds/ 模板 + 示例文章），然后提示你 npm install。**它不是什么魔法安装器，就是拉了个模板目录。**

**本站踩坑**：搭建时本机到 github.com 的连接被阻断（见第 3 节），`hexo init` 克隆失败。替代方案是**手动创建与 hexo-starter 等价的文件**，效果完全一致：

| 文件/目录 | 作用 |
| ---- | ---- |
| `package.json` | 声明依赖：hexo 核心、分类/标签/归档生成器、三个渲染器、本地服务器 |
| `_config.yml` | 站点配置：语言 `zh-CN`、时区 `Asia/Shanghai`、永久链接规则等 |
| `scaffolds/*.md` | 文章模板，`hexo new` 时把 `{{ title }}`、`{{ date }}` 替换成实际值生成新文件 |
| `source/_posts/` | 文章目录，Hexo 扫描这里的 Markdown 生成页面 |

然后 `npm install` 安装依赖即可，全程只依赖 npm 镜像、不需要 GitHub。

### 1.3 主题：npm 安装方式与双配置文件原理

```bash
npm install hexo-theme-butterfly hexo-renderer-pug
```

**为什么加 `hexo-renderer-pug`**：Butterfly 的页面模板用 Pug 语法写、样式用 Stylus 写，Hexo 需要对应的渲染器才能把它们编译成 HTML/CSS。默认 starter 已含 stylus 渲染器，pug 渲染器需要额外装。

**主题的两种安装方式对比**：

- `npm install`（本站采用）：主题作为依赖装进 `node_modules/`，仓库里不存主题源码。优点：CI 环境装依赖即得主题、干净、`npm update` 即升级；缺点：不方便直接魔改主题源码。
- `git clone` 到 `themes/` 目录：主题源码完整入库，随便改；缺点：仓库巨大、升级要手动合并。

**双配置文件机制**：`_config.yml` 里 `theme: butterfly` 指定主题后，Hexo 5+ 支持**在站点根目录放 `_config.butterfly.yml`**——它的配置项会**覆盖**主题包内的同名配置，且不改动主题包本身。这样主题可以随时升级，自定义配置永不丢失。本站的做法是把主题默认 `_config.yml` 整份复制为 `_config.butterfly.yml` 再按需修改。

### 1.4 Hexo 构建管线（`hexo generate` 时发生了什么）

```
source/_posts/*.md
   │  hexo-renderer-marked：Markdown → HTML 片段
   ▼
文章对象（含 front-matter 解析出的 title/date/tags）
   │  主题的 Pug 模板 + hexo-renderer-pug：套布局、拼页面
   │  hexo-renderer-stylus：编译样式
   │  hexo-generator-index/archive/category/tag：生成首页分页、归档、分类、标签页
   ▼
public/ 目录 = 一个完整纯静态网站（可直接用任何 HTTP 服务器分发）
```

front-matter 是每篇文章顶部 `---` 包裹的元数据块（title/date/tags/categories），Hexo 解析后用于归档、分类页和页面标题，写法见 `source/_posts/hello-world.md`。

## 2. 版本管理设计（.gitignore 的原理）

`.gitignore` 决定哪些文件进 Git 仓库，判断标准是**"源文件"还是"可再生的产物"**：

| 排除项 | 原因 |
| ---- | ---- |
| `node_modules/` | 依赖可由 package.json + package-lock.json（精确锁版本）在任何机器 `npm install` 重建，入库纯属浪费体积 |
| `public/` | 构建产物，由源文件 `hexo generate` 再生；入库会造成每次构建海量 diff，且 Cloudflare 会自己构建，根本用不到 |
| `db.json` | Hexo 本地缓存数据库 |
| `.deploy*/` | `hexo deploy` 插件的部署残留 |

`.nvmrc` 写着 `22`：声明项目需要的 Node 主版本。云构建环境读到它就用对应 Node 跑构建，避免 Node 大版本差异导致依赖行为不一致。

## 3. GitHub 仓库与大陆网络问题

### 3.1 建仓与推送

GitHub 网页新建**空仓库**（不勾选任何初始化选项，避免和本地历史冲突），然后：

```bash
git remote add origin https://github.com/<用户名>/personal-blog.git
git push -u origin main
```

**原理**：`remote add` 只是在本地登记远程地址；push 时凭据由 Git Credential Manager（Git for Windows 自带）管理——首次会弹浏览器让你登录 GitHub 并生成 Personal Access Token 存进 Windows 凭据管理器，之后 push 免密。

### 3.2 大陆访问 GitHub 的典型故障与定位

本站搭建过程中实测遇到的网络现象，以及定位方法（`curl -v`、`nslookup` 观察）：

| 现象 | 原因 | 解法 |
| ---- | ---- | ---- |
| DNS 解析出真实 IP（如 20.205.243.166）但 443 端口连不上、curl 超时 | **SNI 阻断**：防火墙识别 TLS 握手中的域名（SNI 字段）后切断连接。实测 api.github.com 可达而 github.com 不可达，且把 github.com 强制解析到 api 的 IP 依旧被断，证明阻断按域名而非按 IP | 开代理/VPN 后再 push |
| 代理开了浏览器能用、git 依然超时 | 代理软件只设置了"系统代理"（仅浏览器遵循），命令行 git 不读系统代理 | 切 TUN/增强模式（全局接管）；或 `git config --global http.proxy http://127.0.0.1:7890`（端口按代理软件实际值，Clash 默认 7890），用完 `git config --global --unset http.proxy` 恢复 |
| 连 bing 等正常网站都打不开，但命令行 curl 一切正常 | **浏览器层残留代理**：浏览器代理扩展（SwitchyOmega 等）指向已关闭的本地代理端口 | 扩展切回"直接连接"，或换浏览器对照定位 |

## 4. Cloudflare 自动部署

### 4.1 Workers 与 Pages 的关系

Cloudflare 传统上用 **Pages** 托管静态站（`*.pages.dev`），**Workers** 跑边缘函数（`*.workers.dev`）。两者现已合并：新版控制台统一从 "Workers & Pages" 创建，Workers 也原生支持静态资源托管（Static Assets）。**本站实际部署的就是一个 Workers 项目**，功能与 Pages 等价（Git 集成、预览、全球 CDN）。

### 4.2 Git 集成的构建流水线

在控制台把仓库连接到项目后，每次 `git push`：

```
push 到 main → GitHub 发 webhook 通知 Cloudflare
→ CF 构建容器：npm install（读 package-lock.json 精确还原依赖）
→ npx hexo generate（产出 public/）
→ public/ 内容分发到 Cloudflare 全球边缘网络（约 300 个节点）
→ 读者访问时由就近节点返回
```

本项目的构建配置：

- 构建命令：`npx hexo generate`
- 输出目录：`public`
- 生产分支：`main`

**踩坑记录**：Cloudflare Workers Builds 构建完成后**不会向 GitHub 回写 commit 状态**（deployments/checks API 查不到记录），这与传统 CI（GitHub Actions、旧版 Pages）行为不同。所以"GitHub 上没有绿勾"不代表部署失败，验证要去 CF 控制台看 Deployments，或直接访问站点。

### 4.3 默认域名在大陆不可用

`*.workers.dev` 整个域名在大陆被 SNI 阻断（实测直连 0.3 秒被重置）。**这只影响访问默认域名，不影响部署本身**——Cloudflare 该项目的部署照常成功。解决办法就是下一节的自定义域名。

## 5. 自定义域名（DNS 原理 + 绑定）

### 5.1 DNS 体系：注册商、NS、权威服务器

理解绑定过程需要先理清 DNS 的分层职责：

```
注册商（如阿里云/Namesilo）     你花钱买下域名的地方，管理"这个域名委托给谁解析"（NS 记录）
      │ NS 委托
      ▼
权威 DNS 服务器               该域名所有解析记录（A/CNAME/MX…）的权威数据源
      ▲ 查询
      │
递归 DNS（运营商 8.8.8.8 等）  代用户向权威查询并缓存（按 TTL）
```

**买域名 ≠ 能解析**：注册商默认会送一套自家权威 DNS。把 NS 改成 Cloudflare 分配的两个地址（本站为 `bart.ns.cloudflare.com` / `reza.ns.cloudflare.com`），等于把"解析权"委托给 Cloudflare——之后所有解析记录都在 CF 控制台管理，Custom Domain 的自动配置也依赖这一点。

**验证是否生效的最快方法**：直接向权威服务器查询（绕过各级缓存，零延迟看到真实状态）：

```bash
nslookup -type=NS stackoverflowing.com bart.ns.cloudflare.com
```

### 5.2 绑定：Custom Domain vs Route（重要区别）

Workers 项目的 Settings → Domains & Routes 里，Add 有两种选项：

| | **Custom Domain（本站使用）** | Route |
| ---- | ---- | ---- |
| 本质 | 完整托管：**自动创建 DNS 解析记录 + 自动签发 HTTPS 证书** | 仅一条转发规则：告诉 Worker"匹配该域名的请求由我处理" |
| DNS 记录 | CF 自动写入（对外表现为 Cloudflare Anycast IP） | **不会创建**，需自己去 DNS 页手动加，否则域名根本无法解析 |
| 证书 | 自动（Universal SSL） | 需自行保证 |

**踩坑记录**：首次误加了 Route，导致权威 DNS 查不到 `blog` 子域（Route 不建解析）。删除 Route、改用 Custom Domain 后，记录立即生成。

绑定完成后权威查询实测：

```
blog.stackoverflowing.com → 172.67.220.164 / 104.21.59.88（Cloudflare Anycast IP）
```

### 5.3 HTTPS 证书与大陆可达性

- **证书**：Custom Domain 绑定后，Cloudflare 自动为域名签发 Universal SSL 边缘证书，用户浏览器到 CF 节点全程加密，无需手动续期。
- **大陆可达的原理**：被阻断的是 `workers.dev` 这个**域名**（SNI 检测），而不是 Cloudflare 的 IP。自定义域名解析到 CF 的 Anycast IP（同一 IP 服务海量网站，无法按 IP 阻断），因此大陆可以直连——实测 HTTP 200、证书验证通过。

### 5.4 站点 url 配置

域名绑好后，把 `_config.yml` 的 `url` 改成正式域名并推送一次。原理：`url` 是 Hexo 生成**绝对链接**的基准——页面的 canonical、og:image 等元数据都由它拼出，配置错了会导致分享到社交平台时缩略图/链接错误、搜索引擎收录混乱。本站当前配置：`url: https://blog.stackoverflowing.com`。

## 6. 完整搭建顺序清单（复刻用）

```
1. npm 镜像配置 → 2. Hexo 站点初始化（或手动脚手架） → 3. 主题 npm 安装 + _config.butterfly.yml
→ 4. 写首篇文章 → 5. .gitignore/.nvmrc/README → 6. git init + commit
→ 7. GitHub 建空仓库 + push（需代理时见 3.2）
→ 8. Cloudflare: Workers & Pages → Create → 连接仓库，构建命令 npx hexo generate，输出 public
→ 9. 验证默认域名部署成功（控制台 Deployments 绿色）
→ 10. 注册商买域名 → CF Add a domain（Free 套餐）→ 回注册商改 NS → 等 Active
→ 11. Workers 项目 Settings → Domains & Routes → Add → Custom domain → blog.域名
→ 12. _config.yml 的 url 改成 https://blog.域名 → push → 完成
```

## 踩坑速查表

| 症状 | 根因 | 处理 |
| ---- | ---- | ---- |
| `hexo init` 卡在克隆 GitHub | 大陆到 github.com 被阻断 | 手动创建 hexo-starter 等价结构（见 1.2） |
| Git Bash 里 npm 报 fd/管道错误 | Git Bash 与 npm 的 shell 脚本兼容问题 | `cmd //c "npm ..."` 调用；或直接用 hexo 命令 |
| git push 超时但浏览器正常 | git 不走系统代理 | TUN 模式或 `git config --global http.proxy` |
| Cloudflare 部署成功但 GitHub 无记录 | Workers Builds 不回写 commit 状态 | 看 CF 控制台或直接访问站点验证 |
| workers.dev 打不开 | 域名级 SNI 阻断 | 绑定自定义域名 |
| 加了 Route 域名不通 | Route 不创建 DNS 记录 | 删 Route，改用 Custom Domain |
| 新域名绑完浏览器打不开 | 本地 DNS 缓存未更新 | `ipconfig /flushdns` 或稍等几分钟 |

## 相关文档

- [Hexo 文档](https://hexo.io/zh-cn/docs/)
- [Butterfly 主题文档](https://butterfly.js.org/)
- [Cloudflare Workers 文档](https://developers.cloudflare.com/workers/)
