# OpenAI Chat Completions 工具调用 — 后端实施计划

## Global Constraints

- 仅修改 `POST /v1/chat/completions` 的 Function Calling；不得改变 `/v1/responses`、`/v1/messages`、图片聊天或既有 Web Search 编排。
- 遵循 proposal `P-001`–`P-006` 与 backend design `D-001`–`D-007`；调用方负责执行函数并用 `role: "tool"` 回传结果。
- 新协议逻辑集中在 `services/protocol/openai_tool_calls.py`；不得把 OpenAI 工具协议放入通用 `conversation.py`，也不得复用或修改 Anthropic XML 工具协议。
- function 工具请求绕过 Chat Completion Cache；普通文本请求继续使用现有缓存路径。
- 只做结构性 JSON 校验，不增加 `jsonschema` 或 OpenAI SDK。上限固定为 128 个工具、64 字符函数名、256 KiB 工具定义和 256 KiB 输出信封。
- 工具结果完整内容参与 `filter_or_log`，但 `LoggedCall.request_text` 只能持久化 `tool_call_id` 与省略占位符；nonce、内部信封和完整工具结果不得出现在公开响应或 Call Record。
- 流式 function 请求完整消费上游文本后再发送标准 Chat Completions SSE chunks；解析失败安全退化为普通 assistant 文本。
- 测试源码放在项目规则指定且 Git 忽略的本地 `tests/`；生产代码与文档可纳入版本控制。
- 当前用户只授权实现，未授权 Git 提交。各任务的 Commit 步骤仅记录建议命令，不得执行，除非用户另行明确授权。

## File Structure

| Path | Action | Responsibility |
| --- | --- | --- |
| `services/protocol/openai_tool_calls.py` | Create | 请求校验、nonce 提示、历史适配、有界解析、日志脱敏 |
| `services/protocol/openai_v1_chat_complete.py` | Modify | Function Calling 分支、非流式/流式投影、缓存绕过 |
| `api/ai.py` | Modify | 显式请求字段、内容检查预览与日志脱敏预览 |
| `tests/protocol/test_openai_tool_calls.py` | Create | 工具 adapter 单元测试 |
| `tests/protocol/test_openai_v1_chat_complete_tools.py` | Create | Chat Completions 工具分支与回归测试 |
| `tests/api/test_ai_tool_calls.py` | Create | 422/400 与日志边界契约测试 |
| `tests/docs/test_function_calling_docs.py` | Create | 双语说明和 CHANGELOG 验收测试 |
| `README.md`, `README_EN.md`, `CHANGELOG.md` | Modify | 调用闭环、兼容限制与变更记录 |

明确不修改：`services/protocol/conversation.py`、`services/protocol/chat_completion_cache.py`、`services/protocol/anthropic_v1_messages.py`、数据库、配置和前端。

## Backend Interface Ledger

| Interface | Owner and contract |
| --- | --- |
| `FunctionToolSpec` | `openai_tool_calls.py`；规范化 function `name`、`description`、`parameters`、`strict` |
| `ToolChoicePolicy` | `openai_tool_calls.py`；`auto`、`none`、`required` 或指定函数策略 |
| `ToolCallPlan` | `openai_tool_calls.py`；当前工具集合、选择策略、并行策略和请求级 nonce |
| `ParsedToolCall`, `ToolOutput` | `openai_tool_calls.py`；已校验调用及可见文本/calls/fallback reason |
| `ToolRequestError` | `openai_tool_calls.py`；可映射 HTTP 400 且不依赖 FastAPI 的语义错误 |
| `build_tool_plan(body)` | 无 function 工具返回 `None`；否则在访问上游前校验并生成 nonce |
| `adapt_tool_messages(messages, plan)` | 校验 assistant/tool 关联，插入 adapter system message，输出 role/content messages |
| `parse_tool_output(text, plan)` | 仅接受当前 nonce 的单个有界 JSON 信封；失败时清除内部标记并退化文本 |
| `redact_tool_results_for_log(messages)` | 保持消息结构，将 tool content 换成含关联 ID 的省略占位符 |
| `tool_completion_response(...)` | `openai_v1_chat_complete.py`；生成标准非流式 tool_calls 或文本投影及 usage |
| `stream_tool_chat_completion(...)` | 惰性缓冲上游文本，生成完整 tool_calls/content chunk 与 terminal chunk |
| `ChatCompletionRequest.tools/tool_choice/parallel_tool_calls` | `api/ai.py`；字段形状错误为 422，深层语义错误为 400 |

