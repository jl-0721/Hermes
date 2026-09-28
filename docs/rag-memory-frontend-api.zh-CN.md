# RAG 与 Memory 前端接口文档

> 面向 Hermes Dashboard/Web 前端。Memory 与 RAG provider 管理接口均已实现；Session Chat 也支持按单次请求覆盖 RAG provider。

## 1. 范围和术语

本文只覆盖两类 provider：

- **RAG / Knowledge provider**：检索外部知识。当前代码实现 `openviking`、`llm-wiki`。
- **Memory provider**：管理跨会话记忆。内置记忆以空字符串 `""` 表示；外部 provider 由插件动态发现，具体选项以 API 返回为准，可能包括 OpenViking、TencentDB 等已安装插件。

二者互相独立。选择 RAG provider 不会切换 Memory provider，反之亦然。模型服务商（如 OpenAI、Anthropic）也不是这两类 provider。

## 2. 通用约定

- 基础路径：Dashboard Web Server 的 `/api`。
- 请求/响应：JSON，UTF-8。
- Web Dashboard 若启用了会话令牌，应在请求头中发送 `X-Session-Token: <token>`；前端现有 `fetchJSON` 封装会自动注入。不要把 token 写入 URL。
- Gateway Session API 使用其自身鉴权配置（如 `API_SERVER_KEY` / `Authorization: Bearer ...`）；与 Dashboard 的 `X-Session-Token` 不是同一个凭据。具体以部署时 API Server 配置为准。
- Memory provider 配置接口支持 `profile` 查询参数；读取/写入配置文件接口 `/api/config` 也支持 `profile`。但 `/api/memory` 与 `/api/memory/provider` 当前没有 profile 参数，状态和激活操作作用于当前运行配置。不要假设它们已支持按 profile 单独切换。
- 错误通常为 FastAPI JSON：`{"detail":"..."}`。常见状态码：`400` 参数无效或 provider 未就绪，`404` provider 不存在，`500` 服务端错误。

## 3. Memory：现有接口

### 3.1 查询当前选择和 provider 状态

`GET /api/memory`

响应示例：

```json
{
  "active": "openviking",
  "providers": [
    {
      "name": "openviking",
      "description": "...",
      "available": true,
      "configured": true,
      "status": "ready",
      "setup": {"dependencies_installed": true}
    },
    {
      "name": "memory_tencentdb",
      "description": "...",
      "available": true,
      "configured": false,
      "status": "needs_config",
      "setup": {"dependencies_installed": true}
    }
  ],
  "builtin_files": {"memory": 1200, "user": 300}
}
```

`providers` 是运行时发现结果，示例中的名称仅用于说明；前端必须动态渲染，不要硬编码 provider 列表。

状态值：

| `status` | 含义 | UI 建议 |
|---|---|---|
| `ready` | 可用且必填配置齐全 | 可选择 |
| `needs_config` | 缺少必填配置或 provider 尚未就绪 | 显示“需配置”，引导配置后再选 |
| `unavailable` | 依赖缺失或 provider 当前不可用 | 禁止启用，展示原因/安装入口 |
| `missing` | 当前配置引用的 provider 已不在安装目录 | 显示异常状态，不要静默改写配置 |

### 3.2 获取某 provider 的配置表单

`GET /api/memory/providers/{name}/config[?profile={profile}]`

响应字段由 provider 动态提供：

```json
{
  "name": "openviking",
  "label": "OpenViking",
  "fields": [
    {
      "key": "endpoint",
      "label": "Endpoint",
      "kind": "text",
      "description": "...",
      "placeholder": "https://...",
      "required": true,
      "value": "https://...",
      "is_set": true,
      "options": [],
      "url": "",
      "when": null
    },
    {
      "key": "api_key",
      "label": "API Key",
      "kind": "secret",
      "required": true,
      "value": "",
      "is_set": true
    }
  ],
  "setup": {}
}
```

`kind` 可为 `text`、`boolean`、`select`、`secret`。密钥值不会回显；`is_set` 表示服务器已有密钥。`when` 可用于条件显示字段。

另有 `surface=declared` 参数读取 Dashboard 声明式配置面板：

`GET /api/memory/providers/{name}/config?surface=declared[&profile={profile}]`

### 3.3 保存 provider 配置并激活

`PUT /api/memory/providers/{name}/config[?profile={profile}]`

请求：

```json
{
  "values": {
    "endpoint": "https://memory.example.com",
    "api_key": "用户新输入的密钥"
  }
}
```

成功响应：

```json
{"ok": true, "active": "openviking"}
```

