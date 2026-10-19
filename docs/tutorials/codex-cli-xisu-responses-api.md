# 2026 年 10 月 Codex CLI 接入犀速AI：Responses API 配置与实测指南

> 本文整理自 [犀速-AI 官网原文](https://www.xisu.ai/resources/tutorials/codex-cli)，原文更新于 2026-10-16。

## 结论先看

- **装好 Codex CLI，写一个 `config.toml`、设置环境变量 `XISU_API_KEY`，就能通过犀速AI 调用 Responses API。**
- **Codex CLI 0.162.1 实测可用**，配置文件和环境变量与 Codex 桌面版共用，命令行版和桌面版读取的是同一个配置。
- **Codex CLI 使用 `/v1/responses`，Chat Completions 客户端使用 `/v1/chat/completions`**；两套接口共用同一把密钥和同一个余额，但请求与响应格式不同。

## 为什么现在值得看

本文基于 Codex CLI 0.162.1 实测，模型名截至 2026-10-16。近期使用中常见的问题包括 Windows 沙箱、`--full-auto` 参数变化、环境变量未生效，以及 Responses 与 Chat Completions 的字段差异；现在按本文配置，可以直接避开这些问题。

Codex CLI 是 OpenAI 开源的命令行编程助手，能在终端里读代码、改文件、跑命令。它通过 Responses API（`/v1/responses`）调用模型，犀速AI 原样提供这个接口，所以只要写好配置文件、设好密钥，Codex CLI 就能直接用，**不需要登录 ChatGPT 账号**。

## 最短路径

- 安装：`npm i -g @openai/codex`
- 在 `~/.codex/config.toml` 里写入犀速AI 的配置（见第 2 步）
- 把密钥设成环境变量 `XISU_API_KEY`，打开新终端运行 `codex`

开始前请先准备好 API 密钥，还没有的话先看「新手第一步：注册、充值、创建密钥」。已经按 Codex 桌面版教程配置过的，第 2、3 步已经完成，**命令行版和桌面版读的是同一个配置文件和环境变量**，装好就能用。

本文基于 Codex CLI 0.162.1 实测，模型名截至 2026-10-16。

## 1. 安装

Windows、macOS、Linux 都用同一条命令。需要先装好 Node.js（建议 LTS 版）：

```bash
npm i -g @openai/codex
```

npm 下载慢的话，可以临时换成国内镜像源：

```bash
npm i -g @openai/codex --registry=https://registry.npmmirror.com
```

装好后检查版本：

```bash
codex --version
```

看到类似 `codex-cli 0.162.1` 就说明装好了（版本号可能更新）。

## 2. 写配置文件 config.toml

配置文件在用户目录下的 `.codex\\config.toml`。内容和桌面版完全一样：

```toml
model_provider = "xisu"
model = "gpt-5.6-terra"
[model_providers.xisu]
name = "犀速AI"
base_url = "https://api.xisu.ai/v1"
wire_api = "responses"
env_key = "XISU_API_KEY"
```

- `base_url` 带 `/v1`。
- `wire_api = "responses"` 表示走 Responses 接口。
- `env_key` 是存放密钥的环境变量名。
- `model` 是默认模型，可以换成模型广场里的其他模型名，推荐见第 5 步。

文件原来就有内容的话，把前两行放到文件最上面，`[model_providers.xisu]` 那一段放到文件末尾。

### Windows

在 PowerShell 里运行下面两行，记事本提示「是否创建新文件」时点「是」，粘贴上面的内容后保存：

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\\.codex" | Out-Null
notepad "$env:USERPROFILE\\.codex\\config.toml"
```

### macOS / Linux

整段粘贴到终端：

```bash
mkdir -p ~/.codex
cat > ~/.codex/config.toml <<'EOF'
model_provider = "xisu"
model = "gpt-5.6-terra"
[model_providers.xisu]
name = "犀速AI"
base_url = "https://api.xisu.ai/v1"
wire_api = "responses"
env_key = "XISU_API_KEY"
EOF
```

注意这里用的是 `>`，会覆盖已有的 `config.toml`。原来有内容的话，请用编辑器手动合并。

## 3. 设置密钥环境变量

密钥不写进配置文件，而是放在环境变量 `XISU_API_KEY` 里，配置文件给别人看也不会泄露密钥。

### Windows（PowerShell）

```powershell
[Environment]::SetEnvironmentVariable("XISU_API_KEY", "这里换成你的密钥", "User")
```

没有输出就是成功了。关掉终端重新打开才能读到新变量。

### macOS / Linux

bash 用户把 `~/.zshrc` 换成 `~/.bashrc`：

```bash
echo 'export XISU_API_KEY="你的密钥"' >> ~/.zshrc
source ~/.zshrc
```

## 4. 开始使用

### 交互模式

进入项目目录，运行：

```bash
cd 你的项目目录
codex
```

第一次启动可能会看到这些提示：

- 「Meet GPT-6 Sol」这类新模型推荐：Codex 会按配置里的模型推荐一个新版本（用 `gpt-5.6-terra` 时推荐 GPT-6 Sol，用 `gpt-5.6-luna` 时推荐 GPT-6 Luna）。选 `2. Use existing model` 保留配置文件里的模型；选 `1` 会改用推荐的模型，它们也都在售，用量和费用会不同。
- 「Update available」：提示有新版本时，选 Skip 继续即可，之后再自行升级。
- Windows 上的沙箱设置：Codex 会先设置沙箱（Setting up sandbox）。如果出现「Couldn't set up your sandbox with Administrator permissions」，选 `2. Use Codex with non-admin sandbox`，Codex 会在 `config.toml` 里写入 `[windows]` 下的 `sandbox = "unelevated"`。

进入后输入 `/status` 回车，看到 `Model provider: xisu` 就说明走的是犀速AI。

### 非交互模式：codex exec

`codex exec` 跑完一个任务就退出，适合脚本和自动化：

```bash
codex exec "用一句话介绍这个项目"
```

输出开头会列出本次的配置，确认 `provider: xisu`：

```text
model: gpt-5.6-terra
provider: xisu
approval: never
sandbox: read-only
```

几个要点：

- 默认是只读沙箱（`sandbox: read-only`）。让它改文件时会报 `writing is blocked by read-only sandbox`，文件不会被创建。
- 需要写文件时加 `--sandbox workspace-write`（简写 `-s workspace-write`），允许改当前目录里的文件：

```bash
codex exec -s workspace-write "创建文件 hello.txt，内容为 hi"
```

- Windows 上还要配好沙箱，否则即使加了 `-s workspace-write`，开头仍显示 `sandbox: read-only`。在交互模式里完成过一次沙箱设置就行，或者直接在 `config.toml` 末尾加上：

```toml
[windows]
sandbox = "unelevated"
```

- 旧教程里的 `--full-auto` 在 0.162.1 里已经没有了，会报 `unexpected argument '--full-auto' found`，用 `-s workspace-write` 代替。
- 不在 Git 仓库里运行会报 `Not inside a trusted directory and --skip-git-repo-check was not specified.`，加上 `--skip-git-repo-check` 即可。
- 临时换模型用 `-m`，例如 `codex exec -m gpt-5.6-luna "…"`。
- 输出里夹杂的 WARN 行（例如访问 chatgpt.com 插件接口返回 401）是 Codex 自身的功能检查，和犀速AI 无关，可以忽略。
- 一次简单的 `codex exec` 任务大约消耗 1 万输入 token，其中大部分命中缓存，按缓存价计费。每次调用都能在控制台「调用明细」里查到。**数据来自 犀速-AI 官网公布的实测（品牌方自测）。**

## 5. 选择模型

截至 2026-10-16，下面这些模型实测都能通过 Responses 接口调用。**数据来自 犀速-AI 官网公布的实测（品牌方自测）。**

| 场景 | 推荐模型 |
|---|---|
| 日常写代码（默认） | `gpt-5.6-terra` |
| 更难的任务 | `gpt-6-sol`、`gpt-6.1-sol`、`gpt-5.6-sol` |
| 简单任务、最省钱 | `gpt-6-luna`、`gpt-5.6-luna` |

Codex 和 GPT 系列配合最好，建议用 GPT 模型。单价见模型广场。

换模型有三种方式：

- 改 `config.toml` 里的 `model`
- 启动时加 `-m` 模型名
- 交互模式里输入 `/model`

## 6. Responses 和 Chat Completions 有什么区别

犀速AI 的 OpenAI 兼容地址同时提供两套对话接口。Codex 用 `/v1/responses`，Cherry Studio、Chatbox 等大多数客户端用 `/v1/chat/completions`。两套用同一把密钥、同一个余额，区别在请求和响应的格式：

| 项目 | `/v1/chat/completions` | `/v1/responses` |
|---|---|---|
| 对话内容 | `messages`（消息数组） | `input`（字符串，或消息数组） |
| 系统提示 | `messages` 里 `role: "system"` 的消息 | `instructions` 字段 |
| 输出长度上限 | `max_tokens` | `max_output_tokens` |
| 取回答 | `choices[0].message.content` | `output` 里 `type: "message"` 的内容；SDK 里直接用 `output_text` |
| 用量字段 | `usage.prompt_tokens / usage.completion_tokens` | `usage.input_tokens / usage.output_tokens` |
| 缓存命中 | `usage.prompt_tokens_details.cached_tokens` | `usage.input_tokens_details.cached_tokens` |
| 流式格式 | 每块 `data: {…}`，读 `choices[0].delta.content`，以 `data: [DONE]` 结尾 | 带 `event:` 的事件流，读 `response.output_text.delta`，以 `response.completed` 结束，没有 `[DONE]` |
| 流式用量 | 最后一个数据块里的 `usage` | 只在 `response.completed` 事件里出现一次 |
| 多轮对话 | 每次带上完整 `messages` | 每次带上完整 `input`，不支持 `previous_response_id` |

## 同一个请求的两种写法：curl

### Chat Completions

```bash
curl https://api.xisu.ai/v1/chat/completions \\
-H "Authorization: Bearer $XISU_API_KEY" \\
-H "Content-Type: application/json" \\
-d '{
"model": "gpt-5.6-luna",
"messages": [
{"role": "system", "content": "你是简洁的助手"},
{"role": "user", "content": "用一句话解释什么是 API"}
],
"max_tokens": 200
}'
```

返回（节选）：

```json
{
"object": "chat.completion",
"model": "gpt-5.6-luna",
"choices": [
{"message": {"role": "assistant", "content": "API 是让不同软件或系统之间能够相互通信和调用功能的一组规则与接口。"}, "finish_reason": "stop"}
],
"usage": {"prompt_tokens": 358, "completion_tokens": 25, "total_tokens": 383}
}
```

### Responses

```bash
curl https://api.xisu.ai/v1/responses \\
-H "Authorization: Bearer $XISU_API_KEY" \\
-H "Content-Type: application/json" \\
-d '{
"model": "gpt-5.6-luna",
"instructions": "你是简洁的助手",
"input": "用一句话解释什么是 API",
"max_output_tokens": 200
}'
```

返回（节选）：

```json
{
"object": "response",
"status": "completed",
"model": "gpt-5.6-luna",
"output": [
{"type": "message", "role": "assistant", "content": [{"type": "output_text", "text": "API 是让不同软件之间能够互相通信和调用功能的一组规则和接口。"}]}
],
"usage": {"input_tokens": 358, "output_tokens": 23, "total_tokens": 381}
}
```

这两段 curl 命令适用于 macOS / Linux 终端，以及 Windows 上的 Git Bash 或 WSL。

## 同一个请求的两种写法：Python

先安装 SDK：`pip install openai`。

```python
import os
from openai import OpenAI
client = OpenAI(api_key=os.environ["XISU_API_KEY"], base_url="https://api.xisu.ai/v1")
# Chat Completions：对话放在 messages 里
chat = client.chat.completions.create(
    model="gpt-5.6-luna",
    messages=[
        {"role": "system", "content": "你是简洁的助手"},
        {"role": "user", "content": "用一句话解释什么是 API"},
    ],
    max_tokens=200,
)
print(chat.choices[0].message.content)
print(chat.usage.prompt_tokens, chat.usage.completion_tokens)
# Responses：系统提示放 instructions，用户输入放 input
resp = client.responses.create(
    model="gpt-5.6-luna",
    instructions="你是简洁的助手",
    input="用一句话解释什么是 API",
    max_output_tokens=200,
)
print(resp.output_text)
print(resp.usage.input_tokens, resp.usage.output_tokens)
```

## 流式输出的区别

```python
import os
from openai import OpenAI
client = OpenAI(api_key=os.environ["XISU_API_KEY"], base_url="https://api.xisu.ai/v1")
# Chat Completions 流式：读 choices[0].delta.content，最后一块带 usage
stream = client.chat.completions.create(
    model="gpt-5.6-luna",
    messages=[{"role": "user", "content": "数到3"}],
    stream=True,
)
for chunk in stream:
    if chunk.choices and chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="", flush=True)
    if chunk.usage:
        print("\\nusage:", chunk.usage.prompt_tokens, chunk.usage.completion_tokens)
