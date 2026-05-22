---
name: fix-or-extend-app
description: 修改一个现有 App —— 加字段 / 改逻辑 / 升级模板 / 改触发参数。重点：files 是全量替换不是增量。
allowed-tools: list_apps, get_app, update_app, try_run_endpoint
---

# 🔧 修改 / 扩展现有 App

## 黄金原则

`update_app(files=[...])` 是**全量替换**。没列出来的文件会消失。改一行也要把所有文件一起传。

## 标准流程

### 1. 读全部当前文件

```
> get_app slug=xxx
[AI → get_app → 拿到 files[]: 完整的文件树 + 内容]
```

### 2. 在本地（AI 的 working memory）改

把 files 数组拿出来，定位要改的文件，修改 content 字段。其余文件原样保留。

### 3. PUT 全量

```
> update_app slug=xxx, files=[全量数组], comment="fix: ..."
[AI → 新版本号 + build pending]
```

写好的 comment 是好习惯 —— 出问题翻 plugin_versions 表能看到。

### 4. 等 build → try_run_endpoint 验证

```
> get_app 看 build_status
> try_run_endpoint(slug="xxx", endpoint_name="<endpoint>", params={})
> 如果失败，走 debug-failed-run skill
```

## 常见修改场景

### 加一个字段到报告

```python
# 在 emit_report 的 doc 数组里加新 block:
sdk.emit_report([
    {"type": "markdown", "text": "..."},
    {"type": "kv", "data": {"原指标": x, "新指标": y}},  # ← 加 y
    {"type": "table", ...},
])
```

### 改用另一个内置模板

```yaml
# config/plugin.yaml
template: dashboard   # ← 改这个；list_templates 看可选
```

### 加新依赖

```
# config/requirements.txt
requests>=2.32
psycopg2-binary>=2.9.10  # 新加
```

加了之后必须重新 build —— save 即触发。

### 改 cache_ttl

```yaml
cache_ttl: 30   # 秒；/r/<slug> 重复访问的缓存窗口
```

`cache_ttl: 0` = 每次访问都重跑（适合调试，正式用不要）。

## 危险动作

| 动作 | 风险 |
|---|---|
| 改 slug | 旧 `/r/<旧slug>` 链接全死，分享出去的全部失效 |
| 删 emit_dataset 调用 | 老快照 OK，新快照前端少了那个 dataset 可能崩 |
| 改 main.py 的全局 sdk.* 调用顺序 | runner 跑的是模块顶层，import 顺序 / 副作用都要注意 |

如果改动是破坏性的（数据集 rename / required_* 减少），考虑**新建一个 slug**（v2 风格）而不是原地改。
