> 本文由 [犀速-AI](https://xisu.ai/) 整理，更新于 2026-10-30。

# OpenAI API 404 model_not_found 怎么解决：模型名称与 Base URL 排查指南

**一句话结论：** OpenAI API 返回 HTTP 404 并出现 `model_not_found` 时，优先核对请求中的 `model` 是否为当前 Base URL 可用的模型 ID，再检查 Base URL 是否指向正确的 API 根路径；不要先把它当作网络超时或 429 限流处理。

## 适用场景

本文只讨论一个错误方向：**HTTP 404，且响应中的错误类型、错误代码或错误消息指向 `model_not_found`、模型不存在、模型不可用或类似含义**。

常见触发场景包括：

- 从其他平台复制了模型名称，但当前服务并不提供该模型。
- 把模型展示名、产品名称或别名误填进了 `model` 字段。
- 模型 ID 拼写错误，大小写、连字符、版本后缀不一致。
- Base URL 少了 `/v1`，多了错误路径，或者仍然指向另一个服务商。
- 模型属于其他项目、组织或权限范围，当前 API Key 无法访问。
- SDK 配置了一个 Base URL，但实际请求被环境变量、配置文件或代理改写到了另一个地址。

OpenAI 官方错误说明将 `model_not_found` 归为请求所需模型或资源不可用，并建议检查模型 ID、模型所属项目以及资源是否已被删除。官方模型列表接口也说明，模型列表中的模型标识符可以被后续 API 请求引用。因此，排查重点应当是“请求打到了哪里”和“这个地址返回了哪些模型”，而不是盲目重复发送同一个请求。

## 分步骤排查

### 第 1 步：先确认响应确实是 404 model_not_found

先保存完整的响应状态和 JSON 错误对象，但不要在日志中打印 API Key。一个典型响应可能类似下面这样：

```json
{
  "error": {
    "message": "The model `example-model` does not exist or you do not have access to it.",
    "type": "invalid_request_error",
    "param": null,
    "code": "model_not_found"
  }
}
```

这里真正有判断价值的是：

1. HTTP 状态码是否为 `404`；
2. `error.code` 是否为 `model_not_found`，或者 `error.message` 是否明确指出模型不存在、模型不可用；
3. 错误是否发生在发送请求后，而不是客户端还没有建立连接。

如果返回的是 `401`，应转向 API Key 或认证配置排查；如果是 `403`，应检查区域、权限或访问策略；如果是 `429`，则属于限流或额度方向。本文不把这些错误混在 404 处理中。

### 第 2 步：确认当前 Base URL

OpenAI 兼容客户端通常由两部分决定最终请求地址：

- Base URL，例如 `https://api.example.com/v1%60%EF%BC%9B
- 资源路径，例如 `/models` 或 `/chat/completions`。

因此，SDK 中的 Base URL 是否已经包含 `/v1`，必须以服务商文档为准。常见错误包括：

```text
正确示例：   https://api.example.com/v1
可能有误：   https://api.example.com/
可能有误：   https://api.example.com/v1/v1
可能有误：   https://api.example.com/v1/chat/completions
```

不要把完整的 `/chat/completions` 地址当作 SDK 的 Base URL。SDK 通常会在 Base URL 后面自动拼接资源路径；如果已经填入完整接口路径，最终请求可能变成重复路径，从而得到 404。

如果你使用犀速-AI，请以控制台或接入文档中给出的兼容接口地址为准，并不要把其他平台的模型名直接照搬过来。

### 第 3 步：列出当前地址可见的模型

在运行环境中预先配置好 `OPENAI_BASE_URL` 和 `OPENAI_API_KEY` 后，可以使用下面的请求查看当前 API 地址返回的模型列表。示例不会在命令中写入或设置明文密钥：

```bash
curl -sS "$OPENAI_BASE_URL/models" \
  -H "Authorization: Bearer $OPENAI_API_KEY"
```

如果你的变量保存的是不含 `/v1` 的主地址，则应按实际接口拼接：

```bash
curl -sS "$OPENAI_BASE_URL/v1/models" \
  -H "Authorization: Bearer $OPENAI_API_KEY"
```

二者不要同时使用。你需要确认变量中到底保存了哪一种形式，避免出现 `/v1/v1/models`。

成功响应通常会包含一个 `data` 数组，数组元素中的 `id` 才是后续请求应使用的模型标识符。不要使用以下内容替代 `id`：

- 网页上的显示名称；
- 宣传页面中的系列名称；
- 个人记忆中的旧模型名；
- 其他服务商返回的模型 ID；
- 从旧项目配置中复制的已下线版本。

如果模型列表请求本身返回 401，应先处理认证问题；如果返回 404，优先检查 Base URL 和 `/v1` 拼接；如果请求成功但列表中没有目标模型，则不能继续假设该模型可用。

### 第 4 步：用列表中的精确 `id` 替换模型名

假设模型列表返回如下内容：

```json
{
  "data": [
    {
      "id": "example-chat-model",
      "object": "model"
    }
  ]
}
```

那么请求体中应使用完整且完全一致的字符串：

```json
{
  "model": "example-chat-model",
  "messages": [
    {
      "role": "user",
      "content": "请只回复：连接测试成功"
    }
  ]
}
```

不要自行把 `example-chat-model` 改成 `example-chat-model-latest`、`Example-Chat-Model` 或其他推测出来的名称。模型 ID 是字符串标识符，拼写、大小写和版本后缀都可能影响匹配。

### 第 5 步：发送一个最小请求

模型名称确认后，先不要带工具调用、图片、结构化输出、超长上下文或流式参数。使用最小请求，可以把问题限定在模型、认证、Base URL 和基本请求格式上：

```bash
curl -sS "$OPENAI_BASE_URL/chat/completions" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "从 /models 返回结果中复制的精确 id",
    "messages": [
      {
        "role": "user",
        "content": "请回复：ok"
      }
    ],
    "max_tokens": 16
  }'
