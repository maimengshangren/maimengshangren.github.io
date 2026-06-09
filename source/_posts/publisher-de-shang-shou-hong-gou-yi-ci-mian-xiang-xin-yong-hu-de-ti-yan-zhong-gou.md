---
title: publisher 的上手鸿沟：一次面向新用户的体验重构
date: '2026-06-08 22:30:00'
tags:
- 产品设计
- 上手体验
- Tauri
- React
- 自建工具
categories:
- 技术
description: 自己写的工具一开始只对自己友好。把它从"我用着顺"改造到"别人也能跑通"，做了一次 PM 视角的体验审计和一套引导向导，记录这个过程里关于设计、诊断、模板创建、错误回退的思考与坑。
author: Layne
slug: publisher-de-shang-shou-hong-gou-yi-ci-mian-xiang-xin-yong-hu-de-ti-yan-zhong-gou
---

## 起点：工具能用，但只对我自己

发布器 `publisher` 已经能从本地 Markdown / Notion 一路推到 Hexo 博客线上，CLI 跑通了，桌面 GUI（Tauri + Python sidecar）也跑起来了。我自己在用，每天发一篇，挺顺。

直到我假想"如果一个新人想用这个工具会怎么样"。

我在脑子里跑了一遍流程：

```
1. 装 Homebrew、Node、Python、Rust、gh
2. clone publisher
3. python -m venv + pip install
4. cd desktop && npm install
5. npm run tauri dev  ← 这里 Rust 第一次编译 5-10 分钟
6. 终于打开 App，看到空 Articles 列表 + 一个紫色按钮
7. ⚙ 设置 → 看到 `repo_path: ../...` ← 我哪有那个目录？
8. 📊 数据 → 红色 "GitHub Token 未配置"
9. 不知道下一步该干啥
```

第 1–5 步劝退一半人，第 6–9 步劝退另一半。

**这不是功能不全的问题，是上手鸿沟的问题。**

## 戴上 PM 帽子：把摩擦点摘出来

强行换视角，把现状所有让新用户卡壳的地方列下来：

| # | 问题 | 痛点深度 |
|---|---|---|
| 1 | App 默认进 Articles，假设你已经配好 | ⚫⚫⚫ 阻断 |
| 2 | 没博客仓库的人**根本走不下去** | ⚫⚫⚫⚫ 流失 |
| 3 | gh 一键导入按钮藏在 Stats 401 错误里 | ⚫⚫ 反直觉 |
| 4 | 没有"配置完整度"指示 | ⚫⚫ 焦虑 |
| 5 | 发布失败就是个 stderr toast | ⚫⚫⚫ 看不懂 |

修补方向也清晰：

- **引导向导**：第一次打开时强制走一遍关键配置
- **诊断页**：把工具的健康状态做成一个可看可修的清单
- **草稿模板**：第一篇有内容可参考，不至于对着空 md 发愁

## 引导向导：4 步全屏 wizard

4 步的设计是经过取舍的——3 步太赶，5 步嫌啰嗦：

```
欢迎  →  GitHub  →  博客  →  验证
 1       2         3       4
```

### Step 1 · 欢迎

故意做得**只读**——一段说明 + 3 个 bullet。让用户先理解工具在帮 ta 解决什么，再开始做事。没有 input、没有按钮，只有「下一步」。

### Step 2 · 连接 GitHub

3 张选项卡：

```
(  ) 从 gh CLI 自动导入（推荐）
(  ) 手动粘贴 Personal Access Token
(  ) 还没账户 — 去注册
```

第一条是关键路径：`Command::new("gh").arg("auth").arg("token")` 抓出来 → 写 macOS Keychain → 重启 App 时自动注入 sidecar `env`。

让用户走"配置 token"这件事**变成一个按钮**，而不是去 GitHub 网页生成 PAT、回来粘贴这种 6 步动作。

### Step 3 · 选博客

最难设计的一步。一开始我只放了 2 个选项（"已 clone" 和 "需要 clone"），上线后被新用户教做人——"我完全没有博客怎么办"。

后来加了第 3 张：

```
(  ) 还没博客 — 用模板新建一个
    └─ 新仓库名（默认 <user>.github.io）
    └─ 父目录
    └─ 站点标题 / 作者（可选）
    [ 一键创建（repo + clone + 装依赖）]
```

后端逻辑：

