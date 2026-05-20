---
name: daily-report-business-concepts
description: 日报平台的核心业务概念 —— 插件 / 版本 / 运行 / 触发器 / 凭据 / 活报告 / 静态快照 / KV / Daily 归档。改任何 plugin 前先理解这些。
---

# 📚 日报平台业务概念

## 核心实体

```
Plugin ── 1:N ── PluginVersion ── 1:N ── PluginFile
   │
   └── 1:N ── Trigger ── 1:N ── Run ── 1:1 ── RunArtifact
                                            ├── doc        (emit_report)
                                            ├── datasets   (emit_dataset)
                                            ├── stdout/stderr
                                            └── static_html (chromedp 烤的快照)
```

### Plugin

逻辑上的一个"报告"。有 `slug`（URL）/ `name` / `current_version_id`。

### PluginVersion + PluginFile

**版本不可变**。每次保存 = 新版本号 + 新文件树。CLAUDE.md §4.3 红线。
- `build_status`: pending → building → ready / build_failed
- build = 在版本的 `lib_dir` 跑 `pip install -r config/requirements.txt --target=...`
- build 成功 → 自动 promote 为 plugin.current_version_id

### Trigger

- `kind: cron` 走调度（robfig/cron）
- `kind: manual_only` 只能从 UI / API 手动触发
- `credential_bindings` 是显式绑定 alias → credential_id（覆盖自动匹配）

### Run + RunArtifact

每次执行的实例。`triggered_by`：`cron` / `manual` / `manual_dryrun` / `view`。
- `view` 是 /r/<slug> 触发的 —— **不写 runs 表**（活报告不留历史）
- 其他都写 + 生成 static_html 快照（chromedp 渲染当前模板 + 数据）

## 三种"数据持久化"

### 1. emit_report → run_artifacts.doc

结构化文档（markdown / kv / table blocks）。给 default 模板的主体。每次 run 重新生成。

### 2. emit_dataset → run_artifacts.datasets

命名数据集，前端 `window.report.datasets[name]` 可读。**全部数据一次性 ship 到前端**，模板用 JS 切换视图无需重跑。**静态快照天然完整**。

### 3. sdk.kv / sdk.daily

跨执行持久化：
- `sdk.kv.get/set/delete(key)` —— 64KB/key，按 plugin 隔离
- `sdk.daily.write/read(date)` —— 按天归档 JSON，5MB/day 上限

是给插件**自己**用的状态（last_total / cache / 缓存中间结果）。不影响报告渲染。

## 活报告 vs 静态快照

| 维度 | 活报告 `/r/:slug` | 静态快照 `/r/:slug/runs/:run_id` |
|---|---|---|
| 触发 | 每次访问可重跑 | 不再跑，读 run_artifacts.static_html |
| 数据时效 | 实时 | 固定到那次 run 的时刻 |
| 凭据 | 自动匹配 | 不需要（数据已烤） |
| 可分享 | 可（受限于凭据可用性） | 100% 自包含 HTML |

## SDK 入口

`from report_sdk import sdk` —— 单例。子进程启动时由 runner 注入 env：
- `RESULT_PATH` —— atexit 写 doc/datasets 到这里
- `DR_RUN_TOKEN` —— HMAC 签的 token，调 /api/internal/* 用
- `DB_<ALIAS>_DSN` —— 数据库 DSN
- `MCP_<ALIAS>_URL` + `MCP_<ALIAS>_KEY` 或 `MCP_<ALIAS>_ACCESS_TOKEN`
- `AI_<ALIAS>_BASE_URL` + `_KEY` + `_MODEL`

## 模板系统

三档 fallback：
1. plugin.yaml `template: default` → 用内置（推荐，AI 不必写 HTML）
2. 否则用插件 ship 的 `view/template.html`（iframe srcdoc 沙箱）
3. 否则用平台默认编辑器主题

模板 = JS 读 `window.report.{doc,datasets,...}` 自行渲染 DOM。

## 红线（不能违反）

- 凭据明文不落日志 / 不入 credentials_snapshot（只存 id+name）
- 版本永不可变 —— 一旦保存 plugin_versions 行，不要 UPDATE
- runner 是**唯一**写 runs / run_artifacts 的地方
- DB migrations 只用 golang-migrate，禁 GORM AutoMigrate
- cron 不补跑 —— 错过的触发直接丢，依赖 /r/<slug> 实时性兜底
