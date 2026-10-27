# GitHub Pages 技术博客

此目录是公众号仓库中的独立 Jekyll 站点源文件。文章使用 Markdown，放在 `_posts/` 中，文件名格式为 `YYYY-MM-DD-title.md`。

## 发布到 GitHub Pages

仓库根目录的 `.github/workflows/github-pages.yml` 会从 `github-pages/` 构建站点并部署。将变更推送到 `main` 或 `master` 分支后触发；也可以在 GitHub Actions 页面手动运行。

首次发布前，在 GitHub 仓库 Settings → Pages → Build and deployment 中选择 **GitHub Actions**。默认 GitHub Pages 地址会使用仓库名作为路径前缀；如绑定自定义域名，再按需调整 `_config.yml` 的 `url`、`baseurl` 和 DNS 设置。

## 本地预览

安装 Ruby 和 Jekyll 后，在本目录运行 `jekyll serve`，然后访问终端显示的本地地址。
