# 2026 年 10 月 Responses API 与 Chat Completions 怎么选：OpenAI 兼容接口排错指南

> 本文整理自 [犀速-AI 官网原文](https://www.xisu.ai/resources/articles/responses-api-chat-completions-choose)，原文更新于 2026-10-17。

## 结论先看

- **Responses API 和 Chat Completions 是两套不同的接口，不能只修改路径就混用请求体。**
- 使用平台的 OpenAI 兼容入口时，当前可直接使用的兼容端点包括 `/chat/completions`、`/embeddings`、`/models`、`/messages` 和 `/messages/count_tokens`。
- 如果 SDK 把请求发到了 `/responses`，却配置的是兼容接口地址，常见结果是 404、400 或返回字段与代码预期不一致；应先确认最终请求路径，再用不带工具和复杂参数的最小 Chat Completions 请求验证链路。

## 为什么现在值得看

本文更新于 2026-10-17。当前部分 SDK 或较新的封装会优先提供 Responses API 方法，而平台兼容入口支持的端点、模型能力和请求结构需要分别确认。遇到 404、400、空响应、工具参数不兼容或 SDK 与命令行结果不一致时，先按本文验证最终路径和最小请求，通常比直接修改业务代码更快。

## 一、先判断你实际调用的是哪套接口

Responses API 与 Chat Completions 的请求路径、字段结构和返回格式不同。OpenAI 官方文档将 Chat Completions 定义为基于 `messages` 生成回复的接口，而 Responses API 使用 `POST /responses`，支持文本、图片、内置工具和函数调用等能力。两者应分别按照对应文档组装请求。可参考 OpenAI 兼容接口接入教程和接口与端点说明。

### 查看最终请求 URL

Chat Completions 的路径通常是 `/v1/chat/completions`；Responses API 的路径是 `/v1/responses`。如果 SDK 自动拼接了 `/v1`，不要在 `base_url` 中重复填写。

### 查看请求体顶层字段

Chat Completions 使用 `messages`；Responses API 使用 `input`，并可以配置 Responses 专用的工具和会话能力。

### 查看代码读取的返回字段

Chat Completions 常从 `choices[0].message.content` 读取文本；Responses API 使用自己的响应对象，不能直接按 `choices` 解析。

### 确认模型和端点匹配

平台可用模型以模型广场页面为准，不同模型对图片、工具、推理参数和输出格式的支持可能不同。

| 对比项 | Chat Completions | Responses API |
|---|---|---|
| 典型路径 | `/v1/chat/completions` | `/v1/responses` |
| 主要输入字段 | `messages` | `input` |
| 常见文本读取方式 | `choices[0].message.content` | 按 Responses 返回对象读取输出内容 |
| 适合场景 | 已有 OpenAI SDK、聊天消息、常规工具调用和兼容旧代码 | 使用 Responses 专用能力、内置工具或新的状态管理方式 |
| 平台接入判断 | 可使用平台已公布的 OpenAI 兼容端点 | 需确认目标服务明确支持该端点，不能仅凭 SDK 名称判断 |

官方文档仍提供 Chat Completions 参考，同时建议新项目了解 Responses API 的能力和差异。具体以对应官方文档和实际服务端返回为准：Chat Completions Overview、Responses Overview。

## 二、用最小请求验证兼容接口

下面的检测请求只使用模型名和一条用户消息，不带工具、结构化输出、图片和复杂参数。将 `YOUR_API_KEY` 替换为平台密钥，将 `MODEL_NAME` 替换为模型广场中可用的模型名。

### PowerShell

```powershell
$uri = "https://api.xisu.ai/v1/chat/completions"
$apiKey = "YOUR_API_KEY"
$model = "MODEL_NAME"
$body = @{
    model = $model
    messages = @(
        @{ role = "user"; content = "只回复：ok" }
    )
} | ConvertTo-Json -Depth 6
try {
    $response = Invoke-WebRequest `
        -Uri $uri `
        -Method Post `
        -Headers @{ Authorization = "Bearer $apiKey" } `
        -ContentType "application/json; charset=utf-8" `
        -Body ([System.Text.Encoding]::UTF8.GetBytes($body)) `
        -UseBasicParsing
    "HTTP: $($response.StatusCode)"
    $response.Headers | Format-List
    $response.Content
} catch {
    if ($_.Exception.Response) {
        $httpResponse = $_.Exception.Response
        "HTTP: $([int]$httpResponse.StatusCode)"
        $httpResponse.Headers | Format-List
        if ($_.ErrorDetails.Message) {
            $_.ErrorDetails.Message
        } else {
            $_.Exception.Message
        }
    } else {
        $_.Exception.Message
    }
}
```

### 怎么看输出

HTTP 200 且响应中出现 `choices`，说明 Chat Completions 基础调用已经成功；如果返回 404，优先检查是否误用了 `/responses` 或重复拼接了 `/v1`；如果返回 400，先读取错误正文中的字段路径，再逐项恢复业务参数。

### macOS/Linux

```bash
export XISU_API_KEY='YOUR_API_KEY'
export XISU_MODEL='MODEL_NAME'
curl -i https://api.xisu.ai/v1/chat/completions \
  -H "Authorization: Bearer ${XISU_API_KEY}" \
  -H "Content-Type: application/json" \
  -d "{\"model\":\"${XISU_MODEL}\",\"messages\":[{\"role\":\"user\",\"content\":\"只回复：ok\"}]}"
```

`curl -i` 会同时显示响应头和响应体。重点查看第一行的 HTTP 状态码、`request-id`、`x-request-id` 或正文中的 `request_id`，具体字段以实际响应为准。成功响应通常包含 `choices`；若需要核对 token 和费用，再查看响应中的 `usage` 以及平台请求日志。

## 三、根据现象判断接口混用位置

| 现象 | 常见原因 | 处理方式 |
|---|---|---|
| 请求返回 404 | 把 `/responses` 发到只配置了兼容 Chat Completions 的地址，或重复拼接了 `/v1` | 打印最终 URL，确认使用 `/v1/chat/completions` 或服务明确支持的端点。 |
| 返回 400，提示缺少 `input` | 当前服务端点是 Responses API，但请求体使用了 `messages` | 按照目标服务的 Responses 文档改用 `input`；如果使用平台兼容入口，则改回 Chat Completions 请求。 |
| 返回 400，提示缺少 `messages` | 把 Responses API 的请求体发送到了 Chat Completions | 改用 `messages` 数组，并确认每条消息包含正确的 `role` 和 `content`。 |
| HTTP 200 但代码读不到文本 | 请求成功，但解析器仍按另一套响应结构读取 | 先打印完整响应的非敏感部分，分别适配 `choices` 或 Responses 返回对象，不要只改模型名。 |
| 最小请求成功，加入工具后失败 | 工具 schema、工具调用字段或模型能力不匹配 | 一次只加入一个工具，检查 `name`、`description`、`parameters` 和 JSON Schema 类型。可参考工具 schema 被拒绝时的排查教程。 |
| 命令行成功，SDK 失败 | SDK 自动拼接路径、覆盖了 `base_url`，或把平台密钥读取成了其他环境变量 | 输出 SDK 最终 URL、请求方法、状态码和响应正文，隐藏 Authorization 值。 |

## 四、正确配置 OpenAI 兼容 SDK

- **把 `base_url` 指向版本根路径。** 使用平台兼容接口时，`base_url` 设置为 `https://api.xisu.ai/v1`，不要直接写成 `https://api.xisu.ai/v1/chat/completions`，除非该 SDK 明确要求完整 endpoint。
- **让 SDK 负责拼接资源路径。** 选择 Chat Completions 相关方法时，确认它最终调用的是 `/chat/completions`，而不是自动改成 `/responses`。
- **先关闭非必要功能。** 暂时移除 `tools`、图片、结构化输出和供应商专用字段，先验证普通文本请求。
- **不要把两套字段混在一起。** 例如不要把 `input`、Responses 专用工具配置直接放进 Chat Completions 请求，也不要把 `choices` 解析逻辑用于 Responses 返回。
- **按模型能力恢复参数。** 最小请求成功后，再逐项加入 system 或 developer 指令、历史消息、工具和输出格式。出现错误时记录 `request_id`，便于在请求日志中核对实际模型、token 和费用。

如果你使用的是 Anthropic Messages 接口，则应使用对应的 `/messages` 请求结构，而不是把 OpenAI 的 `messages` 数组直接当作完全相同的协议。Messages API 的 system 指令位于顶层 `system` 字段，官方文档也明确说明该接口没有 system 角色。需要接入时可参考 Anthropic Messages 接入教程。

## 五、地区、状态页与平台调用要分开判断

如果请求发送到平台 API 地址，优先排查平台密钥、`base_url`、端点、模型名和请求体；官方直连服务的地区支持页面不能替代平台请求日志，也不能直接作为平台调用失败的首要判断依据。

如果你使用官方直连入口，OpenAI 官方支持地区页面列出了当前支持访问 API 的国家和地区；地区列表和服务条件可能更新，应以页面当前内容为准：OpenAI API 支持的国家和地区。使用平台入口时，先完成本文的端点和请求体检查。

当多个最小请求同时出现 5xx 或连接异常时，再查看对应官方状态页。OpenAI 状态页会按 API、Chat Completions、Responses 等组件展示整体状态；状态页反映的是官方服务整体情况，不代表每个模型、账户或平台入口的单次请求结果：OpenAI Status。

## 六、上线前做一次接口自检

- 记录 SDK 版本、最终 `base_url`、请求路径和模型名。
- 用最小请求分别验证普通文本、历史消息和工具调用，不要一次加入所有功能。
- 确认代码能处理 HTTP 错误、空内容、工具调用和流式中断。
- 在日志中保存 `request_id`、模型名、token 和费用，但不要保存完整密钥。
- 切换模型前重新查看模型广场中的可用端点和能力，不要假设所有模型支持相同参数。

## 常见问题

### 我使用 OpenAI SDK，为什么它自动请求了 `/responses`？

部分 SDK 或较新的封装会优先提供 Responses API 方法。检查你调用的方法名称和 SDK 的请求日志；如果目标是平台 OpenAI 兼容接口，请使用 Chat Completions 方法，并确认最终请求路径为 `/v1/chat/completions`。

### `base_url` 应该写到 `/v1` 还是 `/chat/completions`？

对于常见 OpenAI SDK，`base_url` 通常写到 `https://api.xisu.ai/v1`，由 SDK 自动拼接资源路径。若出现双重 `/v1` 或路径重复，打印最终 URL 后按 SDK 文档调整。

### Chat Completions 能不能直接使用 Responses 的 `input` 字段？

不能假设可以。Chat Completions 的请求主体以 `messages` 为核心，Responses API 使用自己的 `input` 和响应结构。两套字段应按目标端点分别组装。

### 为什么返回 200，但程序显示没有内容？

最常见原因是解析字段不匹配，或者模型返回了工具调用而不是普通文本。先打印响应中的 `choices`、`message`、`tool_calls` 和 `finish_reason`，再根据实际结构处理。

### 平台支持 Responses API 吗？如何核对请求是否真的用了目标模型？

已知平台兼容端点包括 `/chat/completions`、`/embeddings`、`/models`、`/messages` 和 `/messages/count_tokens`。可用端点和模型以平台接口说明及模型广场页面为准，不要仅凭官方 SDK 的方法名称推断平台入口一定支持 `/responses`。

读取响应中的模型字段，并在平台请求日志中按 `request_id` 核对模型名、token 和费用。平台按实际 token 用量计费，计费精度到小数点后 10 位，充值积分不过期。

## 参考资料

- OpenAI Chat Completions Overview
- OpenAI Create chat completion
- OpenAI Responses Overview
- OpenAI Create a model response
- OpenAI API 支持的国家和地区
- OpenAI Status
- Claude Messages API Reference

如果你已经确认应用应使用 OpenAI 兼容接口，可以先按快速开始教程完成最小接入，再到套餐与费率页面查看可用模型和计费方式，并结合请求日志核对实际用量。

免责声明：本文仅用于技术排查，接口能力、模型列表和官方支持地区可能随服务更新而变化，请以对应官方页面和实际响应为准。

## 关于 犀速-AI

犀速-AI 提供 OpenAI 兼容接口及相关接入文档，方便开发者根据端点、模型能力和实际响应完成接口排查。

官网：[https://xisu.ai/](https://xisu.ai/)