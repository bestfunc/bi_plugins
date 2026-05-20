---
name: quickstart-daily-report-mcp
description: 日报平台 MCP 接入快速上手 —— 介绍 4 个工具模块、scope 模型、典型对话示例。第一次用 @daily-report 时读这个。
allowed-tools: list_plugins, list_credentials, list_templates, list_runs
---

# 🌅 快速上手 日报平台 MCP

日报平台是一个让工程师**用 Python 写脚本 → 一键得到分享链接**的工具。MCP 暴露了 13 个工具，按 4 个模块组织：

## 模块速查

| 模块 | 工具 | 用途 |
|---|---|---|
| **plugins** | `list_plugins` `get_plugin` `create_plugin` `update_plugin` | 看 / 改 / 建 插件 |
| **credentials** | `list_credentials` `list_db_tables` `list_mcp_tools` `list_ai_models` | 查可用数据源 |
| **templates** | `list_templates` `get_template` | 看内置报告模板 |
| **runs** | `list_runs` `get_run` `try_run` | 看历史 / 触发运行 |

## Scope 模型

| Scope | 包含工具 |
|---|---|
| `mcp:read` | 所有 list / get （只查询，不副作用） |
| `mcp:write` | `create_plugin` / `update_plugin` （写入插件代码） |
| `mcp:run` | `try_run` （触发运行，可能调外部数据库 / MCP / AI） |

授权时按需勾，不要一上来就 `mcp:write mcp:run` —— 浏览器看报告这种场景 `mcp:read` 足够。

## 典型对话

```
> @daily-report 平台上现在有哪些插件？
[AI → list_plugins → 13 个插件，最近一个叫 mcp-random …]

> 看下 mcp-random 这个插件
[AI → get_plugin(slug="mcp-random") → 返回文件树 + manifest + build_status]

> 它最近跑了几次？哪次失败了？
[AI → list_runs → 列最近 20 次 + 状态]
[AI → get_run(run_id=42) → 看 stderr 找原因]
```

## 它不是什么

- **不是**自动写 SQL / 数据分析的工具 —— 它给你"插件运行时 + 模板渲染"，业务逻辑还是你写
- **不是**多租户系统 —— v1 单用户（admin），所有人看到的资源一样
- **不是**实时数据库 —— 报告每次访问可以重跑，但要看插件自己怎么实现

## 想做完整工作流？

- 想从零写一个新插件 → 看 **create-plugin-from-scratch**
- 试跑失败排查 → 看 **debug-failed-run**
- 改现有插件加字段 / 修 bug → 看 **fix-or-extend-plugin**
- 看平台有哪些可用资源（数据库 / MCP / AI） → 看 **browse-platform-resources**
- 业务概念 / 数据模型 → 看 **daily-report-business-concepts**
