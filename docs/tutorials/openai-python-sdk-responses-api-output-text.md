> 本文由 [犀速-AI](https://xisu.ai/) 整理，更新于 2026-11-08。

# OpenAI Python SDK Responses API：最小请求与 output_text 读取

**一句话结论：** 使用 OpenAI Python SDK 调用 Responses API 时，最小请求只需要准备 `model` 和 `input`；读取普通文本优先使用 `response.output_text`，不要默认把 `response.output[0].content[0].text` 当成固定结构。

## 适用场景

本文适合以下情况：

- 第一次使用 OpenAI Python SDK 的 Responses API；
- 从 Chat Completions 切换到 Responses API，需要确认返回文本应该怎么读取；
- 需要在 OpenAI 兼容 API 中验证 `Base URL`、API Key、模型名和最小请求是否可用；
- 想同时记录响应 ID 和 token 用量，方便日志排查与用量核对。

Responses API 用于向模型发起请求。Python SDK 的基本调用方式是创建 `OpenAI` 客户端，再调用 `client.responses.create(...)`；官方文本生成示例也使用 `response.output_text` 读取便捷的文本结果。

## 第 1 步：准备 Python 环境

建议使用虚拟环境，并安装官方 Python SDK：

```bash
python -m venv .venv
```

在 macOS 或 Linux 的终端中激活虚拟环境：

```bash
source .venv/bin/activate
```

在 Windows PowerShell 中激活虚拟环境：

```powershell
.\.venv\Scripts\Activate.ps1
```

Python 官方文档给出的 PowerShell 激活形式是 `<venv>\Scripts\Activate.ps1`。当虚拟环境目录名为 `.venv`，且当前终端位于项目根目录时，应写成 `\.venv\Scripts\Activate.ps1`，其中开头的 `.` 表示当前目录。若 PowerShell 提示脚本执行被禁止，应按照本机 Python 和 Windows 的执行策略处理；不要把 PowerShell 命令复制到 macOS 或 Linux 的 Bash 终端中执行。

安装或升级 SDK：

```bash
python -m pip install --upgrade openai
```

不同系统的激活命令不同，但后续 Python 文件内容可以保持一致。

## 第 2 步：配置 API Key、Base URL 和模型名

不要把真实密钥直接写进源码。可以在当前终端设置环境变量。

### macOS / Linux Bash 或 Zsh

```bash
export OPENAI_API_KEY="你的 API Key"
export OPENAI_BASE_URL="https://xn--6qq22fb6e3b063ixgv/v1%22
export OPENAI_MODEL="你的模型名"
```

### Windows PowerShell

```powershell
$env:OPENAI_API_KEY = "你的 API Key"
$env:OPENAI_BASE_URL = "https://xn--6qq22fb6e3b063ixgv/v1%22
$env:OPENAI_MODEL = "你的模型名"
```

上面两组命令不要混用：`export` 是 Bash/Zsh 常见写法，`$env:变量名` 是 PowerShell 写法。设置环境变量只对当前终端会话有效；如果关闭终端后重新运行程序，需要再次设置，或者改用你所在项目采用的环境变量管理方式。

如果你使用犀速-AI，请先从其控制台或接入说明中复制对应的 Base URL、API Key 和可用模型名，再填入上述环境变量；不要根据其他平台的地址或模型列表自行猜测。

`OPENAI_BASE_URL` 是否需要包含 `/v1`，应以所接入服务的官方说明为准。不同 OpenAI 兼容服务的网关路径可能不同；如果地址重复出现 `/v1/v1`，应检查 Base URL 和 SDK 是否分别又添加了一次版本路径。

## 第 3 步：发送一个最小 Responses API 请求

新建 `minimal_response.py`：

```python
import os

from openai import OpenAI


api_key = os.environ["OPENAI_API_KEY"]
base_url = os.environ.get("OPENAI_BASE_URL")
model = os.environ["OPENAI_MODEL"]

client_kwargs = {"api_key": api_key}
if base_url:
    client_kwargs["base_url"] = base_url

client = OpenAI(**client_kwargs)

response = client.responses.create(
    model=model,
    input="请用一句话说明什么是 API。",
)

print(response.output_text)
print("response_id:", response.id)

if response.usage is not None:
    print("input_tokens:", response.usage.input_tokens)
    print("output_tokens:", response.usage.output_tokens)
    print("total_tokens:", response.usage.total_tokens)
```

运行：

```bash
python minimal_response.py
```

官方 Python 示例展示了 `OpenAI` 客户端与 `client.responses.create(...)` 的组合用法；Responses API 的文本生成示例也使用 `response.output_text` 输出结果。

## 第 4 步：为什么推荐 `response.output_text`

下面这种写法看起来直观，但不适合作为通用解析逻辑：

```python
text = response.output[0].content[0].text
```

原因是 `output` 是一个数组，可能包含多个输出项目。官方文档说明，输出数组可能包含工具调用、推理相关项目或其他项目，因此不能安全地假设文本一定位于 `output[0].content[0].text`。部分官方 SDK 提供 `output_text` 便捷属性，用于聚合模型返回的文本输出。

因此，普通文本任务建议使用：

```python
text = response.output_text
```

如果程序需要处理工具调用、拒答、结构化输出或其他非普通文本结果，则应进一步检查 `response.output` 中各项目的类型，并按照实际返回结构处理，而不是简单地把响应对象转换为字符串。

## 第 5 步：验证最小请求是否真正成功

可以按以下顺序确认结果：

1. **请求没有抛出异常**：说明客户端完成了请求发送和响应解析；
2. **`response.id` 有值**：可以把它写入应用日志，用于后续定位本次响应；
3. **`response.output_text` 有内容**：说明当前响应中存在可由便捷属性聚合的文本输出；
4. **读取 `usage` 前先判断是否为空**：不要假设所有兼容服务或所有响应场景都会提供完整用量字段；
5. **模型名来自当前服务的支持列表**：不要把某个平台的模型名直接复制到另一个兼容服务中。

响应对象的 `id`、`output` 和 `usage` 等字段可以在 API 参考和 Python SDK 类型定义中查看。实际使用兼容服务时，字段是否完整返回还取决于该服务的接口实现，应以服务方文档和实际响应为准。

## 常见问题

### 1. Bash 里的 `export` 能直接放到 PowerShell 中吗？

不能直接混用。macOS/Linux 示例应使用：

```bash
export OPENAI_API_KEY="你的 API Key"
```

Windows PowerShell 示例应使用：

```powershell
$env:OPENAI_API_KEY = "你的 API Key"
```

如果你在 Windows 上使用的是 CMD，而不是 PowerShell，应使用该 shell 自己的环境变量语法；本文不把 CMD 命令混入 Bash 或 PowerShell 代码块。最重要的是，设置变量和运行 Python 程序必须在同一个终端会话中完成，或者确认变量已经被传递到 Python 进程。

### 2. PowerShell 激活虚拟环境时报路径错误怎么办？

如果虚拟环境目录位于当前项目下，并且目录名是 `.venv`，应在项目根目录执行：

```powershell
.\.venv\Scripts\Activate.ps1
```

常见错误是漏掉开头的 `.`，或者误把 Bash 写法 `source .venv/bin/activate` 复制到 PowerShell。还应确认当前目录确实包含 `.venv` 文件夹，可以先执行：

```powershell
Get-ChildItem .venv\Scripts
```

如果列表中没有 `Activate.ps1`，请确认虚拟环境是否创建成功，或重新执行：

```powershell
python -m venv .venv
```

如果文件存在但执行策略阻止运行，应按照本机 Windows 安全策略处理，不要为了绕过错误而把密钥或脚本内容复制到不受信任的位置。

### 3. `AttributeError: 'Response' object has no attribute 'output_text'` 怎么办？

先确认当前导入的确实是目标 Python SDK，而不是项目目录中名为 `openai.py` 的本地文件或名为 `openai` 的本地目录。可以执行：

```bash
python -c "import openai; print(openai.__file__)"
```

如果输出路径指向当前项目下的 `openai.py` 或 `openai/` 目录，先重命名本地文件或目录，再重新运行。随后检查实际安装的 SDK、调用的接口以及服务返回格式。不同兼容实现对 Responses API 的支持程度可能不同，不能只凭接口名称推断所有字段都会完全一致。

### 4. `response.output_text` 为空，但请求没有报错，正常吗？

不一定表示请求失败。Responses API 的 `output` 可以包含非文本项目，例如工具调用或其他类型的输出；当响应中没有可聚合的普通文本时，便捷文本属性可能无法提供你期待的内容。此时可以在脱敏后检查 `response.output`，确认返回项目的类型和结构。官方文档也提醒，不应把 `output[0].content[0].text` 当成固定路径。

不要把 API Key、完整提示词、用户隐私数据或业务敏感内容直接写入公共日志。

### 5. `usage` 是空值，读取 `input_tokens` 报错怎么办？

读取前先判断：

```python
if response.usage is not None:
    total_tokens = response.usage.total_tokens
else:
    total_tokens = None
```

如果你使用的是 OpenAI 兼容服务，还需要确认该服务是否在当前接口和模型下返回用量信息。缺少 `usage` 不能直接推断“没有计费”或“没有消耗”；计费和用量口径应以服务方提供的用量记录为准。

### 6. 报错提示模型不存在，应该先改 Python 代码吗？

通常先检查 `OPENAI_MODEL` 的值是否属于当前服务支持的模型，并确认 `OPENAI_BASE_URL` 指向正确的 API 根地址。模型名、Base URL 和接口路径是三个独立配置项；仅修改 `output_text` 解析代码，无法解决模型名或路由配置问题。

如果服务只实现了 Chat Completions，而没有实现 Responses API，那么即使 Python SDK 安装正确，也可能无法按本文示例调用。此时应以服务方公布的兼容接口范围为准，不要仅根据“OpenAI 兼容”几个字判断所有接口都可用。

### 7. 为什么示例没有直接把 API Key 写在 `OpenAI(api_key=...)` 中？

为了避免密钥进入 Git 提交、截图、日志或聊天记录。示例使用环境变量读取密钥，代码中只保留变量名，不包含真实凭据。无论采用哪种配置方式，都不要把真实 API Key 提交到公开仓库；如果密钥已经泄露，应按照发放该密钥的平台说明处理。

## 参考资料

- [Python 文档：venv——创建虚拟环境](https://docs.python.org/3.12/library/venv.html)
- [OpenAI API：Developer quickstart](https://developers.openai.com/api/docs/quickstart)
- [OpenAI API：Text generation](https://developers.openai.com/api/docs/guides/text)
- [OpenAI API Reference：Create a model response](https://developers.openai.com/api/reference/python/resources/responses/methods/create)
- [OpenAI Python API library：Python API Reference](https://developers.openai.com/api/reference/python)

## 关于 犀速-AI

犀速-AI 面向需要接入 OpenAI 兼容 API 的开发者，提供 API 接入与常用 AI 工具配置相关服务。使用前请以官方页面公布的 Base URL、模型和密钥配置说明为准，并遵守当地法律法规。

官网：[https://xisu.ai/](https://xisu.ai/)