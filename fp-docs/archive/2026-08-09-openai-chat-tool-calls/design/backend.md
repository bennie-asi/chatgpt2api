# OpenAI Chat Completions 工具调用 — 技术方案设计

## 第一部分：架构决策

### Decision Ledger

| ID | Decision | Source | Blocking | Status | Evidence / explicit confirmation |
| --- | --- | --- | --- | --- | --- |
| D-001 | 新建 OpenAI 专用工具适配 Module，内部使用请求级 nonce 与 JSON 信封 | proposal `What Changes` 2 / design question D-001 | yes | `user-confirmed` | D-001: selected dedicated `services/protocol/openai_tool_calls.py` and nonce JSON envelope; user message `确认` immediately following D-001 |
| D-002 | 所有携带 function 工具的 Chat Completions 请求绕过响应缓存 | proposal `What Changes` 5 / design question D-002 | yes | `user-confirmed` | D-002: selected cache bypass for Function Calling while preserving ordinary chat cache; user message `确认` immediately following D-002 |
| D-003 | 限制为 128 个工具、64 位函数名、256 KiB 工具定义和输出信封 | proposal `What Changes` 1 / design question D-003 | yes | `user-confirmed` | D-003: selected the proposed compatibility and memory limits; user message `确认` immediately following D-003 |
| D-004 | 不新增 JSON Schema 执行依赖，业务参数 schema 由调用方校验 | proposal `Out of Scope` / design question D-004 | yes | `user-confirmed` | D-004: selected structural JSON validation only and caller-side business validation; user message `确认` immediately following D-004 |
| D-005 | 总体采用 endpoint-scoped adapter 方案 A | design alternatives A/B/C | yes | `user-confirmed` | D-005: selected approach `A`; user message `A` immediately following the architecture trade-off comparison |
| D-006 | 仅生成 backend small form 与直接 design index | proposal `Out of Scope` / design question D-006 | yes | `user-confirmed` | D-006: selected `design/00-index.md` plus small `design/backend.md`, with no frontend or migration; user message `确认` immediately following D-006 |
| D-007 | 工具结果参与内容检查，但完整内容不持久化到 Call Record | backend security rule / design question D-007 | yes | `user-confirmed` | D-007: selected tool-result redaction for Call Records while retaining content review; user message `确认` immediately following D-007 |

### 决策 1：OpenAI 专用工具适配 Module

- **ID**：`D-001`
- **选择**：新增 `services/protocol/openai_tool_calls.py`，由它统一拥有工具校验、请求级 nonce、内部提示、历史转换和输出解析。
- **理由**：现有 Anthropic 适配器使用 endpoint-specific XML 和宽松正则；直接复用会把不同公开契约绑定在一起。现有 `openai_v1_chat_complete.py` 已同时编排文本、图片和 Web Search，继续内联会扩大 Interface 与回归面。
- **来源**：proposal `What Changes` 2、`services/protocol/anthropic_v1_messages.py:16`、`services/protocol/openai_v1_chat_complete.py:339`、用户对 D-001 的确认。
- **状态**：`user-confirmed`
- **是否阻塞**：是

### 决策 2：Function Calling 不使用响应缓存

- **ID**：`D-002`
- **选择**：请求中存在 `type: "function"` 工具时不调用 `chat_completion_cache`；无函数工具的现有路径保持原状。
- **理由**：缓存完整响应会重放旧 `tool_call_id`，并可能促使调用方重复执行有副作用的函数。缓存语义结果再重新投影 ID 会引入第二套状态和额外复杂度。
- **来源**：proposal `What Changes` 5、`services/protocol/openai_v1_chat_complete.py:347`、`services/protocol/openai_v1_chat_complete.py:358`、用户对 D-002 的确认。
- **状态**：`user-confirmed`
- **是否阻塞**：是

### 决策 3：兼容性与内存安全上限

- **ID**：`D-003`
- **选择**：最多 128 个函数；函数名使用 1–64 位 `[A-Za-z0-9_-]`；工具定义序列化 JSON 和模型工具信封分别不超过 256 KiB。
- **理由**：保留常见客户端兼容空间，同时限制注入上游上下文和 `json.loads` 的内存输入。OpenAI 官方建议起始工具少于 20 个以提高准确性，但该建议不作为本项目的硬拒绝阈值。
- **来源**：proposal `What Changes` 1、OpenAI Function Calling 官方文档、用户对 D-003 的确认。
- **状态**：`user-confirmed`
- **是否阻塞**：是

### 决策 4：参数 schema 的执行边界

