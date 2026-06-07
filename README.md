# Layne's Blog

> 个人博客，基于 Hexo + Butterfly 主题，部署于 GitHub Pages。

**🌐 访问地址：[https://maimengshangren.github.io](https://maimengshangren.github.io)**

---

## 技术栈

| 组件 | 版本 |
|------|------|
| [Hexo](https://hexo.io) | ^8.0.0 |
| [Butterfly 主题](https://butterfly.js.org) | ^5.5.4 |
| 部署平台 | GitHub Pages |
| CI/CD | GitHub Actions |

## 项目结构

```
.
├── source/
│   ├── _posts/          # 文章目录（Markdown）
│   ├── about/           # 关于我页面
│   ├── categories/      # 分类页
│   ├── tags/            # 标签页
│   └── img/             # 图片资源
├── _config.yml          # Hexo 主配置
├── _config.butterfly.yml # Butterfly 主题配置
├── .github/workflows/   # 自动部署脚本
└── BLOG_GUIDE.md        # 写文章与自动化脚本指南
```

## 本地开发

```bash
# 安装依赖
npm install

# 启动本地预览（访问 http://localhost:4000）
npx hexo server

# 新建文章
npx hexo new post "文章标题"

# 构建静态文件
npx hexo generate
```

## 发布文章

编辑 `source/_posts/` 下的 Markdown 文件后，推送到 `main` 分支即可自动部署：

```bash
git add source/_posts/
git commit -m "新文章：文章标题"
git push
```

推送后 GitHub Actions 自动构建并发布，约 1-2 分钟生效。

## 自动化

详见 [BLOG_GUIDE.md](./BLOG_GUIDE.md)，包含自动写博客脚本所需的路径、Front Matter 格式和 Python 示例代码。

## 部署流程

```
git push main
    └─► GitHub Actions 构建 (hexo generate)
            └─► 推送静态文件到 gh-pages 分支
                    └─► GitHub Pages 对外服务
```
