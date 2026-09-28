# 本地 RAG 代码位置、主要改动与合并前检查

## 1. 本地代码在哪里

本次修改所在仓库：

```text
C:\Users\huang\Documents\ChatGPT\Hermes 2\hermes-agent-enhanced-rag
```

当前 Git 状态：

| 项目 | 当前值 |
|---|---|
| 分支 | `codex/rag-backends` |
| 修改基线 HEAD | `c036c94c5` |
| 已跟踪但未提交的修改 | 17 个文件 |
| 新增但未跟踪的文件 | 11 个文件 |
| Commit | 未创建 |
| Push/PR | 未执行 |

也就是说，代码目前只存在于上述本地工作区。切换分支、清理工作区或删除目录前，应先备份或自行提交。

相关文档：

- [详细实现与设计说明](rag-backends-implementation-details.zh-CN.md)
- [独立接口文档](rag-backend-api-reference.zh-CN.md)
- [精简使用说明](rag-backends.md)
- [最初架构设计](rag-backend-integration-design.md)

## 2. 这次实现了什么

本次把 OpenViking 和 LLM-Wiki 接入 Hermes 的统一 Knowledge/RAG 边界，主要能力包括：

1. 调用方可以在 request、session 或 Profile 层选择 `openviking` 或 `llm-wiki`。
2. 选择顺序为 request > session > profile > legacy。
3. 后端失败时返回结构化错误，不静默切换到其他后端。
4. OpenViking RAG 与 `memory.provider` 独立。
5. Memory 和 Knowledge 都使用 OpenViking 时才共享实例。
6. OpenViking 自动 RAG 只检索 Resources，不混入 memories 或 skills。
7. LLM-Wiki 使用隔离的子 Agent 和私有 search/read 工具执行多步导航。
8. `top_k`、字符预算、Wiki 路径范围和 LLM-Wiki 返回 schema 都由代码强制执行。
9. 检索证据、来源、延迟、告警、错误和指纹被写入结构化 `result["rag"]`。
10. 增加 `hermes --rag-backend ... -z ...` one-shot 入口，没有增加第二个 HTTP 服务。

## 3. 主要修改了哪些代码

### 3.1 核心协议和路由

#### `agent/knowledge_provider.py`

主要修改：

- 新增 `KnowledgeRequest`；
- 新增 `KnowledgeEvidence`；
- 扩展 `KnowledgeSearchResult`；
- 增加 `configure_knowledge()` 和 `retrieve_knowledge()`；
- 保留旧 `search_knowledge()` 的兼容路径。

这是所有 RAG Provider 共用的数据协议。

#### `agent/knowledge_registry.py`（新增）

主要功能：

- 注册 Provider 工厂；
- 规范化 Provider 名称；
- 延迟导入和创建 OpenViking/LLM-Wiki；
- 拒绝重复名称和工厂返回类型错误。

#### `agent/knowledge_base_manager.py`

这是本次核心改动最多的文件，主要完成：

- 从单 Provider 管理扩展为多 Provider 注册；
- request/session/profile/legacy 四级选择；
- RLock 保护的延迟加载；
- 独占与共享实例的生命周期管理；
- 无静默 fallback；
- top-K 和 evidence 字符预算二次强制执行；
- evidence 到 content/citation 的统一转换；
- query hash、selection source、延迟和 fallback 状态等诊断；
- Provider 错误脱敏。

### 3.2 Agent 初始化与 Turn 流程

#### `agent/agent_init.py`

- 在 Memory 初始化之后独立创建 Knowledge Manager；
- 构造 `ProviderScope`；
- 当前 Memory Provider 同时支持 Knowledge 时允许非 owning 共享；
- 新增 `skip_knowledge`，供 LLM-Wiki 内层 Agent 防止递归初始化。

#### `run_agent.py`

- `AIAgent` 构造增加 `skip_knowledge`；
- `run_conversation()` 增加 `rag_backend`；
- 新增 `set_rag_backend()`；
- Agent 关闭时关闭 Knowledge 独占实例。

#### `agent/conversation_loop.py`