- **ID**：`D-004`
- **选择**：项目验证信封、函数名和 `arguments` JSON object，不执行完整 JSON Schema 校验；调用方在执行函数前验证业务 schema。
- **理由**：项目不执行工具，也不承诺原生 Structured Outputs。引入通用 schema 执行器会增加依赖与不属于本 Adapter 的业务语义。
- **来源**：proposal `Out of Scope`、用户对 D-004 的确认。
- **状态**：`user-confirmed`
- **是否阻塞**：是

### 决策 5：总体方案 A

- **ID**：`D-005`
- **选择**：采用 endpoint-scoped adapter：专用 Module、nonce JSON 信封、缓存绕过、工具流式整轮缓冲。
- **理由**：与方案 B 的跨协议 XML 共享相比隔离性更好；与方案 C 的单文件内联相比，能以更小的 Interface 集中复杂度并独立测试。
- **来源**：本轮方案 A/B/C trade-off、用户回复 `A`。
- **状态**：`user-confirmed`
- **是否阻塞**：是

### 决策 6：设计产物形式

- **ID**：`D-006`
- **选择**：backend-only small form，使用 `design/00-index.md` 直接链接 `design/backend.md`；不生成 frontend form。
- **理由**：proposal 明确排除 `web-vue/`，完整后端设计低于 500 行和 30,000 字符，不需要 split form。
- **来源**：proposal `Out of Scope`、artifact-layout contract、用户对 D-006 的确认。
- **状态**：`user-confirmed`
- **是否阻塞**：是

### 决策 7：工具结果日志脱敏

- **ID**：`D-007`
- **选择**：工具结果完整内容仍用于现有内容检查；Call Record 的 `request_text` 只保留 `tool_call_id` 和省略占位。
- **理由**：工具结果可能包含外部系统数据。当前 `request_text` 会递归提取消息 `content`，直接复用会把结果持久化到 Call Record。
- **来源**：`services/content_filter.py:26`、`services/log_service.py:696`、用户对 D-007 的确认。
- **状态**：`user-confirmed`
- **是否阻塞**：是

### Pre-write Confirmation Evidence

- Covered IDs: `D-001`, `D-002`, `D-003`, `D-004`, `D-005`, `D-006`, `D-007`
- Outstanding blocking decisions: `none`
- Explicit user authorization to write: the user separately confirmed all four reviewed design sections, then replied `确认` to the request authorizing approach A, backend small form, `design/00-index.md`, and `design/backend.md`, with no conversion or removal.

## 第二部分：技术方案详述

### 后端模块设计

| 文件 | 变更 | 唯一职责 |
| --- | --- | --- |
| `api/ai.py` | 修改 | 显式接收工具字段；分别构造内容检查文本和脱敏 Call Record 摘要；继续负责 HTTP 鉴权与路由 |
| `services/protocol/openai_tool_calls.py` | 新增 | 工具请求识别、语义校验、`ToolCallPlan`、nonce 提示、历史转换、输出解析和日志脱敏投影 |
| `services/protocol/openai_v1_chat_complete.py` | 修改 | 在图片、Web Search、普通文本与 Function Calling 之间编排；生成 Chat Completions JSON/SSE 投影 |
| `services/protocol/conversation.py` | 不修改 | 继续消费已经转换为文本/多模态内容的消息，不承担 OpenAI 工具语义 |
| `services/protocol/chat_completion_cache.py` | 不修改 | 普通聊天继续使用现有缓存；Function Calling 分支在调用缓存前被截断 |
| `services/protocol/anthropic_v1_messages.py` | 不修改 | 保持现有 Anthropic XML 行为，不参与新 Module |
| `README.md`、`README_EN.md`、`CHANGELOG.md` | 修改 | 公开兼容范围、调用循环、流式延迟和 best-effort 边界 |

`openai_tool_calls.py` 暴露以下窄 Interface，具体类名可在实现时保持同等语义：

```python
def build_tool_plan(body: dict[str, Any]) -> ToolCallPlan | None: ...
def adapt_tool_messages(messages: list[dict[str, Any]], plan: ToolCallPlan) -> list[dict[str, Any]]: ...
def parse_tool_output(text: str, plan: ToolCallPlan) -> ToolOutput: ...
def redact_tool_results_for_log(messages: object) -> object: ...
```

- `build_tool_plan` 只在请求包含 function 工具或 function `tool_choice` 时建立计划；纯 Web Search 继续走现有分支。
- 请求同时包含 function 与非 function 工具时返回 400，不尝试同轮编排。
- `tool_choice: "none"` 仍建立计划并绕过缓存，但不向上游暴露本轮可调用函数。
- 公开响应结构仍由 `openai_v1_chat_complete.py` 拥有；工具 Module 返回中立的 `ToolOutput`，不依赖 FastAPI Response 类型。

### 数据模型

本次不新增数据库、配置、持久化实体或跨请求状态。请求内使用不可变数据结构：