```python
gh_create = ["gh", "repo", "create", req.name,
             "--public",
             "--template", req.template,
             "--clone"]
```

模板仓库是我自己博客的副本（`gh api -X PATCH /repos/.../... -f is_template=true`）。任何人现在都可以基于我的博客 30 秒内立一个新站。

### Step 4 · 验证

复用诊断清单的组件。**这是这条产品的灵魂**：

```
✓  gh CLI 已安装
✓  gh 已登录 — 账户：alice
✓  GitHub Token 已注入 sidecar
✓  博客仓库存在 — /Users/alice/alice.github.io
✓  是 git 仓库
✓  远端 origin 已配置
✓  检测到 _config.yml
✓  GitHub Actions 部署 workflow
⚠  hexo 依赖已安装 — 还没跑 `npm install` [怎么装？]
·  Notion 集成（可选）
```

每项除了状态，还可能带一个 `fix` 字段——前端按 fix id 决定渲染什么按钮：

```typescript
if (fix === "import-gh-token") return wrap("从 gh 自动导入", importFromGh);
if (fix === "choose-blog-path") return wrap("选择目录", pickAndSave);
if (fix === "install-gh")       return wrap("如何安装 gh", openDocs);
```

诊断清单**永久存在**——sidebar 多了一个「诊断」入口，配置坏了随时回来看。

## 几个实现上的小心思

### 用 kv 表存"是否完成引导"

不想为单个布尔值再加张表，直接在 `state/published.sqlite` 里加一张通用的 kv：

```sql
CREATE TABLE IF NOT EXISTS kv (
    key        TEXT PRIMARY KEY,
    value      TEXT NOT NULL,
    updated_at TEXT NOT NULL
);
```

```python
@router.get("/onboarding/state")
def get_state() -> dict:
    return {"onboarded": store.kv_get("onboarded") == "true"}
```

App 启动时 `GET /onboarding/state`，返回 `false` 就渲染 `<OnboardingWizard />` 把整个窗口盖住。

### 全屏 overlay 而不是 modal

第一次写的时候做成了居中 modal，调起来发现新用户**真会去点旁边那一圈**。改成全屏 overlay 强制聚焦，背景直接是 App 背景色——不让你有机会逃。

CSS 干净到这种程度：

```css
.wizard-overlay {
  position: fixed; inset: 0;
  background: var(--bg);
  z-index: 100;
  display: flex;
  align-items: center;
  justify-content: center;
  animation: overlayIn var(--dur) var(--ease);
}
```

### 进度条只标"已完成"和"当前"

很常见的进度条做法是 1/4 / 2/4 / 3/4 数字，我觉得密度太高。改成 4 条 4px 圆角条，已完成是品牌色、当前是品牌色、未来是浅灰。一眼能看完。

```jsx
<div className="wizard-progress">
  {Array.from({ length: 4 }).map((_, i) => (
    <div className={`dot ${i < step ? "done" : i === step ? "active" : ""}`} />
  ))}
</div>
```

### 选项卡片不是 radio 是按钮

按钮整张可点击命中，UI 现代且大区域好按。`.ring` 那个圈 + `::after` 实现"选中点"，比 native radio 更可控：

```css
.opt-card.selected .ring::after {
  content: "";
  position: absolute;
  inset: 3px;
  background: var(--brand);
  border-radius: 50%;
}
```

## 跑通之后立刻露馅：4 个新坑

写完上面这套，本以为终于"对新用户友好"。

让 wizard 重置 `onboarded` 标志再跑一遍，自己模拟新用户视角，**立刻看到 4 个仍然会卡的地方**：

### 坑 1 · 模板创建博客后没自动装依赖

`gh repo create --template` 把仓库 clone 下来了，但 `node_modules/` 是空的。`hexo s` 启不动，诊断会显示一条黄色 ⚠。

解法是把"装依赖"也封进流程：

```python
@router.post("/blog/install-deps")
def install_deps(req):
    out = subprocess.run(
        ["npm", "install"], cwd=root,
        capture_output=True, text=True, timeout=360,
    )
    if out.returncode != 0:
        tail = "\n".join((out.stderr or out.stdout).splitlines()[-30:])
        raise HTTPException(400, f"npm install 失败：{tail}")
    return {"ok": True}
```

前端按钮文案变成**分阶段**：

