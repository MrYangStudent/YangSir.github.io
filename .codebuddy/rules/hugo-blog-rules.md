---
description: Hugo 博客项目强制约束规则 — 防止重复出现同样的配置错误、404、首页缺失等问题
alwaysApply: true
---

# Hugo GitHub Pages 博客项目规则

## 🚨 高度敏感区域（禁止随意修改）

### 1. `hugo.yaml` 关键配置 — 禁止删除/关闭

以下字段是首页和导航正常工作的基础，**修改前必须先确认影响范围**：

```yaml
params:
  # ⚠️ enabled 必须为 true，否则首页个人介绍、按钮、社交图标全部消失
  profileMode:
    enabled: true        # ← 禁止改为 false

  # ⚠️ 社交图标配置，首页会渲染
  socialIcons:           # ← 禁止删除整个段落
    - name: github
      url: "https://github.com/MrYangStudent"
    - name: email
      url: "mailto:yxiansheng907@gmail.com"

  # ⚠️ 按钮链接必须去掉前导 /（详见 relURL 陷阱说明）
  profileMode:
    buttons:             # ← 禁止删除整个段落
      - name: "阅读文章"
        url: "posts/"    # ← 不带前导 /，relURL 才会拼接 subpath
      - name: "关于我"
        url: "about/"

# ⚠️ 导航菜单也要存在
menu:
  main:                  # ← 禁止删除
    - identifier: posts
      name: 文章
      url: posts/
    - identifier: about
      name: 关于
      url: about/
```

### 2. Subpath 处理规则 — 所有链接必须用 `relURL`

- **baseURL**: `https://mryangstudent.github.io/YangSir.github.io/`
- **部署子路径**: `/YangSir.github.io/`
- **规则**: `layouts/` 中的所有自定义链接必须用 `{{ $url | relURL }}`，禁止硬编码路径
- **容易遗漏的地方**: `index.html` 的按钮链接、`partials/` 中的链接、自定义菜单模板

```html
<!-- ✅ 正确 — 去掉前导 /，relURL 才会拼接 subpath -->
<a href="{{ "posts/" | relURL }}">文章</a>

<!-- ❌ 错误 1 — 硬编码路径，部署后 404 -->
<a href="/posts/">文章</a>

<!-- ❌ 错误 2 — 带前导 /，relURL 不会拼接 subpath！（见下方陷阱说明） -->
<a href="{{ "/posts/" | relURL }}">文章</a>
```

### ⚠️ relURL 前导斜杠陷阱（导致 subpath 不生效的根因）

**Hugo `relURL` 反直觉行为**：当输入以 `/` 开头时，`relURL` 认为这是一个「相对于服务器根目录」的绝对路径，**不会**拼接 baseURL 的 subpath。

| 输入 | `relURL` 输出 | subpath 是否生效 |
|------|--------------|:--:|
| `"posts/"` | `/YangSir.github.io/posts/` | ✅ 生效 |
| `"/posts/"` | `/posts/` | ❌ 不生效 |

**因此所有传递给 `relURL` 的路径都必须去掉前导斜杠**：
- `hugo.yaml` 中 `buttons[].url`、`menu.main[].url` 都**不带**前导 `/`
- `layouts/` 模板中所有 `{{ "xxx" | relURL }}` 都**不带**前导 `/`

### 3. 首页完整依赖关系

首页（`layouts/index.html`）依赖以下文件和配置，**缺一不可**：

| 依赖项 | 作用 | 缺失后果 |
|--------|------|----------|
| `hugo.yaml` → `profileMode` | 个人介绍、按钮、标题 | 首页空白 |
| `hugo.yaml` → `socialIcons` | GitHub/Email 社交图标 | 图标不显示 |
| `layouts/partials/social_icons.html` | SVG 图标渲染 | 图标不显示 |
| `hugo.yaml` → `profileMode.enabled: true` | 控制渲染开关 | 个人介绍区域消失 |
| `content/about.md` | "关于我"页面 | 点"关于我"按钮 404 |
| `content/posts/_index.md` | 文章列表页面 | 点"阅读文章"按钮 404 |
| `content/search.md` | 搜索页面 | 导航"搜索" 404 |

### 4. `layouts/partials/` vs `_partials/` 区分