- 将 `rag_backend` 从公开入口传入 TurnContext。

#### `agent/turn_context.py`

- 每个 Turn 开始时执行一次 Knowledge 检索；
- 将证据包装为独立的 `<knowledge-context>`；
- 保持 Memory 和 Knowledge 上下文分离；
- 把实际 API 内容写入 `api_content` sidecar，保持 Prompt Cache 重放一致。

#### `agent/turn_finalizer.py`

- 将 RAG 的 requested/actual backend、证据、来源、延迟、告警、错误、metadata 和 trace ID 写入最终结果。

### 3.3 OpenViking

#### `plugins/memory/openviking/__init__.py`

新增或修改：

- Knowledge 专用配置，不修改 Memory 配置；
- 独立 `initialize_knowledge()` 生命周期；
- 结构化 `retrieve_knowledge()`；
- 自动检索固定使用 resource 类型；
- `target_uri`、阈值、总超时、单请求超时、全文读取次数和摘要优先配置；
- Provider 内 top-K/字符预算；
- 配置指纹、搜索延迟和读取延迟；
- Memory/Knowledge 同时选择 OpenViking 时支持共享实例。

### 3.4 LLM-Wiki

#### `plugins/knowledge/__init__.py`（新增）

Knowledge Provider 插件包入口。

#### `plugins/knowledge/llm_wiki/__init__.py`（新增）

主要功能：

- 创建隔离的 non-interactive Hermes 子 Agent；
- 只启用 `llm-wiki-internal` 工具集；
- 禁用 Memory、Knowledge、上下文文件、Session DB 和 trajectory；
- 加载仓库内置 LLM-Wiki Skill；
- 要求严格单个 JSON 对象；
- 校验引用页面确实被工具读取过；
- 将实际 read trace 转换为统一 Evidence；
- 记录模型调用、token、页面、语料和 Skill 指纹。

#### `tools/llm_wiki_tools.py`（新增）

提供两个内部只读工具：

- `llm_wiki_search`
- `llm_wiki_read`

并在代码层实施：

- Wiki root 限制；
- 禁止绝对路径和 `..`；
- 禁止符号链接逃逸；
- 只读 Markdown；
- 逻辑 source top-K；
- evidence 字符预算；
- 页面读取 trace 和阶段延迟。

### 3.5 配置与 CLI

#### `hermes_cli/config_defaults.py`

新增完整 `knowledge` 默认配置，包括 OpenViking 与 LLM-Wiki 子配置。

#### `hermes_cli/_parser.py`

新增：

```text
--rag-backend {openviking,llm-wiki}
```

#### `hermes_cli/main.py`

把 `rag_backend` 传入 one-shot 路径。

#### `hermes_cli/oneshot.py`

- one-shot 接受 `rag_backend`；
- 将其传给 `AIAgent.run_conversation()`；
- `--usage-file` 结果中加入 `rag`。

### 3.6 脱敏 smoke 与测试

#### `scripts/smoke_rag_retrieval.py`（新增）

通过公共 Manager 路径执行单次真实后端检索，只输出白名单字段、URI 和正文长度，不输出 API Key 或证据正文。

测试修改或新增：

- `tests/agent/test_knowledge_base_manager.py`
- `tests/agent/test_llm_wiki_provider.py`
- `tests/plugins/memory/test_openviking_provider.py`
- `tests/agent/test_turn_finalizer_iteration_limit_exit.py`
- `tests/hermes_cli/test_argparse_flag_propagation.py`
- `tests/hermes_cli/test_oneshot_usage_file.py`

## 4. 当前新增文件

```text
agent/knowledge_registry.py
docs/local-rag-code-handoff.zh-CN.md
docs/rag-backend-api-reference.zh-CN.md
docs/rag-backend-integration-design.md
docs/rag-backends-implementation-details.zh-CN.md
docs/rag-backends.md
plugins/knowledge/__init__.py
plugins/knowledge/llm_wiki/__init__.py
scripts/smoke_rag_retrieval.py
tests/agent/test_llm_wiki_provider.py
tools/llm_wiki_tools.py
```