## Tasks

- [x] **Task backend-001: 建立 function 请求规范化与有界校验**

  **Files:** Create `services/protocol/openai_tool_calls.py`; create `tests/protocol/test_openai_tool_calls.py`.

  **Reasoning:** 先建立与 HTTP/上游无关的稳定计划对象，让语义错误在账号选择和生成前终止。后续历史适配与输出解析只消费已验证的 `ToolCallPlan`。

  **Depends on:** none.

  **Interfaces:** `FunctionToolSpec`, `ToolChoicePolicy`, `ToolCallPlan`, `ToolRequestError`, `build_tool_plan(body)`.

  **Step 1 — Add failing tests:** 覆盖嵌套/扁平 named choice、默认 `auto`/parallel、`none`、重复名、未知指定函数、混合类型、129 个工具、非法/超长名称、超过 256 KiB 定义。

  ```python
  def test_build_tool_plan_normalizes_named_choice():
      plan = build_tool_plan({
          "tools": [{"type": "function", "function": {
              "name": "get_weather", "description": "Get weather",
              "parameters": {"type": "object"}, "strict": True,
          }}],
          "tool_choice": {"type": "function", "function": {"name": "get_weather"}},
          "parallel_tool_calls": False,
      })
      assert plan.choice.mode == "function"
      assert plan.choice.function_name == "get_weather"
      assert plan.parallel is False and plan.nonce

  @pytest.mark.parametrize("body", invalid_tool_bodies())
  def test_build_tool_plan_rejects_semantic_errors(body):
      with pytest.raises(ToolRequestError):
          build_tool_plan(body)
  ```

  **Step 2 — Verify red:** Run `python -m pytest tests/protocol/test_openai_tool_calls.py -v`. Expected: collection fails because the module/interfaces do not exist.

  **Step 3 — Implement minimally:** 创建 frozen dataclasses、限制常量与异常。`build_tool_plan` 用 UTF-8 紧凑 JSON 计算定义大小，用 `[A-Za-z0-9_-]{1,64}` 校验唯一名称，接受四类 choice 与两种 named 形状，并生成安全随机 nonce。完全无工具时返回 `None`；混合或未知工具类型抛错。

  **Step 4 — Verify green:** Run `python -m pytest tests/protocol/test_openai_tool_calls.py -v`. Expected: 新增规范化和边界测试全部通过。

  **Step 5 — Commit:** 当前不执行。获明确授权后建议依次执行 `git add services/protocol/openai_tool_calls.py` 和 `git commit -m "feat: validate OpenAI function tool requests"`。

- [x] **Task backend-002: 适配工具历史、协议提示与日志脱敏**

  **Files:** Modify `services/protocol/openai_tool_calls.py` and `tests/protocol/test_openai_tool_calls.py`.

  **Reasoning:** Conversation 只理解 role/content，assistant `tool_calls` 与 `role: "tool"` 必须在 adapter 边界转换。nonce 和不可信结果标识防止普通模型文本伪造调用，并让日志保留关联而不持久化结果。

  **Depends on:** backend-001.

  **Interfaces:** `adapt_tool_messages(messages, plan)`, `redact_tool_results_for_log(messages)`.

  **Step 1 — Add failing tests:** 验证 adapter system message 位于调用方 system messages 后、首个非 system message 前；assistant 调用和 tool 结果按 ID/名称转换；孤立、重复或未知 ID 抛 `ToolRequestError`；日志副本不改变原对象且不保留结果。

  ```python
  def test_adapt_tool_messages_preserves_call_relationships():
      plan = build_tool_plan(tool_body())
      adapted = adapt_tool_messages([
          {"role": "system", "content": "caller policy"},
          {"role": "user", "content": "weather?"},
          {"role": "assistant", "content": None, "tool_calls": [{
              "id": "call_old", "type": "function",
              "function": {"name": "get_weather", "arguments": "{\"city\":\"Paris\"}"},
          }]},
          {"role": "tool", "tool_call_id": "call_old", "content": "sunny"},
      ], plan)
      assert adapted[1]["role"] == "system" and plan.nonce in adapted[1]["content"]
      assert '"type":"tool_call_history"' in adapted[3]["content"]
      assert '"type":"tool_result"' in adapted[4]["content"]

  def test_redact_tool_results_for_log_keeps_only_association():
      original = [{"role": "tool", "tool_call_id": "call_old", "content": "secret"}]
      redacted = redact_tool_results_for_log(original)
      assert "secret" not in str(redacted) and "call_old" in str(redacted)
      assert original[0]["content"] == "secret"
  ```

  **Step 2 — Verify red:** Run `python -m pytest tests/protocol/test_openai_tool_calls.py -k "adapt or redact or relationship" -v`. Expected: helper 缺失导致失败。

  **Step 3 — Implement minimally:** 构造含定义、choice、parallel 与精确 nonce 信封的 adapter system message。建立调用 ID→名称映射，把 assistant/tool 历史转换为紧凑 JSON role/content；arguments 必须是 JSON object 字符串，工具结果标记为 untrusted。日志 helper 深拷贝列表，只替换 tool content。

  **Step 4 — Verify green:** Run `python -m pytest tests/protocol/test_openai_tool_calls.py -v`. Expected: 规范化、历史关联、nonce 提示和脱敏测试全部通过。

  **Step 5 — Commit:** 当前不执行。获授权后建议依次执行 `git add services/protocol/openai_tool_calls.py` 和 `git commit -m "feat: adapt OpenAI tool history safely"`。

