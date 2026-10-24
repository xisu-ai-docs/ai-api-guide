# 工具 / SDK × 端点兼容表

最后核对：2026-10-24。事实来源：犀速AI官方文档「端点与鉴权」「Anthropic 原生端点」「OpenAI Responses 端点」「用现成的 SDK 与客户端接入」。表里写“未验证”的格子表示我们还没有自己跑通，不代表不支持。

## 先记住两条

1. **OpenAI 兼容那一套带 /v1**：`https://api.xisu.ai/v1`（chat/completions、responses、embeddings、models 都在这下面）。
2. **Anthropic 原生那一套不带 /v1**：`https://api.xisu.ai`。Anthropic SDK 和 Claude Code 会自己往后拼 `/v1/messages`，多写一个 /v1 就会请求到 `…/v1/v1/messages` 返回 404。

两套协议共用同一把密钥和同一个余额，只是地址形状不同。

## 兼容表

| 工具/SDK | 协议与端点 | Base URL 怎么填 | 鉴权 | 流式注意事项 | 验证情况 |
|---|---|---|---|---|---|
| OpenAI Python SDK | OpenAI 兼容：/v1/chat/completions、/v1/responses、/v1/embeddings | 带 /v1：https://api.xisu.ai/v1 | Authorization: Bearer <密钥> | 流末尾多一个 choices 为空数组的 usage 块；官方 SDK 可正常处理 | 官网文档列为已跑通（2026-10-22）；本仓库自测：未验证 |
| OpenAI Node SDK | 同上 | 带 /v1：https://api.xisu.ai/v1 | Authorization: Bearer <密钥> | 同上 | 官网文档列为已跑通（2026-10-22）；本仓库自测：未验证 |
| Anthropic SDK | Anthropic 原生：/v1/messages、/v1/messages/count_tokens | 不带 /v1：https://api.xisu.ai（SDK 自己拼 /v1/messages） | x-api-key: <密钥>（也接受 Bearer，两者都给时以 x-api-key 为准） | 原生事件流（message_start … message_stop），没有 data: [DONE] | 官网文档说明（2026-10-04）；本仓库自测：未验证 |
| Claude Code | Anthropic 原生：/v1/messages | ANTHROPIC_BASE_URL=https://api.xisu.ai（不带 /v1） | ANTHROPIC_AUTH_TOKEN=<密钥> | 同 Anthropic SDK | 官网文档说明（2026-10-04）；本仓库自测：未验证 |
| Codex CLI | OpenAI Responses：/v1/responses | 带 /v1：https://api.xisu.ai/v1 | Authorization: Bearer <密钥> | Responses 事件流（response.created … response.completed），没有 data: [DONE]；用量只在 response.completed 里 | 官网文档说明（2026-10-08）；本仓库自测：未验证 |
| Cherry Studio | OpenAI 兼容 | 整串填 https://api.xisu.ai/v1；若客户端自动再补 /v1，删掉你填的 /v1 | API Key 输入框 | 未验证 | 未验证（按官网“任何能自定义 base_url 的客户端”通用规则） |
| Cursor | OpenAI 兼容（自定义 OpenAI Base URL） | 未验证 | 未验证 | 未验证 | 未验证 |
| Cline | OpenAI 兼容或 Anthropic 原生（取决于所选 Provider） | 未验证 | 未验证 | 未验证 | 未验证 |
| LangChain | OpenAI 兼容（ChatOpenAI 的 base_url） | 未验证 | 未验证 | 未验证 | 未验证 |
| LlamaIndex | OpenAI 兼容 | 未验证 | 未验证 | 未验证 | 未验证 |
| Vercel AI SDK | OpenAI 兼容（自定义 provider baseURL） | 未验证 | 未验证 | 未验证 | 未验证 |
| Dify | OpenAI-API-compatible 模型供应商 | 未验证 | 未验证 | 未验证 | 未验证 |

机器可读版本见同目录的 `compatibility-matrix.csv`。

## 什么时候走 Anthropic 原生端点

需要 `cache_control`（提示缓存）、`thinking`、document 块或 `tool_result` 原样到达上游时，走 `/v1/messages`。走 `/v1/chat/completions` 时，顶层的 `thinking` 一类参数会被直接 400 拒掉；嵌在 content 里的 `cache_control` 会被静默丢掉，请求照常成功，但提示缓存不会命中。

## Responses 端点的三个限制

- 网关不保存会话：`previous_response_id`、`background`、`store: true` 会返回 400，把完整对话放进 `input` 即可。
- web_search、file_search、code_interpreter、mcp 这类服务端工具按次另外计费，没配价的模型返回 400 `server_tool_not_priced`；function 工具照常放行。
- 模型是子集：404 `model_not_on_endpoint` 是永久的（换模型或改走 chat/completions），503 `no_available_channel` 是暂时的（按 Retry-After 重试）。

## 更正与补充

发现某个工具的实际行为和表里不一致，请开 issue 并附上工具版本、请求地址（隐去密钥）和 `request_id`。
