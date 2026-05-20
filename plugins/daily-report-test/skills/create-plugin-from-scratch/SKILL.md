---
name: create-plugin-from-scratch
description: 从零创建一个日报平台插件。包括 main.py 入口约定、plugin.yaml manifest 字段、required_mcps / required_dbs 声明、内置模板选择、build + try_run 校验。
allowed-tools: list_plugins, list_credentials, list_templates, get_template, create_plugin, get_plugin, try_run
---

# 🛠 从零创建一个日报插件

## 核心理念

日报插件是一份 **Python 文件 + plugin.yaml**，放进平台 → 自动 pip install → 给一个 `/r/<slug>` 链接 → 每次打开重新跑出最新数据。

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
> 平台上有什么插件？我先避开 slug 冲突
[AI → list_plugins]

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

### 第 3 步：写最小可用代码

```python
# main.py
from report_sdk import sdk
import datetime as dt

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
```

### 第 5 步：调 create_plugin

```
> 用 create_plugin 建一个 slug=my-report, name="我的报告" 的插件，文件树包含 main.py / config/plugin.yaml / config/requirements.txt（依赖空）
[AI → create_plugin(...)
     → 返回 plugin_id + version_id + "build 已排队"]
```

### 第 6 步：等 build → 试跑

```
> get_plugin 看下 build_status
[AI → get_plugin → ready]

> try_run（不要传 credential_ids，平台自动绑）
[AI → try_run(slug="my-report")
     → status=success，doc + datasets 直接返回]
```

## 平台约定

- **每次 update_plugin = 新版本**：版本不可变（CLAUDE.md 红线），改一个字也要 PUT 全量
- **凭据自动绑定**：不传 credential_ids 时按 manifest alias → 凭据 type 自动匹配最近 verified 的
- **失败时**：try_run 自动带 stderr_jsonl 尾，从里面找异常 traceback

## 常见坑

| 现象 | 原因 |
|---|---|
| `MCP[xxx] url not set` | 没声明 `required_mcps` 但代码用了 `sdk.mcp[xxx]` |
| build 卡 pending | pip install 跑慢，等 1-2 分钟 |
| build_failed | 看 `build_log_tail` —— 通常是 requirements.txt 包名拼写 |
| try_run 跑了 71ms 就 success 但啥都没渲染 | main() 包了函数没调，或代码在 `if __name__ == '__main__'` 后面 |
