---
title: 自建博客发布器 publisher 的实现思路
date: '2026-06-07 20:00:00'
tags:
- Python
- Hexo
- Notion
- 工具链
categories:
- 技术
description: 一个把本地 Markdown / Notion 文章一键发布到 Hexo 博客（后续接入微信公众号）的小工具，从架构到关键细节的完整记录。
author: Layne
slug: zi-jian-bo-ke-fa-bu-qi-publisher-de-shi-xian-si-lu
---

## 为什么自己写

我的博客是 Hexo + Butterfly 主题，托管在 GitHub Pages，push 到 `main` 由 GitHub Actions 自动 build 然后发 `gh-pages`。链路本身已经很顺，但写作侧仍然碎：

- 草稿可能在 Notion 里，也可能在本地 `*.md` 里，每次手动复制粘贴、改 frontmatter、传图片
- 同一篇要发到公众号，又得重排版、再传一遍图
- 没有"已发"状态机，重复发布的概率不小

于是写一个发布器 `publisher`，跟博客仓库同级目录，定位是**单人写作的发布前置层**：把"内容来自哪里"和"要发到哪里"解耦，自己只关心写作本身。

本文记录这个工具的实现思路。M1（本地 → Hexo）和 M2（Notion → Hexo）已经完成，M3（公众号草稿）排在后面。

## 总体架构

核心是三段式：

```
┌─────────────┐      ┌──────────────┐      ┌─────────────────┐
│ Source      │      │ Article      │      │ Target          │
│ - 本地 md   │ ───▶ │ (标准化模型) │ ───▶ │ - Hexo (Git)    │
│ - Notion    │      │ 标题/正文/   │      │ - 微信公众号    │
└─────────────┘      │ 标签/图片…   │      └─────────────────┘
                     └──────────────┘
```

**Source** 只做一件事：把任意来源的内容拉成统一的 `Article` 模型。**Target** 只关心怎么把 `Article` 落到某个发布渠道。中间的 `Article` 不依赖任何具体来源或渠道，方便后续加新源（语雀、Obsidian）或新渠道（公众号、知乎）。

工程目录：

```
publisher/
├── pyproject.toml
├── config.yaml             # 路径、字段映射
├── .env.example            # NOTION_TOKEN 等敏感配置
└── src/publisher/
    ├── cli.py              # typer 入口
    ├── models.py           # Article / Asset
    ├── config.py           # 配置加载
    ├── sources/
    │   ├── local.py        # 本地 md
    │   └── notion.py       # Notion API
    ├── targets/
    │   └── hexo.py         # 写文件 + git push
    ├── render/
    │   └── md_to_hexo.py   # frontmatter + 图片路径重写
    └── utils/{slug,state,image,log}.py
```

## 数据模型

`Article` 故意做得很轻，只保留发布所需的最小信息：

```python
@dataclass
class Article:
    title: str
    body_md: str
    source_id: str       # 用于幂等
    source_kind: str     # "local" | "notion"
    slug: str | None
    date: datetime | None
    tags: list[str]
    categories: list[str]
    cover: str | None
    summary: str | None
    author: str | None
    assets: list[Asset]  # 文章引用的图片
    targets: list[str]   # 来源指定的发布目标（可空）
```

`Asset` 表示一个本地化后的图片：

```python
@dataclass
class Asset:
    local_path: Path      # 一定指向本地真实文件
    remote_url: str | None
    ref_in_body: str      # 正文里出现的原始引用串，用作替换 key
```

关键点：**所有图片在进入 `Article` 之前一定要被下载到本地**。Notion 给的图片 URL 是 1 小时过期的临时 S3 链接，直接写进 Hexo 或公众号就是个定时炸弹。Source 层就把它解决掉，下游不用再担心。

## Local source：本地 Markdown

最简单的源：

1. 用 `python-frontmatter` 解析 YAML 头
2. 缺字段就走 `config.yaml` 的默认值兜底（默认分类 `技术` 等）
3. 用正则 `!\[([^\]]*)\]\(([^)]+)\)` 扫正文里的图片引用，相对路径解析成绝对路径，挂到 `assets`，远程 URL 跳过
4. `source_id` 用 `local:<文件路径 sha1>`，便于幂等去重

