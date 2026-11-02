> 本文由 [犀速-AI](https://xisu.ai/) 整理，更新于 2026-11-02。

# Python OpenAI SDK 401 invalid_api_key：request_id 与最小请求排查

**一句话结论：**遇到 `401 invalid_api_key` 时，先确认 Python 进程读取到了正确的 API Key 和 Base URL，再发送一个不带业务逻辑的最小请求；失败时捕获 `openai.AuthenticationError`，记录 `status_code` 与 `request_id`，不要打印密钥或请求头。

## 适用场景

本文只讨论一个 HTTP 状态码：**401 Unauthorized**，重点是 Python OpenAI SDK 中常见的 `invalid_api_key`、`Incorrect API key provided` 或其他认证失败响应。

适合以下情况：

- Python 程序调用 OpenAI API 或 OpenAI 兼容接口时返回 401；
- 终端中确认设置过变量，但 IDE、Docker、systemd 或 CI 中的程序仍然认证失败；
- 需要向服务方提交请求编号，却不希望泄露 API Key；
- 想先排除认证配置，再继续检查模型名、请求参数或业务代码。

本文不展开 403、404、429、500 和超时的综合排查。如果实际响应不是 401，应根据真实状态码切换到对应的排查方法。官方 Python SDK 将 401 映射为 `AuthenticationError`，而 403 映射为 `PermissionDeniedError`，因此不要把所有访问失败都归类为 `invalid_api_key`。([github.com](https://github.com/openai/openai-python/blob/main/README.md?plain=1))

## 1. 先确认 Python 进程读取到的配置

认证排查的第一步不是打印完整密钥，而是确认启动 Python 的进程是否看到了变量。下面的命令只输出变量是否存在，以及非敏感的 Base URL 和模型名：

```bash
python - <<'PY'
import os

for name in ("OPENAI_API_KEY", "OPENAI_BASE_URL", "OPENAI_MODEL"):
    value = os.getenv(name)
    if name == "OPENAI_API_KEY":
        print(f"{name}: {'set' if value else 'missing'}")
    else:
        print(f"{name}: {value or 'missing'}")
PY
```

如果输出 `OPENAI_API_KEY: missing`，说明问题发生在请求发送前，尚不能称为服务端返回的 401。常见原因包括：

- 在一个终端设置变量，却在另一个终端启动程序；
- IDE 的运行配置没有继承终端环境；
- Docker、systemd、GitHub Actions 或其他 CI 环境没有注入变量；
- 修改了变量后没有重启长期运行的 Python 进程；
- 代码读取的是另一套变量名，或者启动脚本覆盖了原有配置。

可以按照实际部署方式设置环境变量，但不要把真实密钥写入源代码、提交到 Git、放进前端构建产物或写入公开日志。例如：

```bash
export OPENAI_API_KEY='从服务控制台获取的密钥'
export OPENAI_BASE_URL='https://example.com/v1'
export OPENAI_MODEL='服务方文档中确认的模型名'
```

这里的地址和模型名只是占位符。实际接入时，Base URL、路径格式和模型 ID 必须以当前服务方文档为准，不要仅凭域名猜测是否需要 `/v1`。

## 2. 安装并确认使用的是目标 Python 环境

在项目虚拟环境中安装官方 Python SDK：

```bash
python -m pip install openai
```

如果项目已经有 `requirements.txt`、锁定文件或其他依赖管理配置，应优先使用项目已有的依赖版本。排查时还要确认 `python` 和 `pip` 指向同一个虚拟环境：

```bash
python -c "import sys; print(sys.executable)"
python -m pip show openai
```

官方 Python SDK 说明：API 返回 4xx 或 5xx 时，会抛出 `APIStatusError` 的子类；其中 401 对应 `AuthenticationError`。这些异常公开提供 `status_code` 和 `response`，失败请求的请求编号则可以从异常的 `request_id` 读取。([github.com](https://github.com/openai/openai-python/blob/main/README.md?plain=1))

## 3. 发送不泄露密钥的最小请求

下面的示例只从环境变量读取 API Key、Base URL 和模型名，不在代码中赋值真实密钥，也不打印请求头。`OPENAI_BASE_URL` 没有设置时，客户端使用 SDK 默认配置；设置后则使用该变量提供的地址。

```python
import os

import openai
from openai import OpenAI

api_key = os.environ["OPENAI_API_KEY"]
model = os.environ["OPENAI_MODEL"]

client_options = {"api_key": api_key}
base_url = os.getenv("OPENAI_BASE_URL")
if base_url:
    client_options["base_url"] = base_url

client = OpenAI(**client_options)

try:
    response = client.chat.completions.create(
        model=model,
        messages=[
            {"role": "user", "content": "Reply with the word: OK"}
        ],
    )
    print(response.choices[0].message.content)
except openai.AuthenticationError as exc:
    print("HTTP status:", exc.status_code)
    print("request_id:", exc.request_id or "not returned")
    raise
```

这段代码的目的不是验证完整业务，而是缩小问题范围：

1. `OPENAI_API_KEY` 由运行环境提供，源码中没有真实密钥；
2. `OPENAI_BASE_URL` 可以用于 OpenAI 兼容接口，但必须使用服务方明确提供的地址；
3. 请求内容尽量简单，便于把认证问题与工具调用、流式输出、长上下文和业务参数分开；
4. 401 使用 `AuthenticationError` 捕获，避免把认证失败误判成网络超时。

如果服务端返回 401，SDK 会进入异常分支；如果请求成功，才会读取 `response.choices[0].message.content`。不要在异常分支中访问成功响应对象的字段。

## 4. 按结果判断 401 的范围

### 情况 A：请求前出现 `KeyError`

例如：

```text
KeyError: 'OPENAI_API_KEY'
```

这说明 Python 进程没有读取到变量，属于本地启动配置问题，不是服务端返回的 401。先检查启动命令、虚拟环境、IDE 运行配置、容器环境变量和 CI Secret 注入方式。

如果是 `OPENAI_MODEL` 缺失，也应先修正本地配置。模型名缺失不会自动等同于 `invalid_api_key`。

### 情况 B：捕获到 `AuthenticationError`，`status_code` 为 401

这表示 SDK 收到了 401 响应，并将其映射为认证异常。对于 `Incorrect API key provided`，官方帮助文档建议检查当前使用的 API Key，确认没有使用旧 Key 或错误 Key；同时还提示要避免应用中混用了不同的 Key，并在需要时确认请求使用了正确的组织设置。([help.openai.com](https://help.openai.com/en/articles/6882433-incorrect-api-key-provided))

建议按以下顺序核对：

1. 当前 Python 进程读取到的是否是预期环境中的 Key；
2. Key 是否在复制时被截断，或意外带入空格、换行等字符；
3. 应用是否在不同模块、配置文件或启动脚本中切换了另一把 Key；
4. Base URL 是否与这把 Key 属于同一个服务；
5. 如果服务方要求指定组织，是否使用了正确的组织配置；
6. Key 是否已经失效、被撤销，或服务方要求重新创建；
7. 仍无法确认时，在服务控制台生成新的 Key，并只更新安全的运行环境配置。

其中第 3 至第 5 项是配置核对建议，不应在没有对应响应证据时表述为服务端一定采用的错误原因。若实际状态码是 403，应按权限或访问控制问题单独排查，而不是继续套用 401 结论。

### 情况 C：`request_id` 为 `None`

官方 Python SDK 的 `APIStatusError` 会从响应头 `x-request-id` 读取 `request_id`；如果响应没有该请求头，异常属性就可能为空。兼容接口、反向代理或中间网关未保留该响应头时，也不能假定一定能拿到请求编号。([github.com](https://github.com/openai/openai-python/blob/main/src/openai/_exceptions.py))

此时仍应记录以下非敏感信息：

- HTTP 状态码；
- 异常类型；
- 请求发生时间；
- Base URL 的域名，避免记录包含敏感查询参数的完整地址；
- SDK 版本；
- 脱敏后的错误类型或错误摘要。

不要为了获得 `request_id` 而打印完整响应、请求头或环境变量内容。

## 5. 在兼容 OpenAI API 的服务中复现最小配置

使用兼容接口时，先把配置范围控制在三项：API Key、Base URL 和模型名。以犀速-AI为例，应按照其当前接入说明填写服务方提供的 Base URL、Key 和模型名称，然后复用上面的最小请求进行验证；不要把一个服务的 Key 与另一个服务的 Base URL 组合使用。

推荐先完成以下顺序：

1. 在安全的运行环境中注入 `OPENAI_API_KEY`；
2. 按服务方说明设置 `OPENAI_BASE_URL`；
3. 设置已确认可用的 `OPENAI_MODEL`；
4. 执行一次非流式最小请求；
5. 只有认证成功后，再加入系统提示词、流式输出、工具调用和业务重试。

这样可以把“Key 或 Base URL 不匹配”与“模型不存在”“请求参数不兼容”等其他问题分开。若服务方的兼容层使用不同的变量名，也应以其官方接入文档为准，不要假设所有平台都使用完全相同的环境变量名称。

## 常见问题

### 1. `OPENAI_API_KEY` 已经显示为 `set`，为什么仍然是 401？

`set` 只能说明进程读取到了非空字符串，不能证明 Key 有效。继续核对是否使用了旧 Key、不同配置来源中的另一把 Key、错误的 Base URL，或服务方要求的组织设置。不要打印完整 Key 来验证，最多输出 `set` 或 `missing`。

### 2. 401 一定是模型名称错误吗？

不能这样判断。本文讨论的是服务端返回的认证失败；如果返回的是 404，应根据实际响应检查模型名、路径或端点。只有在真实状态码为 401 时，才优先处理 API Key、Base URL 和认证相关配置。

### 3. 失败请求读取 `response.request_id` 还是 `exc.request_id`？

失败请求应读取捕获到的异常对象，例如代码中的 `exc.request_id`。官方 SDK 文档还说明，成功的顶层响应对象使用 `_request_id`；这与失败异常上的 `request_id` 是两个不同场景，不要在 401 异常分支中访问不存在的成功响应对象。([github.com](https://github.com/openai/openai-python/blob/main/README.md?plain=1))

### 4. `request_id` 没有返回，是否说明请求没有发出？

不能仅凭 `request_id` 为空得出这个结论。该字段依赖响应中的 `x-request-id`，而兼容接口或代理可能不返回或不保留它。应结合 `status_code`、异常类型和服务端日志继续判断。

### 5. 可以把 Key 直接写到 `OpenAI(api_key="...")` 中测试吗？

不建议。硬编码的密钥可能进入 Git 历史、终端历史、截图、异常信息或 CI 日志。使用环境变量、部署平台 Secret 或其他安全的密钥管理方式，并限制日志内容。

### 6. 401 需要循环重试吗？

不要把 401 当作普通临时网络错误反复重试。无效或不匹配的认证配置不会因为重复发送同一个请求而自动修复。先核对 Key、Base URL 和组织设置，确认配置变更后再运行一次最小请求。

## 参考资料

1. [openai-python 官方 README：错误处理、状态码与 request ID](https://github.com/openai/openai-python/blob/main/README.md)
2. [openai-python 官方源码：APIStatusError、AuthenticationError 与 request_id](https://github.com/openai/openai-python/blob/main/src/openai/_exceptions.py)
3. [OpenAI Help Center：Incorrect API key provided](https://help.openai.com/en/articles/6882433-incorrect-api-key-provided)

## 关于 犀速-AI

犀速-AI提供面向 OpenAI 兼容 API 的接入说明，覆盖 Base URL、API Key 和常见开发工具配置。更多平台信息与接入入口，请访问[官网](https://xisu.ai/)。