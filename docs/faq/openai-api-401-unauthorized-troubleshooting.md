> 本文由 [犀速-AI](https://xisu.ai/) 整理，更新于 2026-10-16。

# OpenAI API 401 Unauthorized 怎么解决：API Key 与 Base URL 排查指南

一句话结论：**OpenAI API 或兼容接口返回 `401 Unauthorized` 时，先检查实际发送的 API Key 和 `Authorization: Bearer ...` 请求头，再确认 Base URL、项目归属、密钥状态以及 IP 白名单；不要一看到 401 就反复更换模型名称。**

## 适用场景

本文适用于以下情况：

- 使用 Python、Node.js、curl 或其他 OpenAI 兼容 SDK 调用接口时返回 `401 Unauthorized`；
- 已经设置了环境变量，但程序仍提示 `Incorrect API key provided` 或 `Invalid Authentication`；
- 从 OpenAI 官方接口切换到其他 OpenAI 兼容服务后，出现认证失败；
- 在 Cursor、Cherry Studio、Dify、LobeChat、NextChat 等工具中填写 API Key 后，测试连接失败；
- 同一把密钥在一个项目或环境中可用，在另一个项目、服务器或网络环境中不可用。

本文只讨论 OpenAI API 或 OpenAI 兼容接口的认证排查。Claude Code 使用 Anthropic 相关配置和协议，不应直接套用本文的 `OPENAI_API_KEY`、OpenAI 风格 Base URL 或 Chat Completions 示例；如果排查 Claude Code，请以其官方文档中的网关和环境变量说明为准。([docs.anthropic.com](https://docs.anthropic.com/en/docs/claude-code/llm-gateway))

## 一、先区分 401 与其他错误

排查前先查看实际的 HTTP 状态码和响应体，不要只根据客户端显示的“连接失败”判断。OpenAI 官方错误码文档将 `401` 认证错误与 `403` 权限或区域限制、`404` 资源不存在、`429` 速率或额度限制以及 `500`、`503` 服务端问题区分开来。([developers.openai.com](https://developers.openai.com/api/docs/guides/error-codes))

| 状态码 | 常见含义 | 首要检查项 |
| --- | --- | --- |
| `400` | 请求格式或参数错误 | JSON、字段名称、参数类型 |
| `401` | 认证失败 | API Key、Bearer 格式、项目归属、IP 白名单 |
| `403` | 已认证但没有访问权限，或区域不支持 | 组织、项目和资源权限 |
| `404` | 地址或资源不存在 | Base URL、路径、模型名 |
| `429` | 速率、余额或用量限制 | 请求频率、额度、项目限制 |
| `500`、`503` | 服务端错误或模型暂时不可用 | 服务状态、稍后重试 |

如果响应中出现 `401`、`invalid_api_key`、`Incorrect API key provided`、`invalid authentication` 或 `ip_not_authorized`，应先处理认证链路。官方文档建议同时读取 HTTP 状态码、结构化错误中的 `error.code` 和 `error.message`，而不是只记录客户端的一句概括性错误。([developers.openai.com](https://developers.openai.com/api/docs/guides/error-codes))

## 二、先用一个最小请求确认问题范围

不要一开始就从完整业务代码、桌面客户端或复杂 Agent 流程开始排查。先使用模型列表请求，能够快速判断“当前密钥是否被当前服务接受”。下面命令使用单行写法，不依赖 Bash 续行符。

### OpenAI 官方接口

```bash
curl -i "https://api.openai.com/v1/models%22 -H "Authorization: Bearer $OPENAI_API_KEY"
```

### OpenAI 兼容接口

如果使用的是兼容服务，应将地址替换为该服务公开文档提供的 Base URL，并确认路径是否包含 `/v1`。例如，若服务文档明确给出 `https://api.xisu.ai/v1%60%EF%BC%8C%E5%8F%AF%E4%BB%A5%E4%BD%BF%E7%94%A8%EF%BC%9A

```bash
curl -i "https://api.xisu.ai/v1/models%22 -H "Authorization: Bearer $API_KEY"
```

上面的 `$OPENAI_API_KEY` 和 `$API_KEY` 只是环境变量名称，不是实际密钥。返回 `200` 且响应中包含模型列表，通常说明密钥和基础地址已经完成基本连通；如果仍然返回 `401`，继续检查下面各项。具体兼容路径、模型列表和认证规则应以服务商公开文档为准。

## 三、检查 Authorization 请求头

### 1. Bearer 格式必须正确

标准格式是：

```http
Authorization: Bearer sk-example
```

下面这些写法都可能导致认证失败：

```http
Authorization: sk-example
Authorization: Token sk-example
Authorization: Bearer: sk-example
```

`Bearer` 与密钥之间需要一个空格，`Bearer` 后面不要添加冒号。curl、Python SDK 和 Node.js SDK 通常会自动生成请求头，但如果请求经过自定义 HTTP 客户端、代理、中间件或网关，仍应确认最终发出的请求头没有被删除或改写。OpenAI API 参考文档明确要求使用 HTTP Bearer 认证。([developers.openai.com](https://developers.openai.com/api/reference/overview))

### 2. 不要把变量名当成密钥

下面的命令传递的是字面量字符串 `OPENAI_API_KEY`，而不是环境变量中的值：

```bash
curl -i "https://api.openai.com/v1/models%22 -H "Authorization: Bearer OPENAI_API_KEY"
```

读取环境变量时，应使用 `$OPENAI_API_KEY`：

```bash
curl -i "https://api.openai.com/v1/models%22 -H "Authorization: Bearer $OPENAI_API_KEY"
```

### 3. 不要在代码中硬编码密钥

Python 示例应从环境变量读取，而不是写入 `"YOUR_API_KEY"` 或实际密钥：

```python
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["API_KEY"],
    base_url="https://api.example.com/v1%22,
)
```

如果服务或项目约定使用 `OPENAI_API_KEY`，也可以将代码中的变量名统一改为 `os.environ["OPENAI_API_KEY"]`。关键是代码读取的变量名必须与运行环境中实际设置的变量名一致。

Node.js 示例同样从环境变量读取：

```javascript
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: process.env.API_KEY,
  baseURL: "https://api.example.com/v1%22,
});
```

官方文档建议将 API Key 保存在环境变量或密钥管理服务中，不要暴露在浏览器、前端应用、公开仓库、日志和截图中。([developers.openai.com](https://developers.openai.com/api/reference/overview))

## 四、确认环境变量真的生效

可以在终端中判断变量是否存在，但不要直接打印完整密钥：

```bash
if [ -n "$API_KEY" ]; then echo "API_KEY is set"; else echo "API_KEY is empty"; fi
```

如果输出为空，常见原因包括：

1. 只在另一个终端窗口中设置过变量；
2. 修改了 `.env` 文件，但当前程序没有加载该文件；
3. Docker、systemd、CI/CD 或 IDE 没有继承当前 Shell 的环境变量；
4. 代码读取 `API_KEY`，实际设置的是 `OPENAI_API_KEY`，或反过来；
5. 修改变量后没有重启开发服务器、容器或任务进程；
6. 复制密钥时带入了引号、空格、换行或不可见字符。

使用 `.env` 时，应确认项目实际采用的配置加载方式，并将 `.env` 加入 `.gitignore`。如果密钥曾经提交到公开仓库或出现在日志中，不要只删除文本记录，应尽快在密钥管理页面创建新密钥并撤销旧密钥。OpenAI 帮助中心说明，完整密钥通常只会在创建时显示；如果遗失，应创建新的密钥并更新应用配置。([help.openai.com](https://help.openai.com/en/articles/4936850-where-do-i-find-my-openai-api-key))

## 五、检查 Base URL 与路径是否混用

OpenAI SDK 中的 `base_url` 或 `baseURL` 通常应填写服务商文档定义的 API 根地址或版本地址，而不是某个具体接口的完整 URL。例如：

```python
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["API_KEY"],
    base_url="https://api.example.com/v1%22,
)
```

常见错误包括：

- 把完整的 `/chat/completions` 地址填进 `base_url`，导致 SDK 再次拼接路径；
- 把官方 API 地址和第三方兼容服务的 API Key 混用；
- 使用网页控制台地址，而不是 API 地址；
- 忘记服务商要求的版本路径，或重复写成 `/v1/v1`；
- 服务商支持 Chat Completions，但客户端配置成了服务商并未提供的其他 API 类型；
- API Key 来自服务 A，却把请求发到了服务 B 的 Base URL。

OpenAI 官方 API 参考文档说明了认证方式、请求路径和请求 ID 等通用行为；对于兼容服务，具体路径、请求格式和支持的 API 类型应以对应服务的公开文档为准。先用 `/models` 或服务商明确提供的检查接口，再测试聊天请求，可以减少同时变化的配置项。([developers.openai.com](https://developers.openai.com/api/reference/overview))

## 六、检查密钥状态、项目和组织归属

即使密钥字符串看起来正确，也可能因为以下原因返回 401：

- 密钥被删除、撤销、过期或停用；
- 当前程序仍在使用旧密钥，环境变量或 Secret 更新没有被新进程读取；
- 密钥属于另一个组织或项目；
- 请求使用的项目与创建密钥时选择的项目不一致；
- 当前账号已经不再属于原组织或项目；
- 项目启用了 IP 白名单，但请求没有从允许的公网出口 IP 发出。

OpenAI 官方错误码文档列出了无效密钥、组织成员资格和 IP 未授权等 401 原因。启用 IP allowlist 后，即使 API Key 有效，只要请求来源 IP 不在允许范围内，也可能返回 `401` 和 `ip_not_authorized`。([developers.openai.com](https://developers.openai.com/api/docs/guides/error-codes))

如果刚刚修改了密钥、项目权限或 IP 白名单，不要立即假设配置没有生效。官方文档提示，部分认证相关更新可能需要传播时间；IP 白名单页面则说明配置变更可能最多需要等待约 15 分钟。等待期间不要在日志中打印完整密钥，也不要为了“测试”把密钥发给他人。([developers.openai.com](https://developers.openai.com/api/reference/overview))

## 七、在 SDK 和可视化工具中定位配置错误

### Python SDK

重点检查 `api_key`、`base_url` 和 `model` 三个值。下面示例不会把密钥写死在源代码中：

```python
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["API_KEY"],
    base_url="https://api.example.com/v1%22,
)

response = client.chat.completions.create(
    model="YOUR_MODEL",
    messages=[{"role": "user", "content": "请回复 OK"}],
)

print(response.choices[0].message.content)
```

如果 `/models` 可以成功，但聊天请求失败，通常应继续检查模型名、接口路径、请求体格式和该模型的访问权限，而不是继续更换 API Key。

### Node.js SDK

```javascript
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: process.env.API_KEY,
  baseURL: "https://api.example.com/v1%22,
});

