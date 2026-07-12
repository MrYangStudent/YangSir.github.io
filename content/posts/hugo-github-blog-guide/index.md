---
title: "零基础搭建 Hugo 博客：从安装到 GitHub Pages 上线"
date: 2026-07-12T12:00:00+08:00
draft: false
tags:
  - Hugo
  - GitHub Pages
  - 博客
  - 教程
  - GitHub Actions
categories: ["教程"]
description: "手把手教你用 Hugo + PaperMod 主题 + GitHub Actions 搭建个人技术博客,附带踩坑实录和配置详解。"
ShowToc: true
TocOpen: true
---

## 为什么要自己搭博客

写博客这件事，我纠结了很久。

市面上成熟的写作平台很多——掘金、CSDN、知乎专栏，注册就能写。但每次想动笔时，总觉得少了点什么。后来想明白了：**平台会变,域名不会变。** 我希望有一个真正属于自己的地方，一个 GitHub 用户名就能访问的独立空间，所有的内容都由自己掌控。

所以就有了这个博客。

这篇文章会把这整个过程完整记录下来——从安装 Hugo 到最后你用手机浏览器打开 `https://<你的用户名>.github.io` 看到完整页面。我会用**这个博客本身的代码**作为示例，你看到的每一个配置项都是真实在用的。

