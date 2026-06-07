---
title: 为 publisher 接入桌面 GUI：第一步把 CLI 包成 Python API
date: '2026-06-07 21:00:00'
tags:
- Python
- FastAPI
- SSE
- Tauri
categories:
- 技术
description: publisher 桌面化第一阶段（D1）的实现记录——为什么走 Tauri + Python sidecar、API 的形状如何与 7
  条 MVP 对齐，以及发布进度用 job + SSE 模型的几个关键技术细节。
author: Layne
slug: wei-publisher-jie-ru-zhuo-mian-gui-di-yi-bu-ba-cli-bao-cheng-python-api
---

[上一篇](/2026/06/07/zi-jian-bo-ke-fa-bu-qi-publisher-de-shi-xian-si-lu/)我把 publisher 这个发布器的 CLI 版本做出来了：本地 Markdown 或 Notion 文章一键发到 Hexo 博客。CLI 自己用是够了，但要把"草稿箱、已发管理、撤回、站点风格编辑"这些可视化需求落地，得加个桌面 GUI。

这一篇记录 D1 阶段：**把 CLI 的核心能力包成 HTTP API**，给后续 Tauri 前端用。整篇都是技术细节，不长。

## 形态决定：Tauri + Python sidecar

技术栈在动手之前先拍板。备选三个：

| 方案 | 优点 | 缺点 |
|---|---|---|
| Electron + Node 重写 | 生态熟 | 包大、内存高，还得把 publisher 的 Python 逻辑全翻译一遍 |
| 纯 Rust + Tauri | 包最小、最快 | 现有 Python 代码作废 |
| **Tauri + Python sidecar** | 包小、复用 publisher 包、桌面原生体验 | 需要让 Tauri 启动时拉起 Python 子进程，端到端调试稍麻烦 |

选第三个。Tauri 主进程（Rust + WebView）只做壳子，Python 作为 sidecar 跑一个本地 HTTP 服务，前端 fetch 调它。最终 `.app` 里就是 Tauri 二进制 + PyInstaller 打包的 Python 子进程。

工程结构是这样：

```
publisher/
├── src/publisher/          # 已有的 CLI / 核心
│   └── api/                # 新增的 FastAPI 层
└── desktop/                # 下一阶段会出现的 Tauri 工程
    ├── src-tauri/
    └── src/                # 前端
```

API 层的定位是个**适配器**——它不实现业务，只把已有的 publisher 模块（Source / Render / Target / State）包成 HTTP 接口。

## 接口形状：从 7 条 MVP 倒推

之前确定的 7 条 MVP 是：

1. 文章列表（本地 + Notion + 已发）+ 状态/来源徽章
2. 在 VS Code 中打开 + 文件监听
3. 发布前预检
4. Dry-run 预览
5. 发布进度（写 → 拷图 → commit → push → CI）
6. 撤回（归档 + 删 + push）
7. 设置

每条 MVP 对应到一组路由：

| MVP | 路由 | 备注 |
|---|---|---|
| 1 | `GET /articles/{local,notion,published}` | 三个数据源各一个端点，前端各自缓存、按需刷新 |
| 2 | 不需要后端 | "在 VS Code 中打开" 是 Tauri 侧 `Command::new("code")`，文件监听用 `notify` crate |
| 3、4 | `POST /publish/dry-run` | 同步返回 frontmatter、落点、commit message、资产数 |
| 5 | `POST /publish` + `GET /publish/jobs/{id}/events`（SSE） | job 模型 + 进度流 |
| 6 | `POST /withdraw` | 同步：archive + 删 + commit + push |
| 7 | 不需要后端 | 设置直接读写 `config.yaml` 和系统 Keychain |

还有支撑型路由：

```
GET /healthz                 # sidecar 存活探测，Tauri 启动时 poll
GET /status/repo             # 博客仓库 git 状态（dirty/ahead/behind）
GET /status/ci?limit=5       # GitHub Actions 最近运行
GET /status/notion           # Notion 配置是否就绪
GET /preview/status          # D4 的占位
```

## 关键技术决策：发布进度的 job + SSE 模型

文章列表、撤回、状态都是请求-响应一发了之，没什么好说的。**发布这条路是异步的**：写文件、拷图、commit、push 几秒到几十秒，需要把每一步进度推给前端。这块设计要稍微讲讲。

### 为什么不用 WebSocket

WebSocket 是双向，我们这里只需要服务端 → 前端推；用 SSE（Server-Sent Events）足够，浏览器原生 `EventSource` API 直接接，自动重连，不需要前端引第三方库。Tauri 的 webview 一样支持。

