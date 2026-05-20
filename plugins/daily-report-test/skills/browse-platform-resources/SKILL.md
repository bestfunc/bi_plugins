---
name: browse-platform-resources
description: 探索平台上有哪些可用数据源 —— 数据库表 / MCP 工具 / AI 模型。写插件前先用这个看清楚。
allowed-tools: list_credentials, list_db_tables, list_mcp_tools, list_ai_models, list_templates, get_template
---

# 🔍 浏览平台资源

写插件前一定要知道：**有哪些数据能拿到、它的字段叫什么**。这套工具就是给你看的。

## 1. 列凭据

```
> 平台有哪些凭据？
[AI → list_credentials → 返回 id / name / type / environment / verified_at]
```

按 `type` 分类：

| type | 含义 | 用法 |
|---|---|---|
| `database` | PG/MySQL DSN | `sdk.db[alias].dsn` → 用 psycopg2/sqlalchemy 自连 |
| `mcp_apikey` | MCP server + X-API-Key | `sdk.mcp[alias].call(tool, args)` |
| `mcp_oauth2` | MCP server + OAuth2 PKCE | 同上，token 平台自动 refresh |
| `openai_compat` | OpenAI 兼容 base_url + api_key | `sdk.ai[alias].base_url/.api_key/.default_model` |

只用 `verified_at` 非空的凭据 —— 没验证的可能配错了。

## 2. 数据库表 / 列

```
> 凭据 #1（数据库）有哪些表？
[AI → list_db_tables(credential_id=1) → 返回 [{schema, name, columns:[{name, data_type, nullable}]}]]
```

写 SQL 前先看一遍，免得字段名拼错。

## 3. MCP 工具

```
> 凭据 #7（mcp_oauth2 / smart-tpm）暴露哪些工具？
[AI → list_mcp_tools(credential_id=7) → 返回 [{name, description, schema}]]
```

每个 tool 的 `schema` 是 JSON Schema —— 看 `required` 字段免得漏参。

> ⚠️ MCP tool 返回 shape 千差万别。先用 `sdk.log.info("got", resp=resp)` 打一次再写后续逻辑。

## 4. AI 模型

```
> 凭据 #2（openai_compat）有哪些模型？
[AI → list_ai_models(credential_id=2) → 返回 [{id}]]
```

## 5. 内置报告模板

```
> 看下平台内置模板
[AI → list_templates → 返回 [{name, title, description, datasets:[{name, required, description, shape}]}]]

> 看 default 模板的 HTML 源
[AI → get_template(name="default") → 返回 spec + html]
```

模板的 `datasets` 字段告诉你"这个模板想看到什么形状的数据集"。比如 `default` 想看 `range_*` 前缀的 dataset，你 `emit_dataset("range_7d", {...})` 就会自动出现切换按钮。

## 别犯傻

- **不要**用 list_credentials 后假设凭据能用 —— 看 `verified_at` 是否近期
- **不要**列出 20 行表后假设字段类型 —— 用 list_db_tables 看 `data_type`
- **不要**调 MCP tool 不先看 schema —— `required` 字段不传就报 KeyError
