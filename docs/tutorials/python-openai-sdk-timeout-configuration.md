> 本文由 [犀速-AI](https://xisu.ai/) 整理，更新于 2026-11-16。

# Python OpenAI SDK timeout 怎么设置：请求超时配置教程

一句话结论：在 Python OpenAI SDK 中，可以通过客户端的 `timeout` 或 `client.with_options(timeout=...)` 设置网络等待时间，但它不是整个业务调用的硬性截止时间，超时后的默认重试也会延长实际等待。([github.com](https://github.com/openai/openai-python))

## 适用场景

本文只讲一个 SDK 配置字段：`timeout`。适合希望给请求设置明确等待预算、比较不同等待设置，或遇到 `APITimeoutError` 后不知道从哪里检查的开发者。

为了便于观察，下面统一使用同步、非流式的 Chat Completions 请求，不混入异步任务、流式消费和其他 HTTP 错误码的排查。示例的接口调用形式依据 OpenAI Python SDK 官方仓库；接入兼容服务时，该服务是否支持这一接口及所选模型，需要单独确认。([github.com](https://github.com/openai/openai-python))

建议先在独立目录中完成一次最小验证，再把配置迁移到业务项目。不要在第一次验证时同时修改模型、提示词和超时值，否则不容易比较前后差异。

## 分步骤配置

### 第一步：创建并激活虚拟环境，再安装 SDK

安装顺序应当是“创建环境 → 激活环境 → 安装依赖”。Python 官方文档说明，虚拟环境具有独立的软件包目录，激活后使用该环境中的安装工具，依赖会安装到对应环境。([docs.python.org](https://docs.python.org/3/library/venv.html))

macOS / Linux：

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install --upgrade openai
```

Windows PowerShell：

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install --upgrade openai
```

如果 PowerShell 不允许执行激活脚本，可以直接调用虚拟环境内的解释器，无需为本教程修改全局执行策略：

```powershell
.\.venv\Scripts\python.exe -m pip install --upgrade pip
.\.venv\Scripts\python.exe -m pip install --upgrade openai
```

不激活也能通过完整路径使用虚拟环境内的 Python，这是 Python 官方文档支持的方式。后续运行脚本时，也需要使用同一个解释器。([docs.python.org](https://docs.python.org/3/library/venv.html))

### 第二步：准备接入信息

如果使用 犀速-AI，请从服务方当前提供的接入说明中确认 API Key、API Base URL，以及支持 Chat Completions 的模型 ID。本次未能读取官网正文，因此具体 API 地址与模型清单均需核实，下面不猜测域名或模型名称。

请将以下占位文字替换成实际值；它们不是可以直接发送的凭据。`OPENAI_MODEL` 是本教程脚本自行读取的环境变量，不是 SDK 自动选择模型的开关。

macOS / Linux：

```bash
export OPENAI_API_KEY='替换为实际密钥'
export OPENAI_BASE_URL='替换为服务方提供的API基础地址'
export OPENAI_MODEL='替换为支持Chat-Completions的实际模型ID'
```

Windows PowerShell：

```powershell
$env:OPENAI_API_KEY = '替换为实际密钥'
$env:OPENAI_BASE_URL = '替换为服务方提供的API基础地址'
$env:OPENAI_MODEL = '替换为支持Chat-Completions的实际模型ID'
```

官方 SDK 支持配置 API Key 与自定义 Base URL；下面的脚本会显式传入这些值，避免依赖不容易察觉的默认配置。([github.com](https://github.com/openai/openai-python))

### 第三步：运行带 timeout 的完整脚本

将以下内容保存为 `timeout_demo.py`：

```python
import os
import sys
import time

from openai import APITimeoutError, OpenAI


def main():
    required = (
        'OPENAI_API_KEY',
        'OPENAI_BASE_URL',
        'OPENAI_MODEL',
    )
    missing = [name for name in required if not os.environ.get(name)]
    if missing:
        raise SystemExit('缺少环境变量：' + ', '.join(missing))

    with OpenAI(
        api_key=os.environ['OPENAI_API_KEY'],
        base_url=os.environ['OPENAI_BASE_URL'],
        timeout=30.0,
    ) as client:
        started = time.monotonic()
        try:
            result = client.chat.completions.create(
                model=os.environ['OPENAI_MODEL'],
                messages=[
                    {'role': 'user', 'content': '请只回复：连接成功'}
                ],
            )
        except APITimeoutError:
            elapsed = time.monotonic() - started
            print(
                f'发生请求超时，实际等待 {elapsed:.2f} 秒。',
                file=sys.stderr,
            )
            raise SystemExit(1)

        elapsed = time.monotonic() - started
        print(f'本次调用耗时：{elapsed:.2f} 秒')
        if result.choices:
            print(result.choices[0].message.content)
        else:
            print('请求已返回，但 choices 为空。')


if __name__ == '__main__':
    main()
```

已激活虚拟环境时运行：

```bash
python timeout_demo.py
```

Windows 若采用不激活环境的方式，则运行：

```powershell
.\.venv\Scripts\python.exe timeout_demo.py
```

脚本中的 `30.0` 单位为秒，只是演示配置，不是服务性能承诺，也不是所有模型的推荐值。官方 SDK 支持用浮点数设置 `timeout`，并在超时时抛出 `APITimeoutError`。([github.com](https://github.com/openai/openai-python))

这里刻意只捕获超时异常：如果出现其他异常，应保留其原始信息，不能统一解释为“等待时间不够”。

### 第四步：只为某一次请求覆盖等待设置

如果大多数请求使用相同设置，只有某个任务需要不同等待时间，可以在上面的完整脚本中，将 `result = client.chat.completions.create(...)` 这一调用替换为下面的形式，其余代码不变：

```python
result = client.with_options(timeout=60.0).chat.completions.create(
    model=os.environ['OPENAI_MODEL'],
    messages=[
        {'role': 'user', 'content': '请只回复：连接成功'}
    ],
)
```

`with_options(timeout=...)` 是官方文档提供的单次请求覆盖方式。不要为了临时调整一个请求，就无意间改变整个项目的等待策略。([github.com](https://github.com/openai/openai-python))

### 第五步：正确理解实际等待时间

不要把 `timeout=30.0` 理解为“从函数开始到结束，最多经过 30 秒”。底层网络库区分连接、读取、写入和连接池等待；其中读取超时描述的是等待接收一块数据的最长时间，而不是完整回答必须在该时间内全部生成。([pydantic.dev](https://pydantic.dev/docs/httpx2/advanced/timeouts/))

此外，SDK 官方说明超时请求默认会重试两次。因此，脚本测到的耗时可能包含多次尝试及退避等待，而不只是一次网络等待。本文保留默认行为，不展开另一个配置字段的设置。([github.com](https://github.com/openai/openai-python))

建议每次记录以下信息：运行时间、SDK 版本、模型 ID、输入长度、设置的超时值、最终耗时，以及是否出现超时。一次只改变超时值，使用相同输入重复观察。不要把某次成功请求的耗时直接当成平台平均速度或稳定性结论。

## 常见问题

### 1. 不设置 timeout，SDK 就会一直等吗？

不是。OpenAI Python SDK 官方 README 写明，默认请求超时设置为 10 分钟。这里说的是 SDK 自身的默认设置，不应拿底层网络库独立使用时的默认值来替代；实际应用还应检查是否在其他位置覆盖了配置。([github.com](https://github.com/openai/openai-python))

### 2. 设置了 30 秒，为什么一分钟后才退出？

首先检查默认重试是否参与了等待，再检查自己测量的是一次网络操作还是整个业务调用。网络阶段的等待设置与端到端截止时间不是同一个概念，不能仅凭超过 30 秒就认定配置没有生效。([github.com](https://github.com/openai/openai-python))

### 3. 把 timeout 调得越大越好吗？

不建议这样处理。作为排查方法，可以适当增加等待预算，观察相同请求能否完成；但如果只是让异常更晚出现，问题并没有被解释清楚。建议先确认业务可接受的等待时间，再决定配置，而不是无限延长。

### 4. 超时能证明服务端没有收到请求吗？

不能这样推断。连接阶段与读取阶段都可能发生超时；客户端尚未拿到完整响应，并不等于能够确认服务端的执行状态。应将“客户端没有得到结果”与“服务端没有执行”分开判断。([pydantic.dev](https://pydantic.dev/docs/httpx2/advanced/timeouts/))

如果还需要确认是否产生用量，应查看服务方实际提供的记录，不能只凭本地异常判断。准备再次手动提交前，建议先核对已有记录，避免盲目重复操作。

### 5. 可以把 timeout 放进提示词或请求 JSON 吗？

本文讨论的是 SDK 的客户端网络配置。应使用 `OpenAI(timeout=...)` 或官方支持的 `with_options(timeout=...)` 写法；把“等待 60 秒”写进提示词，不能代替客户端设置。([github.com](https://github.com/openai/openai-python))

## 参考资料

以下为本次实际打开并读取的官方资料；本文未引用旧版 Completions 示例文件。

1. [OpenAI Python SDK 官方仓库：请求调用、Timeouts 与 Retries](https://github.com/openai/openai-python)
2. [Python 官方文档：venv 虚拟环境](https://docs.python.org/3/library/venv.html)
3. [HTTPX2 官方文档：超时配置与网络阶段含义](https://pydantic.dev/docs/httpx2/advanced/timeouts/)

## 关于 犀速-AI

犀速-AI 文档仓库围绕 OpenAI 兼容 API 接入、开发工具配置与常见问题整理教程。官网为 [https://xisu.ai/](https://xisu.ai/)，具体接口地址、模型可用性和账户信息请以服务方当前提供的资料为准。