### 任务对象与事件队列

每次 `POST /publish` 创建一个 `Job`，给一个 12 位的 id：

```python
@dataclass
class Job:
    job_id: str
    kind: str             # publish | withdraw
    status: str           # pending | running | success | failed
    events: list[JobEvent]   # 缓冲：方便晚订阅者 replay
    result: Optional[dict]
    error: Optional[str]
    loop: Optional[asyncio.AbstractEventLoop]
    queue: Optional[asyncio.Queue]  # 给 SSE handler 用
```

注意有两套数据结构记事件：

- `events: list` —— 一直缓冲，任何时刻拿来 replay
- `queue: asyncio.Queue` —— SSE handler 从这里阻塞读，每条事件实时推给客户端

`queue` 是懒创建的——只有当**真的有人来订阅 SSE 时**才在那个连接所在的 event loop 上创建。这避免了 worker 启动时还不知道哪个 loop 上会消费的尴尬。

### 跨线程把事件送回事件循环

worker 跑在 `run_in_executor` 的线程池里（同步代码，复用 publish_to_hexo）。事件回调要把进度从**工作线程**送回**主 event loop**。这一步做错就是死锁或数据丢：

```python
def push(self, ev: JobEvent) -> None:
    self.events.append(ev)                              # 缓冲是同步的，OK
    if self.queue is not None and self.loop is not None:
        # 这里是关键：不能直接 self.queue.put_nowait()
        # 因为我们在 worker 线程，queue 绑在主 loop 上
        self.loop.call_soon_threadsafe(self.queue.put_nowait, ev)
```

`call_soon_threadsafe` 是 asyncio 给跨线程通信的官方接口，它会把回调投到目标 loop 的就绪队列，由 loop 自己来执行。错用 `asyncio.run_coroutine_threadsafe` 也行但更重。

### 晚订阅者 replay

如果 SSE 客户端连得慢（前端先 POST 拿到 job_id，再发 GET 订阅 events 的话有时间窗），前几条事件已经发生了。处理方式是 attach 时把 `events` 里已缓冲的全部一次性推进 `queue`：

```python
def attach_queue(self, job_id: str) -> Optional[Job]:
    job = self.get(job_id)
    if job is None: return None
    if job.queue is None:
        job.queue = asyncio.Queue()
        job.loop = asyncio.get_running_loop()
        for ev in job.events:           # 历史事件全部 replay
            job.queue.put_nowait(ev)
    return job
```

这一招让前端不必担心订阅时机——晚连也能看到完整流。

### 心跳

SSE 长连接经过任何中间件都会担心被超时关掉。我在生成器里加了 30 秒超时 ping：

```python
async def gen():
    while True:
        try:
            ev = await asyncio.wait_for(job.queue.get(), timeout=30.0)
        except asyncio.TimeoutError:
            yield {"event": "ping", "data": "{}"}
            if job.status in ("success", "failed"): return
            continue
        yield {"event": ev.step, "data": json.dumps(ev.to_dict(), ensure_ascii=False)}
        if ev.level in ("done", "error"): return
```

## 复用 CLI 核心：progress 回调

我不想让 API 这层自己再写一份"写文件 → 拷图 → commit → push"的流水。原本的 `publish_to_hexo` 已经做了这些，CLI 里就在用。怎么让 API 拿到细粒度进度？给函数加一个可选回调：

```python
ProgressCb = Optional[Callable[[str, str], None]]

def publish_to_hexo(article, cfg, repo_root, *, dry_run=False, push=True,
                    progress: ProgressCb = None) -> PublishResult:
    _emit(progress, "render", f"渲染：{article.title}")
    rendered = render(article, cfg, repo_root)
    ...
    _emit(progress, "write", f"写入 {rendered.post_path}")
    ...
    _emit(progress, "copy-assets", f"已拷贝 {len(article.assets)} 张图片")
    ...
    _emit(progress, "commit", f"{commit.hexsha[:8]} {msg}")
    ...
    _emit(progress, "push", f"已推送到 origin/{cfg.git.branch}")
```

CLI 不传 `progress`，行为完全不变；API 这边传一个把事件 push 进 `Job` 的闭包：

```python
def cb(step: str, message: str) -> None:
    job.push(JobEvent(step=step, message=message))

result = publish_to_hexo(article, cfg.hexo, repo_root,
                         dry_run=False, push=req.push, progress=cb)
```

