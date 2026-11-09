> 本文由 [犀速-AI](https://xisu.ai/) 整理，更新于 2026-11-09。

# Node.js OpenAI SDK Responses API：最小请求与 output_text、_request_id 读取

**一句话结论：**在 Node.js 中使用官方 `openai` SDK 调用 Responses API 时，核心写法是 `client.responses.create({ model, input })`，文本结果优先读取 `response.output_text`，成功响应对应的请求追踪 ID 则读取 `response._request_id`。

## 适用场景

本文适合以下情况：

- 正在用 Node.js 或 TypeScript 接入 OpenAI 兼容 API；
- 想从 Chat Completions 切换到 Responses API；
- 不确定 Responses API 返回结果应该读取 `output` 还是 `output_text`；
- 需要在日志中记录一次成功请求的 request ID，便于后续定位问题；
- 使用犀速-AI等兼容服务时，希望先用一个最小请求确认 Base URL、API Key、模型名和接口路径是否配置正确。

本文只讲一个最小请求及其两个常用字段，不涉及流式输出、工具调用、会话状态、自动重试和多种 HTTP 错误码的综合排查。模型名称、Base URL 和 API Key 是否可用，仍需以实际服务控制台或服务商文档为准。

## 第 1 步：准备 Node.js 项目

先创建一个新的目录并初始化项目：

```bash
mkdir openai-responses-demo
cd openai-responses-demo
npm init -y
npm install openai
```

官方 `openai-node` 仓库提供了 JavaScript/TypeScript SDK 的安装与使用示例。当前官方文档列出的运行时要求包括 Node.js 22 和 24 LTS；如果你的项目使用其他 Node.js 版本，应先根据目标 SDK 版本的要求核实兼容性。([github.com](https://github.com/openai/openai-node/blob/main/README.md))

为了直接运行下面的 ES Module 示例，在 `package.json` 中加入：

```json
{
  "type": "module"
}
```

如果你的项目已经使用 CommonJS，也可以改写为对应的 `require` 形式；本文代码采用官方文档常见的 `import OpenAI from 'openai'` 写法。

## 第 2 步：准备环境变量

不要把密钥直接写入源代码、提交到 Git 仓库或放进前端浏览器代码。可以在当前终端设置环境变量：

```bash
export OPENAI_API_KEY="替换为你的 API Key"
export OPENAI_BASE_URL="https://xn--6qq44cezan7kkc165a9mv3i2a/v1%22
export OPENAI_MODEL="替换为服务端实际可用的模型名"
```

Windows PowerShell 示例：

```powershell
$env:OPENAI_API_KEY = "替换为你的 API Key"
$env:OPENAI_BASE_URL = "https://xn--6qq44cezan7kkc165a9mv3i2a/v1%22
$env:OPENAI_MODEL = "替换为服务端实际可用的模型名"
```

`OPENAI_BASE_URL` 不是固定值。直连 OpenAI 时可以不设置它，让 SDK 使用默认地址；接入 OpenAI 兼容服务时，应填写服务商提供的 Base URL，并确认是否已经包含 `/v1`。如果服务商给出的地址本身已经带有版本路径，不要再重复拼接。

`OPENAI_MODEL` 也不要照抄示例中的模型名。Responses API 的请求体需要提供 `model`，但具体模型是否存在、是否支持 Responses API，需要以目标平台的模型列表或文档为准。

## 第 3 步：编写 Responses API 最小请求

新建 `index.js`：

```js
import OpenAI from 'openai';

const client = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
  ...(process.env.OPENAI_BASE_URL
    ? { baseURL: process.env.OPENAI_BASE_URL }
    : {}),
});

const model = process.env.OPENAI_MODEL;

if (!process.env.OPENAI_API_KEY) {
  throw new Error('缺少 OPENAI_API_KEY');
}

if (!model) {
  throw new Error('缺少 OPENAI_MODEL');
}

const response = await client.responses.create({
  model,
  input: '用一句话解释 JavaScript 中 Promise 的作用。',
});

console.log('回答：');
console.log(response.output_text);

console.log('请求 ID：');
console.log(response._request_id ?? '响应中未提供 request ID');
```

运行：

```bash
node index.js
```

官方 Responses API 示例使用 `client.responses.create()` 创建响应，并通过 `response.output_text` 取得聚合后的文本输出。SDK 源码也会在返回对象属于 Response 时补充文本输出字段，因此对只需要模型文字回答的场景，直接读取 `output_text` 比手动假设 `output[0]` 一定是某种消息结构更稳妥。([github.com](https://github.com/openai/openai-node/blob/main/docs/responses.md))

## 第 4 步：理解 `output_text` 与 `output` 的区别

Responses API 的响应包含有序的 `output` 项。不同请求可能产生消息、推理、工具调用或其他输出项，因此不建议无条件写成下面这样：

```js
const text = response.output[0].content[0].text;
```

这种写法依赖固定的数组位置和内容结构，遇到工具调用、不同输出类型或响应结构变化时容易失效。

如果你的目标只是打印最终文本，应优先使用：

```js
console.log(response.output_text);
```

只有在需要判断输出项类型、读取工具调用、处理注释或保留完整响应结构时，才进一步检查：

```js
for (const item of response.output) {
  console.log(item.type);
}
```

官方 Node.js SDK 文档明确区分了这两种用途：`output_text` 用于合并响应中的文本输出；需要底层消息、推理或工具调用项时，再读取 `output`。([github.com](https://github.com/openai/openai-node/blob/main/docs/responses.md))

## 第 5 步：读取成功响应中的 `_request_id`

成功响应可以这样记录：

```js
const response = await client.responses.create({
  model,
  input: '返回一个简短的问候语。',
});

console.log({
  requestId: response._request_id,
  text: response.output_text,
});
```

官方 SDK README 说明，SDK 对象响应提供 `_request_id`，它来自响应头中的 `x-request-id`。这个字段适合写入应用日志，之后向服务商反馈问题时，可以同时提供请求时间、模型名、HTTP 状态和 request ID，但不要记录或上传完整 API Key。([github.com](https://github.com/openai/openai-node/blob/main/README.md))

需要注意，`_request_id` 是成功解析后的响应对象字段；它与 API 错误对象上的属性不是同一个概念。本文不建议根据其他 SDK、旧版本示例或第三方封装猜测错误对象字段名。排查失败请求时，应优先查看目标 SDK 版本公开文档、类型定义和实际响应头。

## 第 6 步：同时读取 HTTP 状态和 request ID

如果你需要同时取得解析后的数据和底层 Web `Response`，可以使用 SDK 提供的 `withResponse()`：

```js
const { data, response: rawResponse, request_id } = await client.responses
  .create({
    model,
    input: '返回一个简短的测试结果。',
  })
  .withResponse();

console.log('HTTP 状态：', rawResponse.status);
console.log('Request ID：', request_id);
console.log('文本：', data.output_text);
```

官方配置文档说明，`withResponse()` 会同时返回解析后的数据、底层 Web `Response` 和 `request_id`；如果只需要解析后的成功响应，也可以直接读取 `data._request_id`。([github.com](https://github.com/openai/openai-node/blob/main/docs/configuration.md))

这个写法适合接入日志系统或内部诊断代码。例如，你可以只记录如下非敏感信息：

```js
console.log(JSON.stringify({
  status: rawResponse.status,
  requestId: request_id,
  model,
  hasText: Boolean(data.output_text),
}));
```

不要在日志中输出 `OPENAI_API_KEY`，也不要为了排查问题把完整授权请求头粘贴到工单、Issue 或聊天窗口。

## 常见问题

### 1. `response.output_text` 是 `undefined`，应该读取什么？

先确认你调用的是 `client.responses.create()`，而不是 `client.chat.completions.create()`。两套接口的响应结构不同：Responses API 使用 `output_text`，Chat Completions 通常从 `choices` 中读取内容。

还要确认 SDK 版本与代码示例匹配，并打印一次经过脱敏处理的响应结构：

```js
console.dir(response, { depth: 4 });
```

如果服务商只是部分兼容 Responses API，可能返回了不同结构。此时应查看该服务商的兼容范围，而不是直接把 Chat Completions 的字段路径套到 Responses API 上。

### 2. `response._request_id` 没有值，是请求失败了吗？

不一定。`_request_id` 来源于响应头中的 `x-request-id`；如果兼容服务没有返回这个响应头，字段可能为空。应同时检查 `response.output_text`、底层 HTTP 状态以及服务端日志。

如果需要更明确地确认底层响应头，可以使用 `withResponse()`：

```js
const { data, response: raw } = await client.responses
  .create({ model, input: '测试' })
  .withResponse();

console.log(raw.status);
console.log(raw.headers.get('x-request-id'));
console.log(data.output_text);
```

### 3. Base URL 应该写到 `/v1` 还是域名根路径？

不能凭猜测决定。应以目标服务的 OpenAI 兼容 API 文档为准，并确认 SDK 会在 Base URL 后拼接的资源路径。官方 SDK 的 Responses 方法请求路径是 `/responses`；如果 Base URL 配错为重复版本路径或缺少必要版本路径，可能导致请求地址不正确。([github.com](https://github.com/openai/openai-node/blob/main/src/resources/responses/responses.ts))

建议先在服务商控制台复制完整 Base URL，再用一个最小请求验证，不要同时加入流式、工具调用、长上下文等额外参数。

### 4. 为什么模型名正确，但仍然无法调用？

模型名可用不等于该模型支持 Responses API。请分别核对：模型是否存在、当前 API Key 是否有权限、服务端是否实现 `/v1/responses`、模型是否支持该接口，以及 Base URL 是否指向正确的 API 版本。

本文示例使用环境变量传入模型名，正是为了避免把某个服务商或某个时间点的模型 ID 当成所有平台都有效的固定值。

### 5. 能不能把 API Key 直接放在浏览器前端？

不建议。官方 SDK 默认禁止在浏览器中直接使用，以避免把密钥暴露给用户；浏览器中的代码、网络请求和构建产物都可能被查看。更安全的做法是由自己的后端保存密钥，再由后端调用兼容 API。([github.com](https://github.com/openai/openai-node))

## 参考资料

以下资料均为本次检索并打开核对的官方资料：

1. [openai-node README：Request IDs 与错误处理](https://github.com/openai/openai-node/blob/main/README.md)
2. [openai-node Responses API 文档](https://github.com/openai/openai-node/blob/main/docs/responses.md)
3. [openai-node 配置文档：Errors、withResponse 与 request ID](https://github.com/openai/openai-node/blob/main/docs/configuration.md)
4. [openai-node Responses 资源源码](https://github.com/openai/openai-node/blob/main/src/resources/responses/responses.ts)

## 关于 犀速-AI

犀速-AI 提供面向开发者的 OpenAI 兼容 API 接入服务与使用文档，帮助用户配置 Base URL、API Key 并接入常见开发工具。具体可用模型、接口能力和配置方式请以官网及账户页面显示内容为准。

官网：[https://xisu.ai/](https://xisu.ai/)