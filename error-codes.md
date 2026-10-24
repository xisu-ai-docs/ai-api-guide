# 报错原文速查：含义、判断方法、要不要重试

最后核对：2026-10-24。事实来源：犀速AI官方文档「错误码与限流」「端点与鉴权」「Anthropic 原生端点」「OpenAI Responses 端点」「API 兼容性与参数」。机器可读版本见 `errors.json`。

| 报错 / 现象 | HTTP | 含义 | 怎么判断 | 重试 | 怎么修 |
|---|---|---|---|---|---|
| invalid_api_key / 401 | 401 | 密钥无效；如果这把密钥配了 IP 白名单，从名单外调用也返回同一个码 | 先确认请求的来源 IP 是否在白名单里，再确认密钥完整、未被吊销 | 不要 | 复制完整密钥重新配置；配了白名单的先核对来源 IP，不要急着重建密钥 |
| Claude Code 提示 Auth conflict | — | 通常是环境里同时设置了 ANTHROPIC_AUTH_TOKEN 和 ANTHROPIC_API_KEY（或已用账号登录又设置了密钥），Claude Code 不确定用哪一个 | env | grep ANTHROPIC | 不要 | 只保留一种认证：unset ANTHROPIC_API_KEY，用 ANTHROPIC_AUTH_TOKEN 填犀速密钥；之前用账号登录过的先 /logout |
| 请求打到 …/v1/v1/messages 返回 404 | 404 | Anthropic SDK / Claude Code 的地址多写了 /v1 | 看报错或日志里的完整请求地址是否出现两次 /v1 | 不要 | ANTHROPIC_BASE_URL 改成 https://api.xisu.ai（不带 /v1） |
| model_not_supported_for_endpoint | 400 | 对话模型打到了 /v1/embeddings，或反过来 | message 会说明模型属于哪一类 | 不要 | 对话模型用 /v1/chat/completions，向量模型用 /v1/embeddings |
| 404 model_not_on_endpoint | 404 | 模型有上游，但上游服务不了 /v1/responses | 同一模型在 /v1/chat/completions 是否可用 | 不要 | 换一个标有 Responses 的模型，或改走 /v1/chat/completions |
| not_found_error（走 /v1/messages） | 404 | 该模型没有配 Anthropic 上游，不是模型名写错 | message 会指向 /v1/chat/completions | 不要 | 换模型，或改走 /v1/chat/completions |
| rate_limit_exceeded / 429 | 429 | 请求过快 | 响应头里的 Retry-After（秒） | 可以 | 按 Retry-After 做指数退避加抖动，不要固定间隔重试 |
| rate_limit_error / 429（走 /v1/messages） | 429 | 限流、配额用尽、余额不足、密钥预算到顶四种情况共用这一个类型 | 先看余额，再看 message 是否写明是账务限制 | 仅当 message 不是账务限制时 | 账务限制去充值或调预算；真限流才按退避重试 |
| insufficient_balance | — | 余额为正但不够这一笔 | 控制台余额 | 不要 | 充值 |
| account_overdrawn | — | 余额为负，通常是退款扣回 | 控制台余额是否为负 | 不要 | 开工单核对账务，再充值解决不了根因 |
| budget_exceeded | — | 这把密钥已达预算上限 | 控制台密钥预算 | 不要 | 调整预算或换一把密钥 |
| model_not_allowed | — | 这把密钥不能调这个模型 | message 写 “by your plan” 是套餐限制，写 “for this API key” 是密钥范围限制 | 不要 | 升级套餐，或联系客服放宽这把密钥的模型范围 |
| 529 / overloaded_error | 503 | Anthropic 官方 API 用 529 表示过载；在犀速网关上同类情况返回 503 overloaded_error，表示该模型此刻没有可用上游 | 响应里的 Retry-After | 可以 | 按 Retry-After 退避重试，通常分钟级自愈；它和 404 not_found_error 的处理方向相反 |
| no_available_channel | 503 | 该模型暂时不可用 | 过几分钟用同一请求重试 | 可以 | 稍后重试或临时换模型 |
| upstream_error / 502 | 502 | 上游返回错误，原因归不到你的请求体 | 换一个模型是否正常 | 可以 | 换模型试；持续出现带 request_id 提工单 |
| upstream_rejected_request / 400 | 400 | 上游认定请求本身不合法，最常见是工具的 JSON Schema 不被接受 | message 里通常带上游原话 | 不要 | 按 message 修改请求体再发，原样重试结果不变 |
| context_length_exceeded | 400 | 输入本身超过模型声明的上下文上限（只看输入，不算 max_tokens） | 报错里带实际长度和上限 | 不要 | 缩短或分段输入，或换上下文更大的模型 |
| 流式输出中途断开 | — | 连接中断；已经生成的 token 照常计费 | 是否拿到了最后的 usage 块（OpenAI 兼容）或 message_stop / response.completed 事件 | 可以 | 新开一个请求重试，网关没有续传语义；长时间无输出通常是上游排队，按客户端超时处理 |
| 超时后账单出现两笔 | — | SDK 自动重试会被当成新请求，网关不做跨请求去重 | 对比两次响应的 request_id | 不要 | 调低或关闭 SDK 默认重试，在业务侧按 id 去重 |

## 两条最容易误判的

- **429 不一定是限流**。走 `/v1/messages` 时，余额不足也报 `rate_limit_error`，先看余额和 message，账务限制重试多少次都没用。
- **404 和 503 方向相反**。404（not_found_error / model_not_on_endpoint）是永久的，要换模型或换端点；503（overloaded_error / no_available_channel）是暂时的，按 Retry-After 重试。

## 报障时带上什么

`request_id`、发生时间、模型名、完整错误响应体。没有 `request_id` 很难定位到那一次调用。