一份核心，两种入口。这是这次 D1 最让我自己满意的小决定——它把"加 GUI"的成本压在了 API 适配层，**核心模块零迁就**。

## 状态机的两个新方法

撤回需要从已发表里删一条记录，所以 `StateStore` 加了 `list_published` 和 `delete_published`：

```python
def list_published(self, target=None) -> list[dict]:
    sql = "SELECT source_id, target, slug, title, published_at FROM published"
    args = ()
    if target:
        sql += " WHERE target=?"
        args = (target,)
    sql += " ORDER BY published_at DESC"
    ...

def delete_published(self, source_id: str, target: str) -> None:
    ...
```

撤回流程：

1. `archive/<时间戳>/` 拷一份本地备份
2. 从博客仓库删源文件
3. `git commit -m "revert: withdraw <slug>"` + push
4. 从 state DB 里删记录

四步全部在 `POST /withdraw` 里同步完成。归档先做、push 最后做——这个顺序保证即使中途崩了也至少有本地副本，下次能从 archive 恢复。

## 几个踩到的小坑

### 1. 仓库名含点的 origin URL 正则

我的博客仓库叫 `maimengshangren.github.io`——repo 名本身含点，最初的正则 `[^/.]+` 把它截在 `maimengshangren` 就停了。改成：

```python
cleaned = url.rstrip("/")
if cleaned.endswith(".git"):
    cleaned = cleaned[:-4]
m = re.search(r"github\.com[:/]([^/]+)/([^/]+)$", cleaned)
```

只剥 `.git` 后缀，其余允许任何字符。

### 2. GitHub Actions API 的匿名 403

`status/ci` 用的是 `https://api.github.com/repos/<owner>/<repo>/actions/runs`，不带 token 时 GitHub 给的匿名速率限制极低，本机调几次就 403。生产里建议 `.env` 里加 `GITHUB_TOKEN`，API 会自动带上：

```python
token = os.environ.get("GITHUB_TOKEN")
if token:
    headers["Authorization"] = f"Bearer {token}"
```

### 3. CORS

sidecar 监听 127.0.0.1，开发时前端跑在 Vite dev server 上是 `http://localhost:5173`，两个 origin。本来 CORS 是个安全话题，但局限在 loopback 的进程里放开全部 origin 完全可以接受：

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### 4. preview 接口先占位

`POST /preview/{start,stop,restart}` 直接 `raise HTTPException(501)`——前端可以早早把按钮接好，等 D4 真做 hexo s 进程管理时只改 handler 不改路由。

## 最终目录

```
src/publisher/api/
├── app.py            # FastAPI 工厂 + run() 入口
├── jobs.py           # 内存 job 注册表 + 跨线程 SSE 队列
└── routes/
    ├── articles.py   # /articles/{local,notion,published}
    ├── publish.py    # /publish, /publish/jobs/{id}, /publish/jobs/{id}/events
    ├── withdraw.py   # /withdraw
    ├── status.py     # /status/{repo,ci,notion}
    └── preview.py    # 501 占位
```

启动：

```bash
publisher-api --port 8765
```

`pyproject.toml` 那边把 `publisher-api = "publisher.api.app:run"` 注册成 console script，PyInstaller 后还是这个入口。

## 一些不写在代码里的原则

D1 推进时反复用到的几条心法：

- **不破坏 CLI**：CLI 是这个工具的第一形态，加 GUI 不该让 CLI 体验劣化。所以 `publish_to_hexo` 的 `progress` 参数是可选的，所有 API 改动都在 `publisher/api/` 这个新包里。
- **同步 vs 异步看场景**：dry-run 几十毫秒能返回，就同步 JSON；publish 秒级，必须异步 + 流式。一律异步会让简单事情复杂化。
- **状态留在最该留的地方**：job 存内存（sidecar 单进程，重启清空也无所谓），published 存 SQLite（要跨进程持久化），Notion token 之后存 Keychain（不能进 git）。
- **占位 501 比空路由好**：路由形状已经定了的就先建好，handler 写 501 而不是不存在。这样前端可以提前 wire 上按钮，看到 501 比 404 信息量大很多。

## 下一步

D2 是 Tauri 工程脚手架 + sidecar 启停管理 + Settings + Articles Tab 的只读列表。预计是 D1 三倍工作量，主要在前端那一坨。代码会继续放在与博客同级的 `publisher/` 目录里，且这一篇之后会建好独立的 GitHub 仓库做版本管理。

下一篇见。