```python
for _alt, url in _IMG_RE.findall(body):
    if _is_remote(url) or url in seen:
        continue
    abs_path = (path.parent / url).resolve()
    if not abs_path.exists():
        continue
    assets.append(Asset(local_path=abs_path, ref_in_body=url))
```

`ref_in_body` 保留原样（比如 `./img/cover.png`），后面 render 阶段就用它做精确替换。

## Notion source：M2 的核心

Notion 这部分最复杂，三件事：**字段映射、块递归遍历、图片即时落盘**。

### 字段映射

Notion 数据库的字段名因人而异。我在 `config.yaml` 里全配成可改：

```yaml
notion:
  property_title: Name
  property_slug: Slug
  property_tags: Tags
  property_categories: Categories
  property_cover: Cover
  property_status: Status
  property_targets: Targets
  status_ready: 待发布
  status_published: 已发布
```

读取时按类型分发：

```python
def _read_multi_select(page, name) -> list[str]:
    p = page["properties"].get(name)
    if not p or p["type"] != "multi_select":
        return []
    return [opt["name"] for opt in p.get("multi_select") or []]
```

### 块递归 + 分页

`blocks.children.list` 单次最多返回 100 个块，要按 `next_cursor` 一直翻：

```python
def _iter_all_blocks(notion, block_id):
    cursor = None
    while True:
        resp = notion.blocks.children.list(
            block_id=block_id, start_cursor=cursor, page_size=100,
        )
        for b in resp["results"]:
            yield b
        if not resp["has_more"]:
            return
        cursor = resp["next_cursor"]
```

绝大多数块带 `has_children`（toggle、quote、列表项嵌套），递归一层就够。

### 块到 Markdown

我支持的块：`paragraph / heading_1-3 / bulleted_list_item / numbered_list_item / to_do / quote / callout / code / divider / image / bookmark / equation / table`，富文本支持 `bold / italic / code / strikethrough / link / equation`。

实现上有几个小坑：

- **有序列表的编号**要自己维护，连续的 `numbered_list_item` 计数，遇到其他块归零
- **callout 的 emoji** 在 `icon.emoji` 里，兜底用 💡
- **table 不能 fall through 递归**——`table_row` 已经在自己的逻辑里消费了，要 `continue`
- **缩进** 用 `"  " * indent` 实现嵌套列表的对齐

### 图片下载

碰到 `image` 块就立刻下载到本地缓存，再把 `Asset` 塞进 `assets`：

```python
url = _file_url(v)
local = download_image(url, image_cache)
assets.append(Asset(local_path=local, remote_url=url, ref_in_body=url))
lines.append(f"![{caption}]({url})")
```

缓存按 URL 的 sha1 命名，同篇文章重发不会重新下载：

```python
key = hashlib.sha1(url.encode()).hexdigest()[:16]
for existing in cache_dir.glob(f"{key}.*"):
    return existing
```

正文里**仍然写原始 URL**——后面 render 阶段会用 `Asset.ref_in_body` 精确替换成 Hexo 的公开路径。这一招让 Source 不需要关心目标渠道的路径规则。

## Render：Markdown → Hexo

`md_to_hexo.py` 做两件事：

### 1. 生成 frontmatter

按 Hexo 习惯输出标准 YAML：

```python
fm = {
    "title": article.title,
    "date": date.strftime("%Y-%m-%d %H:%M:%S"),
}
if article.tags:       fm["tags"] = article.tags
if article.categories: fm["categories"] = article.categories
if article.cover:      fm["cover"] = article.cover
...
text = yaml.safe_dump(fm, allow_unicode=True, sort_keys=False)
```

`allow_unicode=True` 不能漏，否则中文 tag 会变 `\uXXXX`。

### 2. 改写图片路径

每张 asset 落到 `source/img/<yyyy-mm>/<slug>/NN-原文件名.ext`：

```python
for idx, asset in enumerate(article.assets):
    dest = image_dir / slug / f"{idx:02d}-{asset.local_path.stem}{ext}"
    asset_dest[asset] = dest
    public_url = "/" + str(dest.relative_to(repo_root / "source"))
    body = body.replace(f"]({asset.ref_in_body})", f"]({public_url})")
```

为什么按 `<slug>/` 再开一层子目录？避免不同文章里都叫 `cover.png` 的图互相覆盖。

