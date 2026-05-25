---
name: create-app-from-scratch
description: 从零创建一个日报平台 App。包括 main.py 入口约定、plugin.yaml manifest 字段、required_mcps / required_dbs 声明、endpoints 声明、内置模板选择、build + try_run_endpoint 校验。
allowed-tools: list_apps, list_credentials, list_templates, get_template, create_app, get_app, try_run_endpoint
---

# 🛠 从零创建一个日报 App

## 核心理念

日报 App 是一份 **Python 文件 + plugin.yaml**，放进平台 → 自动 pip install → 给一个 `/r/<slug>` 链接 → 每次打开重新跑出最新数据。

## 模块级执行 ≠ 函数定义

最容易踩的坑：runner 是 `python main.py`，不是 import 你的 main()。**代码必须写在模块顶层**，不要包在 `def main():` 里。

```python
# ❌ 错的
from report_sdk import sdk
def main():
    sdk.emit_report([...])

# ✅ 对的
from report_sdk import sdk
sdk.emit_report([...])
```

## 流程

### 第 1 步：先看现状

```
> 平台上有什么 App？我先避开 slug 冲突
[AI → list_apps]

> 有哪些数据源可用？
[AI → list_credentials → 看到 type=database / mcp_apikey / mcp_oauth2 / openai_compat 的凭据]

> 有哪些内置模板？
[AI → list_templates → 看到 "default" 等]
```

### 第 2 步：决定关键字段

| 字段 | 说明 |
|---|---|
| `slug` | URL 片段（`/r/<slug>`），全局唯一，建议小写连字符 |
| `name` | 显示名 |
| `template` | 用 `default` 走内置（推荐），或省略后自带 `view/template.html` |
| `required_dbs` / `required_mcps` / `required_ai` | 声明需要的凭据 alias（平台自动按类型匹配） |
| `endpoints` | 声明至少一个 endpoint（函数入口），见下文 |

### 第 3 步：写最小可用代码

```python
# main.py
from report_sdk import sdk
import datetime as dt

# endpoint 函数，名字对应 plugin.yaml endpoints[].function
def ping(params):
    return {"pong": True, "ts": dt.datetime.utcnow().isoformat()}

# 1. 拿数据 —— 按需启用其中一种
# DB:
# dsn = sdk.db["db"].dsn
# import psycopg2; with psycopg2.connect(dsn) as conn: ...

# MCP:
# result = sdk.mcp["smart"].call("some_tool", {"key": "value"})

# AI:
# proxy = sdk.ai["gpt"]
# from openai import OpenAI
# client = OpenAI(api_key=proxy.api_key, base_url=proxy.base_url)

# 2. 写报告
sdk.emit_report([
    {"type": "markdown", "text": f"# 我的报告 {dt.date.today()}"},
    {"type": "kv", "data": {"指标 A": 42, "指标 B": "OK"}},
    {"type": "table",
     "columns": ["name", "value"],
     "rows": [{"name": "x", "value": 1}, {"name": "y", "value": 2}]},
])

# 3. 想让前端切视图？把数据集发出去
sdk.emit_dataset("history_7d", {"rows": [...]})
sdk.emit_dataset("history_30d", {"rows": [...]})
```

### 第 4 步：写 plugin.yaml

```yaml
entry: main.py
cache_ttl: 60         # /r/<slug> 缓存秒数
timeout_seconds: 60   # 子进程硬超时
# required_dbs:       # 需要数据库时打开
#   - alias: db
#     description: 主库
# required_mcps:      # 需要 MCP 时打开
#   - alias: smart
#     description: smart-tpm
template: default     # 用内置模板
endpoints:
  - name: ping
    function: ping    # 对应 main.py 里的函数名
```

`endpoints` 是必填 —— 至少声明一个，否则 `try_run_endpoint` 无法调用。

### 第 5 步：调 create_app

```
> 用 create_app 建一个 slug=my-report, name="我的报告" 的 App，文件树包含 main.py / config/plugin.yaml / config/requirements.txt（依赖空）
[AI → create_app(...)
     → 返回 app_id + version_id + "build 已排队"]
```

### 第 6 步：等 build → 试跑

```
> get_app 看下 build_status
[AI → get_app(slug="my-report") → build_status=ready]

> try_run_endpoint 调 ping endpoint
[AI → try_run_endpoint(slug="my-report", endpoint_name="ping", params={})
     → 返回 endpoint 的 JSON，如 {"pong": true, "ts": "..."}]
```

