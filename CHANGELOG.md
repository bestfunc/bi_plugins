# Changelog

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
