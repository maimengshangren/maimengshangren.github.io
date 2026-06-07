# 博客使用与开发指南

## 基本信息

| 项目 | 值 |
|------|-----|
| 网站地址 | https://maimengshangren.github.io |
| GitHub 仓库 | https://github.com/maimengshangren/maimengshangren.github.io |
| 本地项目路径 | `~/agentdev/maimengshangren.github.io` |
| 技术栈 | Hexo 5.x + Butterfly 主题 |
| 部署分支 | `main`（源码）→ `gh-pages`（构建产物） |
| 自动部署 | 推送到 `main` 后 GitHub Actions 自动构建，约 1-2 分钟生效 |

---

## 文章目录结构

```
source/
├── _posts/          # 所有文章存放目录（脚本写入此处）
├── about/           # 关于我页面
├── categories/      # 分类页面（自动聚合，无需手动维护）
└── tags/            # 标签页面（自动聚合，无需手动维护）
```

---

## 文章格式（Front Matter）

每篇文章是一个 `.md` 文件，顶部必须包含 Front Matter：

```markdown
---
title: 文章标题
date: 2026-06-07 12:00:00
updated: 2026-06-07 12:00:00   # 可选，更新时间
categories:
  - 技术                        # 一级分类
  - 编程语言                    # 二级分类（可选）
tags:
  - Python
  - AI
cover: https://example.com/image.jpg  # 可选，封面图
description: 文章摘要，显示在首页卡片上  # 可选
---

正文内容（Markdown 格式）...
```

### 文件命名规则

- 文件名：`文章标题.md`（中文标题可用，Hexo 自动处理）
- 存放位置：`source/_posts/`
- URL 规则：`/:year/:month/:day/:title/`（由 `_config.yml` 的 `permalink` 决定）

---

## 手动发布文章步骤

```bash
# 1. 进入项目目录
cd ~/agentdev/maimengshangren.github.io

# 2. 创建新文章（自动生成带 Front Matter 模板的 md 文件）
hexo new post "文章标题"

# 3. 编辑文章
open source/_posts/文章标题.md

# 4. 本地预览（可选）
hexo server
# 访问 http://localhost:4000 查看效果，Ctrl+C 停止

# 5. 发布
git add source/_posts/
git commit -m "新文章：文章标题"
git push
```

---

## 自动写博客脚本所需信息

### 关键路径

```
BLOG_ROOT     = ~/agentdev/maimengshangren.github.io
POSTS_DIR     = ~/agentdev/maimengshangren.github.io/source/_posts
CONFIG_FILE   = ~/agentdev/maimengshangren.github.io/_config.yml
THEME_CONFIG  = ~/agentdev/maimengshangren.github.io/_config.butterfly.yml
```

### 脚本最小操作流程

```python
import os
import subprocess
from datetime import datetime

BLOG_ROOT = os.path.expanduser("~/agentdev/maimengshangren.github.io")
POSTS_DIR = os.path.join(BLOG_ROOT, "source/_posts")

def create_post(title: str, content: str, categories: list = [], tags: list = []):
    """创建并发布一篇文章"""
    date_str = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    filename = f"{title}.md"
    filepath = os.path.join(POSTS_DIR, filename)

    # 构建 Front Matter
    cats = "\n".join([f"  - {c}" for c in categories])
    tag_list = "\n".join([f"  - {t}" for t in tags])
    front_matter = f"""---
title: {title}
date: {date_str}
categories:
{cats}
tags:
{tag_list}
---

"""
    # 写入文件
    with open(filepath, "w", encoding="utf-8") as f:
        f.write(front_matter + content)

    # Git 提交推送
    subprocess.run(["git", "-C", BLOG_ROOT, "add", filepath])
    subprocess.run(["git", "-C", BLOG_ROOT, "commit", "-m", f"新文章：{title}"])
    subprocess.run(["git", "-C", BLOG_ROOT, "push"])

    return filepath
```

### Git 操作命令参考

```bash
# 添加文章
git -C ~/agentdev/maimengshangren.github.io add source/_posts/

# 提交
git -C ~/agentdev/maimengshangren.github.io commit -m "新文章：标题"

# 推送（触发自动部署）
git -C ~/agentdev/maimengshangren.github.io push origin main
```

### 注意事项

- **触发部署**：只需 push 到 `main` 分支，GitHub Actions 自动处理构建和部署
- **生效时间**：push 后约 1-2 分钟网站更新
- **文章 URL**：`https://maimengshangren.github.io/年/月/日/标题/`
- **图片**：建议用外链（如图床），或放入 `source/images/` 目录后用 `/images/xxx.jpg` 引用
- **草稿**：文件放入 `source/_drafts/` 不会发布，用 `hexo publish "标题"` 发布

---

## 部署架构

```
本地写文章
    │
    │ git push origin main
    ▼
GitHub main 分支
    │
    │ 触发 GitHub Actions (.github/workflows/deploy.yml)
    ▼
Actions 运行环境
    ├── npm install          # 安装依赖
    └── hexo generate        # 构建静态文件
    │
    │ peaceiris/actions-gh-pages 推送到 gh-pages 分支
    ▼
GitHub gh-pages 分支（静态 HTML/CSS/JS）
    │
    │ GitHub Pages 自动服务
    ▼
https://maimengshangren.github.io
```

---

## 本次部署过程记录（2026-06-07）

1. 安装 `hexo-cli`、`hexo-theme-butterfly`、`hexo-renderer-pug`、`hexo-renderer-stylus`
2. `hexo init` 初始化项目，配置中文、时区、Butterfly 主题
3. 创建分类、标签、关于我三个页面，配置导航菜单和 GitHub/Email 社交链接
4. 编写 GitHub Actions 部署脚本（`.github/workflows/deploy.yml`）
5. 使用 `gh` CLI 创建仓库 `maimengshangren/maimengshangren.github.io`
6. 通过 API 开启 Actions 写入权限（`default_workflow_permissions: write`）
7. 推送代码，Actions 构建成功，静态文件推送至 `gh-pages` 分支
8. 通过 API 将 GitHub Pages 源切换为 `gh-pages` 分支
9. 网站上线：https://maimengshangren.github.io
