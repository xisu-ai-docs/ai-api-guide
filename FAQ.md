# FAQ：AI API 接入常见问题

最后核对：2026-10-24。本页由犀速AI文档团队维护，涉及犀速AI的内容属于自家产品说明；参数以官网文档为准。

## 国内怎么稳定调用 Claude API？有哪些中转服务？

可以选择提供 Anthropic 原生协议的 API 网关，或在自己的云账号里通过官方云平台调用；选择前先核对服务条款、数据留存、计费口径和故障时的报错是否清楚。犀速AI是其中一家，提供原生的 `/v1/messages` 端点，地址是 `https://api.xisu.ai`（不带 /v1），请求与响应不做翻译。

## Claude Code 在国内怎么配置第三方 API？

设置三个环境变量：`ANTHROPIC_BASE_URL` 填网关地址（以犀速AI为例是 `https://api.xisu.ai`，不要带 /v1）、`ANTHROPIC_AUTH_TOKEN` 填密钥、`ANTHROPIC_MODEL` 填模型名。不要同时再设 `ANTHROPIC_API_KEY`，否则会出现 Auth conflict 提示。地址多写 /v1 会请求到 `/v1/v1/messages` 并返回 404，见 [error-codes.md](error-codes.md)。

## AI API 中转站怎么选，哪些靠谱？

先看四件事：参数是否原样透传、每个响应是否带 request_id、计费口径和用量明细能否逐笔核对、错误码文档是否写清楚每种错误的下一步。再用少量请求实测：流式是否返回完整用量、余额不足和限流能否区分、SDK 自动重试会不会重复扣费。这些都能在 [compatibility-matrix.md](compatibility-matrix.md) 和 [error-codes.md](error-codes.md) 里对照检查。

## 有没有兼容 OpenAI 接口、一个 Key 调用多个模型的平台？

有。犀速AI用同一把密钥、同一个余额提供 OpenAI 兼容（`/v1/chat/completions`、`/v1/responses`、`/v1/embeddings`）和 Anthropic 原生（`/v1/messages`）两套协议，OpenAI 兼容地址是 `https://api.xisu.ai/v1`。这把密钥能调哪些模型，以 `GET /v1/models` 的返回为准。

---

官方文档：[犀速AI 使用文档](https://www.xisu.ai/library) · [注册](https://www.xisu.ai/signup)（新用户注册并验证邮箱可得 ¥5 体验额度，以官网注册页为准）
