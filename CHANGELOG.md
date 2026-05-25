# Changelog

## 1.2.0-dev — 2026-05-25（dev 分支,未发布）

对应日报平台 dev `v0.1.3-mvpA4+`。本次合并的是 workstation-ng 实战 + trigger 修复后沉淀的新规则。

- **create-app-from-scratch** 末尾追加"v1.x app 模型补遗"：MCP 工具入参形状（files=array、publish_app 用 slug+version_id）、endpoint 函数写法、cron 真触发（endpoint_name 必填）、cron-friendly endpoint 自取数据模式、Snapshot 模板沙箱硬约束
- **fix-or-extend-app** 末尾追加：save ≠ publish 两步流程、endpoint 改动 / 删除注意事项
- **debug-failed-run** 末尾追加 v1.x 错误对照表（11 条新错码：files 形状、publish_app id 字段、endpoint not declared、MCP/AI URL not set、AI base_url /v1 自适应、AI default_model 不可信、worker auto-unpublish、trigger no endpoint_name、vite proxy 白屏、null origin CORS、Windows GBK 乱码）
- daily-report-prod 跟其它 variant 同步更新；**但 175 生产仍 v0.0.20 老 plugin 模型**,prod variant 这套内容**等 175 升级到 v1.x 之后**再 merge 到 main

dev 分支 v1.1.0 → v1.2.0-dev。等用户 review 后再决定何时 merge 回 main + 发版。

## 1.0.0 — 2026-05-20

- 初版发布，3 个 variant（local / test / prod）
- 6 个 Skill：quickstart / create / fix / debug / browse / business-concepts
- 对接 daily-report 平台 v0.0.13-mvp9 的 MCP 端点（13 工具 + OAuth 2.1 + PKCE + DCR）