- [x] **Task backend-003: 实现 nonce JSON 信封的有界解析与文本退化**

  **Files:** Modify `services/protocol/openai_tool_calls.py` and `tests/protocol/test_openai_tool_calls.py`.

  **Reasoning:** 模型输出是不可信文本。解析器只能接受当前请求的精确信封，并保证任何截断、伪造或超限输出都不会泄露内部标记或升级为 5xx。

  **Depends on:** backend-001, backend-002.

  **Interfaces:** `ParsedToolCall`, `ToolOutput`, `parse_tool_output(text, plan)`.

  **Step 1 — Add failing tests:** 覆盖有效多调用、named choice、parallel=false、错误 nonce、多个/截断信封、无效 JSON、未知函数、非 object 参数和超过 256 KiB 信封。

  ```python
  def test_parse_tool_output_accepts_current_nonce_and_multiple_calls():
      plan = build_tool_plan(two_tool_body())
      raw = (f'<tool_calls nonce="{plan.nonce}">'
             '{"calls":[{"name":"get_weather","arguments":{"city":"Paris"}},'
             '{"name":"get_time","arguments":{"zone":"UTC"}}]}'
             '</tool_calls>')
      output = parse_tool_output(raw, plan)
      assert [call.name for call in output.calls] == ["get_weather", "get_time"]
      assert output.visible_text == ""

  @pytest.mark.parametrize("raw_factory", malformed_tool_outputs())
  def test_parse_tool_output_safely_falls_back(raw_factory):
      plan = build_tool_plan(tool_body())
      output = parse_tool_output(raw_factory(plan), plan)
      assert output.calls == () and output.fallback_reason
      assert "<tool_calls" not in output.visible_text and plan.nonce not in output.visible_text
  ```

  **Step 2 — Verify red:** Run `python -m pytest tests/protocol/test_openai_tool_calls.py -k "parse_tool_output" -v`. Expected: bounded parser 未实现导致失败。

  **Step 3 — Implement minimally:** 用精确 nonce 构造定界符并有界扫描，只允许一个完整信封；`json.loads` 前检查 UTF-8 大小。根必须为 object，`calls` 为非空 list，每项名称存在且 arguments 为 object。named 只接受目标函数，parallel=false 保留第一项。失败返回无 calls 的 `ToolOutput` 并删除可识别内部标签/信封内容，不使用宽松 XML 正则。

  **Step 4 — Verify green:** Run `python -m pytest tests/protocol/test_openai_tool_calls.py -v`. Expected: adapter 全部测试通过，公开文本不含 nonce/信封。

  **Step 5 — Commit:** 当前不执行。获授权后建议依次执行 `git add services/protocol/openai_tool_calls.py` 和 `git commit -m "feat: parse bounded OpenAI tool envelopes"`。