| 模型 | 字段 | 用途 |
| --- | --- | --- |
| `FunctionToolSpec` | `name`, `description`, `parameters`, `strict` | 规范化后的单个 function 工具 |
| `ToolChoicePolicy` | `mode`, `function_name` | 表示 `auto`、`none`、`required` 或指定函数 |
| `ToolCallPlan` | `tools`, `choice`, `parallel`, `nonce` | 当前请求的权威工具计划 |
| `ParsedToolCall` | `name`, `arguments` | 已通过信封、名称和 JSON object 校验的模型调用意图 |
| `ToolOutput` | `visible_text`, `calls`, `fallback_reason` | 解析后的普通文本或工具调用结果 |

- nonce 使用安全随机值生成，只存在于当前 `ToolCallPlan`，不进入日志、缓存或公开响应。
- 对外 `tool_call_id` 在投影时生成 `call_<uuid>`；客户端必须在下一轮原样回传。
- token usage 的输入基于适配后的实际上游 messages，输出基于上游原始生成文本计数，避免忽略工具提示和信封 token。

### API 接口

路由和鉴权保持不变：

```text
POST /v1/chat/completions
Authorization: Bearer <User Key>
```

新增/明确请求字段：

| 字段 | 类型 | 规则 |
| --- | --- | --- |
| `tools` | `list[object]` | Function Calling 分支仅接受 `type: "function"`；每项含 `function.name`，可含 `description`、`parameters`、`strict` |
| `tool_choice` | string/object | `auto`、`none`、`required`；兼容嵌套 `function.name` 和扁平 `name` 的指定函数形式 |
| `parallel_tool_calls` | boolean | 默认 `true`；`false` 时最多投影第一个有效调用 |
| `messages[].tool_calls` | list | assistant 历史调用，必须包含唯一 ID、function 名称与 JSON 字符串参数 |
| `messages[].tool_call_id` | string | `role: "tool"` 的结果关联，必须引用此前 assistant 调用 |

输入错误映射：

- Pydantic 字段形状错误沿用 FastAPI 422。
- 工具数量、名称、大小、重名、指定函数不存在、混合工具类型和历史关联错误返回 HTTP 400。
- 语义错误在调用 Upstream Account 之前完成，不消耗上游请求。

非流式工具调用投影：

```json
{
  "choices": [{
    "index": 0,
    "message": {
      "role": "assistant",
      "content": null,
      "tool_calls": [{
        "id": "call_<uuid>",
        "type": "function",
        "function": {"name": "get_weather", "arguments": "{\"city\":\"Shanghai\"}"}
      }]
    },
    "finish_reason": "tool_calls"
  }]
}
```

流式工具调用至少包含两个 Chat Completion chunks：

1. assistant delta：一次返回完整 `tool_calls` 数组；每项含 `index`、ID、类型、函数名和完整参数字符串。
2. terminal delta：`delta` 为空，`finish_reason` 为 `tool_calls`。

项目现有 SSE writer 继续负责 JSON 编码和 `[DONE]`。不新增独立流传输层，也不改变无函数工具请求的逐 token 输出。

### 业务逻辑要点

#### 1. 请求分类与校验

1. `api/ai.py` 解析请求并保留现有 User Key 鉴权。
2. `openai_v1_chat_complete.handle` 先处理图片请求；随后构造 `ToolCallPlan`。
3. 无 function 计划时保持现有 Web Search/普通文本分支顺序。
4. 有 function 计划时执行数量、名称、定义大小、选择策略和历史关联校验。
5. 校验完成后直接进入 Function Calling 文本分支，不读取或写入 Chat Completion Cache。

#### 2. 内部提示与历史转换

工具输出格式固定为：

```text
<tool_calls nonce="<request-nonce>">
{"calls":[{"name":"<function-name>","arguments":{}}]}
</tool_calls>
```

- `auto` 允许普通文本或信封；`required` 要求至少一个调用；指定函数只暴露目标函数并要求恰好一次；`none` 不附加可调用函数格式。
- 工具定义、选择策略和并行策略写入一个 adapter-owned system message。它位于调用方 system messages 之后、首个非 system message 之前，使协议约束具有稳定顺序。
- assistant 历史调用转换为 JSON `tool_call_history` 记录，保留 ID、名称和参数。
- tool 结果转换为 JSON `tool_result` 记录，包含 ID、由历史映射出的名称和结果字符串；结果被明确标记为不可信数据，不得作为系统指令执行。
- 转换结果只包含现有 Conversation Module 支持的 role 与文本 content，再交给 `normalize_messages`。

#### 3. 有界解析

