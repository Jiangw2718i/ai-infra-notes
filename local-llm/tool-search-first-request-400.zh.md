# Claude Code 开着 Tool Search 接 vLLM，每个会话的第一个请求都被 400 拒掉

**TL;DR**  Claude Code 开着 Tool Search 时，会话第一个请求的消息里带着 `tool_addition` 内容块，属于 Anthropic 的一个 beta 功能（`mid-conversation-tool-changes-2026-07-01`）。vLLM 的 Anthropic 兼容端点不认识这种块，整个请求被拒，错误正文两万字节左右。Claude Code 静默吞掉这个400错误、去掉这些块重发，cli看不出任何异常。

我一开始以为是工具缺 `input_schema`，还给vllm官方库提了 issue。后来被提醒，装了别人的补丁发现 400 还在，但是原因不对，抓了真实请求才知道。之前调查不完全。

调查也推翻了我之前的一个判断。读 GitHub issue 的时候我以为开着 Tool Search 在本地推理上会更慢，因为claude code中途动态修改 `tools[]` 会废掉整个 KV cache，造成缓存无法命中。第一次结果和想象的相反，Tool Search on比off少算 5559 个 token。但补了第二次测试后结果又反过来了，下面有过程，没有结果。

硬件：NVIDIA DGX Spark（GB10，128 GB 统一内存）。vLLM 跑 Qwen3.8-Flash-Next（NVFP4，旁路层 blockwise fp8），`--max-model-len 262144 --enable-prefix-caching --enable-chunked-prefill`，投机解码 MTP=2，前面挂 llama-swap。客户端是 Mac mini 上的 Claude Code 2.1.273，用 `ANTHROPIC_BASE_URL` 指到本地，挂着 chrome-devtools MCP。

---

本来只是想评估本地 harness。测了一下 `ENABLE_TOOL_SEARCH` 对提示词体积的影响，方法很土：读一次 vLLM 的 `prompt_tokens_total`，跑一个全新的 `claude -p` 会话，再读一次。

默认 22204，Tool Search on之后 10708。省掉一半。

按我自己 bench 出来的 prefill 速率 1348.98 tok/s 换算，首字延迟从 16.5 秒降到 8 秒。本地推理和云端不同，本地没有云端那么多算力，所以速度快感觉明显。

然后读到 claude-code #81967。有人用 mitmproxy 录了 1821 个请求，发现 Tool Search 在会话中途加载一个工具 schema 时，`tools[]` 会从 32398 字节涨到 49133 字节。而 `tools[]` 排在缓存前缀的最前面。

KV cache 是前缀缓存，注意力的计算结构里，第 0 层每个 token 的 K/V 由它自己决定，但是从第 1 层起，token i 的输入是前一层对 token 1..i 做完注意力的结果，所以深层每个 token 的 K/V 都是它和它之前所有内容。token 3 变了，token 4 的 K/V 就是错的，哪怕 token 4 没动。vLLM 的实现是把 token 按块切、对每块算链式哈希，要复用一块必须它前面所有块都逐字节相同。

所以在提示词最前面改东西，缓存几乎都没办法命中。

如果 #81967 是对的，那么开着 Tool Search 在本地有可能缩短冷启动时间，但是可能会话开头省一万一千个 token，中途每加载一个工具说明书就赔一次全量重算。

我要测一下一整个真实任务跑下来的累计 prefill 和缓存命中率。

vLLM 的 metrics 端点正好有两个计数器。

```
vllm:prompt_tokens_total          请求里的提示词 token 总量
vllm:prompt_tokens_cached_total   其中从缓存里捞出来的部分
```

两者相减就是真正算过的量。命中率用 `prefix_cache_hits_total / prefix_cache_queries_total`。

任务设计成必须用到五种不同的工具：找出所有提到 `RETRY_LIMIT` 的文件、读一个 markdown、列出所有 .py、对每个 .py 跑 `wc -l`、把结果写进 summary.md。这样 Grep、Read、Glob、Bash、Write 都会被用到，开着 Tool Search 时每个都要单独加载说明书。

第一对结果出来，非常"完美"：

```
                         ACTUALLY_PREFILLED    hit_rate   wall
Tool Search off            81269                 72%      106s
Tool Search on             33473                 40%       45s
```

结果来看，开Tool Search节省 2.4 倍。但其实两次跑的工作量根本不一样，Tool Search off那次生成了 1845 个 token，开着的只有 962，提示词总量 290869 对 55873，差五倍。估计是轮次数不同造成的，Tool Search off的那次可能因为失败或者其他什么原因多跑了几轮。

轮次数才是这里的主导变量，缓存是次要的。这两个数字没什么可比性。

给脚本加上 `--output-format json`，从返回的信封里取 `num_turns`，同时记录 vLLM 的 `request_success_total` 增量，也就是这次会话实际打了多少个请求。重跑一次，这次对齐了：

```
                          turns  requests  每请求提示词   hit_rate    计算了的  wall
Tool Search off             6        5        58361        94%      16605    52s
Tool Search on              6        5        11169        80%      11046    34s
```

轮数一样，请求次数一样，任务都完成了，两边最后的回答内容也一致。