> 本文参考了 [github-blog-publisher](https://github.com) 技能中的 Hugo 搭建指南，感谢开源社区。

---

## 技术选型

| 组件 | 选型 | 理由 |
|------|------|------|
| 静态站点生成器 | Hugo | Go 编写,单二进制,毫秒级构建,支持 CJK |
| 主题 | PaperMod | 极简、加载快、暗色模式、SEO 友好 |
| 托管 | GitHub Pages | 免费、HTTPS、全球 CDN |
| 自动部署 | GitHub Actions | push 即部署,无需手动操作 |

构建速度对比：Hugo 的构建是毫秒级,100 篇文章基本感觉不到延迟;同类工具可能需要几秒到几十秒。

---

## 第一步：安装 Hugo

### Windows

```powershell
# winget（推荐）
winget install Hugo.Hugo.Extended

# 或者直接用 Scoop / Chocolatey
scoop install hugo-extended
choco install hugo-extended
```

### macOS

```bash
brew install hugo
```

### Linux

```bash
# Ubuntu/Debian
sudo apt install hugo

# 或从 GitHub Releases 下载预编译二进制
# https://github.com/gohugoio/hugo/releases
```

验证安装：

```bash
hugo version
# 输出: hugo v0.164.0+extended windows/amd64 ...
```

> **必须安装 Extended 版本**，大多数主题依赖 SCSS/SASS 编译，非 Extended 版本无法处理。

---

## 第二步：创建站点

```bash
hugo new site my-blog --format yaml
cd my-blog
git init
```

`--format yaml` 让 Hugo 用 YAML 格式生成配置文件（比 TOML 更直观）。

此时目录结构：

```
my-blog/
├── archetypes/    # 文章模板
├── assets/        # CSS/JS（经 Hugo Pipes 处理）
├── content/       # 所有文章放这里
├── data/          # 数据文件
├── layouts/       # 自定义布局
├── static/        # 静态文件（原样复制）
├── themes/        # 主题
└── hugo.yaml      # 站点配置
```

---

## 第三步：安装 PaperMod 主题

用 git submodule 管理主题,方便后续更新：

```bash
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

> 如果你用 `git clone` 主题再手动复制，每次主题更新都得重新操作。submodule 一键搞定：
> ```bash
> git submodule update --remote themes/PaperMod
> ```

---

## 第四步：配置 hugo.yaml

**这是最关键的一步**。以下是本博客实际在用的配置，逐项解释：

```yaml
baseURL: "https://<你的用户名>.github.io/"  # 🚨 如果要部署到子路径，填完整路径
locale: "zh-cn"                               # 中文站点
title: "YangSir's Tech Blog"                  # 浏览器标签页标题
theme: "PaperMod"                             # 主题名称

# Hugo 内置功能开关
enableInlineShortcodes: true
enableRobotsTXT: true
buildDrafts: false                            # 生产环境不构建草稿
buildFuture: false                            # 不构建未来日期的文章
enableEmoji: true
hasCJKLanguage: true                          # 🚨 中文博客必须开启

params:
  author: "YangSir"
  description: "记录技术学习与项目实践的个人博客"

  # 个人主页
  profileMode:
    enabled: true
    title: "Hi there 👋"
    subtitle: "你的个人介绍..."
    buttons:
      - name: "阅读文章"
        url: "/posts/"
      - name: "关于我"
        url: "/about/"

  # 社交链接（显示在页面底部和关于页）
  socialIcons:
    - name: github
      url: "https://github.com/你的用户名"
    - name: email
      url: "mailto:你的邮箱"

  # 阅读体验
  ShowReadingTime: true        # 显示预估阅读时间
  ShowShareButtons: true       # 社交分享按钮
  ShowPostNavLinks: true       # 上一篇/下一篇
  ShowBreadCrumbs: true        # 面包屑导航
  ShowCodeCopyButtons: true    # 代码块复制按钮 🚨
  ShowWordCount: true          # 字数统计
  ShowToc: true                # 目录

  defaultTheme: auto           # 跟随系统主题
  disableThemeToggle: false

# 代码高亮
markup:
  highlight:
    style: "monokai"           # 代码高亮主题
    noClasses: false
    lineNos: true              # 显示行号
  goldmark:
    renderer:
      unsafe: true             # 允许在 Markdown 中插入 HTML

# 导航菜单
menu:
  main:
    - identifier: posts
      name: 文章
      url: /posts/
      weight: 10
    - identifier: tags
      name: 标签
      url: /tags/
      weight: 20
    - identifier: search
      name: 搜索
      url: /search/
      weight: 30
    - identifier: about
      name: 关于
      url: /about/
      weight: 40
```

### 关键配置说明

- **`hasCJKLanguage: true`**：不开启的话，中文字数统计和阅读时间估算会不准
- **`unsafe: true`**：允许在 Markdown 中写 HTML，写嵌入组件时必不可少
- **`ShowCodeCopyButtons: true`**：技术博客必备，代码块右上角的复制按钮

---

## 第五步：创建基础页面

```bash
# 关于页面
hugo new about.md

# 第一篇文章
hugo new posts/my-first-post.md
```

Hugo 的文章 frontmatter 格式：

```yaml
---
title: "文章标题"
date: 2026-07-12T12:00:00+08:00
draft: false
tags: ["标签1", "标签2"]
categories: ["分类"]
description: "文章摘要，会显示在列表页"
---
```

> `draft: true` 时文章在本地可见，但生产环境不会构建。写一半先存着，改天继续。

---

## 第六步：本地预览

```bash
# 启动开发服务器（包含草稿用 -D）
hugo server -D

# 打开 http://localhost:1313
```

Hugo 的热重载极快——保存文件后几十毫秒页面就刷新了，完全不会打断思路。

---

## 第七步：配置 GitHub Actions 自动部署

Hugo 不像 Jekyll 那样被 GitHub Pages 原生支持，需要自定义 workflow。

创建 `.github/workflows/hugo-deploy.yml`：

```yaml
name: Deploy Hugo site to Pages

on:
  push:
    branches: ["master"]      # 🚨 改成你的实际分支名
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

defaults:
  run:
    shell: bash

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive    # 🚨 主题是 submodule，必须递归拉取

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: ${{ env.HUGO_VERSION }}
          extended: true

      - name: Build
        run: hugo --minify

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

Workflow 分两个 Job：
1. **build**：拉代码 → 装 Hugo → 构建 → 上传产物
2. **deploy**：把构建产物部署到 Pages

---

## 第八步：在 GitHub 开启 Pages

这是**最容易忘的一步**，也是 404 最常见的根因。

1. 打开仓库的 **Settings → Pages**
2. **Source** 选择 **GitHub Actions**（不是 "Deploy from a branch"）
3. 保存

![GitHub Pages 设置示意](/images/github-pages-settings.png)

---

## 第九步：推送到 GitHub

```bash
git add .
git commit -m "feat: initialize Hugo blog"
git remote add origin https://github.com/<用户名>/<仓库名>.git
git push -u origin master
```

推送后：
1. 打开仓库的 **Actions** 标签页，确认 workflow 已触发
2. 等待 1-2 分钟构建完成
3. 访问 `https://<用户名>.github.io`（或子路径的完整地址）

看到页面就是成功了。

---

## 踩坑实录

下面是我搭建过程中实际遇到的问题，每个都耽误了不少时间。

### 坑一：404——GitHub Pages 没启用

**现象**：push 后访问域名始终 404，反复检查代码没问题。

**原因**：GitHub Pages 默认不开启，必须去仓库 Settings → Pages 手动选择 Source 为 GitHub Actions。

**修复**：Settings → Pages → Source 选 GitHub Actions → Save。

---

### 坑二：404——Workflow 分支名不匹配

**现象**：仓库用 `master` 分支，但 workflow 监听的是 `main`。

```yaml
on:
  push:
    branches: ["main"]   # ❌ 仓库是 master，永远不会触发
```

**修复**：把 `main` 改成实际分支名：

```yaml
on:
  push:
    branches: ["master"]  # ✅
```

---

### 坑三：首页按钮链接错误

**现象**：首页的"阅读文章"和"关于我"按钮点了跳 404。

**原因**：博客部署在子路径 `https://mryangstudent.github.io/YangSir.github.io/`，但按钮生成了根路径 `/posts/`，浏览器解析成了 `https://mryangstudent.github.io/posts/`，漏掉了子路径。

**修复**：覆盖主题的按钮模板，让 URL 走 Hugo 的 `relURL`：

```html
<!-- layouts/_partials/index_profile.html -->
<a href="{{ $url | relURL }}">{{ $name }}</a>
```

`relURL` 会自动加上 `baseURL` 中的子路径。

---

### 坑四：主题 submodule 拉取失败

**现象**：GitHub Actions 构建时报错 `theme not found`。

**原因**：checkout 时没启用 `submodules: recursive`。

**修复**：

```yaml
- name: Checkout
  uses: actions/checkout@v4
  with:
    submodules: recursive    # 🚨
```

---

### 坑五：文章列表不显示

**现象**：Hugo 配置了文章列表，但首页一片空白。

**原因**：没有指定 `mainSections`，Hugo 不知道从哪个目录取文章。

**修复**：在 `hugo.yaml` 中添加：

```yaml
params:
  mainSections:
    - posts
```

---

## 目录结构速查

```
my-blog/
├── archetypes/
│   └── default.md                  # 新文章的默认 frontmatter
├── assets/
│   └── css/extended/custom.css     # 自定义样式
├── content/
│   ├── posts/                      # 文章（每篇一个目录，支持图片资源）
│   │   └── my-article/
│   │       ├── index.md
│   │       └── screenshot.png
│   └── about.md                    # 关于页面
├── layouts/
│   └── _partials/                  # 覆盖主题的局部模板
├── themes/
│   └── PaperMod/                   # 主题（git submodule）
├── hugo.yaml
└── .github/workflows/
    └── hugo-deploy.yml
```

---

## 常用命令速查

```bash
hugo new posts/文章名.md        # 新建文章
hugo server -D                   # 本地预览（含草稿）
hugo server --port 3131          # 指定端口
hugo                             # 构建（输出到 public/）
hugo --minify                    # 构建并压缩
hugo new about.md                # 新建页面
```

---

## 总结

整个流程其实就九步：

1. 安装 Hugo Extended
2. `hugo new site` 创建站点
3. 安装 PaperMod 主题（git submodule）
4. 配置 `hugo.yaml`
5. 创建几篇基础页面
6. 本地 `hugo server -D` 预览
7. 编写 GitHub Actions workflow
8. 在仓库 Settings 开启 Pages
9. `git push`

难点不在代码，而在那些容易被忽略的细节——分支名匹配、submodule 递归拉取、子路径链接。

搞定这些之后，写文章就简单了：一个 `hugo new`，写完 Markdown，`git push`。剩下的 GitHub Actions 全自动搞定。

---

*这个博客本身就是按照本文搭建的，如果你对某个配置有疑问，直接翻仓库源码比任何文档都直白。如果觉得有用，欢迎给 [本博客的 GitHub 仓库](https://github.com/MrYangStudent/YangSir.github.io) 点个 Star。*
