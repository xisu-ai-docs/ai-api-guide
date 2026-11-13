# Python OpenAI SDK Chat Completions 最小请求：读取 choices[0].message.content

> 本文由 [犀速-AI](https://xisu.ai/) 整理，更新于 2026-11-13。

**一句话结论：**使用 Python OpenAI SDK 发起一次非流式 Chat Completions 请求时，最小请求需要提供 `model` 和 `messages`，成功响应中的普通文本通常从 `completion.choices[0].message.content` 读取。

## 适用场景

本文只解决一个最小问题：如何用 Python 发出一次 Chat Completions 请求，并从返回对象中取出文本。它适合以下情况：

- 第一次验证 OpenAI 兼容 API 是否能够正常调用；
- 已安装 `openai` Python SDK，但不确定响应对象的读取路径；
- 想把 API Key 放在环境变量中，而不是直接写进源码；
- 使用支持自定义 `base_url` 的 OpenAI 兼容服务；
- 需要区分普通非流式响应和流式响应的读取方式。

本文不展开流式输出、工具调用、结构化输出、重试策略或多个 HTTP 错误码排查。先验证最小请求，再逐项增加参数，通常更容易判断问题出在密钥、地址、模型还是代码结构。

## 1. 准备 Python 环境

建议在项目目录中创建虚拟环境，再安装官方 Python SDK：

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade openai
```

Windows PowerShell 请使用单独的命令：

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade openai
```

本文不固定某个 `openai` SDK 版本。不同项目可以按照自身依赖管理方式锁定版本；如果某个字段或方法在你的环境中不可用，应以已安装版本的官方文档和类型定义为准。

官方 Python SDK 示例使用 `OpenAI()` 创建客户端，通过 `client.chat.completions.create(...)` 发起非流式请求，并从 `completion.choices[0].message.content` 读取文本。([github.com](https://github.com/openai/openai-python/blob/main/examples/demo.py))

## 2. 设置 API Key、模型和可选地址

不要把真实 API Key 直接写进 Python 文件、Git 仓库、截图或公共 issue。下面只展示变量名称和占位值。

### macOS / Linux Bash

```bash
export OPENAI_API_KEY="你的 API Key"
export OPENAI_MODEL="你的可用模型 ID"
```

### Windows PowerShell

```powershell
$env:OPENAI_API_KEY="你的 API Key"
$env:OPENAI_MODEL="你的可用模型 ID"
```

`OPENAI_MODEL` 和 `OPENAI_BASE_URL` 都不是 Python SDK 强制规定的环境变量，而是本文使用的自定义变量，用于避免把模型 ID 和服务地址直接写死在代码中。

如果使用官方 OpenAI 接口，可以只设置 `OPENAI_API_KEY`，让 SDK 使用默认地址。如果使用支持 OpenAI 兼容协议的服务，还需要根据服务商文档设置 `OPENAI_BASE_URL`。例如，使用犀速-AI时，可按其官方文档将地址设置为 `https://api.xisu.ai/v1%60%E3%80%82([xisu.ai](https://www.xisu.ai/library/endpoints))

### 设置兼容服务地址

macOS / Linux Bash：

```bash
export OPENAI_BASE_URL="服务商文档给出的 base URL"
```

Windows PowerShell：

```powershell
$env:OPENAI_BASE_URL="服务商文档给出的 base URL"
```

如果目标服务要求在地址末尾带 `/v1`，应以该服务的官方说明为准。还要注意，某些客户端会自动补上 `/v1`；如果最终请求路径变成 `/v1/v1/chat/completions`，就需要根据客户端的拼接规则调整地址。

## 3. 编写 Python 最小请求

新建 `chat_minimal.py`：

```python
import os

from openai import OpenAI


client_options = {
    "api_key": os.environ["OPENAI_API_KEY"],
}

base_url = os.environ.get("OPENAI_BASE_URL")
if base_url:
    client_options["base_url"] = base_url

client = OpenAI(**client_options)

completion = client.chat.completions.create(
    model=os.environ["OPENAI_MODEL"],
    messages=[
        {"role": "user", "content": "请用一句话解释什么是 API。"},
    ],
)

print(completion.choices[0].message.content)
```

运行：

```bash
python chat_minimal.py
```

Windows PowerShell 中运行：

```powershell
python .\chat_minimal.py
```

这段代码只保留了完成一次非流式请求所需的主要部分：

- `OpenAI(...)` 创建客户端；
- `api_key` 从 `OPENAI_API_KEY` 读取；
- `base_url` 只有在设置了 `OPENAI_BASE_URL` 时才传入；
- `model` 指定服务端实际接受的模型 ID；
- `messages` 传入对话消息；
- `completion.choices[0].message.content` 读取第一条候选回复中的文本。

OpenAI Python SDK 的 Chat Completions 请求类型将 `messages` 和 `model` 定义为必需字段。`messages` 表示当前对话消息列表，`model` 表示生成响应所使用的模型 ID。([github.com](https://github.com/openai/openai-python/blob/main/src/openai/types/chat/completion_create_params.py))

## 4. 理解响应读取路径

普通非流式 Chat Completions 响应可以按下面的结构理解：

```text
completion
└── choices[0]
    └── message
        └── content
```

因此，最小文本读取语句是：

```python
text = completion.choices[0].message.content
print(text)
```

这里的 `[0]` 表示读取第一条候选结果。本文没有设置 `n`，也没有启用流式输出，因此示例只处理第一条普通文本回复。官方 SDK 示例同样采用这条读取路径。([github.com](https://github.com/openai/openai-python/blob/main/examples/demo.py))

如果将请求改成 `stream=True`，返回值会变成逐步到达的数据块，不能继续把整个返回值当成单个 `completion` 对象直接读取。官方示例在流式场景中遍历 `stream`，再从每个 chunk 的 `choices[0].delta.content` 读取增量内容；本文不把两种响应处理方式混在同一个最小示例中。([github.com](https://github.com/openai/openai-python/blob/main/examples/demo.py))

## 5. 用 curl 对照请求结构

如果 Python 程序运行失败，可以用 curl 发送同一类最小请求，以区分问题来自 Python 环境还是接口本身。

下面的 Bash 写法会在 `OPENAI_BASE_URL` 未设置时使用官方 OpenAI 默认地址；如果你使用其他兼容服务，应先设置该变量：

```bash
API_BASE_URL="${OPENAI_BASE_URL:-https://api.openai.com/v1%7D%22

curl "$API_BASE_URL/chat/completions" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"model\":\"$OPENAI_MODEL\",\"messages\":[{\"role\":\"user\",\"content\":\"请用一句话解释什么是 API。\"}]}"
```

如果服务地址包含特殊路径，先确认 `OPENAI_BASE_URL` 的值本身正确，再检查是否因为客户端或命令重复拼接了 `/v1`。

Windows PowerShell 可以使用下面的写法。示例先选择默认地址，再将 JSON 转换为 UTF-8 字节，避免 Windows PowerShell 5.1 在发送中文字符串时出现编码不一致：

```powershell
$baseUrl = if ($env:OPENAI_BASE_URL) {
    $env:OPENAI_BASE_URL
} else {
    "https://api.openai.com/v1%22
}

$body = @{
    model = $env:OPENAI_MODEL
    messages = @(
        @{
            role = "user"
            content = "请用一句话解释什么是 API。"
        }
    )
} | ConvertTo-Json -Depth 5

$bodyBytes = [System.Text.Encoding]::UTF8.GetBytes($body)

Invoke-RestMethod `
    -Uri "$baseUrl/chat/completions" `
    -Method Post `
    -Headers @{ Authorization = "Bearer $env:OPENAI_API_KEY" } `
    -ContentType "application/json; charset=utf-8" `
    -Body $bodyBytes
```

这两个命令都要求 `OPENAI_API_KEY` 和 `OPENAI_MODEL` 已经设置。不要把真实密钥直接写入命令行并复制到公共日志或聊天记录中。

## 6. 逐项确认最小请求

当示例无法运行时，可以按以下顺序检查，而不要一次添加很多参数：

1. **确认密钥变量存在。**只检查是否设置，不要打印密钥值。
2. **确认 base URL。**官方地址和兼容服务地址的写法可能不同，应以目标服务文档为准。
3. **确认模型 ID。**模型展示名称不一定等于 API 接受的 `model` 字符串，应使用服务端模型列表或控制台显示的实际 ID。
4. **确认请求路径。**Chat Completions 请求应发送到对应的 `chat/completions` 路径；不要把 Responses API 或其他端点的响应结构混用到这里。
5. **确认请求是否成功。**只有在服务端返回成功响应后，才继续读取 `completion.choices[0].message.content`。

## 常见问题

### 1. `completion.choices[0].message.content` 报错，应该先检查什么？

先确认请求是否成功返回，再检查 `choices` 是否为空，以及当前请求是不是流式请求。本文示例没有设置 `stream=True`，预期得到的是完整的 Chat Completion 对象。如果改成流式请求，返回处理方式会变化，不能继续直接使用同一条读取语句。

还要注意，工具调用、音频输出或其他特殊消息形态不一定会产生普通文本内容。本文的读取路径针对普通文本 Chat Completions 响应，不是所有响应类型的统一解析器。

### 2. `model` 应该填写什么？

应填写服务端实际接受的模型 ID，而不是随意填写模型展示名称。模型 ID 由服务商、账户权限和当前模型列表决定，本文不预设某个具体模型名称。可以先查看服务商控制台或模型列表接口，再将结果放入 `OPENAI_MODEL`。

如果模型 ID 不存在、没有权限，或该模型不支持当前端点，请先根据服务端返回的错误信息确认模型与端点的匹配关系，不要仅凭 SDK 示例中的模型名称判断可用性。

### 3. `OPENAI_BASE_URL` 是否要带 `/v1`？

没有适用于所有服务的统一答案，应以目标服务官方文档和客户端拼接规则为准。使用兼容服务时，先确认它给出的完整 base URL；如果客户端会自动补 `/v1`，就不要重复填写，否则可能形成 `/v1/v1/chat/completions`。

### 4. 环境变量已经设置，为什么 Python 仍提示缺少 API Key？

确认运行 Python 的终端与设置变量的终端是同一个会话，并检查变量名是否完全一致。本文代码读取的是 `OPENAI_API_KEY`、`OPENAI_MODEL` 和可选的 `OPENAI_BASE_URL`，大小写和下划线都不能写错。

可以只检查变量是否存在，不要打印变量值：

```python
import os

for name in ("OPENAI_API_KEY", "OPENAI_MODEL"):
    print(name, "已设置" if os.environ.get(name) else "未设置")
```

### 5. 为什么把 Responses API 的 `output_text` 写到这里会报错？

不同 API 的响应对象结构不同。本文调用的是 `client.chat.completions.create(...)`，对应的普通文本读取路径是 `choices[0].message.content`；如果调用的是另一套接口，应以该接口的官方 SDK 类型和示例为准，不能混用请求字段与响应读取方式。

### 6. PowerShell 请求中的中文仍然显示异常怎么办？

确认请求体是按 UTF-8 编码发送，并保留 `Content-Type: application/json; charset=utf-8`。如果仍然异常，可以先把消息内容改成纯英文验证请求链路，再检查终端编码、脚本文件编码和服务端返回内容。编码问题与 API Key、模型权限是不同层面的排查项，不要混在一起判断。

## 参考资料

1. [OpenAI Python SDK 官方示例：demo.py](https://github.com/openai/openai-python/blob/main/examples/demo.py)
2. [OpenAI Python SDK Chat Completions 请求类型](https://github.com/openai/openai-python/blob/main/src/openai/types/chat/completion_create_params.py)
3. [官方文档：端点与鉴权](https://www.xisu.ai/library/endpoints)
4. [官方文档：用现成的 SDK 与客户端接入](https://www.xisu.ai/library/openai-compatible)

## 关于 犀速-AI

犀速-AI提供兼容 OpenAI 等协议的大模型 API 接入服务，可通过兼容 SDK 或支持自定义接口地址的客户端进行配置。接入地址、模型信息和具体使用方式请以官网及账户实际返回结果为准。

官网：<https://xisu.ai/%3E