## 5. 验证做到了什么程度

### 5.1 已通过的验证

| 验证项 | 结果 |
|---|---|
| 本次所有 Python 改动 Ruff 检查 | 通过 |
| RAG 核心聚焦测试 | 75/75 通过 |
| TurnContext/TurnFinalizer 补充回归 | 29/29 通过 |
| 配置相关测试 | 101 通过、1 个既有 Windows 路径断言失败 |
| `git diff --check` | 通过，仅有 Windows 行尾提示 |
| OpenViking 真实服务 smoke | 通过 |
| LLM-Wiki 真实模型 smoke | 通过 |
| OpenViking 容器测试后状态恢复 | 已恢复为停止状态 |

OpenViking 真实 smoke：

- 查询成功；
- `actual_backend=openviking`；
- `degraded=false`；
- `fallback_used=false`；
- 返回 4 条证据；
- 证据严格限制到 12,000 字符；
- 总延迟约 849 ms。

LLM-Wiki 真实 smoke：

- 查询成功；
- `actual_backend=llm-wiki`；
- `degraded=false`；
- `fallback_used=false`；
- 返回 1 个实际读取页面、5,846 字符；
- 完成 6 次额外模型调用；
- 总延迟约 40.3 秒。

所以可以确认：代码不是只写了没有运行。两个真实后端都通过了公共 Manager 路径。

### 5.2 没有得到“全仓全绿”的验证

尝试运行过全仓测试，但当前 Windows 环境出现多项与本次 RAG 无关的基线问题，例如：

- 缺少可选 `acp` 包；
- 缺少可选 `anthropic` SDK；
- Windows HOME/路径行为与部分测试的 Unix 假设不一致；
- 部分用例尝试访问受限网络；
- 并行全仓测试被主动中止后，尚未执行的文件会被 runner 标记为未完成，不能用于判断本次改动。

因此，全仓测试结果不是有效的“全绿”证明，也不能把这些环境失败直接归因到 RAG 代码。

此外，没有重新执行付费 fixed-500。旧 fixed-500 存在 6 次预算违规和 2 次证据归档失败，正式状态为 `invalid_experiment`，不能拿来证明当前实现。

## 6. 现在能否合并

客观结论：

> 当前代码已经达到“可以提交 Code Review 和 CI”的状态，但现有证据不足以承诺“无需进一步检查即可直接合并”。

支持进入评审的证据：

- 核心接口、两种 Provider、选择优先级、预算、路径安全、生命周期和 CLI 都有针对性测试；
- 104 个直接相关的核心/Turn 测试通过；
- 配置回归除一个已知 Windows 基线问题外通过；
- 两个真实后端均实际跑通；
- 静态检查和 diff 完整性检查通过。

仍需在正式合并门禁完成：

1. 在项目标准 CI/Linux 环境安装完整 dev/optional 依赖。
2. 从干净基线和当前分支分别运行完整测试，区分基线失败和新增失败。
3. 运行仓库正式 lint/type/check 门禁。
4. 由维护者复核 OpenViking 现有显式资源工具的兼容行为。
5. 如合并标准要求效果评测，先做小样本协议预检；不要直接复用旧 fixed-500。

如果上述 CI 门禁通过，可以合并。当前本地验证没有发现阻止进入评审的 RAG 功能问题，但不能代替正式 CI 和代码评审。

## 7. 建议的合并前命令

聚焦测试必须通过仓库脚本运行：

```text
scripts/run_tests.sh \
  tests/agent/test_knowledge_base_manager.py \
  tests/agent/test_llm_wiki_provider.py \
  tests/plugins/memory/test_openviking_provider.py \
  tests/agent/test_turn_finalizer_iteration_limit_exit.py \
  tests/hermes_cli/test_argparse_flag_propagation.py \
  tests/hermes_cli/test_oneshot_usage_file.py
```

随后在完整依赖环境运行：

```text
scripts/run_tests.sh
```

不要直接调用 `pytest`，以免绕过仓库要求的测试环境和并行 runner。
