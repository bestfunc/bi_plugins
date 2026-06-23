---
name: daily-report-business-concepts
description: Smart Quality 的核心业务概念 —— 三类 App / 版本 / 触发器 / 凭据 / 活报告 / 静态快照 / KV / Daily / v3 节点 / 常驻服务 / API Key。改任何 App 前先理解这些。
---

# 📚 Smart Quality 业务概念

> 平台 UI 已改名 **Smart Quality 3.0**（代号原"日报平台"）。配置文件名、表名、slug、MCP 工具名等机器 ID 不变。

## App 的三种类型（v3 起）

平台从"只有 endpoint 一种 App"进化成三类，决定 manifest 文件、调用方式、产出形态：

| 类型 | manifest | 调用 | 产出 |
|---|---|---|---|
| **endpoint（日报应用）** | `plugin.yaml` | `POST /api/dr/<slug>/<endpoint>` | 活报告 `/a/<slug>` + 邮件 + 快照 |
| **dag（系统应用）** | `tinia-repo.yaml` + `nodes/` | `POST /api/dag/<slug>/<dag_key>`（`main`=全 pipeline / 其他=单节点） | 节点图编排 + UI 挂载页 |
| **service（常驻服务）** | `tinia-repo.yaml` + `service:` 块 | always-on，**非请求驱动** | 持续出站 WS，浏览器走 `/rtstream/<slug>` |

> "日报应用 / 日报"是 endpoint 类型这一**功能类目**的名字，保留；产品名是 Smart Quality。

## 核心实体（endpoint 类型）

```
App (plugin.yaml) ── 1:N ── AppVersion ── 1:N ── AppFile
   │
   └── 1:N ── Trigger ── 每次触发 → 执行 endpoint → 返回 JSON / 烤快照
```

### App

逻辑上的一个"报告 / 系统应用 / 服务"。有 `slug`（URL）/ `name` / `current_version_id` / `published_version_id`。endpoint 类型配置文件叫 `plugin.yaml`，dag / service 类型叫 `tinia-repo.yaml`（文件名是平台 derive App type 的依据）。

### AppVersion + AppFile

**版本不可变**。每次保存 = 新版本号 + 新文件树。CLAUDE.md §4.3 红线。
- `build_status`: pending → building → ready / failed
- build = 在版本的 lib_dir 跑 `pip install -r requirements.txt --target=...`
- **save ≠ publish**：build 成功 promote `current_version_id`，但 `published_version_id` 要显式 `publish_app` 才切（除非 `auto_publish: true`）

### Trigger

- `kind: cron` 走调度（robfig/cron，时区 Asia/Shanghai，**不补跑**）
- cron trigger 必须带 `endpoint_name`，否则 scheduler 跳过
- `credential_bindings` 是显式绑定 alias → credential_id（覆盖自动匹配）

### Endpoint

在 `plugin.yaml` 的 `endpoints:` 段声明的函数入口。`try_run_endpoint` 必须指定 `endpoint_name`。每个 endpoint 对应 main.py 里一个顶层 Python 函数，**直接 return JSON**。

## 三种"数据持久化"

### 1. emit_report → doc

结构化文档（markdown / kv / table blocks）。给 default 模板的主体。⚠️ `emit_report` **只传 list 永远不传 dict**（dict 会让活报告白屏）。

### 2. emit_dataset → datasets

命名数据集，前端 `window.report.datasets[name]` 可读。**全部数据一次性 ship 到前端**，模板用 JS 切换视图无需重跑。name 必须 `^[A-Za-z_][A-Za-z0-9_]{0,63}$`。

### 3. sdk.kv / sdk.daily

跨执行持久化：
- `sdk.kv.get/set/delete(key)` —— 64KB/key，按 App 隔离
- `sdk.daily.write/read(date)` —— 按天归档 JSON，5MB/day 上限

是给 App **自己**用的状态（last_total / cache / 中间结果）。不影响报告渲染。

## 活报告 vs 静态快照

| 维度 | 活报告 `/a/<slug>` | 静态快照 `/a/<slug>/snap/<id>` |
|---|---|---|
| 触发 | 每次访问可重跑（`POST /api/dr/<slug>/<endpoint>`） | 不再跑，读烤好的 static_html |
| 数据时效 | 实时 | 固定到那次执行的时刻 |
| 凭据 | 自动匹配 | 不需要（数据已烤） |
| 可分享 | 可（受限于凭据可用性） | 100% 自包含 HTML |

> ⚠️ 活报告页是 `/a/<slug>`（不是 `/r/<slug>`）；`POST /api/dr/<slug>/<endpoint>` 是 endpoint dispatch。

## v3 概念速查（dag / service 类型）

| 概念 | 说明 |
|---|---|
| **节点（node）** | dag App 里一个 Python 函数 + node.yaml；多节点可在 DAG 编辑器拖拽编排 |
| **DAG 调用** | `POST /api/dag/<slug>/<dag_key>`，body **直接是 params**（不 wrap `{params:...}`） |
| **UI 挂载（ui_mount）** | dag App 自带前端页。`kind: tsx`（单 .tsx 运行时编译，挂主站 `/plugin/<slug>/<mount>`）或 `kind: dist`（整 SPA 静态服务 `/_ui/<slug>/<mount>/`） |
| **常驻服务（service）** | `service:` 块声明 always-on 进程；平台 supervisor 托管 + 崩溃指数退避自愈，**绝不 auto-unpublish**；浏览器走 `/rtstream/<slug>` WS |
| **工位维度 DSN（桥1）** | dag App 可声明 workstation datasource，per-request 注入 `_datasource`（不走 env，防并发串库） |
| **插件 migration（桥2）** | dag App 可在平台 PG 建表，`tinia-repo.yaml` 声明 `table_prefix` + `migrations:` |
| **对外接口 API Key** | `require_api_key` 按 App opt-in（默认 false）；开启后 `/api/dr`+`/api/dag` 需 `Authorization: Bearer dr_...`，scope 细到单接口 |

## SDK 入口（endpoint 类型）

`from report_sdk import sdk` —— 单例。子进程启动时由平台注入 env：
- `DB_<ALIAS>_DSN` —— 数据库 DSN
- `MCP_<ALIAS>_URL` + `MCP_<ALIAS>_KEY` 或 `MCP_<ALIAS>_ACCESS_TOKEN`
- `AI_<ALIAS>_BASE_URL` + `_KEY` + `_MODEL`

## 模板系统

三档 fallback：
1. plugin.yaml `template: default` → 用内置（推荐，AI 不必写 HTML）
2. 否则用 App ship 的 `view/template.html`（iframe srcdoc 沙箱）
3. 否则用平台默认编辑器主题

模板 = JS 读 `window.report.{doc,datasets,...}` 自行渲染 DOM。

## 红线（不能违反）

- 凭据明文不落日志 / 不入快照（只存 id+name）
- 版本永不可变 —— 一旦保存版本行，不要 UPDATE
- DB migrations 只用 golang-migrate，禁 GORM AutoMigrate
- cron 不补跑 —— 错过的触发直接丢，依赖 `/a/<slug>` 实时性兜底
- `emit_report` 只传 list，永不传 dict