该接口会保存字段，并在 provider 已就绪时把它设为活动 Memory provider。秘密字段只在用户提供新值时写入；编辑其他字段时，前端应省略未修改的密钥，不能用空字符串覆盖已有密钥。对 `surface=declared`，接口仅写声明式插件配置，不返回 `active`，也不执行上述激活步骤。

### 3.4 单独切换 / 关闭外部 Memory provider

`PUT /api/memory/provider`

请求：

```json
{"provider":"memory_tencentdb"}
```

关闭外部 provider、只用内置记忆：

```json
{"provider":""}
```

`"built-in"`、`"builtin"`、`"none"` 也会规范化为内置记忆。响应：

```json
{"ok":true,"active":"memory_tencentdb"}
```

未配置或不可用的 provider 会返回 `400`；前端应先检查 `GET /api/memory` 的 `status`，成功后再刷新状态。

### 3.5 安装缺失依赖（可选）

`POST /api/memory/providers/{name}/setup`

请求体可选：

```json
{"values":{"endpoint":"https://memory.example.com"}}
```

这是会触发依赖安装/外部 setup 步骤的管理操作，不是普通配置保存。仅在用户明确点击安装/初始化后调用，并将响应中的 `results` 展示出来。不可把它放进 provider 下拉框的自动选择流程。

### 3.6 可选的配置 schema 查询

`GET /api/config/schema[?profile={profile}]` 返回动态表单 schema，其中 `memory.provider.options` 是当次发现的 Memory provider 名称列表，空字符串表示内置记忆。若只需可用性、配置状态，优先使用 `GET /api/memory`。

通用配置读取/写入：

- `GET /api/config[?profile={profile}]`
- `PUT /api/config[?profile={profile}]`，请求体 `{"config":{...}}`

`PUT /api/config` 是面向整个 Dashboard 配置对象的深度合并接口。Memory 选择优先使用专用 `PUT /api/memory/provider`，避免前端读改写大配置对象造成并发覆盖。

## 4. RAG：现有管理接口

当前内置 RAG provider 为 `openviking` 和 `llm-wiki`，由服务端 Registry 返回。Dashboard provider 管理接口与 Memory 的模式一致，配置按 profile 隔离。

### 4.1 获取 provider 状态

`GET /api/rag[?profile={profile}]`

```json
{
  "active":"openviking",
  "enabled":true,
  "providers":[
    {"name":"openviking","label":"OpenViking","available":true,"configured":true,"status":"ready","reason":""},
    {"name":"llm-wiki","label":"LLM-Wiki","available":false,"configured":false,"status":"unavailable","reason":"Provider is not available in the current profile."}
  ],
  "settings":{"top_k":5,"evidence_char_budget":12000,"allow_fallback":false}
}
```

`status` 为 `ready`、`unavailable` 或 `missing`。当前 `available` 表示 provider 的本地配置/语料可用性，不保证远端服务在后续检索瞬间一定可达；检索错误会在 Chat 返回的 `rag` 元数据中报告。

### 4.2 获取 provider 配置表单

`GET /api/rag/providers/{name}/config[?profile={profile}]`

响应示例：

```json
{
  "name":"llm-wiki",
  "label":"LLM-Wiki",
  "fields":[
    {"key":"root","label":"Wiki root","kind":"text","default":"","value":"","required":false,"is_set":false},
    {"key":"max_iterations","label":"Maximum iterations","kind":"integer","default":20,"value":20,"required":false,"is_set":true}
  ]
}
```

字段定义随 provider 变化。当前 `openviking` RAG 字段：`target_uri`、`score_threshold`、`timeout_seconds`、`request_timeout_seconds`、`full_read_limit`、`prefer_abstract`；`llm-wiki` 字段：`root`、`manifest`、`model`、`model_provider`、`max_iterations`。连接凭据不在该表单中返回，沿用 OpenViking 现有凭据配置。

### 4.3 保存 provider 专属配置

`PUT /api/rag/providers/{name}/config[?profile={profile}]`

```json
{"values":{"root":"/data/wiki","max_iterations":16}}
```

响应：`{"ok":true,"provider":"llm-wiki","config":{...}}`。只更新传入字段，不会自动改变当前默认 provider；用户明确选择后再调用 4.4。未知字段、越界数字返回 `400`，未知 provider 返回 `404`。

数值边界：`score_threshold` 为 `[0,1]`；超时为 `0.25–60` 秒；`max_iterations` 为 `3–1000`；`full_read_limit` 为 `0–1000`。

### 4.4 设置 Profile 默认 provider

`PUT /api/rag/provider[?profile={profile}]`

