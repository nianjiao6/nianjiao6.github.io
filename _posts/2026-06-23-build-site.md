---
title: GitHub Pages + Chirpy 建站全流程
date: 2026-06-23 10:00:00 +0800
categories: [Tech, Jekyll]
tags: [jekyll, github-pages]
pin: true
---

这篇文章记录本站从零搭建的完整过程:基于 [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) 主题、用 gem 方式集成、通过 GitHub Actions 部署到 GitHub Pages,并补上在 macOS 上本地预览的环境配置。既是建站笔记,也可作为复刻模板。

## 一、技术选型

本站是一个 GitHub Pages 用户站点(`用户名.github.io`),技术栈如下:

| 组件 | 选择 | 说明 |
| --- | --- | --- |
| 静态生成器 | Jekyll 4 | GitHub Pages 官方支持 |
| 主题 | Chirpy v7.6 | 以 **gem** 方式引入,而非拷贝源码 |
| 部署 | GitHub Actions | Chirpy 必须用 Actions(见下) |
| 本地 Ruby | 3.4.9(rbenv 管理) | 与 CI 保持一致 |

### 为什么用 gem 方式而不是 fork 主题源码

Chirpy 提供两种用法:fork 整个主题仓库,或用官方的 [chirpy-starter](https://github.com/cotes2020/chirpy-starter) 模板只引一个 gem。本站选后者——仓库里只保留配置和内容,主题代码由 `jekyll-theme-chirpy` gem 提供,升级时只需改 `Gemfile` 里的版本号,不必处理大量源码冲突。

```ruby
# Gemfile
gem "jekyll-theme-chirpy", "~> 7.6"
gem "html-proofer", "~> 5.0", group: :test
```

### 为什么必须用 GitHub Actions 部署

> Chirpy 用到了 `jekyll-archives` 插件和自定义 Ruby 插件,这些**不在 GitHub Pages 原生构建的白名单内**。如果用 Pages 默认的"从分支构建",线上会直接构建失败。
{: .prompt-warning }

解决办法是改用 GitHub Actions 自行构建后再发布。仓库里的 `.github/workflows/pages-deploy.yml` 就负责这件事:push 到 `master`/`main` 时用 Ruby 3.4 构建、跑 html-proofer 校验、再通过 `actions/deploy-pages` 发布。

## 二、项目结构

从 chirpy-starter 对齐后,仓库的核心结构是这样的:

```
.
├── _config.yml              # 站点配置(标题、语言、社交链接等)
├── _data/                   # 联系方式、分享按钮等数据
├── _plugins/                # 自定义插件(按 git 历史推算文章更新时间)
├── _posts/                  # 文章(就是这个目录)
├── _tabs/                   # 侧边栏导航页(About / Archives / Categories / Tags)
├── assets/img/favicons/     # 网站图标
├── tools/                   # 本地预览 / 测试脚本
├── .github/workflows/       # GitHub Actions 部署工作流
├── Gemfile                  # Ruby 依赖声明
└── index.html               # 首页(只有一行 layout: home)
```

`_layouts`、`_includes`、`_sass` 这些主题模板**不在仓库里**——它们由 gem 提供,这正是 gem 方式的好处。

## 三、本地环境:Ruby 版本是第一个坑

macOS 自带的 Ruby 通常很旧(本机是 2.6.10),而 Chirpy v7.6 要求 `Ruby ~> 3.1`(即 ≥3.1 且 <4.0)。所以第一步是装一个合适版本的 Ruby。

这里用 [rbenv](https://github.com/rbenv/rbenv) 而非 `brew install ruby`,原因是 **rbenv 按目录隔离 Ruby 版本,不污染系统环境**,也不会影响其他项目。

```bash
# 1. 安装 rbenv 与编译工具
brew install rbenv ruby-build

# 2. 让 shell 启用 rbenv(写入 ~/.zshrc 后重开终端)
echo 'eval "$(rbenv init - zsh)"' >> ~/.zshrc
source ~/.zshrc

# 3. 安装 Ruby 3.4.9(与 CI 一致;rbenv 会现场编译,约 3-5 分钟)
rbenv install 3.4.9

# 4. 在本仓库锁定版本(生成 .ruby-version 文件)
cd /path/to/your-site
rbenv local 3.4.9
```

> `rbenv install` 是**源码编译**:官方 Ruby 只发源码包,且要链接本机的 OpenSSL、libyaml 等库,所以必须在本机编译。但这是一次性成本——**每个版本只编译一次,之后所有项目共享**。其他项目想用 3.4.9,只需 `rbenv local 3.4.9`,无需再编译。
{: .prompt-info }

> 不要执行 `rbenv global 3.4.9`。保持全局默认为 `system`,这样没有 `.ruby-version` 的其他项目仍用系统 Ruby,完全不受影响。
{: .prompt-tip }

## 四、安装依赖

Ruby 就绪后,用 Bundler 安装主题及其依赖:

```bash
gem install bundler   # 若自带的 bundler 过旧
bundle install        # 安装 jekyll-theme-chirpy 及全部依赖
```

完成后会生成 `Gemfile.lock`(已在 `.gitignore` 中忽略,由 bundler 管理,不进仓库)。

## 五、本地启动预览

两种方式任选其一:

```bash
# 方式一:Chirpy 自带脚本(推荐,带 livereload 热更新)
bash tools/run.sh

# 方式二:原生命令
bundle exec jekyll serve -l
```

然后浏览器打开 <http://127.0.0.1:4000>。修改任何文件都会自动重建并刷新页面,`Ctrl+C` 停止。

如需在提交前做一次生产模式构建 + 死链检查:

```bash
bash tools/test.sh
```

## 六、个性化配置

站点几乎所有个性化都集中在 `_config.yml`。下面按功能分区块说明常用项。文件中 `# The following options are not recommended...` 那一行往后的内容是底层构建逻辑,**不建议修改**(见本章末尾)。

#### 站点基础

```yaml
lang: zh-CN                  # 界面语言,需匹配 _data/locales/ 下的文件名
timezone: Asia/Shanghai      # 文章时间所用时区
title: 你的站点标题           # 主标题(侧边栏顶部、浏览器标签页)
tagline: 一句话副标题          # 副标题(主标题下方小字)
description: >-              # 用于 SEO meta 和 RSS feed 的站点描述
  这里写一句站点简介。
url: "https://用户名.github.io"   # 站点完整地址,末尾不要带 /
baseurl: ""                  # 用户主页站点留空;子路径部署才填(如 /blog)
```

> `lang` 的值必须能在主题的 `_data/locales/` 里找到对应文件(如 `zh-CN.yml`),归档、分类等界面文字才会本地化。
{: .prompt-tip }

#### 身份与社交

```yaml
github:
  username: 你的GitHub用户名

social:
  name: 你的名字              # 显示为文章默认作者 + 页脚版权所有者
  email: your-email@example.com
  links:                     # 第一个链接会被用作版权链接
    - https://github.com/你的用户名
    # - https://twitter.com/你的用户名

avatar: /assets/images/avatar.jpg   # 侧边栏头像,可填本地路径或外链 URL
```

> 侧边栏底部那排联系图标(GitHub / 邮箱 / RSS 等)由 `_data/contact.yml` 控制,**不在** `_config.yml` 里;要增删图标去那个文件改。
{: .prompt-info }

#### 外观与目录

```yaml
theme_mode:                  # 留空=跟随系统并显示明暗切换按钮;light / dark = 强制某一模式
social_preview_image:        # 分享到社交平台时的预览大图(og:image)
toc: true                    # 文章右侧目录(Table of Contents)总开关
```

#### 评论系统

内置三选一,设 `provider` 并填对应子项即可,留空则关闭评论:

```yaml
comments:
  provider:                  # disqus / utterances / giscus
  giscus:
    repo:                    # 用户名/仓库名
    repo_id:
    category:
    category_id:
```

> 个人站最常用 **giscus**(基于 GitHub Discussions,免费、无广告)。到 [giscus.app](https://giscus.app/) 按提示授权仓库并生成上面这几个 id,填进来即可。
{: .prompt-info }

#### 访问统计与站长验证

```yaml
analytics:
  google:
    id:                      # Google Analytics 衡量 ID
  goatcounter:
    id:
  # 还支持 umami / matomo / cloudflare / fathom,填对应 id 即生效

pageviews:
  provider:                  # 文章阅读量显示(目前支持 goatcounter)

webmaster_verifications:     # 搜索引擎收录验证码,做 SEO 时填
  google:
  bing:
  baidu:
```

#### 其他常用开关

```yaml
pwa:
  enabled: true              # 渐进式 Web 应用:可"安装"到桌面 + 离线缓存
paginate: 10                 # 首页每页显示的文章数
```

> `# The following options are not recommended to be modified` 之后的 `kramdown`、`collections`、`defaults`、`sass`、`compress_html`、`exclude`、`jekyll-archives` 等,是 Markdown 解析、固定链接规则、资源压缩、归档生成等底层机制。**除非清楚连带影响,否则不要改**——尤其 `defaults` 里的 `permalink: /posts/:title/`,一旦改动会导致所有文章链接和站内引用失效。
{: .prompt-warning }

### 自定义网站图标(favicon)

Chirpy 默认用 gem 自带的图标。换成自己的步骤:

1. 准备一张方形图片(≥ 512×512),用 [realfavicongenerator.net](https://realfavicongenerator.net/) 生成整套图标并下载;
2. 把解压出的全部文件放进 `assets/img/favicons/`;
3. 提交推送即可。

> Chirpy v7.6 用的是新版图标格式,`favicon.ico`、`favicon.svg`、`favicon-96x96.png`、`apple-touch-icon.png`、`web-app-manifest-*.png` 都用得上,不必删任何文件。`site.webmanifest` 则**保留 gem 自带的模板版**即可——它会自动用 `site.title`/`site.description` 填充,无需手动维护。
{: .prompt-tip }

## 七、部署到 GitHub Pages

最后一步是把 Pages 的部署源切到 Actions(只需做一次):

1. 进入仓库 **Settings → Pages → Build and deployment**;
2. **Source** 选择 **GitHub Actions**。

此后每次 `git push` 到 `master`,工作流会自动构建并部署。在仓库 **Actions** 标签可看到 "Build and Deploy" 的运行状态,成功后访问站点即可看到更新。

> favicon 缓存非常顽固。若更新后浏览器仍显示旧图标,强制刷新(<kbd>Cmd</kbd>+<kbd>Shift</kbd>+<kbd>R</kbd>)或用无痕窗口确认。
{: .prompt-info }

## 八、日常写作流程

在 `_posts/` 下新建 `YYYY-MM-DD-标题.md`,文件头如下:

```yaml
---
title: 文章标题
date: 2026-06-23 10:00:00 +0800
categories: [一级分类, 二级分类]   # 最多两级
tags: [标签一, 标签二]            # 数量不限,建议小写
pin: true                        # 可选:置顶到首页
---
```

> 生产模式构建**不会发布发布时间在未来的文章**。如果写好的文章线上看不到,先检查 `date` 是不是晚于当前时间。
{: .prompt-warning }

正文支持标准 Markdown,以及 Chirpy 特色的提示框语法:

```markdown
> 这是一条提示。
{: .prompt-tip }
```

至此,从环境、配置、本地预览到线上部署的完整链路就跑通了。
