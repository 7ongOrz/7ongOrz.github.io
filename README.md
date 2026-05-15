# AhTong's blog

这是 Hexo 源站仓库。GitHub Actions 会在推送到 `master` 或 `main` 后安装依赖、生成 `public/`，再把 `public/` 作为 GitHub Pages artifact 发布。

发布前需要在仓库 Settings -> Pages -> Build and deployment 中把 Source 设置为 `GitHub Actions`。

## 目录逻辑

- `_config.yml`: Hexo 站点配置，包含域名、永久链接、文章资源目录、搜索、字数统计、Giscus 评论等。
- `_config.next.yml`: NexT 主题外置配置。主题本体来自 npm 包 `hexo-theme-next`，不要直接改 `node_modules/hexo-theme-next`。
- `source/_posts`: 文章 Markdown 和每篇文章自己的图片资源目录。
- `source/_data`: NexT 自定义注入片段，目前 `body-end.njk` 保留了 canvas 背景、站点运行时间和可选鼠标点击特效。
- `source/images`: 站点级图片，比如侧栏头像。
- `source/js/cursor`: 旧主题里自定义的鼠标点击特效脚本。
- `scaffolds`: `hexo new` 新建文章时使用的模板。
- `.github/workflows/pages.yml`: GitHub Pages 构建和发布流程。
- `.github/dependabot.yml`: 定期检查 Hexo、NexT、插件和 Actions 更新。

## 常用命令

本地有 Node.js/npm 时可以运行：

```sh
npm install
npm run server
npm run build
```

首次在有 npm 的环境中确认构建成功后，建议提交 `package-lock.json`，并把 `.github/workflows/pages.yml` 里的安装命令从 `npm install --no-audit --no-fund` 改成 `npm ci --no-audit --no-fund`，让 CI 构建完全可复现。

当前仓库没有提交 `public/` 和 `node_modules/`。`public/` 是构建产物，由 GitHub Actions 临时生成并发布。