- [x] **Task backend-004: 接入非流式 Chat Completions 工具投影并绕过缓存**

  **Files:** Modify `services/protocol/openai_v1_chat_complete.py`; create `tests/protocol/test_openai_v1_chat_complete_tools.py`.

  **Reasoning:** endpoint-scoped 分支可复用账号切换、token 统计和日志元数据，同时保持普通文本、图片与 Web Search 不变。随机 nonce 请求不能进入响应缓存。

  **Depends on:** backend-001, backend-002, backend-003.

  **Interfaces:** `tool_completion_response(...)`, `openai_v1_chat_complete.handle(body)`.

  **Step 1 — Add failing tests:** monkeypatch `text_backend`/`collect_text` 返回当前 nonce 信封；断言标准响应、多调用、arguments JSON 字符串、finish reason、usage、账号元数据与缓存未调用。另测解析失败文本和普通请求仍使用缓存。

  ```python
  def test_handle_projects_non_stream_tool_calls_without_cache(monkeypatch):
      monkeypatch.setattr(chat_module.chat_completion_cache,
                          "get_or_compute_response", fail_if_called)
      monkeypatch.setattr(chat_module, "text_backend",
                          lambda: FakeBackend("account@example.com"))
      monkeypatch.setattr(chat_module, "collect_text", envelope_for_request_plan)
      response = chat_module.handle(tool_request(stream=False))
      choice = response["choices"][0]
      assert choice["message"]["content"] is None
      assert choice["message"]["tool_calls"][0]["type"] == "function"
      assert json.loads(choice["message"]["tool_calls"][0]["function"]["arguments"])
      assert choice["finish_reason"] == "tool_calls"
      assert response["_account_email"] == "account@example.com"
  ```

  **Step 2 — Verify red:** Run `python -m pytest tests/protocol/test_openai_v1_chat_complete_tools.py -k "non_stream or cache" -v`. Expected: function 请求进入缓存文本路径或只返回 assistant content。

  **Step 3 — Implement minimally:** 在图片判断后构造 plan。无 plan 时保留当前顺序；有 plan 时适配 messages、`collect_text`、解析并投影。每个公开调用生成 `call_<uuid>`，arguments 为紧凑 JSON；tool output finish reason 为 `tool_calls`，fallback 为 `stop`。usage 输入按适配 messages、输出按原始上游文本。`ToolRequestError` 映射 400；function 分支不碰 cache。

  **Step 4 — Verify green:** Run `python -m pytest tests/protocol/test_openai_v1_chat_complete_tools.py -k "non_stream or cache or fallback" -v` then `python -m pytest tests/protocol/test_openai_tool_calls.py -v`. Expected: 非流式契约、adapter 和普通缓存回归通过。

  **Step 5 — Commit:** 当前不执行。获授权后建议依次执行 `git add services/protocol/openai_v1_chat_complete.py services/protocol/openai_tool_calls.py` 和 `git commit -m "feat: return OpenAI chat tool calls"`。

- [x] **Task backend-005: 接入整轮缓冲的流式工具投影**

  **Files:** Modify `services/protocol/openai_v1_chat_complete.py` and `tests/protocol/test_openai_v1_chat_complete_tools.py`.

  **Reasoning:** 上游只提供文本 delta，无法在完整信封前判断 content/tool_calls。惰性 generator 在首个公开 chunk 前消费整轮，满足标准 SSE，也保留 `LoggedCall.run` 对首次迭代异常的处理。

  **Depends on:** backend-004.

  **Interfaces:** `stream_tool_chat_completion(...)`, `completion_chunk(...)`.

  **Step 1 — Add failing tests:** monkeypatch `stream_text_deltas` 产生拆分信封，断言迭代前不消费；迭代后只产生完整 tool_calls chunk 和空 delta terminal。另测文本 fallback、terminal stop、公开 chunk 无 nonce。

  ```python
  def test_stream_tool_request_buffers_then_emits_standard_chunks(monkeypatch):
      consumed = []
      monkeypatch.setattr(chat_module, "stream_text_deltas", split_envelope_stream(consumed))
      events = chat_module.handle(tool_request(stream=True))
      assert consumed == []
      chunks = list(events)
      first, terminal = chunks[0]["choices"][0], chunks[-1]["choices"][0]
      assert first["delta"]["role"] == "assistant"
      assert first["delta"]["tool_calls"][0]["index"] == 0
      assert terminal == {"index": 0, "delta": {}, "finish_reason": "tool_calls"}
      assert "nonce" not in json.dumps(chunks)
  ```

  **Step 2 — Verify red:** Run `python -m pytest tests/protocol/test_openai_v1_chat_complete_tools.py -k "stream" -v`. Expected: stream 仍把原始信封作为 content delta 并以 stop 结束。

  **Step 3 — Implement minimally:** 新增惰性 generator，内部经既有 `stream_text_deltas` 完整收集后解析。有效调用一次发送完整 `delta.tool_calls`（index/id/type/name/arguments），随后空 delta + `tool_calls`；fallback 一次发送 role/content，随后空 delta + `stop`。复用同一 completion ID/created 与账号日志元数据；function stream 绕过 cache。

  **Step 4 — Verify green:** Run `python -m pytest tests/protocol/test_openai_v1_chat_complete_tools.py -v`. Expected: 非流式、流式、缓存绕过、退化和普通聊天回归通过。

  **Step 5 — Commit:** 当前不执行。获授权后建议依次执行 `git add services/protocol/openai_v1_chat_complete.py` 和 `git commit -m "feat: stream buffered OpenAI tool calls"`。