为什么用 `00-原文件名` 前缀？保留可读性的同时，让同名图也有唯一文件名。

slug 的处理也有兜底：`python-slugify` 对纯中文返回空串，所以用"中文片段 + 文件路径 sha1 前 10 位"：

```python
cjk_part = "".join(_CJK_RE.findall(title))[:20]
return f"{cjk_part}-{digest}" if cjk_part else f"post-{digest}"
```

## Hexo target：写文件 + git push

落盘的事不多：

```python
rendered.post_path.write_text(rendered.body, encoding="utf-8")
copy_assets(article, rendered)
```

然后用 `GitPython` 做提交：

```python
repo = git.Repo(repo_root)
repo.git.add(*rel_paths)
commit = repo.index.commit(msg)
origin.push(refspec=f"HEAD:{branch}")
```

提交信息模板可配（`commit_message_template: "post: {title}"`）。push 完剩下的就交给原本的 GitHub Actions：build → 推 `gh-pages` → Pages 上线。

`--dry-run` 时不写仓库，只把渲染产物落到 `publisher/out/`，便于先肉眼检查再正式发。`--no-push` 介于两者之间：写、提交但不 push，方便本地最后改一手再推。

## 幂等与状态

为了避免重复发，用 SQLite 存"已发表"记录：

```sql
CREATE TABLE published (
    source_id    TEXT NOT NULL,
    target       TEXT NOT NULL,
    slug         TEXT NOT NULL,
    title        TEXT,
    published_at TEXT NOT NULL,
    PRIMARY KEY (source_id, target)
);
```

`source_id` 是 `local:<sha1>` 或 `notion:<page-id>`。`target` 是 `hexo` 或 `wechat`。同篇文章发两个渠道独立计算。强制重发用 `--force`。

Notion 那侧成功后会把 `Status` 字段更新为 `已发布` 做状态机闭环：

```python
notion.pages.update(
    page_id=page_id,
    properties={"Status": {"select": {"name": "已发布"}}},
)
```

回写失败只打 warn 不阻塞主流程——博客已经推出去了，状态字段稍后人工改也无所谓。

## CLI

用 `typer` 起手，三个命令：

```bash
# 看 Notion 里 "待发布" 列表
publisher list --source notion

# 本地 md → Hexo（最常用）
publisher publish --source local --file path/to/article.md

# Notion 单页 / 全部待发布
publisher publish --source notion --page <id>
publisher publish --source notion --all

# 调试 / 安全开关
publisher publish --source local --file x.md --dry-run    # 只渲染不推
publisher publish --source local --file x.md --no-push    # 提交不推
publisher publish --source notion --page <id> --force     # 已发的也重推
```

`--dry-run` 在这种"写一次就改别人仓库 + push 到公网"的工具里是必须的，第一次跑、改 render 逻辑、加新块类型时都用得上。

## 留给 M3（公众号）的钩子

接公众号时只动两层：

- `targets/wechat.py`：access_token 缓存、永久素材上传、Markdown → 带 inline CSS 的 HTML、`/cgi-bin/draft/add`
- `render/md_to_wx_html.py`：Markdown 渲染 + `premailer` 内联 CSS

CLI 那边 `--to hexo,wechat` 的入口形态已经留好。`Article` 模型不需要改——Hexo 用得到的字段，公众号草稿基本都用得到。

短期内只发草稿、不直接走 `/freepublish/submit`，避免误发。

## 一些原则上的取舍

- **不在源层做渠道相关的事**。Notion 拉下来的图片不直接重写成 Hexo 路径，因为后面公众号要走自己的素材上传链路，提前重写会让流程多绕一层。
- **`Article` 不可变信息越少越好**。比如 `date` 来源放在 Source，渲染层不去猜。
- **配置和敏感信息分离**。`config.yaml` 进 git，`.env` 不进。
- **失败要响但不要炸**。Notion 回写、图片下载这类副作用环节失败只打 warn，主链路（推到 Hexo）不阻断。

## 下一步

- M3：公众号草稿
- 给 `Article` 加 `excerpt` 自动从首段截，公众号摘要用得到
- 加 `publisher new` 命令，从 scaffold 生成草稿框架
- 多文章批量发的进度条 / 失败汇总

整套代码在与博客仓库同级的 `publisher/` 目录里。下一篇会写公众号那段的具体踩坑。