```

如果你的服务使用 Responses API，应使用该服务明确支持的路径和请求格式，不要把 Chat Completions 的字段与 Responses API 混用。OpenAI 官方 API 参考将不同 API 表面的路径、请求结构和客户端方法分别列出；兼容服务也可能只实现其中一部分，因此应以目标服务公布的兼容范围为准。

### 第 6 步：核对 SDK 的实际配置

在 Python 中，建议把 Base URL、API Key 和模型名从环境变量或密钥管理服务读取：

```python
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["OPENAI_API_KEY"],
    base_url=os.environ["OPENAI_BASE_URL"],
)

response = client.chat.completions.create(
    model=os.environ["OPENAI_MODEL"],
    messages=[{"role": "user", "content": "请回复：ok"}],
    max_tokens=16,
)

print(response.choices[0].message.content)
```

在 Node.js 中：

```js
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
  baseURL: process.env.OPENAI_BASE_URL,
});

const response = await client.chat.completions.create({
  model: process.env.OPENAI_MODEL,
  messages: [{ role: "user", content: "请回复：ok" }],
  max_tokens: 16,
});

console.log(response.choices[0].message.content);
```

重点检查三件事：

- 程序运行时读取到的 Base URL 是否就是你刚才用 `curl` 测试的地址；
- `OPENAI_MODEL` 是否是当前地址的模型列表中真实存在的 `id`；
- 是否有 `.env`、启动脚本、容器环境变量或 IDE 配置覆盖了你以为正在使用的值。

OpenAI 官方 Python SDK 将 HTTP 404 映射为 `NotFoundError`。如果程序捕获了通用异常，建议至少记录状态码、错误代码和请求 ID；不要只记录“请求失败”，否则无法判断是模型不存在还是其他问题。

## 常见问题

### 1. `/models` 能访问，但目标模型仍然返回 404，为什么？

`/models` 返回成功，只能说明当前认证信息可以访问模型列表，不代表任意模型名都可用。请在响应的 `data` 数组中查找精确的 `id`，并确认请求使用的是同一个 Base URL、同一个 API Key 和同一个项目环境。模型列表可能因服务、项目或权限范围不同而不同。

### 2. 模型名称在网页上能看到，为什么 API 仍提示 model_not_found？

网页显示名称不一定等于 API 模型 ID。网页可能展示系列名、营销名称、别名或产品入口名称，而 API 请求要求使用接口实际接受的标识符。以当前 `/models` 返回的 `id` 或服务商明确公布的 API 模型名为准，不要根据网页标题自行推断。

### 3. 把 `/v1` 加上或删掉后，还是 404，怎么办？

先分别确认 Base URL 的定义和客户端的拼接规则。对于保存了 `https://api.example.com/v1%60 的变量，请求模型列表通常是 `"$OPENAI_BASE_URL/models"`；对于只保存了 `https://api.example.com`/ 的变量，才可能需要 `"$OPENAI_BASE_URL/v1/models"`。如果两种写法都失败，检查域名是否正确，以及该服务是否真的提供 OpenAI 兼容的 `/models` 路径。

### 4. 需要不断重试 404 请求吗？

通常不应把固定的 `model_not_found` 当作瞬时网络故障反复重试。先修正模型 ID、Base URL、项目配置或访问权限；只有在错误信息明确表明模型暂时不可用，且服务文档提供了相应恢复方式时，才按照服务方说明处理。

### 5. 旧代码以前能用，现在突然出现 404，如何定位？

先记录最近发生的配置变化：模型名是否被替换、API Key 是否切换、Base URL 是否变化、项目是否变化，以及模型是否仍出现在当前 `/models` 返回结果中。若使用固定版本模型，检查该版本是否仍被目标服务支持；若使用别名，确认别名在当前服务中仍存在。必要时保留失败请求的时间、HTTP 状态码、错误代码和请求 ID，便于向服务方提交可核验信息。

## 参考资料

1. [OpenAI API Error codes](https://developers.openai.com/api/docs/guides/error-codes) —— 错误代码、HTTP 状态码及 `model_not_found` 的判断依据。
2. [OpenAI API List models](https://developers.openai.com/api/reference/resources/models/methods/list) —— 模型列表接口和模型 `id` 的使用方式。
3. [OpenAI API Reference Overview](https://developers.openai.com/api/reference/overview) —— Base URL、Bearer 认证、请求调试和 API 参考说明。
4. [OpenAI Python API Reference](https://developers.openai.com/api/reference/python) —— Python SDK 对 404 `NotFoundError` 及请求错误的处理方式。

## 关于 犀速-AI

犀速-AI 提供面向开发者的 OpenAI 兼容 API 接入服务与配置指引，帮助用户在常见 SDK 和 AI 工具中完成连接与排错。更多服务说明与接入信息请访问官网：[https://xisu.ai/](https://xisu.ai/)。