缓存命中减少，命中率从 94% 掉到 80%。Tool Search off的时候每个请求要 58361 个 token，Tool Search on只需要 11169，差 5倍多，Tool Search off的状态下，即使 缓存94%都命中，实际计算的token还是大于Tool Search on的状态，这样看来，Tool Search on更节省算力。

还有个细节。这一轮的关的那次是后跑的，缓存已经热起来了，本来应该有优势的。

为了写文章，中途隔了几小时用同一套脚本又补测了一下，这次，Tool Search off的命中率涨到 96%，但是实际参与计算的只有 9496，比Tool Search on的 11179 还少，上面的结论就不成立了。

两次Tool Search off，提示词总量几乎一样，291805 和 291096，差异都在缓存命中量，275200 涨到 281600。命中率 94% 到 96%，虽然只有2%，但是在 29 万的基数上就是七千多个 token。

Tool Search off在多次测试后，缓存热起来了，跑的测试越多参与计算的token就会越少。

测试途中，翻 llama-swap 日志想确认端点健康状态，看到下面两行。

```
9月 17 14:54:49 node-a llama-swap[219117]: [WARN] non-200 response, recording partial metrics: status=400, path=/v1/messages
9月 17 14:54:49 node-a llama-swap[219117]: [INFO] Request 192.0.2.33 "POST /v1/messages HTTP/1.1" 400 20214 "claude-cli/2.1.273 (external, sdk-cli)" 19.705389ms
```

发的请求19.7 毫秒就被拒了，没进推理。

看了一下vLLM 容器日志，

```
(APIServer pid=1) INFO:     172.17.0.1:42798 - "POST /v1/messages?beta=true HTTP/1.1" 400 Bad Request
(APIServer pid=1) INFO:     172.17.0.1:42798 - "POST /v1/messages?beta=true HTTP/1.1" 200 OK
```

400 Bad Request之后同样的API返回200。claude code客户端遇到错误后，静默重试，之后成功。命令行上什么都看不出来，只有去log中看。

log显示测试当天三次400，翻看日志发现三次 400 的响应体大小完全一样，都是 20214 字节。

对一下log时间，这三次都是Tool Search on的状态下跑的，而且都是第一个请求。

claude code 不把 HTTP 错误正文写进会话记录，stderr 和 JSON 里也搜不到，所以拿不到原始错误。于是自己发请求复现。

试了四个形状。命中了一个：

```
{"error":{"message":"1 validation error:\n  {'type': 'missing', 'loc': ('body', 'tools', 0, 'input_schema'), 'msg': 'Field required', 'input': {'type': 'tool_search_tool_20250101', 'name': 'tool_search'}}","type":"Bad Request","param":"body.tools.0.input_schema","code":400}}
```

`input_schema` 是必填的。而 Tool Search 的全部意义就是先只发工具名、不发说明书，也就是没有 `input_schema`。

感觉发现了新大陆，迫不及待的开始给vllm和claude code的库发issue。

 [vllm-project/vllm#57324](https://github.com/vllm-project/vllm/issues/57324)，[anthropics/claude-code#95087](https://github.com/anthropics/claude-code/issues/95087)。vLLM 之前有人提交过PR #46797，也是关于 `input_schema` 这里的，看了一下，虽然和我的问题有些区别，但是PR #46797应该可以解决我的问题。

issue 发出去几个小时就有人回了。其中一位手上也有一台 DGX Spark，拿我描述的请求形状测了 #46797，说修好了，问我要不要在自己机器上端到端确认一下。

我把补丁装上，重启，然后开着 Tool Search 跑了一次真实会话。400 还在。

实在搞不清楚了，于是在claude code和LLM API之间架了个代理，抓个包，看看请求和错误到底是什么。

然后发现了真正的错误，还真就不是input_schema的问题。

```
{'type': 'literal_error', 'loc': ('body', 'messages', 1, 'content', 'list[AnthropicContentBlock]', 1, 'type'),
 'msg': "Input should be 'text', 'image', 'tool_use', 'tool_result', 'tool_reference', 'thinking' or 'redacted_thinking'",
 'input': 'tool_addition', ...}
```

vLLM 根本就无法识别 `tool_addition`。

保险起见，同一个请求拿去测原版和打了补丁的 vLLM，报的错一样。删掉 `tool_addition`之后，两边都能通过。Tool Search off的情况下，请求里没有 tool_addition，当然不报错。

还有个疑问，为什么 claude code 重发的时候会自己把 `tool_addition` 去掉。claude code 没开源，不过程序文件里的 JS 没加密，能搜到相关的代码片段，里面有一段专门处理这个 400 的逻辑，先给错误分类，再回退，

```
[late-tool-additions] tool_addition rejected (${n}) — falling back to the ToolSearch
announcement (or to inline tools where there is no ToolSearch),
sticky-rejecting the beta until /clear or /compact
```

分类的依据是错误正文里有没有 `tool_addition` 这个词，vLLM 的报错正好把它回显了出来。错误命中之后，不再用 tool_addition 这种特殊块，改成直接在提示词里用普通文字写上那几个工具的名字 ，去掉 beta 头，然后记住这个端点不接受它。记住的方式是放在memory里，`/clear` 或 `/compact` 就清掉，进程退出也没了。所以 `claude -p` 每次都是新进程，每次都要遇到一次400错误。

根据新的调查结果修改了两个issue。

调查结束。