- [x] **Task backend-006: 固化 API 字段、400/422 边界和 Call Record 脱敏**

  **Files:** Modify `api/ai.py`; create `tests/api/test_ai_tool_calls.py`; modify `tests/protocol/test_openai_v1_chat_complete_tools.py`.

  **Reasoning:** FastAPI 层公开字段形状并产生稳定 422，深层工具语义由 adapter 产生 400。内容检查和持久化日志使用不同的瞬时预览，兼顾安全审查与最小化持久化。

  **Depends on:** backend-002, backend-004, backend-005.

  **Interfaces:** `ChatCompletionRequest.tools`, `.tool_choice`, `.parallel_tool_calls`, `create_chat_completion(...)`.

  **Step 1 — Add failing tests:** 错误顶层字段形状返回 422；未知 named function、混合类型、错误历史关联返回 400 且不访问 backend；合法请求完整结果交给内容检查，但 LoggedCall 只含 ID/占位符。

  ```python
  def test_chat_route_filters_full_tool_result_but_logs_redacted_copy(monkeypatch, client):
      captured = install_chat_route_spies(monkeypatch)
      response = client.post("/v1/chat/completions", headers=auth_header(), json={
          "model": "auto",
          "messages": [{"role": "tool", "tool_call_id": "call_old",
                        "content": "private-result"}],
          "tools": [function_tool("get_weather")],
      })
      assert response.status_code != 422
      assert "private-result" in captured.filter_text
      assert "private-result" not in captured.logged_request_text
      assert "call_old" in captured.logged_request_text

  def test_tool_semantic_error_maps_to_400_before_backend(monkeypatch):
      monkeypatch.setattr(chat_module, "text_backend", fail_if_called)
      with pytest.raises(HTTPException) as exc_info:
          chat_module.handle(request_with_unknown_named_function())
      assert exc_info.value.status_code == 400
  ```

  **Step 2 — Verify red:** Run `python -m pytest tests/api/test_ai_tool_calls.py tests/protocol/test_openai_v1_chat_complete_tools.py -k "422 or 400 or redact or semantic" -v`. Expected: 显式字段、日志分层或 400 映射测试失败。

  **Step 3 — Implement minimally:** 给请求模型增加 `tools: list[dict[str, object]] | None`、`tool_choice: str | dict[str, object] | None`、`parallel_tool_calls: bool | None`。原 messages 生成 filter preview，`redact_tool_results_for_log` 副本生成 LoggedCall preview。保留 payload、request shape、trace、鉴权和图片分类。只把 `ToolRequestError` 转 400，不捕获上游异常。

  **Step 4 — Verify green:** Run `python -m pytest tests/api/test_ai_tool_calls.py -v` then `python -m pytest tests/protocol/test_openai_tool_calls.py tests/protocol/test_openai_v1_chat_complete_tools.py -v`. Expected: 422/400、内容检查、日志脱敏及协议测试通过。

  **Step 5 — Commit:** 当前不执行。获授权后建议依次执行 `git add api/ai.py services/protocol/openai_v1_chat_complete.py services/protocol/openai_tool_calls.py` 和 `git commit -m "feat: expose safe chat tool call contracts"`。

