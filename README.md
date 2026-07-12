
# YangSir's Tech Blog

> 基于 [Hugo](https://gohugo.io/) + [PaperMod](https://github.com/adityatelange/hugo-PaperMod) 主题的个人技术博客，通过 GitHub Actions 自动部署到 GitHub Pages。

[![Hugo](https://img.shields.io/badge/Hugo-0.140+-FF4088?logo=hugo)](https://gohugo.io/)
[![Theme](https://img.shields.io/badge/Theme-PaperMod-blue)](https://github.com/adityatelange/hugo-PaperMod)
[![GitHub Pages](https://img.shields.io/badge/GitHub%20Pages-Active-success?logo=github)](https://mryangstudent.github.io/YangSir.github.io/)

## 博客内容

| 文章 | 简介 |
|------|------|
| [从零微调大模型：用 Qwen2.5-7B 训练一个会写小说的 AI](https://mryangstudent.github.io/YangSir.github.io/posts/llm-novel-finetune/) | 零基础 LLM 微调实战指南。从 CUDA 环境搭建到模型推理，手把手教你用 QLoRA 微调 Qwen2.5-7B |

**覆盖领域**：LLM · Fine-tuning · QLoRA/LoRA · NLP · 深度学习 · 项目实践

---

## 本地运行

```bash
# 克隆仓库（含子模块）
git clone --recurse-submodules https://github.com/MrYangStudent/YangSir.github.io.git
cd YangSir.github.io

# 启动开发服务器
hugo serve -D

# 访问 http://localhost:1313
```

> 需要安装 [Hugo Extended](https://gohugo.io/installation/) v0.140+

## 项目结构

```
├── archetypes/       # 文章模板
├── assets/           # 自定义 CSS 样式
├── content/          # 博客文章（Markdown）
│   └── posts/        # 文章目录
├── layouts/          # 自定义页面模板
├── themes/           # PaperMod 主题（submodule）
├── hugo.yaml         # Hugo 配置文件
└── .github/          # GitHub Actions CI/CD
```

## 写新文章

```bash
hugo new posts/文章名称/index.md
```

Markdown 文件头部加上 front matter：

```yaml
---
title: "文章标题"
date: 2026-07-12
tags: ["Go", "LLM"]
draft: true
---
```

将 `draft: true` 改为 `false` 或删掉即可发布。

## 部署

推送 `master` 分支后，GitHub Actions 会自动构建并部署到 GitHub Pages。

## 许可证

博客文章版权归作者所有，代码部分采用 MIT License。