const response = await client.chat.completions.create({
  model: "YOUR_MODEL",
  messages: [{ role: "user", content: "请回复 OK" }],
});

console.log(response.choices[0].message.content);
```

### 桌面客户端和可视化工具

在 Cursor、Cherry Studio、LobeChat、NextChat 或 Dify 中，可以按照下面顺序检查：

1. Provider 类型是否选择了 OpenAI Compatible 或工具对应的兼容接口类型；
2. API Key 是否包含前后空格、换行、引号或复制时带入的其他字符；
3. Base URL 是否填写为 API 地址，而不是网页地址；
4. Base URL 是否按工具要求填写版本路径，是否出现重复 `/v1`；
5. 模型名称是否来自当前服务实际支持或返回的模型列表；
6. 测试请求是否使用了当前配置，而不是旧的默认 Provider；
7. 如果工具支持查看请求日志，应只检查请求地址、状态码和错误类型，避免导出完整 Authorization 请求头。

## 常见问题

### 1. API Key 明明正确，为什么还是 401？

先确认“正确”指的是当前请求实际使用的密钥，而不是你记忆中的密钥。检查 Shell、IDE、容器、部署平台和 CI/CD Secret 中的变量值是否一致，确认密钥没有被删除或撤销，也没有在复制时带入空格和换行。如果服务启用了项目或 IP 限制，还要确认当前请求属于正确项目并从允许的出口 IP 发出。

### 2. 401 和 403 有什么区别？

`401` 主要表示认证没有通过，例如密钥缺失、密钥无效或来源 IP 未获授权；`403` 通常表示服务已经识别请求者，但请求者没有访问某项资源的权限，或者请求受到区域或组织策略限制。实际排查时仍应以响应中的 `error.code` 和 `error.message` 为准，因为不同兼容服务的错误结构可能不同。([developers.openai.com](https://developers.openai.com/api/docs/guides/error-codes))

### 3. `/models` 返回 200，但聊天请求仍然失败，说明什么？

这通常说明当前密钥和基础地址至少能够完成一次认证，但聊天请求还可能存在模型名、接口路径、请求体格式或模型权限问题。继续检查 `model` 是否确实存在、`messages` 是否为数组，以及服务是否支持你调用的 API 类型。不要因为 `/models` 成功，就认为所有模型和所有接口都自动可用。

### 4. 401 是否可以通过重试解决？

通常不应该盲目重试。无效密钥、错误 Bearer 格式、错误项目归属和未授权 IP 都属于配置或凭据问题，重复发送相同请求不会使配置自动正确。只有在刚刚完成密钥、项目或 IP 白名单变更，并且已经确认配置内容正确时，才可以等待传播后重新测试。对于明确的认证错误，应先修复配置；对于临时网络错误或部分服务端错误，才考虑根据官方建议进行有限重试。([developers.openai.com](https://developers.openai.com/api/docs/guides/error-codes))

### 5. 排查时应该提供哪些信息？

向服务商反馈时，可以提供 HTTP 状态码、错误类型、错误代码、请求时间、时区、请求路径、使用的模型以及响应中的 request ID。不要提供完整 API Key、完整 `Authorization` 请求头、用户隐私数据或业务提示词。OpenAI API 参考文档说明，`x-request-id` 可用于定位单次请求；如果请求因为网络或超时没有获得响应，也可以记录客户端请求 ID，具体以服务实现为准。([developers.openai.com](https://developers.openai.com/api/reference/overview))

## 排查清单

可以按照下面顺序快速复核：

- [ ] HTTP 状态码确实是 `401`；
- [ ] 请求头格式为 `Authorization: Bearer API_KEY`；
- [ ] Shell、IDE、容器或 CI/CD 中的密钥变量非空；
- [ ] 代码读取的变量名与实际设置的变量名一致；
- [ ] 密钥没有前后空格、引号或换行；
- [ ] Base URL 指向正确的 API 服务，没有重复版本路径；
- [ ] API Key 与 Base URL 属于同一个服务；
- [ ] 密钥未被删除、撤销或停用；
- [ ] 组织、项目和权限配置正确；
- [ ] IP 白名单包含当前请求的公网出口 IP；
- [ ] `/models` 和实际业务请求使用的是同一套配置；
- [ ] 日志、截图和错误报告中没有泄露完整密钥。

## 参考资料

- https://developers.openai.com/api/docs/guides/error-codes
- https://developers.openai.com/api/reference/overview
- https://developers.openai.com/api/reference/typescript
- https://developers.openai.com/api/docs/quickstart
- https://developers.openai.com/api/docs/guides/ip-allowlist
- https://help.openai.com/en/articles/4936850-where-do-i-find-my-openai-api-key
- https://code.claude.com/docs/en/llm-gateway

## 关于 犀速-AI

犀速-AI 提供兼容 OpenAI API 的模型接入方式，支持通过 Base URL 和 API Key 接入 SDK、客户端及开发工具。更多平台信息与使用入口请访问官网：https://xisu.ai/%E3%80%82