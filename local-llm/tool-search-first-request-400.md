# Claude Code with tool search gets a 400 on the first request of every session against vLLM

**Symptom.** With `ENABLE_TOOL_SEARCH=true` and `ANTHROPIC_BASE_URL` pointed at vLLM, the first request of every session is rejected. Nothing shows up on the command line, because the client swallows the error and resends. The proxy log is where it surfaces:

```
[WARN] non-200 response, recording partial metrics: status=400, path=/v1/messages
[INFO] Request 192.0.2.33 "POST /v1/messages HTTP/1.1" 400 20214 "claude-cli/2.1.273 (external, sdk-cli)" 19.705389ms
```

vLLM's own log shows the retry landing on the same connection:

```
(APIServer pid=1) INFO:     172.17.0.1:42798 - "POST /v1/messages?beta=true HTTP/1.1" 400 Bad Request
(APIServer pid=1) INFO:     172.17.0.1:42798 - "POST /v1/messages?beta=true HTTP/1.1" 200 OK
```

Over one afternoon: 29 × 200, 3 × 400. Each 400 was the first request of a session with tool search on; sessions with it off produced none. All three bodies were 20214 bytes.

**Cause.** The first request carries `tool_addition` content blocks on a `role: system` message, and the request header lists the beta `mid-conversation-tool-changes-2026-07-01`:

```
tools: 15 entries, every one with an input_schema (4 have defer_loading: true)

messages[0]  role=user    [text, text, text]
messages[1]  role=system  [text, tool_addition, tool_addition, tool_addition]
```

```json
{"type": "tool_addition", "tool": {"type": "tool_reference", "name": "mcp__..."}}
```

vLLM's `AnthropicContentBlock.type` accepts seven values and `tool_addition` is not one of them, so the whole request fails validation:

```
{'type': 'literal_error', 'loc': ('body', 'messages', 1, 'content', 'list[AnthropicContentBlock]', 1, 'type'),
 'msg': "Input should be 'text', 'image', 'tool_use', 'tool_result', 'tool_reference', 'thinking' or 'redacted_thinking'",
 'input': 'tool_addition', ...}
```

The ~20 KB body is pydantic echoing the message content back in the error. Same on `main` (`75c71390d5b3`, 2026-09-17): no `tool_addition` or `tool_removal` anywhere in the tree.

**What the retry changes.** `tools` and `system` are identical between the rejected request and the one that gets the 200. The system message is rewritten without the `tool_addition` blocks (the three tools are listed in its text instead), and `mid-conversation-tool-changes-2026-07-01` is dropped from `anthropic-beta`. So the client already has a fallback for an endpoint without this beta; it just discovers the need for it with a 400, once per session, every session. Same on Claude Code 2.1.273 and 2.1.274.

**Wrong first diagnosis.** I could not get the error body from the client side (Claude Code does not log it, and it is in neither stderr nor the JSON envelope), so I probed the endpoint with hand-written payloads and hit a different validation error, `AnthropicTool.input_schema` being required, on a name-only tool. I filed both issues on that basis. Someone with the same hardware tested vLLM PR [#46797](https://github.com/vllm-project/vllm/pull/46797) (which makes `input_schema` optional) against my described payload and it passed; I installed it and the real session still got the 400. Only then did I put a logging proxy between the client and the endpoint and capture the actual request. Validated offline, the captured request is rejected identically by the stock and patched builds, and passes on both once the three `tool_addition` blocks are removed. Deferred tools are sent with their full definition on every request; the tool search docs say so, and the capture confirms it.

**Fix.** None yet. `ENABLE_TOOL_SEARCH=false` removes the rejected round-trip and restores the full prompt (22,204 tokens at session start versus 10,708 on this setup). Leaving it on costs one rejected round-trip and a ~20 KB error body per session. Server side would need `/v1/messages` to accept `tool_addition` (and presumably `tool_removal`); client side, the beta could be gated on the endpoint instead of discovered through a 400. Filed and since corrected on both sides: [vllm-project/vllm#57324](https://github.com/vllm-project/vllm/issues/57324), [anthropics/claude-code#95087](https://github.com/anthropics/claude-code/issues/95087).

**Context.** NVIDIA DGX Spark (GB10), 128 GB unified memory. vLLM serving Qwen3.8-Flash-Next (NVFP4), `--max-model-len 262144 --enable-prefix-caching --enable-chunked-prefill`, speculative decoding MTP=2, llama-swap in front. Client is Claude Code 2.1.273 / 2.1.274 on macOS.

中文原文：[Claude Code 开着 Tool Search 接 vLLM，每个会话的第一个请求都被 400 拒掉](tool-search-first-request-400.zh.md)
