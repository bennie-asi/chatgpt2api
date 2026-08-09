# FeaturePilot History

## 2026-08-09: openai-chat-tool-calls

**目标：** 为 `/v1/chat/completions` 增加客户端执行型 OpenAI Function Calling，使 SDK、Agent 和业务客户端能够完成“模型请求调用 → 调用方执行 → 回传工具结果 → 模型继续回答”的标准循环，同时保持既有文本、图片和 Web Search 行为。

**变更点：**
- 明确接收 function `tools`、`tool_choice` 与 `parallel_tool_calls`，并在访问上游前完成语义校验。
- 使用请求级 nonce 和有界 JSON 信封适配工具定义、assistant `tool_calls` 与 `role: "tool"` 历史。
- 投影标准非流式 `message.tool_calls` 与整轮缓冲后的流式 `delta.tool_calls`，解析失败时安全退化为普通文本。
- Function Calling 请求绕过 Chat Completion Cache；工具结果参与内容检查，但完整内容不写入 Call Record。
- 补充双语调用闭环、兼容边界、变更日志和本地契约测试证据。

**结构冲突：** None

**归档路径：** `fp-docs/archive/2026-08-09-openai-chat-tool-calls/`
