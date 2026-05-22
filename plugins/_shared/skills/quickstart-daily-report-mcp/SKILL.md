---
name: quickstart-daily-report-mcp
description: 日报平台 MCP 接入快速上手 —— 介绍 4 个工具模块、scope 模型、典型对话示例。第一次用 @daily-report 时读这个。
allowed-tools: list_apps, list_credentials, list_templates
---

# 🌅 快速上手 日报平台 MCP

日报平台是一个让工程师**用 Python 写脚本 → 一键得到分享链接**的工具。MCP 暴露了若干工具，按 3 个模块组织：

## 模块速查

| 模块 | 工具 | 用途 |
|---|---|---|
| **apps** | `list_apps` `get_app` `create_app` `update_app` `delete_app` | 看 / 改 / 建 / 删 App |
| **credentials** | `list_credentials` `list_db_tables` `list_mcp_tools` `list_ai_models` | 查可用数据源 |
| **templates** | `list_templates` `get_template` | 看内置报告模板 |
| **run** | `try_run_endpoint` | 触发 App endpoint 运行，返回 endpoint JSON |

## Scope 模型

| Scope | 包含工具 |
|---|---|
| `mcp:read` | 所有 list / get （只查询，不副作用） |
| `mcp:write` | `create_app` / `update_app` / `delete_app` （写入 App 代码） |
| `mcp:run` | `try_run_endpoint` （触发运行，可能调外部数据库 / MCP / AI） |

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
[AI → list_apps → 13 个 App，最近一个叫 mcp-random …]

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

- 想从零写一个新 App → 看 **create-app-from-scratch**
- 试跑失败排查 → 看 **debug-failed-run**
- 改现有 App 加字段 / 修 bug → 看 **fix-or-extend-app**
- 看平台有哪些可用资源（数据库 / MCP / AI） → 看 **browse-platform-resources**
- 业务概念 / 数据模型 → 看 **daily-report-business-concepts**