**try_run_endpoint 签名**：
```
try_run_endpoint(slug, endpoint_name, params)
  - slug: App slug，如 "my-report"
  - endpoint_name: 对应 plugin.yaml endpoints[].name
  - params: 传给 endpoint 函数的 JSON 对象
  返回：endpoint 函数 return 的 JSON
```

## 平台约定

- **每次 update_app = 新版本**：版本不可变（CLAUDE.md 红线），改一个字也要 PUT 全量
- **凭据自动绑定**：不传 credential_ids 时按 manifest alias → 凭据 type 自动匹配最近 verified 的
- **失败时**：stderr_jsonl 里找异常 traceback

## 常见坑

| 现象 | 原因 |
|---|---|
| `MCP[xxx] url not set` | 没声明 `required_mcps` 但代码用了 `sdk.mcp[xxx]` |
| build 卡 pending | pip install 跑慢，等 1-2 分钟 |
| build_failed | 看 `build_log_tail` —— 通常是 requirements.txt 包名拼写 |
| try_run_endpoint 报 endpoint not found | plugin.yaml 里 `endpoints[].name` 拼错，或忘了加 `endpoints` 段 |
| App 跑了 71ms 就 success 但啥都没渲染 | main() 包了函数没调，或代码在 `if __name__ == '__main__'` 后面 |

---

## v1.x app 模型补遗（dev 分支必看）

> dev 分支已切到 v1.x app 小程序模型。175 生产仍是 v0.0.20 老 plugin 模型 —— 以下规则**仅适用于 dev / test 本地连**，连生产时这些细节不一定适用。

### MCP 工具入参形状（最容易写错）

| 工具 | arguments 形状 | 易错点 |
|---|---|---|
| `create_app` / `update_app` | `{slug或id, files: [{path, content}, ...]}` | `files` 是 **array of {path, content}**，不是 map！传成 dict 后端报 `cannot unmarshal object into ... []FileInput` |
| `publish_app` | `{slug, version_id}` | **不接受 id 字段**；version_id 从 `get_app.current_version_id` 拿。错传 id 报 `slug 和 version_id 都必填` |
| `try_run_endpoint` | `{slug, endpoint_name, params?, credential_ids?}` | 字段名是 **`endpoint_name`** 不是 `endpoint`（写错报 `endpoint_name 必填`）;名字必须在 manifest `endpoints[].name` 里声明 |

### endpoint 函数（v1.x 写法）

每个 endpoint 是 main.py 顶层函数，**直接 return JSON**，不再用 `sdk.emit_report`：

```python
def summary(params):
    return {"stations": [...], "totals": {...}}
```

manifest 里登记：
```yaml
endpoints:
  - name: summary       # URL / cron 用的
    function: summary   # main.py 里的函数名
```

### cron 真触发（dev ≥ commit 04ade70 才生效）

历史 bug：`trigger.CreateInput` 之前漏 `endpoint_name` 字段 → HTTP/UI 建的 cron 都没 endpoint_name → scheduler 跳过 "no endpoint_name" → 不会真触发。dev 已修。

建 cron：
```bash
POST /api/triggers
{
  "plugin_id": 14,
  "name": "daily-8am",
  "kind": "cron",
  "cron_expr": "0 8 * * *",
  "endpoint_name": "take_snapshot",   ← 必填
  "params": {},
  "enabled": true
}
```

**cron-friendly endpoint 设计**：cron 触发拿到的 params 就是 trigger 里 `params` 字段（默认 `{}`），没上下文。需要数据的 endpoint 必须能**自取**：

```python
def take_snapshot(params):
    import report_sdk.snapshot as snap
    data = params.get("data")
    if not data:
        data = summary({"limit": 200})  # cron 调用走这条，自跑 summary
    return {"snapshot_id": snap.create(template="dashboard", data=data, title="...")}
```

反例：endpoint 强制要 params 里有完整 data → cron 永远拿不到，trigger 每次都 error。

### Snapshot 模板硬约束

`sdk.snapshot.create` 走 chromedp sidecar 烤静态 HTML，**沙箱 ≤30s 渲染窗口、禁止 fetch / 跨域 / 动态 import**。

- 模板能拿到的数据：`window.__SNAPSHOT__ = { data, title, tags }`
- **想要交互（点击展开 modal）→ 把详情数据在 endpoint payload 里就 ship 过去**，模板不能再 fetch
- `view/template.html`（动态，用 `dr.call()`）跟 `snapshots/<name>.html`（静态，用 `__SNAPSHOT__`）**分开维护**，别复用同一份 HTML