```
"创建 GitHub 仓库…"    ← 5-10s
"装依赖中…（1-3 分钟）" ← 60-180s
"✓ 已完成"
```

新用户拉条进度看着等就行，不用打开终端。

### 坑 2 · wizard 完成后用户不知道下一步干啥

Step 4 「完成」按钮按下去直接关闭——结果用户看到空 Articles 列表 + 一个紫色按钮，茫然。

加一屏「就绪页」：

```
✨ 准备好开始了

[ ✍️ 写第一篇 ]   [ 🏠 看看主页 ]   [ 🎨 改改主题 ]
```

点 「写第一篇」 → 跳到 Articles Tab + **自动弹新建草稿框**。从"配置完成"到"键盘可以开始敲字"压缩到一次点击。

实现上用一个 `pendingAction` signal 跨组件传：

```tsx
<ArticlesTab pendingAction={pendingAction} onActionConsumed={...} />

useEffect(() => {
  if (pendingAction === "new-draft") {
    setShowNew(true);
    onActionConsumed?.();
  }
}, [pendingAction]);
```

### 坑 3 · gh 不存在时的体验

`gh` 不在 PATH 上时，Step 2 那张「从 gh CLI 导入」卡片**点了会失败**，错误信息又是 stderr 那种工程师视角的。

加一个前置探测，没装时**直接把卡片禁用**+ 弹一个黄色提示：

```
⚠ 未检测到 gh CLI

在终端里跑下面这条命令装上（一次性，全程 30 秒）：

[ brew install gh ]   [复制]   [重新检测]

还没装 Homebrew？ brew.sh
```

复制按钮调 `navigator.clipboard.writeText`，重新检测调 `/diagnostics` 看 `gh.installed` 是否变绿。让"装 gh"这件事变成 3 个具体动作而不是一句"gh not found"。

### 坑 4 · 编辑器命令默认是 `code`，但很多人没装 shell 命令

我在 `config.yaml` 里把 `editor_command` 默认值放成 `code`，背后假设是 VS Code 用户都装过 `Shell Command: Install 'code' command in PATH`。

实际上很多人没装过——结果"在 VS Code 中打开"会 fallback 到 `open <file>`，文件被 Typora（或别的默认应用）打开，看上去像 App 在乱搞。

加了个 startup hook：

```rust
fn auto_detect_editor_at_startup(project_root: &Path) {
    // 如果用户已经手动设过非 `code` 的值，尊重它
    if let Some(cur) = current { /* 跳过 */ }

    // 否则按优先级探一遍
    const CANDIDATES: &[&str] = &["code", "cursor", "subl", "mate", "atom", "idea"];
    for c in CANDIDATES {
        if which::which(c).is_ok() {
            // 写入 config.yaml
        }
    }
}
```

这个 hook 在 `tauri::Builder::setup` 里跑，**比 sidecar 启动更早**——保证 `/load_config` 第一次返回时就已经是检测好的值。

**完全静默**。不打扰用户。这种系统判断"哪个编辑器是合理默认"的事，最好的 UX 就是用户根本没意识到它发生过。

## 反思：从"我用着顺"到"别人能跑通"

这一段重构让我对自建工具的"完成度"重新校准了一下：

| 我以为完成度是 | 实际上完成度是 |
|---|---|
| 功能跑通了 | 新用户能在 10 分钟内独立用起来 |
| CLI 有 help | UI 有 zero-config 默认值 |
| 错误能抛 | 错误能被普通用户读懂、知道下一步做啥 |
| 我能配置 | 配置错了能自动检测出来并指条路 |

写工具的时候很容易陷入"功能驱动"——做了 A，做了 B，做了 C，列表越长越满足。但**功能列表不等于产品**。同样的功能集，加一层引导和一个诊断页，对新用户来说是天壤之别。

下一步是**打包成 `.app`**——前面所有 UX 努力，对装不上 Rust/Python/Node 的真人来说还是看不到。这一关过了，才算真正把工具递到别人手里。

## 后续清单

- [ ] PyInstaller 打 Python sidecar 单二进制
- [ ] tauri.conf.json 配 `externalBin` 把 sidecar bundle 进 `.app`
- [ ] Apple Developer ID 签名 + DMG 包装
- [ ] 发布失败错误规则匹配翻译
- [ ] 样式 Tab 保存前自动 `git stash` 兜底

源码：<https://github.com/maimengshangren/publisher>。
