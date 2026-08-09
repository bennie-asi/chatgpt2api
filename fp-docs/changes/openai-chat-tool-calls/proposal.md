# OpenAI Chat Completions 工具调用

## Why

当前 `/v1/chat/completions` 虽然通过宽松请求模型接收 `tools`、`tool_choice` 等字段，但文本链路会把非 Web Search 工具视为不受支持，并向上游注入“工具不可用”的系统提示；返回投影也只有普通助手文本，无法产生 OpenAI 兼容的 `tool_calls`。这使依赖 Function Calling 的 SDK、Agent 和业务客户端无法通过本项目完成“模型请求调用 → 客户端执行 → 回传工具结果 → 模型继续回答”的标准循环。

本次变更为 Chat Completions 增加客户端执行型 Function Calling：项目负责验证公开请求、把工具上下文适配到上游文本会话、解析模型产生的调用意图，并投影为稳定的 OpenAI JSON/SSE 契约；项目自身不执行调用方提供的函数。

## What Changes

### 1. 扩展 Chat Completions 请求契约

- 在 `/v1/chat/completions` 明确接收 `type: "function"` 的 `tools` 定义、`tool_choice` 和 `parallel_tool_calls`。
- `tool_choice` 支持 `auto`、`none`、`required` 以及指定函数；指定函数必须存在于本次请求的工具集合中。
- 工具定义缺少名称、名称重复、schema 形状无效或出现本次能力范围外的工具类型时，返回明确的 4xx 输入错误，不进入上游会话。
- 保持不携带函数工具的普通文本、图片和现有 Web Search 路径行为不变。

### 2. 增加上游文本工具协议适配

- 把函数名称、描述、JSON Schema 与 `tool_choice` 约束合成为内部工具指令，要求上游模型用机器可解析的内部格式表达零个、一个或多个调用。
- 将历史 assistant `tool_calls` 和调用方回传的 `role: "tool"` 消息转换为有调用 ID 关联的上游文本记录，使下一轮模型能够消费工具结果。
- 内部标记只用于上游适配，不作为公开 API 契约，也不得泄漏到普通 `content`。
- 工具仍由调用方执行；本项目不加载、运行或授权调用方函数。

### 3. 投影非流式工具调用结果

- 成功解析调用意图时，在 `choices[0].message.tool_calls` 中返回稳定调用 ID、`type: "function"`、函数名和 JSON 字符串参数，并将 `finish_reason` 设为 `tool_calls`。
- 支持单轮多个调用；`parallel_tool_calls: false` 时最多投影一个调用。
- 上游返回普通回答或工具标记无法完整解析时，将清理后的可见文本作为普通助手消息返回，`finish_reason` 为 `stop`。
- 工具参数只做安全、可序列化的 JSON 规范化；不承诺模拟 OpenAI 原生 Structured Outputs 对 `strict: true` 的强保证。

### 4. 投影流式工具调用结果

- 对携带函数工具的 `stream: true` 请求缓冲完整上游轮次，先判定文本或工具调用，再输出 OpenAI 兼容 SSE。
- 工具调用通过 `choices[0].delta.tool_calls` 返回，包含调用索引、ID、类型、函数名与参数；终止块使用 `finish_reason: "tool_calls"`。
- 普通回答以标准文本 delta 返回并以 `finish_reason: "stop"` 结束；任何内部工具标记都不得在中间 chunk 中泄漏。
- 不携带函数工具的现有流式文本路径继续逐段转发，不因本次改造增加整轮缓冲延迟。

### 5. 固定兼容性、缓存与文档边界

- 缓存键必须区分工具定义、选择策略、并行策略和包含工具结果的消息历史，避免跨工具上下文复用响应。
- 为请求校验、内部协议构造与解析、非流式投影、流式 SSE、多调用、工具结果续轮和失败退化增加契约测试。
- 更新公开接口文档、示例和变更日志，明确调用方执行工具、工具请求流式缓冲以及 `strict` 语义边界。

