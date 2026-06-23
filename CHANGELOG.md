# Changelog

## 3.0.0 — 2026-06-23

对应 Smart Quality 平台 master `v0.3.x` + `v3.0` 分支（三类 App / DAG / 常驻服务 / 对外接口 API Key）。

- **品牌**：产品名"日报平台" → **Smart Quality 3.0**（散文 / 描述 / skill 正文）。机器 ID 全部不变（marketplace `bi_plugins`、variant ID `daily-report-{local,test,prod}`、`mcpServers` key `daily-report`、skill 目录名、入口 URL、GitHub repo）。"日报应用"作为 endpoint 类型的功能类目名保留。
- **修过期事实**：活报告路径 `/r/<slug>` → **`/a/<slug>`**（静态快照 `/a/<slug>/snap/<id>`）；MCP 工具数 **13 → 23**（按平台 `mcpserver/tools.go` 实数核对，新增 `update_credential` / `export_app` / `import_app` / 4 个 email 工具）；移除杜撰的 `delete_app` MCP 工具（删 App 走 HTTP `DELETE /api/apps/:id`）。
- **v3 能力覆盖**（基于平台真实文档写）：
  - quickstart + business-concepts：三类 App（endpoint / dag / service）总览、真实 23 工具清单、v3 概念速查（节点 / DAG 调用 / UI mount tsx vs dist / 常驻服务 / 工位 DSN / 插件 migration / API Key）。
  - create-app-from-scratch：末尾新增 dag（`tinia-repo.yaml` + nodes + stdin/stdout JSON-RPC）/ service（`service:` 块 + supervisor 托管 + `/rtstream`）创建路线 + API Key opt-in。
  - debug-failed-run：新增 v3 + API Key 错误对照（DAG 404 / body 不 wrap / PATH·TZ / dist 404 / tsx import 限制 / 401·403·429 / circuit_open）。
  - 反复强调 **DAG / 常驻服务 / API Key 没有 MCP 工具**，走 HTTP——零编造。

## 1.2.2 — 2026-05-27

- **daily-report-test** 入口从 `https://daily-report-test.bestfunc.com/api/mcp` 切到内网 `http://192.168.2.121:8332/api/mcp`（test 环境换部署位置）。local / prod / marketplace 跟随锁步同版。

## 1.2.1 — 2026-05-27

- **daily-report-prod** 入口从内网 `http://192.168.2.175:8332` 切到正式域名 `https://bi.bestfunc.com/api/mcp`（platform 已迁域名）。local / test / marketplace 跟随锁步同版。
- ⚠️ `bi.bestfunc.com` 当前为自签 / 不受信任根证书，SDK 严格 TLS 会拒连；授权要通需服务端换公网受信任证书，或客户端设 `NODE_EXTRA_CA_CERTS` 信任该 CA。

## 1.2.0 — 2026-05-25

对应日报平台 `v0.1.5`（175 已升级到 v1.x app 模型,本次跟随发版）。

- **create-app-from-scratch** 末尾追加"v1.x app 模型补遗"：MCP 工具入参形状（files=array、publish_app 用 slug+version_id、try_run_endpoint 字段是 endpoint_name 非 endpoint）、endpoint 函数写法、cron 真触发（endpoint_name 必填）、cron-friendly endpoint 自取数据模式、Snapshot 模板沙箱硬约束
- **fix-or-extend-app** 末尾追加：save ≠ publish 两步流程、endpoint 改动 / 删除注意事项
- **debug-failed-run** 末尾追加 v1.x 错误对照表（11 条新错码）
- daily-report-prod 跟其它 variant 同步更新（175 已升 v1.x,prod 与 local/test 内容现在一致）

## 1.0.0 — 2026-05-20

- 初版发布，3 个 variant（local / test / prod）
- 6 个 Skill：quickstart / create / fix / debug / browse / business-concepts
- 对接 daily-report 平台 v0.0.13-mvp9 的 MCP 端点（13 工具 + OAuth 2.1 + PKCE + DCR）
