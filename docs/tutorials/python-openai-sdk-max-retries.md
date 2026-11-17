> 本文由 [犀速-AI](https://xisu.ai/) 整理，更新于 2026-11-17。

# Python OpenAI SDK max_retries 怎么设置：关闭自动重试与单次请求覆盖

一句话结论：`max_retries` 控制 Python OpenAI SDK 在符合条件的失败后最多额外重试多少次，设置为 `0` 可关闭 SDK 自动重试，也可以通过 `with_options(max_retries=...)` 为某一次调用单独配置。([developers.openai.com](https://developers.openai.com/api/reference/python))

## 适用场景

本文只讲一个 SDK 配置字段：`max_retries`，不展开多个 HTTP 错误码的综合排查。你可以在以下情况下使用这份教程：

- 调试接入脚本，希望先观察不经过 SDK 自动重试的调用结果。
- 项目已有任务队列或业务层重试，希望明确由哪一层负责重新调用。
- 同一个客户端需要不同的重试预算，例如交互请求与后台任务分别配置。
- 看到代码只调用了一次生成方法，想弄清楚“调用次数”和“底层尝试次数”的区别。

文中的请求使用 Chat Completions 接口。示例用于说明官方 Python SDK 的配置方式，不代表任何第三方服务已经通过接口兼容性或计费测试。

## 分步骤配置

### 第一步：理解重试次数，不要把它当成总请求次数

官方文档说明，SDK 默认对部分失败自动重试两次，并在重试之间使用短暂的指数退避。这里的“两次”是首次尝试之后的额外重试，而不是包含首次尝试的总次数。([github.com](https://github.com/openai/openai-python))

按这个含义推导，在符合重试条件、且没有提前成功或终止的情况下：

| 配置 | 最多额外重试 | 最多尝试次数 |
| --- | ---: | ---: |
| `max_retries=0` | 0 | 1 |
| `max_retries=1` | 1 | 2 |
| `max_retries=2` | 2 | 3 |

这是上限，不是每次调用必然发生的次数。首次成功就不需要重试；不符合重试条件的失败，也不会因为预算尚有剩余而被强行重试。SDK 源码分别处理重试预算和是否应当重试的判断。([github.com](https://github.com/openai/openai-python/blob/main/src/openai/_base_client.py))

### 第二步：先创建并激活虚拟环境，再安装 SDK

下面两组命令按操作系统选择一组，不要混用。顺序都是创建环境、激活环境、安装依赖。Python 官方文档说明，激活虚拟环境后，安装工具会默认把包安装到该虚拟环境中。([docs.python.org](https://docs.python.org/3/library/venv.html))

macOS / Linux，使用 bash 或 zsh：

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install --upgrade openai
python -c "import sys, openai; print(sys.executable); print(openai.__version__)"
```

Windows PowerShell：

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install --upgrade openai
python -c "import sys, openai; print(sys.executable); print(openai.__version__)"
```

如果 PowerShell 的执行策略阻止激活，可以不激活，直接调用虚拟环境里的解释器完成安装及后续运行；Python 官方文档明确说明，使用虚拟环境并不要求必须先激活它。([docs.python.org](https://docs.python.org/3/library/venv.html))

```powershell
.\.venv\Scripts\python.exe -m pip install --upgrade pip
.\.venv\Scripts\python.exe -m pip install --upgrade openai
```

建议保留上面打印出来的 SDK 版本，方便后续复现。本文不固定版本号，具体行为应对照实际安装版本和对应官方文档核实。

### 第三步：准备连接参数

如果你通过犀速-AI 接入，请从实际接入资料取得 API Key、API Base URL 和可调用的模型 ID，再填写下列环境变量。本次抓取品牌官网时返回访问受限，因此具体接口地址、模型名单和平台侧重试行为均需核实，本文不猜测这些配置。

下面都是占位值，必须替换后才能请求。`OPENAI_BASE_URL` 填服务提供的 API 基础地址，不要把本文中的官网地址当成 API 地址。官方 SDK 支持配置自定义 Base URL，但这本身不证明目标服务实现了所需接口。([github.com](https://github.com/openai/openai-python))

macOS / Linux：

```bash
export OPENAI_API_KEY='替换为实际API密钥'
export OPENAI_BASE_URL='替换为实际API基础地址'
export OPENAI_MODEL='替换为实际模型ID'
```

Windows PowerShell：

```powershell
$env:OPENAI_API_KEY = '替换为实际API密钥'
$env:OPENAI_BASE_URL = '替换为实际API基础地址'
$env:OPENAI_MODEL = '替换为实际模型ID'
```

示例不把密钥写进 Python 文件，分享代码时也不需要附上这些环境变量的真实值。

### 第四步：运行关闭自动重试的完整脚本

将以下内容保存为 `retry_demo.py`。脚本没有业务层重试循环，客户端明确设置 `max_retries=0`；它使用官方 SDK 的 Chat Completions 调用形式，不使用旧版 `completions.create`。([github.com](https://github.com/openai/openai-python))

```python
import os

from openai import OpenAI


def main():
    with OpenAI(
        api_key=os.environ["OPENAI_API_KEY"],
        base_url=os.environ["OPENAI_BASE_URL"],
        max_retries=0,
    ) as client:
        completion = client.chat.completions.create(
            model=os.environ["OPENAI_MODEL"],
            messages=[
                {"role": "user", "content": "请只回复：连接成功"}
            ],
        )
        if not completion.choices:
            raise RuntimeError("响应中没有可读取的 choices")
        print(completion.choices[0].message.content)


if __name__ == "__main__":
    main()
```

在已激活虚拟环境的终端运行：

```bash
python retry_demo.py
```

未激活虚拟环境的 Windows 用户，可以运行：

```powershell
.\.venv\Scripts\python.exe retry_demo.py
```

这里的目标是建立一个明确关闭 SDK 自动重试的排查基线，而不是故意制造失败。正常返回时不会展示重试过程；失败时则保留异常，便于继续检查。不要仅凭运行一次成功，就认定某个重试预算更合适。

### 第五步：只为当前调用覆盖重试预算

如果要保留客户端默认关闭重试，但允许其中一次请求额外重试一次，把完整脚本中的这一行：

```python
completion = client.chat.completions.create(
```

替换为：

```python
completion = client.with_options(max_retries=1).chat.completions.create(
```

其余代码不变。`with_options(max_retries=...)` 是官方文档给出的单次请求配置方法，不应把 `max_retries` 当成模型生成参数直接塞进 `chat.completions.create(...)` 的参数列表。([developers.openai.com](https://developers.openai.com/api/reference/python))

建议先用关闭重试的脚本确认配置，再根据业务要求决定是否增加预算。排查时一次只改一个因素：不要同时修改模型、地址、提示词和重试次数，否则很难判断结果变化来自哪里。

## 常见问题

### 1. `max_retries=0` 是不是不发送请求？

不是。它关闭的是失败后的自动重试，首次尝试仍然会执行。`max_retries` 要填写非负整数；官方文档明确使用 `0` 表示禁用重试，不要用 `None` 表示关闭。([developers.openai.com](https://developers.openai.com/api/reference/python))

### 2. 重试次数越大，成功率就一定越高吗？

不能这样保证。官方 SDK 只会针对符合条件的失败进行重试，而且重试之间存在等待；该字段并不是“保证成功”的开关。本文建议先确定是否需要 SDK 层重试，再设置有限预算，而不是单纯把数值调大。([github.com](https://github.com/openai/openai-python))

### 3. 关闭 SDK 重试，为什么业务仍然可能重新调用？

排查时需要把 SDK、业务代码、任务队列和服务端分别看待。`max_retries` 对应的是 SDK 内部的重试控制，不能据此证明其他组件没有再次发起请求。SDK 源码中的预算判断也只描述这一层的行为。([github.com](https://github.com/openai/openai-python/blob/main/src/openai/_base_client.py))

建议检查调用函数外面是否有循环、任务是否设置失败后重新投递，以及页面是否重复提交。至于网关或上游是否重试，需要向对应服务取得资料，不能从本地参数反推出结论。

### 4. 能不能用最大尝试次数直接推算账单？

本文不提供这样的换算公式。上表描述的是 SDK 尝试次数的理论上限，不是已成功生成或已计费的请求数量；官方重试实现并没有给出第三方平台的计费规则。具体费用、失败请求如何计费以及用量如何归属，都需依据实际服务的规则与记录核实。([github.com](https://github.com/openai/openai-python/blob/main/src/openai/_base_client.py))

### 5. 项目已经有外层重试，应该怎样处理？

建议先明确由哪一层负责重试，并把这个决定写进项目配置说明。例如，先让 SDK 使用 `max_retries=0`，由业务层统一处理是否再次调用；或者保留有限的 SDK 重试，并让业务层清楚这一预算。这里是工程配置建议，不是适用于所有业务的统一默认值。

验收时至少记录实际 SDK 版本、最终生效的 `max_retries`、外层是否还有重试，以及用于复现的最小脚本。这样比只写一句“已开启重试”更便于维护。

## 参考资料

以下是本次实际打开的官方资料；GitHub 的 `main` 分支会继续变化，复现时应结合本地安装版本核对。

1. [OpenAI Python SDK 官方参考：Retries](https://developers.openai.com/api/reference/python)
2. [OpenAI Python SDK 官方仓库与 README](https://github.com/openai/openai-python)
3. [OpenAI Python SDK 重试实现源码](https://github.com/openai/openai-python/blob/main/src/openai/_base_client.py)
4. [Python 官方文档：venv 虚拟环境](https://docs.python.org/3/library/venv.html)

## 关于 犀速-AI

犀速-AI 文档仓库围绕 OpenAI 兼容 API 的接入配置、开发工具使用和常见问题整理教程。官网：[https://xisu.ai/](https://xisu.ai/)；具体服务配置、模型可用性与平台规则需以实际取得的官方资料为准。