1. 先用精确 nonce 构造开始和结束定界符。
2. 只接受至多一个完整信封；信封体超过 256 KiB 时进入安全退化。
3. 使用 `json.loads` 解析 object，并要求 `calls` 为非空 list。
4. 每项函数名必须存在于 `ToolCallPlan`，`arguments` 必须是 JSON object。
5. 指定函数模式只接受该函数；`parallel_tool_calls: false` 时确定性保留第一个有效调用。
6. 为每个调用生成新的 `call_<uuid>`，内部 nonce 永不进入公开字段。

解析不使用跨整段文本的宽松 XML 正则。出现 nonce 不匹配、多信封、截断、无效 JSON、未知函数或无效参数时，使用有界字符串扫描删除可识别的内部标记和信封内容，保留外围可见文本并返回 `finish_reason: "stop"`。即使 `required` 或指定函数被模型违反，也不伪造调用或把协议错误升级为 5xx。

#### 4. 非流式与流式执行

- 非流式：使用现有 `text_backend` 和 `collect_text` 收集上游文本，解析后生成普通 message 或 tool_calls message。
- 流式：返回惰性 generator；generator 内完整消费上游文本，完成解析后才产生第一个公开 chunk，使 `LoggedCall.run` 仍能在首块前映射上游异常。
- 缓冲后的普通文本通过一个 content delta 返回；工具调用通过一个完整 tool_calls delta 返回。
- Upstream Account 选择、鉴权失败后的账号切换和账户使用标记继续由 `stream_text_deltas` 拥有。

#### 5. Call Record 与内容检查

`api/ai.py` 为 function 请求构造两份瞬时文本：

- 内容检查文本：包含工具结果，继续传给 `filter_or_log`。
- Call Record 摘要：通过 `redact_tool_results_for_log` 把 `role: "tool"` 的 content 替换为带 `tool_call_id` 的省略占位，再交给 `LoggedCall.request_text`。

两份文本都不写入请求 body，不建立第二份业务状态。工具定义和 assistant 参数当前不会被 `request_text` 的 content 提取规则展开；本次不新增原始 payload 日志。

#### 6. 失败与兼容边界

- 请求语义错误：HTTP 400。
- 上游网络、鉴权、限流和生成失败：沿用现有错误响应与账号切换行为。
- 工具信封解析失败：普通文本退化，不返回 502。
- `required`、指定函数和 `strict` 是 ChatGPT Web 文本后端上的 best-effort 适配，不宣称等价于 OpenAI 原生模型约束。
- 不执行工具、不注册工具、不授权副作用、不增加数据库或配置开关。
- 纯 Web Search 行为保持不变；function 与非 function 工具混合请求不进入 Web Search 编排。

### 测试与验证

测试源码遵循项目规则放在 Git 忽略的本地 `tests/`，通过与生产调用方一致的 Interface 验证：

| 测试范围 | 关键用例 |
| --- | --- |
| `tests/protocol/test_openai_tool_calls.py` | 工具规范化、选择策略、128/64/256 KiB 边界、nonce、历史配对、日志脱敏、有界解析和退化 |
| `tests/protocol/test_openai_v1_chat_complete_tools.py` | 非流式调用、多调用、`parallel_tool_calls: false`、续轮、普通文本、缓存绕过、token usage |
| API 契约测试 | 422 与 400 区分、标准 `message.tool_calls`、SSE `delta.tool_calls`、`finish_reason` |
| 回归测试 | 无工具文本流、图片聊天、Web Search、Anthropic Messages 不变 |

验证顺序：

1. 运行新工具 Module 的针对性测试。
2. 运行 Chat Completions、Conversation、缓存和 API 契约相关测试。
3. 运行完整 `python -m pytest`。
4. 检查 `git diff`、未跟踪生产文件和文档一致性。

本次无前端构建、浏览器检查或真实工具副作用验证。

### 文档与验收契约

- `README.md` 与 `README_EN.md` 增加完整调用循环：工具定义、assistant `tool_calls`、调用方执行、`role: "tool"` 回传和最终回答。
- `CHANGELOG.md` 记录 `/v1/chat/completions` Function Calling、工具流式缓冲和 best-effort 限制。
- 验收时确认公开响应和 Call Record 均不包含 nonce、内部信封或完整工具结果。
- 普通聊天、图片和现有 Web Search 的请求与响应快照保持兼容。

### Proposal Coverage

| Proposal requirement | Design owner |
| --- | --- |
| P-001：仅 `/v1/chat/completions` | API 接口、后端模块设计 |
| P-002：调用方执行工具 | API 接口、业务逻辑要点 6 |
| P-003：完整声明范围与多调用 | 数据模型、请求分类、有界解析 |
| P-004：工具流式整轮缓冲 | 非流式与流式执行 |
| P-005：解析失败退化文本 | 有界解析、失败与兼容边界 |
| P-006：proposal form | 上游 proposal 已确认；本设计使用 D-006 独立确认设计 form |