# Responses 流式：按事件类型处理，usage 只在 response.completed 里
stream = client.responses.create(model="gpt-5.6-luna", input="数到3", stream=True)
for event in stream:
    if event.type == "response.output_text.delta":
        print(event.delta, end="", flush=True)
    elif event.type == "response.completed":
        u = event.response.usage
        print("\\nusage:", u.input_tokens, u.output_tokens)
```

原始的 Chat Completions 流大致如下（省略了部分字段），最后以 `[DONE]` 收尾：

```text
data: {"object":"chat.completion.chunk","choices":[{"delta":{"content":"1"}}]}
...
data: {"object":"chat.completion.chunk","choices":[{"delta":{},"finish_reason":"stop"}],"usage":{"prompt_tokens":305,"completion_tokens":9,"total_tokens":314}}
data: [DONE]
```

Responses 流是带名字的事件，依次是：

```text
response.created → response.in_progress → response.output_item.added → response.content_part.added
→ response.output_text.delta（多次）→ response.output_text.done → response.content_part.done
→ response.output_item.done → response.completed
```

中间可能夹杂其他事件类型（例如 `response.metadata`），按 `type` 只处理你关心的即可。判断结束请看 `response.completed`，不要等 `[DONE]`。

## 多轮对话

犀速AI 不在服务器上保存对话，Responses 的多轮对话要自己把历史带上：

```python
import os
from openai import OpenAI
client = OpenAI(api_key=os.environ["XISU_API_KEY"], base_url="https://api.xisu.ai/v1")
# 多轮对话：不用 previous_response_id，把历史整段放进 input
history = [
    {"role": "user", "content": "我叫小林"},
    {"role": "assistant", "content": "你好，小林！"},
    {"role": "user", "content": "我叫什么？"},
]
resp = client.responses.create(model="gpt-5.6-luna", input=history, max_output_tokens=100)
print(resp.output_text)
```

## 三个不支持的字段

下面三个依赖服务端存储的字段会直接返回 400，不会被悄悄忽略：

| 字段 | 返回的错误 | 怎么改 |
|---|---|---|
| `previous_response_id` | `'previous_response_id' is not supported on this gateway …` | 把完整对话放进 `input` |
| `background` | `'background' is not supported on this gateway …` | 改用普通请求或流式 |
| `store: true` | `'store: true' is not supported on this gateway …` | 改成 `false` 或不传 |

Codex CLI 本身不依赖这三个字段，正常使用不受影响。Responses 接口的完整说明（服务端工具计费、404 与 503 的区别等）见 OpenAI Responses 端点。

## 常见问题

### Codex CLI 提示 Missing environment variable: XISU_API_KEY 怎么办？

环境变量没设，或者设完没有重新打开终端。按第 3 步重新设置，然后开一个新终端。

### 为什么一直 Reconnecting... 2/5，最后报 401 Unauthorized: invalid API key？

密钥不对。检查是否复制完整、有没有多余空格，或者密钥已被吊销，需要重新创建。

### 文件写不进去，提示 read-only sandbox 怎么办？

`codex exec` 默认只读，加 `-s workspace-write`；Windows 上还要按第 4 步配好沙箱。

### `--full-auto` 报 unexpected argument 怎么解决？

0.162.1 已经去掉这个参数，用 `-s workspace-write` 代替。

### 提示模型不存在或不可用怎么办？

把 `model` 换成模型广场里在售的模型名。返回 `404 model_not_on_endpoint` 说明这个模型不能走 Responses 接口，换一个；返回 `503` 是暂时没有可用资源，稍后重试。

### 想换回 OpenAI 官方账号怎么设置？

删除 `config.toml` 里的 `model_provider = "xisu"` 这一行。

### 桌面版能用吗？

能，配置完全一样，见 Codex 桌面版教程。其他客户端见犀速AI 客户端下载与配置大全。

## 关于 犀速-AI

犀速-AI 提供 OpenAI 兼容接口，本文涉及的 Codex CLI 配置、Responses API 与 Chat Completions 用法均以官网原文为准。

官网：https://xisu.ai/