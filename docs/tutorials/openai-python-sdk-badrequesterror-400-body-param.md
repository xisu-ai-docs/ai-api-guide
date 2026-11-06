> 本文由 [犀速-AI](https://xisu.ai/) 整理，更新于 2026-11-06。

# OpenAI Python SDK BadRequestError 怎么排查：400 错误读取 body 与 param

**一句话结论：** OpenAI Python SDK 返回 `BadRequestError` 时，先确认 HTTP 状态码确实是 `400`，再读取异常对象中的 `body`、`code`、`param` 和 `type`；不要只看异常字符串，也不要通过打印请求头排查。

## 适用场景

本文只讨论一个问题：**使用 OpenAI Python SDK 调用 OpenAI 兼容 API 时，服务端返回 HTTP 400，SDK 抛出 `BadRequestError`，如何判断究竟是哪一个请求参数不符合接口要求。**

适合以下情况：

- Python 项目已经安装并使用 `openai` SDK；
- 使用官方 OpenAI API，或使用兼容 OpenAI API 格式的服务；
- 已经配置了 API Key、Base URL 和模型名称；
- 请求能够到达服务端，但服务端返回 400，而不是网络连接失败、401、403、404 或 429；
- 希望在日志中记录可用于定位的错误字段，同时避免泄露密钥。

OpenAI Python SDK 文档将非成功的 4xx、5xx 响应归为 `APIStatusError`，其中 HTTP 400 对应 `BadRequestError`。SDK 的异常对象还包含 `status_code`、`response` 等属性；异常实现中也公开了 `body`、`code`、`param` 和 `type` 等字段。([github.com](https://github.com/openai/openai-python))

## 第一步：确认当前调用确实使用了 Python SDK

先确认依赖已经安装，并查看项目实际使用的包版本。版本不同，异常层级和部分辅助属性可能存在差异，因此排查时应以当前环境中的公开文档和类型定义为准。

```bash
python -m pip install --upgrade openai
python -c "import openai; print(openai.__version__)"
```

不要把 API Key 直接写入源代码、提交到 Git 仓库，或在日志中输出。官方快速开始文档示例使用环境变量保存密钥，SDK 会自动读取环境变量中的认证信息。([platform.openai.com](https://platform.openai.com/docs/quickstart))

例如，在 macOS 或 Linux 中可以先设置变量：

```bash
export XISU_API_KEY="从服务控制台获取的密钥"
export XISU_BASE_URL="从服务文档获取的兼容 API Base URL"
export XISU_MODEL="从服务控制台或模型列表确认的模型 ID"
```

Windows PowerShell 可以使用：

```powershell
$env:XISU_API_KEY = "从服务控制台获取的密钥"
$env:XISU_BASE_URL = "从服务文档获取的兼容 API Base URL"
$env:XISU_MODEL = "从服务控制台或模型列表确认的模型 ID"
```

上面的 Base URL 和模型 ID 只是环境变量名称，不代表固定值。接入犀速-AI 时，应将它们替换为犀速-AI 控制台或接入文档中实际提供的配置。

## 第二步：使用一个最小的 Chat Completions 请求

为了避免业务代码、工具调用、结构化输出、图片输入等因素干扰，先使用最小请求。下面的代码只从环境变量读取配置，不赋值保存密钥，也不打印请求头。

```python
import os

from openai import OpenAI

client = OpenAI(
    api_key=os.environ["XISU_API_KEY"],
    base_url=os.environ["XISU_BASE_URL"],
)

response = client.chat.completions.create(
    model=os.environ["XISU_MODEL"],
    messages=[
        {"role": "user", "content": "请只回复：测试成功"},
    ],
)

print(response.choices[0].message.content)
```

这个请求的作用不是测试所有能力，而是先验证最基本的请求结构是否能被当前兼容接口接受。如果它返回 400，优先检查请求体字段、模型能力和 Base URL 对应的接口类型；如果最小请求成功，再逐项恢复原有参数。

需要注意，OpenAI 兼容并不一定代表所有服务都支持完全相同的字段。某些参数可能只适用于特定模型或特定接口，是否支持应以服务方文档为准，不能仅依据参数名称推断。

## 第三步：捕获 BadRequestError 并读取关键字段

推荐单独捕获 `BadRequestError`，同时保留一个更宽泛的 `APIStatusError` 分支，用于处理其他 HTTP 状态码。本文重点是 400，因此不要把所有异常都简单转换成“密钥错误”或“网络错误”。

```python
import os

import openai
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["XISU_API_KEY"],
    base_url=os.environ["XISU_BASE_URL"],
)

try:
    response = client.chat.completions.create(
        model=os.environ["XISU_MODEL"],
        messages=[
            {"role": "user", "content": "请只回复：测试成功"},
        ],
    )
    print(response.choices[0].message.content)

except openai.BadRequestError as exc:
    print("HTTP status:", exc.status_code)
    print("error code:", exc.code)
    print("error param:", exc.param)
    print("error type:", exc.type)
    print("error body:", exc.body)

except openai.APIStatusError as exc:
    print("HTTP status:", exc.status_code)
    print("API status error:", type(exc).__name__)
```

对于 400，正常情况下 `exc.status_code` 应为 `400`。`exc.body` 是服务端返回的错误正文；如果正文是可解析的 JSON，SDK 会保留为结构化对象，否则可能是原始文本。`exc.code`、`exc.param` 和 `exc.type` 则取决于服务端是否在错误响应中提供这些字段。([github.com](https://github.com/openai/openai-python/blob/main/src/openai/_exceptions.py))

## 第四步：按字段判断问题位置

### 1. `status_code`：先确认是不是 400

`status_code` 用于确认异常类型。不要因为异常类名中出现 `BadRequest`，就跳过状态码检查。若状态码是 401，应转到认证排查；若是 404，应检查路径或模型；若是 429，则属于限流或用量问题。

本文只覆盖 `status_code == 400` 的情况。

### 2. `body`：优先阅读服务端返回的完整错误正文

`body` 通常是最有价值的字段。常见信息包括：

- 哪个 JSON 字段不符合要求；
- 参数类型错误，例如字符串、数字、数组或对象类型不匹配；
- 某个参数不支持当前模型或接口；
- 请求缺少服务端要求的必填字段；
- 服务端给出的错误说明。

排查时可以记录 `body`，但要注意正文可能包含用户输入、文件名或业务数据。生产环境不应无条件把完整正文写入公共日志，建议先做脱敏或限制日志访问权限。

### 3. `param`：定位具体参数名

如果服务端返回了 `param`，它通常可以帮助你快速定位问题字段，例如 `model`、`messages`、`temperature` 或某个工具配置字段。这里的“通常”很重要：`param` 是否存在、格式如何，取决于服务端的错误响应，兼容 API 不一定全部返回该字段。

当 `param` 不为空时，建议只修改这个参数并重新发送最小请求，而不是同时改动 Base URL、模型、消息内容和其他选项。这样更容易确认真正原因。

### 4. `code` 与 `type`：区分错误类别，但不要混淆两者

`type` 往往是较宽泛的错误类别，`code` 可能是更具体的机器可读标识。两者都可能为空，也可能由不同服务定义不同值。

因此，代码中不要把某个第三方服务的 `code` 当成所有 OpenAI 兼容服务都必须返回的固定枚举。更稳妥的做法是：先判断 HTTP 状态码，再对 `code`、`param` 和 `body` 做存在性检查。

## 第五步：用“逐项恢复”定位 400

如果最小请求成功，而原始业务请求失败，可以按照下面的顺序逐项恢复：

1. 先只保留 `model` 和一条文本消息；
2. 恢复原始系统提示词；
3. 恢复多轮 `messages`；
4. 恢复 `temperature`、`max_tokens` 等普通参数；
5. 最后恢复工具调用、JSON Schema、图片、音频或供应商专属参数。

每次只恢复一组字段，并保留上一次可成功的版本。这样可以将“400”缩小为一个具体字段，而不是在多个参数同时变化时凭经验猜测。

如果错误正文明确指出某个参数不支持，应删除该参数或查阅当前模型的接口说明；不要通过重试来解决参数格式错误。重试不会自动修复一个始终相同的非法请求。

## 常见问题

### 1. `BadRequestError` 是不是 API Key 错了？

通常不能这样判断。Python SDK 将 400 映射为 `BadRequestError`，而认证失败属于其他状态码和异常类型。应先打印 `exc.status_code`，再根据 `exc.body` 判断请求参数问题。不要只凭“调用失败”推断是密钥问题。

### 2. 为什么 `exc.param` 是 `None`？

因为 `param` 是否返回由服务端错误响应决定。服务端可能只返回一段文本，或者只提供 `message`、`type` 等字段。此时应查看 `exc.body`，不能假设所有 400 响应都包含参数名。

### 3. 可以直接打印 `exc.response` 吗？

可以在本地调试时查看，但生产日志不建议直接输出完整响应对象。响应可能包含原始错误正文以及其他元数据。本文示例优先读取 `status_code`、`code`、`param`、`type` 和 `body`，并且不打印请求头或 API Key。

### 4. 为什么官方示例能用，我的兼容接口却返回 400？

官方示例针对官方 API 的接口和模型。兼容服务可能只实现其中一部分字段，或者对模型、消息格式、工具参数有额外限制。应以当前服务的 Base URL、模型列表和 API 兼容说明为准，并先用最小请求验证。

### 5. 应该捕获 `APIError` 还是 `BadRequestError`？

如果只想处理 400，优先捕获 `openai.BadRequestError`；如果要统一记录所有 SDK 错误，可以再捕获 `openai.APIStatusError` 或更上层的 `openai.APIError`。官方 Python SDK 文档列出了 400 到 `BadRequestError` 的映射，并区分了连接错误和 HTTP 状态错误。([github.com](https://github.com/openai/openai-python))

## 排查清单

- [ ] `status_code` 是否确实为 `400`；
- [ ] `body` 是否包含具体字段或错误说明；
- [ ] `param` 是否指出了出错参数；
- [ ] `model` 是否来自当前服务实际提供的模型列表；
- [ ] 请求路径是否与 Base URL 拼接正确；
- [ ] 是否误用了当前服务不支持的参数；
- [ ] 是否使用最小请求复现；
- [ ] 是否只一次修改一个参数；
- [ ] 日志中是否避免输出 API Key、Authorization 请求头和未经脱敏的业务内容。

## 参考资料

- https://developers.openai.com/api/docs/guides/error-codes
- https://platform.openai.com/docs/quickstart
- https://github.com/openai/openai-python
- https://github.com/openai/openai-python/blob/main/src/openai/_exceptions.py

## 关于 犀速-AI

犀速-AI 提供面向开发者的 OpenAI 兼容 API 接入服务与配置说明，适合将常见 SDK、开发工具和 AI 应用连接到统一的 API 接口。具体 Base URL、模型 ID、可用参数和账户规则，请以官方页面当前展示的信息为准。

官网：https://xisu.ai/