- [x] **Task backend-007: 补齐双语文档、变更记录与全量回归验收**

  **Files:** Create `tests/docs/test_function_calling_docs.py`; modify `README.md`, `README_EN.md`, `CHANGELOG.md`.

  **Reasoning:** 兼容 API 的公开契约变化需要完整两轮闭环、流式形状和 best-effort 限制。文档测试把三份说明与实现验收绑定，随后全量回归确认范围外协议不变。

  **Depends on:** backend-004, backend-005, backend-006.

  **Interfaces:** public Function Calling documentation and release-note contract.

  **Step 1 — Add failing tests:** 两份 README 必须含 `tools`、assistant `tool_calls`、`role: "tool"`、`tool_choice`、`parallel_tool_calls`、流式缓冲和调用方执行；CHANGELOG 必须含 endpoint、Function Calling 与 best-effort。

  ```python
  @pytest.mark.parametrize("path", ["README.md", "README_EN.md"])
  def test_function_calling_round_trip_is_documented(path):
      text = Path(path).read_text(encoding="utf-8")
      for phrase in ("tool_calls", 'role: "tool"', "tool_choice", "parallel_tool_calls"):
          assert phrase in text

  def test_function_calling_release_note_is_present():
      text = Path("CHANGELOG.md").read_text(encoding="utf-8")
      assert "/v1/chat/completions" in text
      assert "Function Calling" in text and "best-effort" in text
  ```

  **Step 2 — Verify red:** Run `python -m pytest tests/docs/test_function_calling_docs.py -v`. Expected: 现有文档缺少完整闭环和兼容边界。

  **Step 3 — Implement minimally:** 两份 README 加入等价的请求、tool_calls 响应、调用方执行、tool result 回传及最终回答示例；说明 function-only、128/64/256 KiB 限制、stream 缓冲、解析失败退化，以及 required/named/strict 的 best-effort。CHANGELOG 按现有格式记录，不能宣称等价于原生 OpenAI 模型约束。

  **Step 4 — Verify green and regression:** 依次运行：

  ```text
  python -m pytest tests/docs/test_function_calling_docs.py -v
  python -m pytest tests/protocol/test_openai_tool_calls.py tests/protocol/test_openai_v1_chat_complete_tools.py tests/api/test_ai_tool_calls.py -v
  python -m pytest
  git status --short
  git diff --check
  ```

  Expected: 文档、新增测试和全量回归通过；diff 无空白错误；无意外修改 conversation、cache、Anthropic、数据库、配置或前端。

  **Step 5 — Commit:** 当前不执行。获授权后建议依次执行 `git add README.md README_EN.md CHANGELOG.md` 和 `git commit -m "docs: describe chat function calling"`。

## Coverage Matrix

| Requirement / decision | Tasks | Evidence |
| --- | --- | --- |
| P-001：仅 Chat Completions | backend-004, 005, 006, 007 | handler/API tests、全量回归、范围审计 |
| P-002：调用方执行并回传 tool | backend-002, 004, 006, 007 | 历史关联、两轮契约、双语示例 |
| P-003：function/choice/multi-call | backend-001, 003, 004 | 规范化、choice 与 multi-call tests |
| P-004：stream 整轮缓冲 | backend-005, 007 | 惰性消费、完整 delta、terminal tests |
| P-005：解析失败退化文本 | backend-003, 004, 005 | malformed/oversize/unknown fallback tests |
| P-006 / D-006：small backend form | backend-001–007 | 单一 `plan-backend.md`，无 frontend/migration/overview |
| D-001：专用 adapter + nonce | backend-001, 002, 003 | adapter tests 与 nonce 不外泄断言 |
| D-002：绕过缓存 | backend-004, 005 | stream/non-stream cache spy tests |
| D-003：128/64/256 KiB | backend-001, 003 | 边界与超限 tests |
| D-004：无 JSON Schema 依赖 | backend-001, 003 | structural object tests、依赖 diff 审计 |
| D-005：endpoint-scoped 方案 A | backend-001–006 | 新 adapter + Chat 局部集成，不改 conversation |
| D-007：过滤完整内容、日志脱敏 | backend-002, 006 | filter spy 与 LoggedCall 断言 |
| 普通聊天、图片、Web Search、Anthropic 回归 | backend-004, 005, 007 | targeted tests 与完整 pytest |

## Execution Order

`backend-001 → backend-002 → backend-003 → backend-004 → backend-005 → backend-006 → backend-007`

每个任务完成后先运行 targeted tests；只有新增测试、相关回归、全量 pytest、`git diff --check` 和范围审计全部通过，才可报告实现完成。
