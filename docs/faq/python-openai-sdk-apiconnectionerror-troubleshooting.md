> 本文由 [犀速-AI](https://xisu.ai/) 整理，更新于 2026-11-24。

# Python OpenAI SDK APIConnectionError 怎么排查：Base URL、代理与超时定位指南

## 一句话结论

`APIConnectionError` 表示 Python OpenAI SDK 没有正常完成 API 连接或请求传输，优先检查 Base URL、DNS、网络、代理、TLS 证书和防火墙；如果异常实际是 `APITimeoutError`，应单独按照超时处理，而不是继续当作普通连接失败。官方 Python SDK 将 `APITimeoutError` 定义为 `APIConnectionError` 的子类，因此捕获时必须先捕获 `APITimeoutError`，再捕获一般的 `APIConnectionError`。[官方异常定义](https://github.com/openai/openai-python/blob/main/src/openai/_exceptions.py)

## 适用场景

本文适合以下情况：

- 使用 Python `openai` SDK 调用 OpenAI 兼容 API 时出现 `openai.APIConnectionError`；
- 请求没有拿到可用的 HTTP 响应，日志中只显示 `Connection error`；
- 需要判断问题来自 Base URL、域名解析、代理、TLS 证书还是本机防火墙；
- 使用 `AsyncOpenAI` 编写异步脚本，希望区分普通连接失败和 `APITimeoutError`；
- 使用犀速-AI 或其他 OpenAI 兼容服务时，需要验证 SDK 配置是否已经正确到达接口。

本文不展开 401、403、404、429 或 500 等 HTTP 状态错误。服务端已经返回 4xx 或 5xx 时，SDK 通常会抛出 `APIStatusError` 的具体子类；只有没有正常建立连接、请求传输失败或请求超时，才应重点查看 `APIConnectionError` 及其底层异常。官方 SDK 文档也将连接异常与收到 HTTP 状态响应后的异常分开处理。[错误处理说明](https://github.com/openai/openai-python/blob/main/README.md)

## 1. 先创建虚拟环境，再把 SDK 安装到该环境

为了避免系统 Python、虚拟环境和运行脚本使用不同解释器，建议始终通过虚拟环境中的 Python 调用 pip。下面的安装顺序是：创建 `.venv`，立即升级 pip 和 `openai`，最后再激活环境。即使激活步骤尚未执行，使用 `.venv` 内的 Python 路径也能确保 SDK 被安装到目标虚拟环境中。

### macOS/Linux

```bash
mkdir openai-connection-check
cd openai-connection-check
python3 -m venv .venv
.venv/bin/python -m pip install --upgrade pip
.venv/bin/python -m pip install --upgrade openai
source .venv/bin/activate
```

### Windows PowerShell

```powershell
mkdir openai-connection-check
cd openai-connection-check
python -m venv .venv
.venv\Scripts\python.exe -m pip install --upgrade pip
.venv\Scripts\python.exe -m pip install --upgrade openai
.venv\Scripts\Activate.ps1
```

激活后，确认执行文件和 SDK 文件来自同一个环境：

```bash
python -c "import sys, openai; print(sys.executable); print(openai.__file__)"
```

如果这里无法导入 `openai`，问题还没有进入 API 连接阶段，应先处理 Python 环境、虚拟环境或安装问题。

## 2. 检查 API Key、Base URL 和模型名

OpenAI 兼容 SDK 调用通常需要 API Key、Base URL 和模型名。建议使用环境变量，避免把密钥直接写进脚本或提交到代码仓库。

Base URL 必须以服务商提供的接入文档为准。有些服务要求使用带 `/v1` 的 API 根地址，有些服务会提供不同路径；不要仅凭经验添加或删除路径，也不要把控制台网页地址当作 API 地址。

### macOS/Linux

```bash
export OPENAI_API_KEY='替换为你的_API_Key'
export OPENAI_BASE_URL='https://xn--6qq44cz0cu2ekc165av1ofy0b/v1'
export OPENAI_MODEL='替换为服务商确认可用的模型名'
```

### Windows PowerShell

```powershell
$env:OPENAI_API_KEY = '替换为你的_API_Key'
$env:OPENAI_BASE_URL = 'https://xn--6qq44cz0cu2ekc165av1ofy0b/v1'
$env:OPENAI_MODEL = '替换为服务商确认可用的模型名'
```

逐项检查：

1. Base URL 是否存在拼写错误、前后空格或全角字符；
2. 是否误写成管理后台、模型列表页面或普通网页地址；
3. 是否出现重复路径，例如 `/v1/v1`；
4. 当前进程是否真的读取到了环境变量；
5. 模型名是否来自服务商当前文档或控制台，而不是凭记忆猜测。

API Key 为空通常更容易触发认证错误，但排查连接问题时仍应先确认变量没有为空、没有被错误的 shell 环境覆盖。

## 3. 使用完整的异步最小脚本复现

将下面内容保存为 `check_connection.py`。脚本包含 `import asyncio`、`async def main()` 和 `asyncio.run(main())`，可以直接运行。它还按照异常继承关系，先捕获 `openai.APITimeoutError`，再捕获一般的 `openai.APIConnectionError`，避免把超时统一显示成普通连接失败。

```python
import asyncio
import os

import openai
from openai import AsyncOpenAI


async def main() -> None:
    api_key = os.environ.get("OPENAI_API_KEY")
    base_url = os.environ.get("OPENAI_BASE_URL")
    model = os.environ.get("OPENAI_MODEL")

    missing = [
        name
        for name, value in (
            ("OPENAI_API_KEY", api_key),
            ("OPENAI_BASE_URL", base_url),
            ("OPENAI_MODEL", model),
        )
        if not value
    ]
    if missing:
        raise RuntimeError(f"缺少环境变量: {', '.join(missing)}")

    client = AsyncOpenAI(
        api_key=api_key,
        base_url=base_url,
        timeout=20.0,
        max_retries=0,
    )

    try:
        response = await client.chat.completions.create(
            model=model,
            messages=[
                {"role": "user", "content": "请只回复：连接测试成功"}
            ],
        )
        print(response.choices[0].message.content)
    except openai.APITimeoutError as exc:
        print("APITimeoutError: 请求在设定时间内没有完成")
        print(f"SDK 异常: {exc}")
        print(f"底层异常: {exc.__cause__!r}")
    except openai.APIConnectionError as exc:
        print("APIConnectionError: 无法建立或维持 API 连接")
        print(f"SDK 异常: {exc}")
        print(f"底层异常: {exc.__cause__!r}")
    finally:
        await client.close()


if __name__ == "__main__":
    asyncio.run(main())
```

运行脚本：

```bash
python check_connection.py
```

脚本中将 `max_retries` 设置为 `0`，是为了让排查结果更直接。官方 SDK 对连接错误、超时以及部分可重试的 HTTP 错误默认进行重试；排查阶段关闭重试，可以避免等待时间被放大，并更快看到第一次失败的底层原因。确认配置正确后，再根据业务需求设置重试次数。[重试与超时说明](https://github.com/openai/openai-python/blob/main/README.md)

## 4. 根据异常类型和 `__cause__` 定位

### 4.1 输出 `APITimeoutError`

`APITimeoutError` 表示请求在配置的超时时间内没有完成。它继承自 `APIConnectionError`，所以如果代码只写：

```python
except openai.APIConnectionError:
    ...
```

超时也会被这个分支捕获。若希望区分超时与其他连接错误，必须把 `except openai.APITimeoutError` 放在前面。

超时时可以先检查网络是否稳定、代理是否能够转发 HTTPS 请求、Base URL 是否指向正确入口，以及服务端是否需要更长的响应时间。适当增加 `timeout` 只能解决响应确实较慢的情况，不能修复 DNS 失败、端口错误或证书校验失败。官方 SDK 支持通过 `timeout` 配置请求超时，并在超时时抛出 `APITimeoutError`。[官方超时说明](https://github.com/openai/openai-python/blob/main/README.md)

### 4.2 输出 `Name or service not known` 或类似 DNS 错误

这通常说明当前机器无法解析 Base URL 中的域名。先检查域名拼写和当前网络的 DNS 设置：

```bash
nslookup 你的接口域名
```

也可以确认 URL 的主机名是否能够被当前环境解析。容器、虚拟机、公司网络和家庭网络可能使用不同的 DNS 配置，因此应在实际运行 Python 脚本的环境中检查，而不是只在另一台电脑上测试。

### 4.3 输出 `Connection refused` 或 `ConnectError`

这类信息表示连接目标被拒绝或连接没有成功建立。常见检查方向包括：端口是否正确、代理是否配置错误、出口防火墙是否拦截、服务入口是否需要 HTTPS，以及 Base URL 是否包含了不应出现的资源路径。

如果请求经过代理，应该同时检查 `HTTP_PROXY`、`HTTPS_PROXY` 等环境变量是否被当前进程读取。不要把代理地址、API Key 或完整请求头直接发布到公开日志中。

### 4.4 输出证书、TLS 或 SSL 相关错误

如果 `e.__cause__` 中包含 `certificate`、`SSL` 或 `TLS`，应检查系统时间、根证书、Python 运行环境以及企业代理是否替换了证书链。不要为了临时绕过错误而关闭 TLS 证书校验，否则可能暴露 API Key 或请求内容。

官方错误排查资料将网络设置、代理、SSL 证书、防火墙和容器网络权限列为 `APIConnectionError` 的常见排查方向。[错误码与异常排查资料](https://github.com/openai/openai-cookbook/blob/main/examples/data/oai_docs/error-codes.txt)

## 5. 用 curl 与 SDK 做对照

当 Python SDK 报错时，可以使用相同的 Base URL 和 API Key 发起一次最小请求。下面示例使用 macOS/Linux shell 语法：

```bash
curl "$OPENAI_BASE_URL/chat/completions" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "替换为服务商确认可用的模型名",
    "messages": [
      {"role": "user", "content": "连接测试"}
    ]
  }'
```

可以按照结果进行判断：

- `curl` 也无法解析域名或建立连接：优先检查 DNS、网络、代理、证书和防火墙；
- `curl` 能收到 HTTP 响应，而 Python 报 `APIConnectionError`：检查 Python 进程读取到的 Base URL、代理变量、证书链和 SDK 环境；
- `curl` 正常返回，SDK 仍失败：确认运行脚本的 Python 与安装 `openai` 的 Python 是同一个解释器；
- 两者都收到 HTTP 状态响应：说明请求已经抵达接口，此时不应继续按纯连接失败处理，应根据实际响应内容排查。

## 常见问题

### 1. `APIConnectionError` 一定代表服务商不可用吗？

不一定。它只能说明 SDK 没有成功完成连接或请求传输，原因还可能是本机网络、DNS、代理、TLS 证书、出口防火墙或错误的 Base URL。服务端是否可用，需要结合 curl、其他网络环境和服务商公开状态信息判断。

### 2. 为什么捕获了 `APIConnectionError`，却没有单独看到超时？

因为 `APITimeoutError` 是 `APIConnectionError` 的子类。如果先捕获父类，子类异常会被父类分支提前处理。正确顺序是先捕获 `APITimeoutError`，再捕获 `APIConnectionError`。官方源代码明确展示了这种继承关系。[异常源代码](https://github.com/openai/openai-python/blob/main/src/openai/_exceptions.py)

### 3. 应该直接捕获 `httpx` 的连接异常吗？

通常应优先捕获 SDK 提供的 `openai.APIConnectionError` 和 `openai.APITimeoutError`，再通过 `exc.__cause__` 查看底层异常。这样可以保持应用代码与 SDK 的异常层次一致，也能在需要时保留底层网络错误信息。

### 4. `max_retries=0` 是否适合生产环境？

它适合排查阶段，因为可以减少自动重试带来的等待和日志干扰。生产环境是否关闭重试，要结合请求是否可以安全重放、业务对延迟的容忍度以及服务端限制决定。官方 SDK 默认会对连接错误、超时和部分状态错误进行有限次数的重试，也支持通过 `max_retries` 调整。[官方重试说明](https://github.com/openai/openai-python/blob/main/README.md)

### 5. 连接失败时能否读取 `_request_id`？

不能把 `_request_id` 当作连接失败时的必备字段。它来自成功收到响应后的响应头；如果请求没有建立连接，就可能没有响应头。此时应记录异常类型、`exc.__cause__`、时间、Base URL 的主机名和运行环境，但不要记录完整 API Key。

## 参考资料

- https://github.com/openai/openai-python/blob/main/README.md
- https://github.com/openai/openai-python/blob/main/src/openai/_exceptions.py
- https://github.com/openai/openai-python/blob/main/src/openai/__init__.py
- https://github.com/openai/openai-cookbook/blob/main/examples/data/oai_docs/error-codes.txt

## 关于 犀速-AI

犀速-AI 提供 OpenAI 兼容 API 接入服务，便于在 Python、Node.js 和常见 AI 开发工具中配置 Base URL 与 API Key。具体接口地址、模型名称和使用规则请以官方页面及控制台当前显示内容为准，官网地址为 [https://xisu.ai/](https://xisu.ai/)。