- **`layouts/partials/`**: 自定义 partial，可以被模板中 `partial "xxx.html"` 找到 ✅
- **`layouts/_partials/`**: Hugo 不搜索此目录，用于存放不会直接被调用的碎片文件 ⚠️
- **PaperMod 主题**: 内部 partials 放在 `themes/PaperMod/layouts/_partials/`，自定义模板**不能依赖它们**
- **规则**: 所有项目自定义的 partial 必须放在 `layouts/partials/`，不能在模板中引用 `_partials/` 下的文件

### 5. 先确认服务进程，再诊断页面问题

- **Hugo 进程必须存活**：出现 404 / 页面空白时，**第一步不是改配置，而是先检查 `hugo.exe` 是否在运行**
- 如果进程不存在，必须先按「本地预览」命令启动，等待 `Web Server is available` 后再验证页面
- 只有在服务运行且页面仍 404 时，才允许排查配置、路径、文件缺失等问题

```powershell
# 检查 hugo 是否运行
Get-Process hugo -ErrorAction SilentlyContinue | Select-Object Id, Path
```

---

### 本地预览

```powershell
# 1. 检查 hugo 是否已经在运行（如果已经运行则不要重复启动）
Get-Process hugo -ErrorAction SilentlyContinue | Select-Object Id, Path

# 2. 找到 hugo（可能在 GOPATH/bin 或 winget 目录）
$env:PATH = [Environment]::GetEnvironmentVariable("PATH","Machine") + ";" + 
            [Environment]::GetEnvironmentVariable("PATH","User")

# 3. 使用正确的 baseURL（必须带 subpath）
hugo serve --port 3131 --baseURL "http://localhost:3131/YangSir.github.io/" --bind "127.0.0.1"

# 4. 验证关键页面
#    - 首页:    http://localhost:3131/YangSir.github.io/
#    - 文章列表: http://localhost:3131/YangSir.github.io/posts/
#    - 关于页:   http://localhost:3131/YangSir.github.io/about/
```

### 每次修改后检查清单

修改 `hugo.yaml`、`layouts/`、`content/` 之后：

- [ ] 首页正常显示（个人介绍 + 社交图标 + 按钮 + 文章列表）
- [ ] "阅读文章"按钮跳转到 `/posts/` 返回 200
- [ ] "关于我"按钮跳转到 `/about/` 返回 200
- [ ] 导航栏"文章"/"标签"/"搜索"/"关于"四个链接都可达
- [ ] 社交图标（GitHub、Email）正常渲染

### 部署到 GitHub Pages

```powershell
git push origin master
# GitHub Actions 自动触发，无需手动操作
# 部署后在 https://mryangstudent.github.io/YangSir.github.io/ 验证
```

---

## 🚫 禁止操作

| ❌ 禁止 | 后果 | 正确做法 |
|---------|------|----------|
| 关闭 `profileMode.enabled` | 首页空白 | 保持 `enabled: true` |
| 删除 `socialIcons` 配置 | 社交图标消失 | 如需修改，先确保 `social_icons.html` partial 同步更新 |
| 删除 `profileMode.buttons` | 跳转按钮消失 | 保持至少 `posts` 和 `about` 两个按钮 |
| 硬编码链接路径（不用 relURL） | 部署后 404 | 始终使用 `relURL` |
| `relURL` 传入带前导 `/` 的路径 | subpath 不拼接，链接错误 | 去掉前导 `/`，用 `"posts/"` 而非 `"/posts/"` |
| 在 `_partials/` 添加被模板引用的文件 | 渲染失败 | 放在 `layouts/partials/` |
| 删除 `content/about.md` | "关于我"页面 404 | 保留该文件 |
| 删除 `content/posts/_index.md` | 文章列表 404 | 保留该文件 |
| 修改 `baseURL` 路径 | 全站链接失效 | 保持 `/YangSir.github.io/` subpath |
| 修改 CI 触发分支 | 部署失败 | 保持 `master` 分支 |

---

## 📁 关键文件清单（不可删除）

```
content/about.md          ← "关于我"页面
content/posts/_index.md   ← 文章列表页
content/search.md         ← 搜索页面
layouts/index.html        ← 自定义首页模板
layouts/partials/social_icons.html  ← 社交图标 partial
hugo.yaml                 ← 全站配置
.github/workflows/hugo-deploy.yml   ← CI/CD 部署
```

---

## 🔗 相关资源

- 源项目: https://github.com/MrYangStudent/github-blog-publisher
- 线上地址: https://mryangstudent.github.io/YangSir.github.io/
- PaperMod 文档: https://github.com/adityatelange/hugo-PaperMod/wiki
