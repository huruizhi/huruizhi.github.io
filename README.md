# GitHub Pages 技术博客

此目录是公众号仓库中的独立 Jekyll 站点源文件。文章使用 Markdown，放在 `_posts/` 中，文件名格式为 `YYYY-MM-DD-title.md`。

## 发布到 GitHub Pages

此站点以 Jekyll 源码发布。推送到 GitHub 后，在仓库 Settings → Pages → Build and deployment 中选择 **Deploy from a branch**、分支 `main`、目录 `/(root)`。GitHub Pages 会自动构建仓库根目录中的 Jekyll 站点。

默认 GitHub Pages 地址使用 `huruizhi.github.io`；如绑定自定义域名，再按需调整 `_config.yml` 的 `url`、`baseurl` 和 DNS 设置。

## 本地预览

安装 Ruby 和 Jekyll 后，在本目录运行 `jekyll serve`，然后访问终端显示的本地地址。
