---
name: quickstart-daily-report-mcp
description: Smart Quality MCP 接入快速上手 —— 三类 App、真实 MCP 工具清单、scope 模型、典型对话示例。第一次用 @daily-report 时读这个。
allowed-tools: list_apps, list_credentials, list_templates
---

# 🌅 快速上手 Smart Quality MCP

Smart Quality（平台代号原"日报平台"，UI 已改名 Smart Quality 3.0）是一个让工程师**用 Python 写脚本 / 节点 → 平台托管运行 → 一键得到分享链接或 UI 页面**的工具。MCP 暴露了一组工具让 AI 客户端全程不开浏览器就能建 App、试跑、查资源。

## 平台有三类 App（v3 起）

| 类型 | manifest | 调用方式 | 产出 | 入口 |
|---|---|---|---|---|
| **endpoint（日报应用）** | `plugin.yaml` | `POST /api/dr/<slug>/<endpoint>` | 活报告 `/a/<slug>` + 邮件 / 快照 | 应用列表卡片"进入" |
| **dag（系统应用）** | `tinia-repo.yaml` | `POST /api/dag/<slug>/<dag_key>` | 节点图 DAG runner + UI 挂载 | sidebar 自动菜单 |
| **service（常驻服务）** | `tinia-repo.yaml` + `service:` 块 | 非请求驱动（always-on） | 浏览器 WS `/rtstream/<slug>` | 常驻服务管理页 |

> "日报应用"是 endpoint 类型的功能类目名（不是产品名）。本快速上手 + 各 skill 的工具流程主要覆盖 **endpoint 类型**（最成熟、MCP 工具全覆盖）。dag / service 怎么建见 **create-app-from-scratch** skill 末尾的 v3 章节。

## MCP 工具清单（真实 23 个）

| 模块 | 工具 | 用途 |
|---|---|---|
| **apps** | `list_apps` `get_app` `create_app` `update_app` `publish_app` `unpublish_app` | 看 / 建 / 改 / 发布 / 下架 App |
| **run** | `try_run_endpoint` | 触发 App endpoint 运行，返回 endpoint JSON |
| **import/export** | `export_app` `import_app` | `.dr-app.json` bundle 跨环境搬运（export 可选 `version_no` 取历史版本） |
| **credentials** | `list_credentials` `update_credential` `list_db_tables` `list_mcp_tools` `list_ai_models` | 查 / 改凭据、看数据源 schema |
| **templates** | `list_templates` `get_template` | 看内置报告模板 |
| **snapshots** | `list_snapshots` `get_snapshot` | 列 / 看静态快照 |
| **calls** | `list_endpoint_calls` | 某 App endpoint 调用历史（cron 调用一定有 trigger_id） |
| **email** | `list_email_subscribers` `create_email_subscription` `list_email_sends` `test_email_send` | 邮件 publisher 订阅 / 发送记录 / 测试 |

⚠️ **不存在的 MCP 工具**（别调，会找不到）：`delete_app`（删 App 走 HTTP `DELETE /api/apps/:id`）；DAG 跑节点 / 常驻服务管理 / API Key 管理**都没有 MCP 工具**，走 HTTP（见 create-app-from-scratch v3 章节 + EXTERNAL_API_GUIDE）。

## Scope 模型

| Scope | 包含工具 |
|---|---|
| `mcp:read` | 所有 list / get（只查询，不副作用） |
| `mcp:write` | `create_app` / `update_app` / `publish_app` / `unpublish_app` / `update_credential` / `import_app` / `create_email_subscription` 等写入 |
| `mcp:run` | `try_run_endpoint` / `test_email_send`（触发运行，可能调外部数据库 / MCP / AI） |

授权时按需勾，不要一上来就 `mcp:write mcp:run` —— 浏览器看报告这种场景 `mcp:read` 足够。

## try_run_endpoint 签名

```
try_run_endpoint(slug, endpoint_name, params)
  - slug: App slug，如 "hello-app"
  - endpoint_name: plugin.yaml endpoints[].name 里声明的名字
  - params: 传给 endpoint 函数的 JSON 对象
  返回：endpoint 函数 return 的 JSON
```

## 典型对话

```
> @daily-report 平台上现在有哪些 App？
[AI → list_apps → 返回 App 列表 + slug + build/publish 状态]

> 看下 mcp-random 这个 App
[AI → get_app(slug="mcp-random") → 返回文件树 + manifest + build_status]

> 跑一下 mcp-random 的 ping endpoint
[AI → try_run_endpoint(slug="mcp-random", endpoint_name="ping", params={})
     → 返回 endpoint JSON]
```

## 它不是什么

- **不是**自动写 SQL / 数据分析的工具 —— 它给你"App 运行时 + 模板渲染"，业务逻辑还是你写
- **不是**多租户系统 —— v1 单用户（admin），所有人看到的资源一样
- **不是**实时数据库 —— 报告每次访问可以重跑，但要看 App 自己怎么实现

## 想做完整工作流？

- 想从零写一个新 App（含 dag / 常驻服务） → 看 **create-app-from-scratch**
- 试跑失败排查 → 看 **debug-failed-run**
- 改现有 App 加字段 / 修 bug → 看 **fix-or-extend-app**
- 看平台有哪些可用资源（数据库 / MCP / AI） → 看 **browse-platform-resources**
- 业务概念 / 数据模型 → 看 **daily-report-business-concepts**
