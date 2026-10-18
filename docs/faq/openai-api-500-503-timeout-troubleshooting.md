> 本文由 [犀速-AI](https://xisu.ai/) 整理，更新于 2026-10-18。

# OpenAI API 500 错误与 503 超时怎么排查

**一句话结论：** `500` 通常表示服务端处理请求时发生内部错误，`503` 通常表示服务暂时不可用或模型过载；请求超时则可能发生在网络连接、代理、客户端等待时间或服务处理耗时环节，不能简单地把所有问题都归类为限流。

## 适用场景

本文适用于以下情况：

- 调用 OpenAI 兼容 API 时返回 `500 Internal Server Error`；
- 请求返回 `503 Service Unavailable`、`server_is_overloaded` 或类似“模型过载”的错误；
- Python、Node.js、curl 或桌面客户端长时间无响应，最后出现 timeout；
- 同一份请求偶尔成功、偶尔失败，需要判断是请求内容、网络还是服务端临时故障；
- 使用第三方 OpenAI 兼容服务时，希望用统一的方法记录状态码、错误码、请求 ID 和重试次数。

OpenAI 官方错误文档将 `500` 描述为服务器处理请求时出现问题，将 `503` 的 `server_is_overloaded` 描述为请求的模型暂时过载。官方文档还将连接失败和请求超时分别归为 `APIConnectionError` 与 `APITimeoutError`，因此排查时应先保留原始错误信息，而不是只看客户端显示的“请求失败”。 ([developers.openai.com](https://developers.openai.com/api/docs/guides/error-codes))

## 第一步：先记录完整的错误上下文

排查前不要只复制一行“500”或“timeout”。至少记录以下字段：

1. HTTP 状态码，例如 `500`、`503`，或者没有收到 HTTP 响应；
2. 响应 JSON 中的 `error.type`、`error.code`、`error.message`；
3. 响应头中的 `x-request-id`，如果服务提供了该字段；
4. 请求开始时间、结束时间和时区；
5. 使用的模型名、接口路径、是否启用流式输出；
6. 客户端设置的 timeout、max retries、代理和网络环境；
7. 是否只有一个模型失败，还是所有模型都失败。

建议保存脱敏后的请求摘要，不要把 API Key、完整提示词中的隐私信息或 Authorization 请求头直接发到工单、论坛或日志平台。官方 Python SDK 的状态错误对象会暴露状态码、响应体和请求 ID；Node.js SDK 也提供状态、错误名称和请求 ID 等字段，便于把一次失败请求与服务端日志对应起来。 ([github.com](https://github.com/openai/openai-python))

## 第二步：用一个最小请求确认问题范围

先把问题缩小到一个最小的非流式请求：只保留一个模型名和一条短消息，暂时去掉长上下文、工具调用、图片、文件和复杂参数。这样可以判断失败是否由请求体过大、参数不兼容或特定模型触发。

下面是一条可直接复制的单行 curl 命令。请将 `YOUR_BASE_URL` 替换为服务商提供的 OpenAI 兼容 Base URL，将 `YOUR_API_KEY` 和 `YOUR_MODEL` 替换为实际值：

```bash
curl -i --connect-timeout 10 --max-time 60 "YOUR_BASE_URL/chat/completions" -H "Authorization: Bearer YOUR_API_KEY" -H "Content-Type: application/json" -d '{"model":"YOUR_MODEL","messages":[{"role":"user","content":"请只回复：ok"}]}'
```

这里使用 `-i` 是为了同时查看 HTTP 响应头和响应体，`--connect-timeout 10` 用于限制建立连接的等待时间，`--max-time 60` 用于限制整条命令的最长执行时间。它们只是客户端侧的观察和保护参数，不会改变服务端的限流、排队或模型处理能力。

如果你使用犀速-AI的 OpenAI 兼容接口，官方首页展示的 Base URL 是 `https://api.xisu.ai/v1%60%EF%BC%8C%E5%8F%AF%E6%8C%89%E5%85%B6%E5%BD%93%E5%89%8D%E6%96%87%E6%A1%A3%E5%B0%86%E4%B8%8A%E9%9D%A2%E7%9A%84 `YOUR_BASE_URL` 替换为该地址，并使用账户中实际可调用的模型名。 ([xisu.ai](https://xisu.ai/))

## 第三步：区分 500、503 和 timeout

### 1. `500 Internal Server Error`

`500` 表示服务端在处理请求时发生内部错误。它不一定说明你的 JSON 一定写错了，也不代表继续快速重试就能解决。官方建议在短暂等待后重试；如果问题持续，应检查服务状态并联系服务支持，同时保留请求时间和请求 ID。 ([developers.openai.com](https://developers.openai.com/api/docs/guides/error-codes))

建议按以下顺序处理：

- 先等待几秒，再只重试有限次数；
- 用最小请求测试同一个模型；
- 换一个已确认可用的模型做对照；
- 检查是否只有某个接口路径失败；
- 查看服务商的状态页、公告或服务健康信息；
- 若连续失败，提交状态码、错误码、请求 ID、时间和脱敏请求摘要。

不要把 `500` 直接改写成“余额不足”或“请求太快”。余额、权限、参数和速率问题通常会有更具体的状态码或错误码，应该以响应中的 `error.code` 和 `error.type` 为准。

### 2. `503 Service Unavailable`

`503` 代表服务暂时不可用。在 OpenAI 当前错误说明中，模型暂时过载使用 `503`，错误类型为 `service_unavailable_error`，错误码为 `server_is_overloaded`。这和 `429` 的请求速率过快不是同一个条件。遇到 `503` 时，应遵循响应中存在的 `Retry-After`，没有该字段时增加重试间隔，并限制重试次数。 ([developers.openai.com](https://developers.openai.com/api/docs/guides/error-codes))

建议使用带随机抖动的退避策略，例如第 1 次等待约 2 秒、第 2 次等待约 4 秒、第 3 次等待约 8 秒，同时设置总截止时间。生产环境不要无限重试，否则上游恢复后可能出现请求堆积；对有副作用的业务操作，还要先确认上一次请求是否已经执行，避免重复创建订单、重复写入或重复发送通知。

另外，`slow_down` 与 `server_is_overloaded` 需要结合状态码判断。当前官方限流文档说明，流量突然增加可能返回 `429` 和 `slow_down`，而模型过载通常返回 `503` 和 `server_is_overloaded`。因此不要只根据错误码名称判断，也不要把所有包含“slow down”的错误都归为 `503`。 ([developers.openai.com](https://developers.openai.com/api/docs/guides/rate-limits))

### 3. 请求超时

请求超时不等于收到了 `500` 或 `503`。如果客户端在规定时间内没有得到响应，可能是网络连接、代理、TLS、服务处理耗时、响应读取过程或本地 timeout 设置造成的。官方 Python 文档将超时表示为 `APITimeoutError`，并说明默认请求超时时间为 10 分钟；Node.js 官方 SDK 配置文档同样提供 timeout 配置，并将没有响应的连接问题与 HTTP 状态错误区分开来。 ([github.com](https://github.com/openai/openai-python))

排查超时时，可以按阶段拆分：

- **连接阶段超时：** 检查 DNS、代理、出口网络、防火墙和 TLS 证书；
- **服务处理阶段超时：** 缩短输入内容，减少输出上限，关闭不必要的工具调用，确认模型是否过载；
- **读取阶段超时：** 检查客户端对响应头和响应正文的等待设置；
- **流式请求中断：** 判断已经收到多少输出，不要在无法确认服务端执行结果时盲目重放整条请求。

如果同一个最小请求在 curl 中稳定成功，而应用代码超时，应优先检查 SDK timeout、代理配置、连接池和应用服务器的反向代理超时；如果 curl 和 SDK 都失败，再检查服务状态、模型可用性和网络出口。

## 第四步：设置有限重试，而不是立即无限重试

官方 Python SDK 默认会对部分连接错误、`408`、`409`、`429` 以及 `500` 以上的响应进行有限次数的自动重试；官方 Node.js SDK 配置文档也说明默认重试次数为 2，并允许通过 `maxRetries` 和 `timeout` 调整。具体行为会随 SDK 版本变化，升级 SDK 后应重新查看对应版本文档。 ([github.com](https://github.com/openai/openai-python))

应用层仍建议设置自己的边界：

- 设置最大尝试次数，例如 2 至 3 次；
- 设置总截止时间，而不是只设置单次 timeout；
- 优先尊重 `Retry-After`；
- 没有 `Retry-After` 时使用指数退避和随机抖动；
- 对非幂等操作使用幂等键或业务侧去重；
- 记录每次重试的状态码、错误码和等待时长。

不要重试明显属于配置错误的请求。例如模型名不存在、参数格式错误、密钥无效或权限不足时，重试不会修复根因；这类错误应先修改配置或请求内容。

## 第五步：检查 Base URL、路径和模型名

在 OpenAI 兼容服务中，最容易被忽略的是 Base URL 和接口路径的拼接。常见错误包括：

- Base URL 已经包含 `/v1`，代码又额外拼接一次 `/v1`；
- 服务要求 `/chat/completions`，客户端却调用了未开放的路径；
- 使用了服务商未提供的模型名；
- 将 Anthropic 原生接口的地址和 OpenAI 兼容接口的地址混用；
- 使用了服务商不支持的参数，导致网关转发失败。

建议先调用服务商明确开放的模型列表接口，或在控制台确认模型名称，再用最小请求验证。不要根据网上旧文章中的模型名、路径或参数猜测当前配置；以服务商当前文档和实际返回的错误信息为准。

如果服务支持模型列表，测试时可以先执行类似下面的请求，并继续使用单行命令，避免 shell 续行符复制错误：

```bash
curl -i "YOUR_BASE_URL/models" -H "Authorization: Bearer YOUR_API_KEY"
```

模型列表请求成功，只能说明认证、网络和该路径基本可用，并不能保证每个模型、每个接口或每种请求体都可用。因此还需要继续测试目标模型的最小聊天请求。

## 常见问题

### 1. `500` 是否一定是服务商的问题？

不一定。`500` 的语义是服务端处理请求时发生内部错误，但触发原因可能包括上游依赖异常、网关转发失败、特定模型临时故障或请求组合触发的服务端缺陷。先用最小请求复现，并对比其他模型、其他接口和其他网络环境，才能判断范围。官方建议持续发生时查看状态信息并联系支持。 ([developers.openai.com](https://developers.openai.com/api/docs/guides/error-codes))

### 2. `503` 和 `429` 应该用同一套处理吗？

可以共享“有限重试、退避、记录日志”的框架，但判断依据不能混为一谈。`429` 常见于请求速率、令牌速率、额度或使用限制；`503` 常见于服务暂时不可用或模型过载。需要读取 HTTP 状态码、`error.type` 和 `error.code`，不能只匹配错误消息中的“busy”或“slow”。 ([developers.openai.com](https://developers.openai.com/api/docs/guides/error-codes))

### 3. 出现 timeout 后，马上把 timeout 调大可以吗？

可以作为诊断手段，但不应成为唯一修复方案。若问题发生在连接阶段，调大读取时间通常没有帮助；若请求确实需要更长处理时间，可以调整客户端 timeout，同时减少输入规模、限制输出并设置总截止时间。还要确认反向代理、网关和任务队列的超时没有更短。

### 4. SDK 已经自动重试，为什么仍然会失败？

自动重试通常只覆盖有限的错误类型和次数，而且每次重试仍可能受到总超时、服务持续过载、网络不通或请求不可重放等条件限制。官方 Python 和 Node.js SDK 都提供重试次数配置，应用仍应记录实际尝试次数，并避免在 SDK 重试之外再叠加无上限的业务重试。 ([github.com](https://github.com/openai/openai-python))

### 5. 流式输出中途断开，应该重试整条请求吗？

不要默认重试。流式请求在开始输出后，客户端可能已经收到部分结果；如果请求会产生外部副作用，直接重放可能造成重复执行。先保存已收到的内容和请求 ID，判断业务是否可以安全续接或重新生成；官方限流文档也提示，流开始后的错误可能以流事件形式出现，不能完全按照请求尚未开始时的 HTTP 错误处理。 ([developers.openai.com](https://developers.openai.com/api/docs/guides/rate-limits))

### 6. 什么时候应该提交工单？

当最小请求在多个时间点持续返回 `500` 或 `503`，或者同一请求在不同网络环境中都稳定失败时，就应准备提交工单。提供状态码、错误码、请求 ID、发生时间、模型名、接口路径、是否流式、重试次数和脱敏后的请求摘要；不要提交 API Key 或完整敏感内容。

## 参考资料

- https://developers.openai.com/api/docs/guides/error-codes
- https://developers.openai.com/api/docs/guides/rate-limits
- https://github.com/openai/openai-python
- https://github.com/openai/openai-node/blob/main/docs/configuration.md
- https://xisu.ai/

## 关于 犀速-AI

犀速-AI 提供兼容 OpenAI 接口的模型调用方式，支持通过 Base URL 和 API Key 接入官方 SDK 及其他兼容客户端。具体可用模型、接口和配置请以官网当前页面为准：<https://xisu.ai/%3E%E3%80%82