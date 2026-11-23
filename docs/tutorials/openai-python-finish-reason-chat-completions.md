> 本文由 [犀速-AI](https://xisu.ai/) 整理，更新于 2026-11-23。

# OpenAI Python SDK finish_reason 怎么读取：Chat Completions 完成原因判断

**一句话结论：**在 OpenAI Python SDK 的 Chat Completions 响应中，使用 `completion.choices[0].finish_reason` 读取模型本次生成的结束原因，不要仅凭返回文本长度判断请求是否正常完成。

## 适用场景

本文只讲一个 SDK 响应字段：`finish_reason`。它适合以下场景：

- 你需要区分模型是自然结束，还是因为达到输出上限而提前停止。
- 你在接入 OpenAI 兼容 API 后，想在程序中记录每次生成的完成状态。
- 你正在开发聊天机器人、批量生成脚本或内容审核流程，需要判断是否应该重试或提示用户。
- 你使用 Python OpenAI SDK 的 Chat Completions 接口，而不是旧版 Completions 接口或 Responses API。

OpenAI Python SDK 将 Chat Completions 的结果建模为 `ChatCompletion`，其中每个候选结果位于 `choices` 数组中；每个 `Choice` 对象包含 `finish_reason` 字段。官方类型定义列出的常见值包括 `stop`、`length`、`tool_calls`、`content_filter` 和已弃用的 `function_call`。([github.com](https://github.com/openai/openai-python/blob/main/src/openai/types/chat/chat_completion.py))

## 一、finish_reason 是什么

`finish_reason` 表示模型为什么停止生成。最常见的判断方式是：

| 值 | 含义 | 应用层处理建议 |
| --- | --- | --- |
| `stop` | 模型自然完成，或遇到了请求中提供的停止序列 | 通常可以直接继续处理结果 |
| `length` | 达到了请求或模型允许的输出长度 | 检查输出限制，必要时提示内容可能不完整 |
| `tool_calls` | 模型生成了工具调用 | 读取工具调用信息并执行后续流程 |
| `content_filter` | 输出因内容过滤而被截断或省略 | 不要把结果当作普通完整文本处理 |
| `function_call` | 旧版函数调用结束原因，当前定义中标记为已弃用 | 维护旧代码时兼容读取，新代码优先使用工具调用流程 |

其中，`stop` 不等于“返回内容一定符合业务要求”，它只说明生成过程以正常停止原因结束；你的程序仍然需要根据业务规则校验内容。相反，`length` 也不一定代表请求失败，而是说明生成受到了长度限制。官方 SDK 的 `Choice` 类型对这些结束原因进行了明确说明。([github.com](https://github.com/openai/openai-python/blob/main/src/openai/types/chat/chat_completion.py))

## 二、安装 OpenAI Python SDK

以下命令使用虚拟环境，并将依赖安装到虚拟环境对应的 Python 解释器中。

### macOS 或 Linux

```bash
python -m venv .venv
.venv/bin/python -m pip install --upgrade pip
.venv/bin/python -m pip install --upgrade openai
source .venv/bin/activate
```

### Windows PowerShell

```powershell
python -m venv .venv
.venv\Scripts\python.exe -m pip install --upgrade pip
.venv\Scripts\python.exe -m pip install --upgrade openai
.venv\Scripts\Activate.ps1
```

安装完成后，可以确认 SDK 已进入当前虚拟环境：

```bash
python -c "import openai; print(openai.__file__)"
```

官方 Python SDK 支持使用 `OpenAI` 客户端调用 `chat.completions.create`，并通过 `completion.choices[0]` 访问候选结果。([github.com](https://github.com/openai/openai-python))

## 三、配置 OpenAI 兼容 API

建议通过环境变量保存 API Key、Base URL 和模型名，不要把密钥直接写入源代码或提交到 Git 仓库。

### macOS 或 Linux

```bash
export OPENAI_API_KEY="你的 API Key"
export OPENAI_BASE_URL="你的 OpenAI 兼容 Base URL"
export OPENAI_MODEL="控制台中显示的模型名称"
```

### Windows PowerShell

```powershell
$env:OPENAI_API_KEY = "你的 API Key"
$env:OPENAI_BASE_URL = "你的 OpenAI 兼容 Base URL"
$env:OPENAI_MODEL = "控制台中显示的模型名称"
```

Base URL 和模型名称必须以服务控制台或接口文档给出的值为准。不要因为某个模型名称在其他平台可用，就直接复制到当前服务中。接入犀速-AI 时，可以从对应控制台或接入说明中复制 Base URL、API Key 和可用模型名称，再替换上面的环境变量。

## 四、最小可运行示例：读取 finish_reason

下面是一个同步脚本。它使用 Chat Completions 接口，不使用旧版 `client.completions.create`，也不使用 Responses API。保存为 `read_finish_reason.py` 后运行即可。

```python
import os

from openai import OpenAI


def main() -> None:
    api_key = os.environ["OPENAI_API_KEY"]
    base_url = os.environ["OPENAI_BASE_URL"]
    model = os.environ["OPENAI_MODEL"]

    client = OpenAI(
        api_key=api_key,
        base_url=base_url,
    )

    completion = client.chat.completions.create(
        model=model,
        messages=[
            {
                "role": "user",
                "content": "用一句话说明什么是 API。",
            }
        ],
    )

    finish_reason = completion.choices[0].finish_reason
    print(f"finish_reason={finish_reason}")

    if finish_reason == "stop":
        print("模型正常结束生成。")
    elif finish_reason == "length":
        print("模型因长度限制结束，结果可能不完整。")
    elif finish_reason == "tool_calls":
        print("模型请求执行工具调用。")
    elif finish_reason == "content_filter":
        print("结果受到内容过滤影响。")
    else:
        print("返回了当前脚本未单独处理的结束原因。")


if __name__ == "__main__":
    main()
```

运行脚本：

```bash
python read_finish_reason.py
```

在真实服务中，返回值通常应以当前服务实际响应为准。OpenAI 兼容并不意味着所有平台、模型和参数都完全相同，因此程序最好保留未知值分支，而不是只写 `if finish_reason == "stop"` 后把其他情况全部当成失败。

## 五、如何把 finish_reason 用在业务逻辑中

### 1. 对 `length` 做完整性提示

如果你的应用要求回答必须完整，可以把 `length` 记录为“需要检查”的状态：

```python
finish_reason = completion.choices[0].finish_reason

if finish_reason == "length":
    status = "incomplete"
else:
    status = "completed"

print(status)
```

是否自动发起第二次请求，需要结合业务内容决定。自动重试可能增加用量，也可能造成重复文本，因此不应无条件重试。更稳妥的做法是先记录 `finish_reason`，再根据输出是否完整、任务是否允许续写来决定下一步。

### 2. 对 `tool_calls` 单独进入工具流程

当 `finish_reason` 为 `tool_calls` 时，不能把这次响应简单当作普通文本回复。它表示模型希望应用执行工具调用。你的程序应读取对应的工具调用结构，执行允许的函数，再把工具结果提交给模型继续生成。

如果当前模型或兼容服务不支持工具调用，程序中可以保留分支，但不要仅凭 `finish_reason` 为 `tool_calls` 就假定所有工具字段都存在；还应根据实际响应结构进行校验。

### 3. 对 `content_filter` 保留可解释状态

`content_filter` 应当与普通的 `stop` 区分开。面向用户时，可以显示“内容未能完整生成”之类的通用提示；日志中则记录结束原因，方便后续分析。不要为了让程序看起来成功，就把它强行改写成 `stop`。

## 六、常见问题

### 1. 为什么 `finish_reason` 是 `None`？

首先确认你读取的是完整的非流式 Chat Completions 响应，而不是某个流式 chunk。流式响应会分多段返回，每一段的数据结构可能不同，结束原因通常出现在最终结束事件或最后一个有效 chunk 中。还要确认当前兼容服务确实返回了该字段；如果服务端没有返回，SDK 不能凭空补出真实值。

### 2. `finish_reason=stop` 是否代表回答一定正确？

不是。它只表示模型按正常停止原因结束生成，不能证明事实正确、格式符合要求或满足你的业务约束。仍然需要在应用层检查 JSON 格式、必填字段、长度和业务规则。

### 3. `finish_reason=length` 是接口报错吗？

通常不应直接等同于 HTTP 错误。它表示生成过程触及长度限制，响应可能仍然包含部分内容。你可以根据任务类型选择增加输出限制、缩短输入、拆分任务，或在用户确认后继续生成。

### 4. 为什么我的平台返回了表格中没有的值？

不同 OpenAI 兼容服务可能对扩展字段、模型能力和响应格式有差异。应优先查看当前服务的接口文档，并在代码中保留未知值处理逻辑。不要把官方 SDK 当前类型定义中的枚举值当成所有第三方兼容服务的完整保证。

### 5. Chat Completions 和 Responses API 的 finish_reason 一样吗？

不要直接假设两种接口的响应字段完全相同。本文针对的是 `client.chat.completions.create` 返回的 Chat Completions 对象，读取路径是 `completion.choices[0].finish_reason`。如果项目使用 Responses API，应按照该接口自身的响应结构编写代码，不能机械复制本文路径。

## 参考资料

1. [openai/openai-python README](https://github.com/openai/openai-python/blob/main/README.md)
2. [openai-python 官方 Chat Completions 示例](https://github.com/openai/openai-python/blob/main/examples/async_demo.py)
3. [openai-python 官方仓库](https://github.com/openai/openai-python)
4. [ChatCompletion 类型定义与 finish_reason 字段](https://github.com/openai/openai-python/blob/main/src/openai/types/chat/chat_completion.py)

## 关于 犀速-AI

犀速-AI 提供面向开发者的 OpenAI 兼容 API 接入服务与使用文档，帮助用户在 Python、Node.js 及常见 AI 工具中完成配置和调用。模型名称、Base URL、API Key 使用方式以及可用能力，请以官方页面和控制台显示内容为准。

官网：[https://xisu.ai/](https://xisu.ai/)