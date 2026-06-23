# Six Feet Under

> Remember Me

个人博客,基于 [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) Jekyll 主题(gem 方式集成)。

## 本地预览

需要 Ruby 与 Bundler 环境。

```bash
bundle install          # 安装依赖
bundle exec jekyll s    # 或 bash tools/run.sh,访问 http://127.0.0.1:4000
```

## 构建与检查

```bash
bash tools/test.sh      # 生产模式构建 + html-proofer 链接检查
```

## 写文章

在 `_posts/` 下新建 `YYYY-MM-DD-标题.md`,文件头使用 Chirpy front matter:

```yaml
---
title: 文章标题
date: 2026-06-23 12:00:00 +0800
categories: [分类]
tags: [标签]
---
```

## 部署

push 到 `master` 后由 `.github/workflows/pages-deploy.yml` 通过 GitHub Actions 自动构建并部署到 GitHub Pages。

> 注意:仓库 **Settings → Pages → Source** 需设置为 **GitHub Actions**(Chirpy 使用了 GitHub Pages 原生构建不支持的插件)。

## License

本项目代码遵循 [MIT License](LICENSE)。
