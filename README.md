# RickDamon's Blog

使用 **Hexo** + **Matery** 主题搭建的个人博客。

## 项目结构

```
.
├── source/              # 博客源文件目录
│   └── _posts/         # 博客文章（Markdown格式）
├── themes/             # 主题目录
│   └── matery/         # Matery 主题
├── public/             # 生成的静态网站（生产文件）
├── _config.yml         # Hexo 主配置文件
├── package.json        # 项目依赖配置
└── _backup/            # 备份的原始静态文件
```

## 快速开始

### 安装依赖
```bash
npm install
```

### 创建新文章
```bash
hexo new post "文章标题"
```

### 本地预览
```bash
hexo server
# 或简写
hexo s
```
然后访问 `http://localhost:4000`

### 生成静态网站
```bash
hexo generate
# 或简写
hexo g
```

### 清理缓存
```bash
hexo clean
```

### 发布到 GitHub Pages
```bash
hexo deploy
# 或简写
hexo d
```

## 常用命令

| 命令 | 简写 | 说明 |
|------|------|------|
| `hexo new post "标题"` | `hexo n post "标题"` | 创建新文章 |
| `hexo new page "页面名"` | `hexo n page "页面名"` | 创建新页面 |
| `hexo generate` | `hexo g` | 生成静态文件 |
| `hexo server` | `hexo s` | 本地预览 |
| `hexo clean` | - | 清理生成的文件 |
| `hexo deploy` | `hexo d` | 部署到远程仓库 |

## 文章编写

新建文章会在 `source/_posts/` 目录下生成一个 Markdown 文件。文件头部包含文章元数据：

```markdown
---
title: 文章标题
date: 2026-01-06 17:30:00
categories: 分类名
tags: 
  - 标签1
  - 标签2
---

# 文章内容从这里开始
```

## Matery 主题配置

主题配置文件位于 `themes/matery/_config.yml`，可以根据需要自定义：

- 网站标题、描述
- 菜单导航
- 社交链接
- 评论系统
- 分析工具等

详见 [Matery 主题文档](https://github.com/blinkfox/hexo-theme-matery/blob/master/README_CN.md)

## 备份

原始的静态文件已备份在 `_backup/` 目录中，如需查看可以参考。

## 重要提示

- 修改文章后需要重新运行 `hexo generate` 生成新的静态文件
- 发布前建议使用 `hexo server` 本地预览效果
- 建议定期 `git commit` 保存源文件版本

## 更新日志

- 2026-01-06: 重新初始化 Hexo 项目，集成 Matery 主题
