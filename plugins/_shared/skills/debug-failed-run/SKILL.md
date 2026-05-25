---
name: debug-failed-run
description: App 运行失败时排错的标准流程 —— 看 stderr_jsonl traceback、定位异常、修代码、重新试跑。
allowed-tools: get_app, update_app, try_run_endpoint
---

# 🔧 调试失败的运行

## 失败信号

| 现象 | 含义 |
|---|---|
| `try_run_endpoint` 返回 `status=failed` | 子进程 exit code != 0 或 SDK 抛了异常 |
| `try_run_endpoint` 返回 `status=timeout` | 超过 `timeout_seconds` 被 kill |
| `try_run_endpoint` 返回 `status=interrupted` | heartbeat 断了被 reaper 标记 |
| `/r/<slug>` 返回 422 | 同上失败，但通过 live 路径暴露 |

## 标准排错流程

### 1. 直接 try_run_endpoint 看 stderr

```
> try_run_endpoint(slug="xxx", endpoint_name="<endpoint>", params={})
[AI → 返回 status=failed + stderr_jsonl，里面是 SDK 写的 JSON Lines 日志 + 异常 traceback]
```

**stderr_jsonl 阅读要点**：

- 每行是一条 JSON：`{"level": "info|warn|error", "msg": "...", "k": "v", ...}`
- 最末尾通常是 Python traceback（明文格式，不是 JSON）
- `sdk.log.info(msg, k=v)` 调用会出现在这里 —— 用它做断点

### 2. 常见错误模式

| traceback 关键字 | 原因 | 修法 |
|---|---|---|
| `ModuleNotFoundError: No module named 'xxx'` | requirements.txt 漏依赖 | update_app 加包名 |
| `MCP[xxx] url not set` | 用了 sdk.mcp[xxx] 但 manifest 没声明 `required_mcps` | 在 plugin.yaml 加 alias |
| `KeyError: 'version_id'` 类参数错误 | MCP 工具入参不对 | 看 list_mcp_tools 的 schema |
| `urlopen error WinError 10060` 或 `Connection refused` | 外部服务不可达 / token 过期 | 检查凭据是否还 verified |
| `KeyError` 在 dict 取值 | API 返回 shape 跟你以为的不一样 | 用 `_pick_*` 风格写 defensive 取值 |

### 3. 修代码 → update_app → 试跑

```
> 改 main.py 第 38 行，把 mcp.call("xxx", {"id": ds_id}) 改成 mcp.call("xxx", {"version_id": vid})，
  其他文件不动，update_app 重存
[AI → get_app 拿当前所有文件 → update_app(files=[全量], comment="fix: ...")]

> 等 build ready 后 try_run_endpoint
[AI → 轮询 get_app 看 build_status]
[AI → try_run_endpoint(slug="xxx", endpoint_name="<endpoint>", params={})]
```

⚠️ `update_app.files` 是**全量**，不传等于删 —— get_app 先拿全部文件再改要传的那个。

## 不要 X 做 Y

| ❌ 不要 | ✅ 应该 |
|---|---|
| 看到 failed 就猜原因瞎改 | 先看 try_run_endpoint 返回的 stderr_jsonl |
| 修了一个 bug 就直接结束 | 重 try_run_endpoint 验证 |
| 改 main.py 不动 requirements.txt | 加新依赖时一起改 |
| 拿 update_app 只传改动文件 | 拿当前全量 + 替换目标文件 |

---

## v1.x 常见错误对照（dev 分支）

| 错 / 现象 | 含义 | 修法 |
|---|---|---|
| `cannot unmarshal object into ... []FileInput` | `update_app.files` 传成 map | 改成 array of `{path, content}` |
| `slug 和 version_id 都必填` | `publish_app` 用了 id 字段 | 改 `{slug, version_id}` |
| `endpoint not declared in manifest` | endpoint 没在 `plugin.yaml.endpoints[]` 登记，或 trigger 引用了已删 endpoint | 加 `{name, function}` 或 disable trigger |
| `MCP_XXX_URL not set` / `AI_XXX_KEY not set` | alias 没绑上凭据 | 凭据 name 精确等于 alias（小写不敏感） |
| AI 调 `HTTPError 404` | base_url 不含 `/v1` 而代码拼了 `/chat/completions` | 自适应：`base.endswith("/v1") ? base : base + "/v1"` + `/chat/completions` |
| AI `HTTPError 400: Model not exist` | 凭据 `default_model` 字段不可信（可能填了网关下线的型号） | `list_ai_models(credential_id=X)` 看实际清单，hardcode 或修凭据 |
| Worker 反复 crash,5min 3 次后自动 unpublish | python `sys.exit` / 未捕获异常 / stdin 被关 | 修代码 → publish 重新上;别死循环 publish + 调用 |
| `scheduler: trigger N has no endpoint_name; skipping` | 老 trigger 没填 endpoint_name | 删了重建（带 endpoint_name）；dev `04ade70+` 才接受这个字段 |
| 快照页白屏 / `/a/<slug>/snap/<id>` 404 | vite dev 没把这个路径代理到后端 | `vite.config.ts.proxy` 加 regex `^/a/[^/]+/snap/[^/]+$` |
| 沙箱 iframe 报 `from origin 'null' has been blocked by CORS` | `/a/<slug>` 主页的 iframe sandbox 调 endpoint | vite `server.cors: {origin: "*"}`；`dr.call` 用绝对 origin URL |
| 前端 / DB 中文乱码 | Windows worker stdout 默认 cp936/GBK | 已修（dev `5fc220c`）：worker.Spawn 强制 `PYTHONIOENCODING=utf-8` |
