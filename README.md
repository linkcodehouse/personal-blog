# PersonalWeb - 个人技术博客

基于 Hexo + Butterfly 主题搭建的个人技术博客，通过 GitHub 自动部署到 Cloudflare（Workers）。

线上地址：https://personal-blog.2505157114.workers.dev/

## 日常写作流程

```bash
# 1. 新建文章（在 source/_posts/ 下生成 Markdown 文件）
hexo new post "文章标题"

# 2. 本地预览（http://localhost:4000）
npm run server

# 3. 写完后提交并推送，Cloudflare Pages 自动构建发布
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

## 目录结构

```
├── _config.yml            # 站点配置（标题、作者、语言等）
├── _config.butterfly.yml  # Butterfly 主题配置（外观、导航、功能）
├── source/
│   └── _posts/            # 文章目录（Markdown）
├── scaffolds/             # 文章模板
└── themes -> node_modules/hexo-theme-butterfly  # 主题（npm 安装）
```

## 部署架构

```
写作 → git push → GitHub 仓库 → Cloudflare 自动构建（hexo generate）
     → 全球 CDN 发布 → 自定义域名
```

Cloudflare 构建配置（Workers → Settings → Build）：

- 构建命令：`npx hexo generate`
- 输出目录：`public`

## 相关文档

- [Hexo 文档](https://hexo.io/zh-cn/docs/)
- [Butterfly 主题文档](https://butterfly.js.org/)
- [Cloudflare Workers 文档](https://developers.cloudflare.com/workers/)