```json
{"provider":"llm-wiki"}
```

成功响应：`{"ok":true,"active":"llm-wiki"}`。选择非空 provider 会同时确保 `knowledge.enabled=true`。传 `{"provider":""}` 清空显式默认，恢复自动/兼容解析。不可用或不存在的 provider 返回 `400`，不会静默切换到其他后端。

### 4.5 通用 RAG 配置

现有配置文件接口可以保存 `knowledge` 配置：

```http
GET /api/config?profile=default
PUT /api/config?profile=default
Content-Type: application/json
```

```json
{
  "config": {
    "knowledge": {
      "enabled": true,
      "provider": "openviking",
      "top_k": 5,
      "evidence_char_budget": 12000
    }
  }
}
```

provider 默认选择建议使用 `/api/rag/provider` 专用接口；通用设置（例如 `enabled`、`top_k`、`evidence_char_budget`）可用 `/api/config` 读写。provider 专属字段建议使用上述专用配置表单接口。当前已实现的 provider 与配置细节见 [RAG 后端接口说明](rag-backend-api-reference.zh-CN.md)。

## 5. Chat：单次请求选择 RAG provider

Session Chat 接口支持可选 `rag_backend`，只影响当前 Turn，不写回 Profile 或 Memory 配置：

`POST /api/sessions/{session_id}/chat` 或 `/chat/stream`

```json
{"message":"查找部署说明","rag_backend":"openviking"}
```

目标路径：

- `POST /api/sessions/{session_id}/chat`
- `POST /api/sessions/{session_id}/chat/stream`

省略 `rag_backend` 或传空字符串表示使用 Profile 默认/自动解析。显式值目前接受已注册 provider 名称（`openviking`、`llm-wiki`）；未知值返回 `400`，不可静默切换 provider。服务端会将该参数传给 Agent 当前 Turn 的 `run_conversation`。

成功响应建议在 turn metadata 中带回可审计结果：

```json
{
  "rag": {
    "requested_backend":"openviking",
    "actual_backend":"openviking",
    "latency_ms":412.7,
    "degraded":false,
    "error":"",
    "warnings":[],
    "evidence":[],
    "trace_id":"..."
  }
}
```

同步响应包含顶层 `rag`；流式接口在 `assistant.completed` 和 `run.completed` 结束事件中返回同一元数据。

## 6. 前端建议调用顺序

### Memory

1. 页面初始化调用 `GET /api/memory`，按 `providers` 和 `status` 绘制选项。
2. 用户选 provider 后调用 `GET /api/memory/providers/{name}/config` 绘制动态配置表单。
3. 缺少配置时先保存 `PUT /api/memory/providers/{name}/config`；接口成功会一并激活 provider。仅切换到已配置 provider 时可调用 `PUT /api/memory/provider`。
4. 成功后重新请求 `GET /api/memory` 更新 UI。

### RAG

1. 调用 `GET /api/rag?profile=...`，根据 provider 的 `status` 绘制选项。
2. 用户打开设置时调用 `GET /api/rag/providers/{name}/config`；保存专属设置调用对应 `PUT`。
3. 用户设置默认值调用 `PUT /api/rag/provider`；每次对话覆盖则在 Session Chat JSON 中携带 `rag_backend`。
4. Chat 完成后读取响应中的 `rag` 字段，展示实际 provider、耗时、错误/降级和引用证据。

## 7. 当前实现状态速查

| 能力 | Memory | RAG |
|---|---|---|
| provider 动态发现/状态 | 有：`GET /api/memory` | 有：`GET /api/rag` |
| 配置 schema | 有：`GET /api/memory/providers/{name}/config` | 有：`GET /api/rag/providers/{name}/config` |
| 激活/默认选择 | 有：`PUT /api/memory/provider` | 有：`PUT /api/rag/provider` |
| 单次 Chat 请求选择 | 无 per-request Memory 字段 | 有：Session Chat 的 `rag_backend` |
| 运行时状态回传 | `GET /api/memory` | Chat 响应/结束事件的 `rag` metadata |

## 8. 代码依据

- Dashboard Memory 路由：`hermes_cli/web_server.py`
- 请求/响应模型：`hermes_cli/web_models.py`
- 前端 API 封装：`web/src/lib/api.ts`
- RAG Agent 接口与配置：`agent/conversation_loop.py`、`agent/knowledge_base_manager.py`、`agent/knowledge_provider.py`
- Gateway Session Chat 路由：`gateway/platforms/api_server.py`
- RAG provider 细节：[RAG 后端接口说明](rag-backend-api-reference.zh-CN.md)
