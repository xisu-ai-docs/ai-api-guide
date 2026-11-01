> 本文由 [犀速-AI](https://xisu.ai/) 整理，更新于 2026-11-01。

# Node.js OpenAI SDK 401 invalid_api_key：request ID 读取与排查

**一句话结论：** 遇到 `401 invalid_api_key` 时，先确认请求实际使用的 API Key、Base URL 和组织配置；在当前 `openai-node` 文档中，失败请求可读取 `error.requestID`，也可从 `error.headers` 的 `x-request-id` 响应头获取诊断信息，而 `withResponse()` 场景应优先通过原始 `Response` 的响应头读取请求 ID，避免依赖不同版本或二次封装暴露的固定字段。([github.com](https://github.com/openai/openai-node/blob/main/docs/configuration.md))

## 适用场景

本文只讨论一个 HTTP 状态码：**401 Unauthorized**，重点处理错误信息中出现以下内容的情况：

- `invalid_api_key`；
- `Incorrect API key provided`；
- “API 密钥不正确”或类似认证失败提示。

适合以下场景：

- Node.js 项目使用官方 `openai` SDK 调用 Responses API 或 Chat Completions API；
- 项目通过环境变量配置 API Key、Base URL、组织或项目 ID；
- 需要区分“环境变量没有加载”“Key 已失效”“Base URL 与 Key 不匹配”等问题；
- 请求已经返回 401，需要把 request ID 提供给接口服务商支持人员；
- 日志中出现 `request_id`、`requestID`、`_request_id` 多种写法，不确定应该读取哪个字段。

本文不讨论 403、404、429、500、网络超时和模型能力限制。官方 Node.js SDK 将 HTTP 401 映射为 `AuthenticationError`，而 404、429 等状态码对应其他错误类型；因此排查时应先确认状态码，不要只根据一段错误文本判断。([github.com](https://github.com/openai/openai-node/blob/main/docs/configuration.md))

## 第一步：先处理环境变量缺失，再创建 SDK 客户端

`new OpenAI()` 或 `new OpenAI({ apiKey })` 可能在 API Key 没有提供时直接抛出配置错误。如果客户端创建语句位于 `try` 外部，后续请求错误的 `catch` 不会覆盖这个阶段。因此，诊断代码应先显式检查变量是否存在，并把客户端创建放入异常处理范围内。

下面的示例只读取已经配置的环境变量，不在源码中赋值、打印完整密钥，也不输出 `Authorization` 请求头：

```js
import OpenAI from "openai";

async function main() {
  const apiKey = process.env.OPENAI_API_KEY;

  if (!apiKey) {
    console.error("OPENAI_API_KEY: missing");
    process.exitCode = 1;
    return;
  }

  try {
    const client = new OpenAI({
      apiKey,
      baseURL: process.env.OPENAI_BASE_URL || undefined,
      organization: process.env.OPENAI_ORG_ID || undefined,
      project: process.env.OPENAI_PROJECT_ID || undefined,
    });

    await client.responses.create({
      model: process.env.OPENAI_MODEL,
      input: "Return only the word OK.",
    });

    console.log("request succeeded");
  } catch (error) {
    if (error instanceof OpenAI.APIError) {
      const requestId =
        error.requestID ??
        error.headers?.get("x-request-id") ??
        null;

      console.error({
        status: error.status,
        name: error.name,
        request_id: requestId,
      });
    } else {
      console.error({
        name: error?.name,
        message: "request failed before an API response was received",
      });
    }

    process.exitCode = 1;
  }
}

main();
```

当前官方 Node.js SDK 文档将 `error.requestID`、`error.status` 和 `error.headers`列为 API 错误的可用信息；其中 `headers` 使用 Web 标准 `Headers` 接口时，可以通过 `.get("x-request-id")`读取响应头。([github.com](https://github.com/openai/openai-node/blob/main/docs/configuration.md))

如果使用的是第三方 OpenAI 兼容客户端、旧版本 SDK 或经过封装的请求层，不要假定它一定提供相同字段。应以项目实际安装版本的公开类型定义和官方文档为准。

## 第二步：区分 `requestID`、`request_id` 和 `_request_id`

这几个名称表示的上下文不同，不能直接混用。

### 1. 失败的 API 异常：优先看 `error.requestID`

对当前官方 `openai-node` 而言，API 请求返回非成功状态码后，异常对象属于 `OpenAI.APIError` 或其子类。官方错误处理示例使用 `error.requestID`读取请求 ID：

```js
try {
  await client.responses.create({
    model: process.env.OPENAI_MODEL,
    input: "Hello",
  });
} catch (error) {
  if (error instanceof OpenAI.APIError) {
    console.error({
      status: error.status,
      type: error.name,
      request_id: error.requestID ?? null,
    });
  }
}
```

如果 `error.requestID`为空，可以继续检查响应头：

```js
const requestId =
  error.requestID ??
  error.headers?.get("x-request-id") ??
  null;
```

请求 ID缺失并不等于 API Key 一定无效。它是服务端响应中的诊断信息，可能因为目标服务、中间代理或兼容接口没有返回该响应头而为空。判断 401 时，仍应以 `error.status`、错误类型和服务端错误内容为主。

### 2. 成功的解析结果：使用 `_request_id`

官方 Node.js SDK 文档说明，成功返回的解析结果可以暴露 `_request_id`，其值来自响应的 `x-request-id`响应头：

```js
const response = await client.responses.create({
  model: process.env.OPENAI_MODEL,
  input: "Return only OK.",
});

console.log(response._request_id ?? null);
```

这个字段属于成功返回的解析结果，不应写成 401 异常对象上的固定属性：

```js
// 不要假设 401 异常存在 response 对象
console.log(response._request_id);
```

### 3. `.withResponse()`：从原始 `Response` 读取响应头

`.withResponse()`用于同时取得解析后的数据和原始 Web `Response`。当前 SDK 文档示例还展示了返回对象中的 `request_id`，但不同 SDK 版本、兼容客户端或二次封装对该字段的暴露方式可能不同。为了直接依赖标准 Web API，建议在需要读取响应头时使用 `response.headers.get("x-request-id")`：([github.com](https://github.com/openai/openai-node/blob/main/docs/configuration.md))

```js
const { data, response } = await client.responses
  .create({
    model: process.env.OPENAI_MODEL,
    input: "Return only OK.",
  })
  .withResponse();

const requestId = response.headers.get("x-request-id");

console.log({
  status: response.status,
  request_id: requestId,
  output: data.output_text,
});
```

因此，不建议把 `.withResponse()`的结果固定写成只包含某个 `request_id`字段，也不要把 `error.requestID`、`response._request_id` 和原始响应头当作同一个对象属性。更稳妥的区分方式如下：

| 场景 | 建议读取方式 |
| --- | --- |
| 失败的 Node.js API 异常 | `error.requestID`，必要时回退到 `error.headers?.get("x-request-id")` |
| 成功的解析结果 | `response._request_id` |
| `.withResponse()`返回值 | `response.headers.get("x-request-id")` |
| 直接取得原始响应 | `rawResponse.headers.get("x-request-id")` |

## 第三步：确认进程实际加载的配置

API Key 配置正确，不代表当前运行进程就一定拿到了正确值。建议在启动阶段只记录变量是否存在，不记录变量内容：

```js
for (const name of [
  "OPENAI_API_KEY",
  "OPENAI_BASE_URL",
  "OPENAI_ORG_ID",
  "OPENAI_PROJECT_ID",
  "OPENAI_MODEL",
]) {
  console.log(`${name}: ${process.env[name] ? "configured" : "missing"}`);
}
```

重点检查以下情况：

1. 代码读取的是 `OPENAI_API_KEY`，但部署环境配置成了其他变量名；
2. 修改 `.env`后没有重启 Node.js 进程；
3. Shell、IDE、Docker、CI 或进程管理器中存在旧变量；
4. 应用的不同模块分别创建了客户端，并读取了不同的 Key；
5. `OPENAI_BASE_URL`指向了另一个服务，但仍使用原服务的 Key；
6. `OPENAI_ORG_ID`或 `OPENAI_PROJECT_ID`被错误继承，导致请求使用了非预期上下文。

OpenAI 帮助中心建议核对当前请求实际使用的 API Key，确认应用没有在多个 Key 之间切换，并在需要时确认组织设置。([help.openai.com](https://help.openai.com/en/articles/6882433-incorrect-api-key-provided))

如果你的应用通过犀速-AI提供的 OpenAI 兼容接口接入，应将服务商给出的 Base URL 配置到 `OPENAI_BASE_URL`，并将同一服务对应的 Key 配置到 `OPENAI_API_KEY`，不要交叉组合不同服务的地址和密钥。

## 第四步：用最小请求验证认证配置

认证排查时，先移除流式输出、工具调用、文件上传、复杂消息历史和自定义中间件，只保留一个普通文本请求。这样可以减少请求体和客户端配置带来的变量：

```js
import OpenAI from "openai";

async function verify() {
  if (!process.env.OPENAI_API_KEY) {
    console.error("OPENAI_API_KEY: missing");
    process.exitCode = 1;
    return;
  }

  try {
    const client = new OpenAI({
      apiKey: process.env.OPENAI_API_KEY,
      baseURL: process.env.OPENAI_BASE_URL || undefined,
    });

    const result = await client.chat.completions.create({
      model: process.env.OPENAI_MODEL,
      messages: [{ role: "user", content: "Say OK." }],
    });

    console.log({
      output: result.choices?.[0]?.message?.content ?? null,
      request_id: result._request_id ?? null,
    });
  } catch (error) {
    if (error instanceof OpenAI.APIError) {
      console.error({
        status: error.status,
        type: error.name,
        request_id:
          error.requestID ??
          error.headers?.get("x-request-id") ??
          null,
      });
    } else {
      console.error("request failed before an API response was received");
    }

    process.exitCode = 1;
  }
}

verify();
```

如果这个最小请求返回 401，应继续检查 Key、Base URL、组织和项目配置。如果最小请求成功，而包含复杂参数的请求失败，则问题未必是认证，应另行检查模型、请求体或兼容接口支持范围。

## 第五步：无法确认 Key 时，创建新 Key 并更新配置

不要尝试从日志、代码仓库或历史构建产物中恢复完整密钥。OpenAI 帮助中心说明，完整 Secret Key 通常只在创建时显示；如果丢失或没有保存，应创建新的 Key 并更新应用配置，已删除的 Key 不能继续使用。([help.openai.com](https://help.openai.com/en/articles/4936850-where-do-i-find-my-api-key))

建议按以下顺序处理：

1. 在对应的 API Key 管理页面确认当前使用的 Key 来源；
2. 确认运行环境实际加载的是目标环境变量；
3. 如果无法确认 Key 内容或状态，创建新的 Key；
4. 只更新服务端的 Secret 配置，不把 Key 写入 Git、前端代码、工单或 issue；
5. 重启实际处理请求的 Node.js 进程、容器或任务；
6. 使用最小请求重新验证；
7. 若仍然返回 401，记录状态码、错误类型、发生时间和 request ID，提交给接口服务商支持人员，但不要提交密钥。

## 常见问题

### 1. 为什么 `error.request_id` 是 `undefined`？

在当前官方 Node.js SDK 的错误处理示例中，异常对象使用的是 `error.requestID`，不是 `error.request_id`。可以先读取 `error.requestID`，再使用 `error.headers?.get("x-request-id")`作为响应头回退路径。若使用第三方客户端，应以该客户端的公开类型定义为准。

### 2. 为什么 `.withResponse()`不能只按 `request_id`字段处理？

`.withResponse()`返回的是包含解析数据和原始 Web `Response`的结果。当前官方文档和测试示例展示了 `request_id`字段，但不同版本或封装层可能改变可直接访问的字段；而 `response.headers.get("x-request-id")`是原始响应的标准读取方式。需要兼容多个客户端时，优先读取原始响应头。([github.com](https://github.com/openai/openai-node/blob/main/tests/responses.test.ts))

### 3. `error.requestID`为空，是否说明 API Key 无效？

不一定。request ID只是诊断信息，服务端或中间层没有返回 `x-request-id`时可能为空。应结合 `error.status`、`error.name`、错误代码和实际请求地址判断，不能仅凭 request ID缺失判定 Key 无效。

### 4. 401 和 403 是否可以用同一套方法处理？

不应混为一谈。本文只处理 401 认证失败。401 时优先核对 Key、Base URL 和组织配置；403 属于另一类权限或访问限制问题，应按照目标服务的权限说明独立排查。

### 5. 可以把完整 `error.message` 写入共享日志吗？

不建议。认证错误消息有时可能包含经过部分隐藏的 Key 片段，调试日志也可能包含请求上下文。生产日志建议保留状态码、错误类型、时间、目标服务标识和 request ID，并避免输出完整密钥、Authorization 请求头及未经脱敏的错误对象。

### 6. 修改环境变量后为什么仍然返回 401？

常见原因是重启了错误的进程、容器没有重新创建、CI Secret 没有更新，或者进程管理器继续使用旧环境。可以在启动阶段输出变量“configured/missing”状态，并在一次失败请求中记录 `error.status`、`error.name`和 request ID，以确认实际执行路径。

## 参考资料

1. https://github.com/openai/openai-node/blob/main/docs/configuration.md
2. https://github.com/openai/openai-node/blob/main/README.md
3. https://github.com/openai/openai-node/blob/main/tests/responses.test.ts
4. https://help.openai.com/en/articles/6882433-incorrect-api-key-provided
5. https://help.openai.com/en/articles/4936850-where-do-i-find-my-api-key

## 关于 犀速-AI

犀速-AI提供面向开发者的 OpenAI 兼容 API 接入服务与配置指引，适合在服务端应用、Node.js 项目和常见 AI 工具中管理接口配置。更多接入方式与平台说明，请访问[官网](https://xisu.ai/)。