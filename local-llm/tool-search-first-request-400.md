# Claude Code with tool search gets a 400 on the first request of every session against vLLM

**TL;DR** — With `ENABLE_TOOL_SEARCH=true` behind a custom `ANTHROPIC_BASE_URL`, the first request of every Claude Code session carries `tool_addition` content blocks from an Anthropic beta. vLLM's `/v1/messages` does not know that block type and rejects the whole request. The client strips the blocks, drops the beta header, retries, and succeeds, so nothing shows on the command line. Cost: one rejected round trip and a ~20 KB error body per session. I first blamed `input_schema` and filed two issues on that; the correction is below.

Hardware: NVIDIA DGX Spark (GB10, 128 GB unified memory). vLLM `0.1.dev20073+g8e685d198` serving Qwen3.8-Flash-Next (NVFP4), `--max-model-len 262144 --enable-prefix-caching --enable-chunked-prefill`, speculative decoding MTP=2, llama-swap in front. Client: Claude Code 2.1.273 and 2.1.274 on macOS.

## Symptom

The proxy log is where it surfaces:

```
[WARN] non-200 response, recording partial metrics: status=400, path=/v1/messages
[INFO] Request 192.0.2.33 "POST /v1/messages HTTP/1.1" 400 20214 "claude-cli/2.1.273 (external, sdk-cli)" 19.705389ms
```

vLLM's own log shows the retry landing on the same connection:

```
(APIServer pid=1) INFO:     172.17.0.1:42798 - "POST /v1/messages?beta=true HTTP/1.1" 400 Bad Request
(APIServer pid=1) INFO:     172.17.0.1:42798 - "POST /v1/messages?beta=true HTTP/1.1" 200 OK
```

Over one afternoon: 29 × 200, 3 × 400. Each 400 was the first request of a session with tool search on; sessions with it off produced none. All three bodies were 20214 bytes. `claude -p --output-format json` reports `is_error: false` and the transcript records nothing.

## Cause

The rejected request, captured with a logging proxy between the client and the endpoint:

```
tools: 15 entries, every one with an input_schema (4 have defer_loading: true)

messages[0]  role=user    [text, text, text]
messages[1]  role=system  [text, tool_addition, tool_addition, tool_addition]
```

```json
{"type": "tool_addition", "tool": {"type": "tool_reference", "name": "mcp__..."}}
```

The request header lists `mid-conversation-tool-changes-2026-07-01` in `anthropic-beta`. The error body:

```
4 validation errors:
  {'type': 'string_type', 'loc': ('body', 'messages', 1, 'content', 'str'), 'msg': 'Input should be a valid string', ...}
  {'type': 'literal_error', 'loc': ('body', 'messages', 1, 'content', 'list[AnthropicContentBlock]', 1, 'type'),
   'msg': "Input should be 'text', 'image', 'tool_use', 'tool_result', 'tool_reference', 'thinking' or 'redacted_thinking'",
   'input': 'tool_addition', ...}
  (same for blocks 2 and 3)
```

`AnthropicContentBlock.type` is a seven-value literal at `vllm/entrypoints/anthropic/protocol.py:39-47` on `main` (SHA `75c71390d5b3`, 2026-09-17); `tool_addition` and `tool_removal` appear nowhere in the tree. The ~20 KB body is pydantic echoing the message content back.

Checks: the captured request is rejected identically by the build above and by the same build with PR #46797 applied; it validates on both once the three `tool_addition` blocks are removed; Claude Code 2.1.273 and 2.1.274 send the same shape; with `ENABLE_TOOL_SEARCH=false` there are no `tool_addition` blocks and no 400.

## What the retry changes

`tools` and `system` are identical between the rejected request and the retry. The system message is rewritten without the `tool_addition` blocks (the three tools are named in its text instead), and `mid-conversation-tool-changes-2026-07-01` is dropped from `anthropic-beta`.

This is a deliberate fallback. Claude Code is not open source, but the bundled JS in the 2.1.274 binary is not obfuscated. It classifies a 400 whose body mentions `tool_addition`, then logs:

```
[late-tool-additions] tool_addition rejected (${n}) — falling back to the ToolSearch announcement (or to inline tools where there is no ToolSearch), sticky-rejecting the beta until /clear or /compact
```

The rejection is remembered in process memory only, so every new process (every `claude -p` run) pays the round trip again.

## Fix

None yet. `ENABLE_TOOL_SEARCH=false` removes the rejected round trip and restores the full prompt (22,204 tokens at session start versus 10,708 on this setup). Leaving it on costs one rejected round trip and a ~20 KB error body per session.

Server side would need `/v1/messages` to accept `tool_addition` (and presumably `tool_removal`); client side, the beta could be gated on the endpoint, or the rejection persisted, instead of rediscovered through a 400. Filed: [vllm-project/vllm#57324](https://github.com/vllm-project/vllm/issues/57324), [anthropics/claude-code#95087](https://github.com/anthropics/claude-code/issues/95087).

## The diagnosis that was wrong

Both issues were first filed as "deferred tools are sent without `input_schema`", with a curl reproduction hitting `AnthropicTool.input_schema` (required at `protocol.py:79` on the same SHA) with a name-only tool. That error is real, and PR [#46797](https://github.com/vllm-project/vllm/pull/46797) (for [#46790](https://github.com/vllm-project/vllm/issues/46790), server tools) fixes it, but no request from Claude Code triggers it: every tool in the captured request carries an `input_schema`, deferred ones included, which is also what the tool search docs say to send. The curl payload was hand-written; I had not captured the real request because Claude Code does not log the error body. Someone with the same hardware tested #46797 against the described payload and it passed; installed here, the real session still got the 400. The capture above is what disproved the first version. Both issues carry a correction note and a new title.

中文原文：[Claude Code 开着 Tool Search 接 vLLM，每个会话的第一个请求都被 400 拒掉](tool-search-first-request-400.zh.md)