## Capabilities

### New Capabilities

- `openai-chat-function-calling`: `/v1/chat/completions` 能将函数工具定义和工具结果适配到上游文本会话，并返回 OpenAI 兼容的单个或多个 `tool_calls`。

### Modified Capabilities

- `openai-chat-completions`: 扩展请求校验、消息规范化、缓存隔离、非流式响应和流式 SSE 投影，同时保持无函数工具请求的现有行为。

## Out of Scope

- 不改造 `/v1/responses`。
- 不改造现有 `/v1/messages` Anthropic XML 工具兼容层。
- 不在服务端执行调用方函数，也不新增工具注册、权限、沙箱、重试或副作用审计系统。
- 不新增 Shell、文件、MCP、Code Interpreter 等托管工具。
- 不承诺 `strict: true` 等同于 OpenAI 原生 Structured Outputs 的 schema 强制保证。
- 不改变现有 Chat Completions Web Search 的执行实现，也不在同一轮编排 Web Search 与函数工具。
- 不修改 `web-vue/` 页面或交互。

## Impact

- `api/ai.py` - 明确 Chat Completions 工具字段并在进入协议 Module 前完成稳定输入校验。
- `services/protocol/openai_v1_chat_complete.py` - 拥有函数工具协议适配、调用解析以及 JSON/SSE 对外投影。
- `services/protocol/conversation.py` - 保留并规范化 assistant `tool_calls`、tool result 及其调用关联，避免把结构化历史静默压成空文本。
- `services/protocol/chat_completion_cache.py` - 固定函数工具请求和工具结果历史的缓存隔离。
- `tests/` - 在本地忽略的 Python 测试树中覆盖公开 Interface、续轮和回归行为。
- `README.md`、`README_EN.md`、`CHANGELOG.md` - 记录公开兼容能力、限制和调用示例。
- `web-vue/` - 不受影响。

### Handoff Decision Ledger

| ID | Decision | Source | Blocking | Status | Evidence / explicit confirmation |
| --- | --- | --- | --- | --- | --- |
| P-001 | 本次仅改造 `/v1/chat/completions`，不扩展 `/v1/responses` 或 `/v1/messages` | user response to P-001 | yes | `user-confirmed` | P-001: selected option 1 (Chat Completions only); user message `1` immediately following the P-001 question |
| P-002 | 项目只返回工具调用，工具由调用方执行并以 `role: "tool"` 回传 | user response to P-002 | yes | `user-confirmed` | P-002: selected option 1 (caller-side execution); user message `1` immediately following the P-002 question |
| P-003 | 支持 function 工具、完整 `tool_choice` 范围与单轮多个 `tool_calls` | user response to P-003 | yes | `user-confirmed` | P-003: selected option 1 (full declared Function Calling range); user message `1` immediately following the P-003 question |
| P-004 | 携带函数工具的流式请求先缓冲整轮，再发送标准 SSE | user response to P-004 | yes | `user-confirmed` | P-004: selected option 1 (buffer before SSE projection); user message `1` immediately following the P-004 question |
| P-005 | 无法解析工具调用时退化为普通助手文本而非协议错误 | user response to P-005 | yes | `user-confirmed` | P-005: selected option 1 (text fallback); user message `1` immediately following the P-005 question |
| P-006 | 使用 small form，canonical entry 为 `fp-docs/changes/openai-chat-tool-calls/proposal.md` | user response to P-006 | yes | `user-confirmed` | P-006: selected the proposed small form and exact path; user message `确认` immediately following the P-006 question |

### Pre-write Confirmation Evidence

- Covered IDs: `P-001`, `P-002`, `P-003`, `P-004`, `P-005`, `P-006`
- Outstanding blocking decisions: `none`
- Explicit user authorization to write: after reviewing the complete Why / What Changes / Out of Scope / Impact summary, ledger, small form, and exact canonical path, the user replied `确认` to the separate request to write this proposal under P-001 through P-006.
