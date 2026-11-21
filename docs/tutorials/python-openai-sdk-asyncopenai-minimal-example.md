> 本文由 [犀速-AI](https://xisu.ai/) 整理，更新于 2026-11-21。

# Python OpenAI SDK 异步请求：AsyncOpenAI 完整最小示例

一句话结论：使用 `AsyncOpenAI` 创建客户端，在异步函数中通过 `await` 发起请求，再用 `asyncio.run(main())` 启动独立脚本，就能完成一次完整的异步调用。([github.com](https://github.com/openai/openai-python/blob/main/README.md))

## 适用场景

本文围绕一个核心对象 `AsyncOpenAI`，演示一次非流式 Chat Completions 请求，适合以下需求：

- 希望先用独立 Python 脚本验证 API 接入配置；
- 准备把模型调用加入已有的异步程序；
- 使用 OpenAI 兼容接口，需要自行填写连接地址和模型名称；
- 只需要获得完整文本，暂时不涉及流式输出和并发任务。

本文不展开多个 HTTP 错误码、重试策略或性能测试。建议先按步骤跑通最小请求，再增加业务功能。示例是根据官方 SDK 接口编写的独立脚本，不是复制旧版 Completions 示例；实际请求能否成功，还需要由所用服务的接口兼容性、账户权限和模型支持情况确认。

## 一、创建虚拟环境并安装 SDK

请先准备能够运行 `venv` 的 Python 环境。SDK 支持的 Python 版本范围以官方 README 的要求为准，本文不固定版本下限。

**顺序必须是：创建虚拟环境 → 激活虚拟环境 → 安装依赖。** 激活操作会让当前终端优先使用虚拟环境中的解释器；仅创建 `.venv` 并不会自动切换当前解释器。([docs.python.org](https://docs.python.org/3/library/venv.html))

### macOS / Linux：Bash 或 Zsh

```bash
mkdir async-openai-demo
cd async-openai-demo
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install --upgrade openai
```

### Windows PowerShell

```powershell
mkdir async-openai-demo
cd async-openai-demo
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install --upgrade openai
```

上述安装命令应在激活成功后继续执行，不要在激活之前使用全局解释器安装 SDK。官方 SDK 的包名是 `openai`，官方 README 提供了通过 pip 安装的方式。([github.com](https://github.com/openai/openai-python/blob/main/README.md))

### 检查解释器和安装位置

两种系统激活后都可以运行：

```text
python -c "import sys; import openai; print(sys.executable); print(openai.__file__)"
```

检查两条输出路径是否位于当前项目的 `.venv` 目录内。这个检查比只看终端提示符更直接：它能帮助确认后续运行脚本的解释器与安装 SDK 的环境一致。虚拟环境中的解释器和包目录布局可参照 Python 官方 `venv` 文档。([docs.python.org](https://docs.python.org/3/library/venv.html))

## 二、准备连接配置

本教程用环境变量保存接入信息，并由脚本显式传给客户端。请先准备服务方提供的 API Key、接口 Base URL，以及支持 Chat Completions 文本请求的模型标识。

如果你使用犀速-AI，请从其实际接入说明中确认这三项信息后再填写；本文不预设未经核实的地址或模型名称。

下面三个变量是本文脚本读取配置的约定，其中模型变量由脚本读取后传入请求，并不是设置后就会自动影响所有 SDK 调用。

| 环境变量 | 本文中的用途 |
| --- | --- |
| `OPENAI_API_KEY` | 读取后传给客户端作为 API Key |
| `OPENAI_BASE_URL` | 读取后传给客户端作为接口基础地址 |
| `OPENAI_MODEL` | 读取后传给本次请求作为模型标识 |

`AsyncOpenAI` 构造函数支持传入 `api_key` 和 `base_url`，本文采用显式传入方式，避免读者依赖不清楚的默认连接配置。([raw.githubusercontent.com](https://raw.githubusercontent.com/openai/openai-python/main/src/openai/_client.py))

### macOS / Linux

```bash
export OPENAI_API_KEY="替换为实际_API_Key"
export OPENAI_BASE_URL="替换为服务方提供的完整_Base_URL"
export OPENAI_MODEL="替换为支持本接口的模型标识"
```

### Windows PowerShell

```powershell
$env:OPENAI_API_KEY = "替换为实际_API_Key"
$env:OPENAI_BASE_URL = "替换为服务方提供的完整_Base_URL"
$env:OPENAI_MODEL = "替换为支持本接口的模型标识"
```

这些中文值都是占位符，必须替换后才能请求服务。建议在设置变量的同一个终端窗口里继续操作，不要把真实密钥写进即将提交到仓库的源文件。

## 三、编写完整异步脚本

在项目目录中新建 `async_request.py`，复制以下完整代码。这个文件名是本文自行约定的本地文件名，不是 SDK 自带的示例文件。

```python
import asyncio
import os

from openai import AsyncOpenAI


def required_env(name: str) -> str:
    value = os.environ.get(name)
    if value is None or not value.strip():
        raise RuntimeError(f"请先设置环境变量：{name}")
    return value.strip()


async def main() -> None:
    api_key = required_env("OPENAI_API_KEY")
    base_url = required_env("OPENAI_BASE_URL")
    model = required_env("OPENAI_MODEL")

    async with AsyncOpenAI(
        api_key=api_key,
        base_url=base_url,
    ) as client:
        response = await client.chat.completions.create(
            model=model,
            messages=[
                {
                    "role": "user",
                    "content": "请用一句话介绍异步 API 调用。",
                }
            ],
        )

        if not response.choices:
            raise RuntimeError("响应没有返回候选结果，请核对服务的接口兼容性。")

        content = response.choices[0].message.content
        if content:
            print(content)
        else:
            print("请求已返回，但本次响应没有普通文本内容。")


if __name__ == "__main__":
    asyncio.run(main())
```

这里调用的是 `client.chat.completions.create(...)`，请求使用 `messages`，不是旧版 Completions 的 `prompt`。官方 Chat Completions 资源源码包含对应的异步实现，官方 README 也提供了异步客户端调用 Chat Completions 的示例。([raw.githubusercontent.com](https://raw.githubusercontent.com/openai/openai-python/main/src/openai/resources/chat/completions/completions.py))

代码使用 `async with` 管理客户端生命周期。官方 SDK 文档明确建议在事件循环结束前使用异步上下文管理器，或显式等待客户端关闭；因此本文不依赖进程退出时的隐式清理。([github.com](https://github.com/openai/openai-python/blob/main/README.md))

## 四、运行并检查输出

确认依赖安装成功、占位配置已经替换，然后执行：

### macOS / Linux

```bash
python async_request.py
```

### Windows PowerShell

```powershell
python .\async_request.py
```

本脚本会先检查三个环境变量，再尝试发起请求。拿到候选结果后，它会打印第一条消息的普通文本；若没有候选结果或没有普通文本，脚本会给出对应提示。具体回答不是固定字符串，不需要与某一份示例输出完全相同。

如需确认环境问题，建议按以下顺序检查，而不是一次修改所有配置：

1. 重新运行第一节的解释器检查命令，确认仍在项目虚拟环境中；
2. 确认三个占位值均已替换，不要打印完整密钥排查；
3. 对照服务方说明核对 Base URL 和模型标识；
4. 保留异常信息用于定位，但分享日志前先移除密钥等敏感内容。

## 五、理解异步入口

`async def main()` 只是定义异步入口；本教程通过末尾的 `asyncio.run(main())` 将它交给事件循环执行。Python 官方文档说明，`asyncio.run()` 会执行传入的可等待对象，并负责管理事件循环。([docs.python.org](https://docs.python.org/3/library/asyncio-runner.html))

请求前面的 `await` 同样不能遗漏。SDK 官方异步用法要求使用 `AsyncOpenAI`，并在异步 API 调用前使用 `await`。本文只验证这种调用方式，不据此承诺单次响应速度或吞吐量。([github.com](https://github.com/openai/openai-python/blob/main/README.md))

## 常见问题

### 1. 已经安装 SDK，为什么仍然提示找不到 `openai`？

先检查安装和运行是否使用了同一个解释器。虚拟环境默认与基础 Python 环境的包隔离，因此在全局环境安装成功，不代表 `.venv` 中也已经安装。([docs.python.org](https://docs.python.org/3/library/venv.html))

重新激活项目虚拟环境，然后运行：

```text
python -m pip install --upgrade openai
python -c "import sys; import openai; print(sys.executable); print(openai.__file__)"
```

不要通过反复切换 `pip`、`pip3`、`python` 来碰运气，应先确认解释器路径。

### 2. PowerShell 不允许执行激活脚本怎么办？

可以不激活，直接调用虚拟环境中的解释器安装依赖和运行脚本。Python 官方文档说明，使用虚拟环境并不强制要求激活，也可以直接指定解释器路径。([docs.python.org](https://docs.python.org/3/library/venv.html))

在已经创建 `.venv` 的项目目录中执行：

```powershell
.\.venv\Scripts\python.exe -m pip install --upgrade pip
.\.venv\Scripts\python.exe -m pip install --upgrade openai
.\.venv\Scripts\python.exe .\async_request.py
```

运行前仍需完成第二节的环境变量配置。这个方案无需为了本教程修改系统执行策略。

### 3. Base URL 要不要加 `/v1`？

请原样采用所用服务明确提供的 Base URL，不要统一追加或删除版本路径。本文无法替每个兼容服务确定其 URL 规则，因此不提供未经核实的通用地址。SDK 支持自定义 `base_url`，但这不等于所有服务都采用相同的路径约定。([raw.githubusercontent.com](https://raw.githubusercontent.com/openai/openai-python/main/src/openai/_client.py))

### 4. 为什么脚本提示缺少环境变量？

这是本文 `required_env()` 函数主动给出的提示，发生在请求之前。检查提示中的变量名，回到第二节，在当前终端设置对应值后重新运行即可。本文要求三项配置都明确存在，不会在缺少 Base URL 时自动尝试其他服务。

### 5. 已有事件循环的程序可以照搬最后两行吗？

不要直接照搬。Python 官方文档明确规定，同一线程中已经运行事件循环时，不能再次调用 `asyncio.run()`。独立脚本保留本文入口；集成到已有异步程序时，应由该程序的异步入口调用并等待 `main()`，而不是重复创建事件循环。([docs.python.org](https://docs.python.org/3/library/asyncio-runner.html))

## 参考资料

以下资料均为本次实际打开的官方文档或官方仓库文件：

- [OpenAI Python SDK README：安装、异步调用与资源管理](https://github.com/openai/openai-python/blob/main/README.md)
- [OpenAI Python SDK 客户端源码：AsyncOpenAI 构造参数](https://raw.githubusercontent.com/openai/openai-python/main/src/openai/_client.py)
- [OpenAI Python SDK Chat Completions 资源源码](https://raw.githubusercontent.com/openai/openai-python/main/src/openai/resources/chat/completions/completions.py)
- [Python 官方文档：venv 虚拟环境](https://docs.python.org/3/library/venv.html)
- [Python 官方文档：asyncio 运行器](https://docs.python.org/3/library/asyncio-runner.html)

## 关于 犀速-AI

犀速-AI 的文档仓库围绕 OpenAI 兼容 API 接入整理教程，关注 Base URL、API Key 配置及常见开发工具的使用。服务信息与可用能力请以[官网](https://xisu.ai/)及实际接入说明为准，使用时请遵守当地法律法规。