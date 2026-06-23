# bi_plugins

> 日报平台（[bestfunc/daily-report](https://github.com/bestfunc/daily-report)）的 Claude Code 插件包。
> 通过 MCP（OAuth 2.1 + PKCE + Dynamic Client Registration）让 AI 客户端代写日报插件、试跑、查历史。

## 是什么

日报平台让工程师写 Python 脚本 → 平台托管运行 → 拿到 `/r/<slug>` 分享链接。这个 plugin pack 把平台 13 个 MCP 工具 + 6 个 AI Skill 一键注入到 Claude Code / Cursor / Codex / Qwen CLI 里。

**无需复制 token**：插件第一次连接时 Claude Code 自动开浏览器走 OAuth → 平台登录页（默认 admin/admin）→ 同意 scope → 自动回填 token。

## 三个 variant

| variant | 指向 | 用什么场景 |
|---|---|---|
| `daily-report-local` | `http://localhost:18923` | 自己机器 `go run ./cmd/server` 时 |
| `daily-report-test`  | `http://192.168.2.121:8332` | 团队测试环境（内网） |
| `daily-report-prod`  | `https://bi.bestfunc.com` | 生产 |

## 安装（Claude Code）

```bash
# 添加 marketplace
claude /plugin marketplace add bestfunc/bi_plugins

# 安装某个 variant（按你的环境）
claude /plugin add daily-report-local
```

浏览器会自动弹出登录页。默认开发环境用户名密码都是 `admin`（用 `DR_MCP_ADMIN_USER` / `DR_MCP_ADMIN_PASS` env 改）。

## Scope 模型

| Scope | 用途 |
|---|---|
| `mcp:read` | 只查询（list_*, get_*） |
| `mcp:write` | 创建 / 修改插件 |
| `mcp:run` | 触发运行（有副作用） |

授权时按需勾。每次 token 1h 有效，自动 refresh 30 天。

## 6 个 Skill

| Skill | 用途 |
|---|---|
| `quickstart-daily-report-mcp` | 第一次用 MCP 时的总览 |
| `create-plugin-from-scratch` | 从零写一个报告插件 |
| `fix-or-extend-plugin` | 改现有插件 / 升级模板 |
| `debug-failed-run` | 看 stderr_jsonl 排错 |
| `browse-platform-resources` | 探索数据库表 / MCP 工具 / AI 模型 |
| `daily-report-business-concepts` | 业务概念 / 数据模型速查 |

## 自托管

把仓库 fork 一份，改 `plugins/*/.claude-plugin/plugin.json` 里的 `url` 字段即可。OAuth metadata 走 `.well-known` 自动发现。

## 反馈

- 平台 issue: https://github.com/bestfunc/daily-report/issues
- 本仓库 issue: https://github.com/bestfunc/bi_plugins/issues

## License

Apache-2.0
