# 2026 年 10 月 Claude Code、Codex CLI、Cursor 共用 API：配置与报错排查

> 本文整理自 [犀速-AI 官网原文](https://www.xisu.ai/resources/articles/claude-code-codex-cursor-api)，原文更新于 2026-10-17。

## 结论先看

- 一把 API Key 可以同时接入 Claude Code、Codex CLI 和 Cursor，但三个工具必须分别使用正确的协议端点：Claude Code 使用 Anthropic Messages，Codex CLI 使用 OpenAI Responses，Cursor 使用 OpenAI Chat Completions。
- `base_url` 的填写方式不同：Claude Code 使用 `https://api.xisu.ai`，不带 `/v1`；Codex CLI 和 Cursor 使用 `https://api.xisu.ai/v1`。
- 最容易出错的地方是 `base_url` 是否带 `/v1`、模型名是否匹配协议，以及客户端是否真正加载了新的环境变量。

## 为什么现在值得看

本文最后更新于 2026-10-17，内容覆盖 Claude Code 2.1.300、Codex CLI 0.162.1 的配置记录，并结合近期常见的 401、404、429、400、503 报错整理排查顺序。Claude Code、Codex CLI 和 Cursor 走的是不同协议，先确认端点、模型名和环境变量，可以减少晚高峰或接口异常时反复试错的时间。

## Claude Code、Codex CLI、Cursor 各走自己的协议，共用一个账户

一把 API Key 能否同时接入 Claude Code、Codex CLI 和 Cursor？可以共用，但不是把三个工具都填成同一个完整地址：Claude Code 走 Anthropic Messages，Codex CLI 走 OpenAI Responses，Cursor 走 OpenAI Chat Completions。

实际配置时，你可以给三个工具使用同一把密钥，也可以为每个工具分别创建密钥。前者配置简单，后者更方便分开看账单、设置预算和定位请求。

本文适用对象：第一次接触大模型 API 的开发者、想把编码工具接到第三方 API 的个人用户，以及需要排查协议和模型问题的老手。

最后更新：2026-10-17

实测或资料环境：Claude Code 教程记录为 Claude Code 2.1.300；Codex CLI 教程记录为 codex-cli 0.162.1；Cursor 部分依据 Cursor 官方文档整理，本站尚未完成完整实测。版本记录数据来自 犀速-AI 官网公布的实测（品牌方自测）。

## 一、先看结论：三个工具各走自己的协议端点

大白话：API 就像一个前台，`base_url` 是前台地址，API Key 是门禁卡，模型名是你点名要找的服务。三个工具虽然可以共用同一把门禁卡，但它们递交请求的表格不同，所以必须进入各自对应的端点。

| 工具 | 协议 | base_url 填法 | 密钥放哪 | 模型名填法 | 推荐默认模型 | 配置文件位置 |
| --- | --- | --- | --- | --- | --- | --- |
| Claude Code | Anthropic Messages | `https://api.xisu.ai`，不带 `/v1` | `ANTHROPIC_AUTH_TOKEN` 或 `settings.json` 的 `env` | 直接填 Claude 模型 ID | `claude-sonnet-5-5` | `~/.claude/settings.json` |
| Codex CLI | OpenAI Responses | `https://api.xisu.ai/v1` | `XISU_API_KEY` | 直接填 Responses 可用模型 ID | `gpt-5.6-terra` | `~/.codex/config.toml` |
| Cursor | OpenAI Chat Completions | `https://api.xisu.ai/v1` | Cursor Settings > Models > OpenAI API Key | 添加与接口返回一致的模型名 | `gpt-5.6-terra` | Cursor 设置界面 |

本站的 OpenAI 兼容接口说明了自定义客户端通常只需要修改 `base_url` 和 API Key；Anthropic 原生端点则需要把地址指向不带 `/v1` 的根地址。可先阅读 OpenAI 兼容接入文档、Anthropic Messages 文档和端点与鉴权文档。

## 二、先把三个概念弄懂

### 1. base_url 是什么

`base_url` 就是客户端拼接 API 路径时使用的基础地址。客户端往往不会只请求你填写的字符串，而是继续拼接路径。例如 Claude Code 会在地址后面拼接 `/v1/messages`，所以它填写 `https://api.xisu.ai`；Codex CLI 会请求 `/v1/responses`，所以它填写 `https://api.xisu.ai/v1`。

如果 Claude Code 填成 `https://api.xisu.ai/v1`，最终路径可能变成 `/v1/v1/messages`，常见结果是 404，或者提示 “There's an issue with the selected model”。这个提示不一定代表模型名错了，先检查地址。

### 2. API Key 是什么

API Key 是用来识别账号、余额、套餐和预算的密钥。本文中的示例只使用 `sk-xxxx` 或环境变量名，不把真实密钥写进代码、配置仓库或截图。

OpenAI 兼容端点使用 `Authorization: Bearer`；Anthropic Messages 端点使用 `x-api-key`，也接受 Bearer。两套协议共用同一把密钥和同一个余额，但客户端对密钥变量的命名不同。

### 3. 模型名是什么

模型名是请求体中的 `model` 字段，必须是平台实际提供、并且被当前端点支持的模型 ID。模型名不是“Claude”“GPT”这类展示名称，也不是 Cursor 下拉菜单里的任意自定义标签。

建议先调用 `GET /v1/models`，再把返回的模型 ID 原样复制到工具配置中。不同协议可用模型可能是子集：Claude Code 只能通过 Messages 使用 Claude 系列；Responses 端点也有自己的可用模型范围。

## 三、三种协议到底有什么不同

大白话：三种协议都是“发消息、收回答”，但请求字段、响应字段和流式结束方式不同。工具如果写死了某一种协议，不能只靠换模型名解决。

### Chat Completions、Responses、Anthropic Messages 三种协议对照

| 协议 | 端点路径 | 请求体关键字段 | 响应读取方式 | 谁在用 |
| --- | --- | --- | --- | --- |
| OpenAI Chat Completions | `/v1/chat/completions` | `messages`、`max_tokens` | `choices[0].message.content` | Cursor、许多桌面客户端、传统 OpenAI SDK 调用 |
| OpenAI Responses | `/v1/responses` | `input`、`instructions`、`max_output_tokens` | `output` 或 SDK 的 `output_text` | Codex CLI、OpenAI SDK 的 responses 接口 |
| Anthropic Messages | `/v1/messages` | `model`、`messages`、`max_tokens` | Anthropic 原生消息结构 | Claude Code、Anthropic SDK |

Chat Completions 的多轮历史通常放在 `messages`；Responses 的多轮历史要把完整 `input` 带上，本站网关不支持 `previous_response_id`；Anthropic Messages 则按 Anthropic 的消息格式提交。更细的字段差异可参考 Responses 端点文档。

## 四、第 0 步：先用 curl 验证密钥和模型

大白话：不要一上来就调试 Claude Code 或 Codex。先确认密钥能列模型，再分别测试 Chat Completions 和 Messages。这样能把“客户端配置问题”和“接口本身不可用”分开。

### macOS/Linux：列出模型

```bash
export XISU_API_KEY="sk-xxxx"
curl https://api.xisu.ai/v1/models \\
  -H "Authorization: Bearer $XISU_API_KEY"
```

成功时会返回模型列表。重点看 HTTP 状态码和模型 ID；如果是 401，优先检查密钥、环境变量和 IP 白名单。

### Windows PowerShell：列出模型

```powershell
$env:XISU_API_KEY = "sk-xxxx"
Invoke-RestMethod -Uri "https://api.xisu.ai/v1/models" -Headers @{ Authorization = "Bearer $env:XISU_API_KEY" }
```

PowerShell 会把 JSON 解析成对象并显示结果。当前窗口用 `$env:` 设置的变量只对当前会话有效；如果使用永久用户变量，设置后要重新打开 PowerShell。

### macOS/Linux：测试 Chat Completions 和 Anthropic Messages

```bash
curl https://api.xisu.ai/v1/chat/completions \\
  -H "Authorization: Bearer $XISU_API_KEY" \\
  -H "Content-Type: application/json" \\
  -d '{
    "model": "gpt-5.6-terra",
    "messages": [{"role": "user", "content": "用一句话解释 API"}],
    "max_tokens": 100
  }'

curl https://api.xisu.ai/v1/messages \\
  -H "x-api-key: $XISU_API_KEY" \\
  -H "anthropic-version: 2023-06-01" \\
  -H "Content-Type: application/json" \\
  -d '{
    "model": "claude-sonnet-5-5",
    "max_tokens": 100,
    "messages": [{"role": "user", "content": "用一句话解释 API"}]
  }'
```

Chat Completions 成功时通常能看到 `choices` 和回答内容；Messages 成功时能看到 Anthropic 形状的 `content`。Claude 测试必须使用 Claude 系列模型，不能把 GPT 模型发到 Messages。

### Windows PowerShell：测试两个对话端点

```powershell
$headers = @{
  Authorization = "Bearer $env:XISU_API_KEY"
  "Content-Type" = "application/json"
}
$chatBody = @{
  model = "gpt-5.6-terra"
  messages = @(@{ role = "user"; content = "用一句话解释 API" })
  max_tokens = 100
} | ConvertTo-Json -Depth 5
Invoke-RestMethod -Uri "https://api.xisu.ai/v1/chat/completions" -Method Post -Headers $headers -Body $chatBody

$anthropicHeaders = @{
  "x-api-key" = $env:XISU_API_KEY
  "anthropic-version" = "2023-06-01"
  "Content-Type" = "application/json"
}
$messageBody = @{
  model = "claude-sonnet-5-5"
  max_tokens = 100
  messages = @(@{ role = "user"; content = "用一句话解释 API" })
} | ConvertTo-Json -Depth 5
Invoke-RestMethod -Uri "https://api.xisu.ai/v1/messages" -Method Post -Headers $anthropicHeaders -Body $messageBody
```

如果两个请求都能返回回答，密钥和基础接口基本正常。后续某一个工具仍失败时，就重点检查它的协议、`base_url`、模型名和配置加载顺序。

## 五、Claude Code 配置

大白话：Claude Code 只需要知道三件事：去哪里请求、用哪把密钥、默认使用哪个 Claude 模型。

1. 安装 Claude Code。教程记录的版本是 2.1.300；安装包要求 Node.js 22 或更高版本时，可先运行 `node -v` 检查。
2. 设置 `ANTHROPIC_BASE_URL`、`ANTHROPIC_AUTH_TOKEN`、`ANTHROPIC_MODEL` 和 `ANTHROPIC_DEFAULT_HAIKU_MODEL`。
3. 打开新终端，进入项目目录，运行 `claude`。
4. 进入会话后运行 `/status`，确认 base URL、认证变量和模型都符合预期。

### macOS/Linux 环境变量

```bash
cat >> ~/.zshrc <<'EOF'
export ANTHROPIC_BASE_URL="https://api.xisu.ai"
export ANTHROPIC_AUTH_TOKEN="sk-xxxx"
export ANTHROPIC_MODEL="claude-sonnet-5-5"
export ANTHROPIC_DEFAULT_HAIKU_MODEL="claude-haiku-5-5"
EOF
source ~/.zshrc
claude -p "用一句话说明你能做什么"
```

macOS 默认使用 `~/.zshrc`；如果你使用 bash，把文件名替换为 `~/.bashrc`。非交互命令能返回一句话，通常说明基本接通。

### Windows PowerShell 环境变量

```powershell
[Environment]::SetEnvironmentVariable("ANTHROPIC_BASE_URL", "https://api.xisu.ai", "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_AUTH_TOKEN", "sk-xxxx", "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_MODEL", "claude-sonnet-5-5", "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_DEFAULT_HAIKU_MODEL", "claude-haiku-5-5", "User")
# 关闭当前 PowerShell，再打开新窗口
claude -p "用一句话说明你能做什么"
```

没有输出通常表示写入成功。已经打开的终端读不到新写入的用户环境变量，所以必须关闭并重新打开 PowerShell。

### 使用 ~/.claude/settings.json

不想修改系统环境变量时，可以把配置写入 `~/.claude/settings.json`；Windows 对应 `%USERPROFILE%\\.claude\\settings.json`。

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://api.xisu.ai",
    "ANTHROPIC_AUTH_TOKEN": "sk-xxxx",
    "ANTHROPIC_MODEL": "claude-sonnet-5-5",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "claude-haiku-5-5"
  }
}
```

配置文件中的密钥是明文。更稳妥的做法是只在 `settings.json` 中保存地址和模型，把密钥留在环境变量中，并把配置文件加入 Git 检查范围。

### 检查模型和特殊行为

日常写代码可以使用 `claude-sonnet-5-5`；难题和大范围重构可以考虑 `claude-opus-5-5`；简单任务或后台小任务可以使用 `claude-haiku-5-5`。Claude Code 只能使用 Claude 系列模型，非 Claude 模型通过 `/v1/messages` 调用会出现 `404 not_found_error`。

进入会话后运行 `/status`，应能看到类似以下信息：`Auth token: ANTHROPIC_AUTH_TOKEN`、`Anthropic base URL: https://api.xisu.ai`、`Model: 你设置的模型`。

教程记录了几个容易忽略的行为：首次请求可能写入约 3 万 token 的提示缓存，缓存大约 5 分钟后过期；连续工作时后续请求可能命中缓存。Claude Code 的 WebSearch 工具在该端点不支持，可能返回 `400 server-side tool 'web_search_20250305' is not enabled on this endpoint`。如果不希望它继续尝试，可以在 `settings.json` 中加入：

```json
{
  "permissions": {
    "deny": ["WebSearch"]
  }
}
```

密钥错误时，Claude Code 可能长时间静默重试。可以按 `Ctrl+C` 退出，再使用 `claude --debug` 查看 401 线索。

## 六、Codex CLI 配置

大白话：Codex CLI 使用 Responses 协议，因此必须把地址写成带 `/v1` 的形式，并明确指定 `wire_api = "responses"`。

1. 安装：`npm i -g @openai/codex`。
2. 在用户目录创建 `.codex/config.toml`。
3. 设置 `XISU_API_KEY` 环境变量。
4. 运行 `codex` 或 `codex exec`，再通过输出和 `/status` 检查 provider。

### 完整 config.toml

```toml
model_provider = "xisu"
model = "gpt-5.6-terra"
[model_providers.xisu]
name = "犀速AI"
base_url = "https://api.xisu.ai/v1"
wire_api = "responses"
env_key = "XISU_API_KEY"
[windows]
sandbox = "unelevated"
```

macOS/Linux 文件位置是 `~/.codex/config.toml`；Windows 也使用用户目录下的 `.codex\\config.toml`。如果文件原来有内容，把前两行放到文件顶部，把 provider 段落放到文件末尾；不要无意覆盖已有配置。

### 创建文件和设置密钥

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
echo 'export XISU_API_KEY="sk-xxxx"' >> ~/.zshrc
source ~/.zshrc
codex --version
codex
```

看到类似 `codex-cli 0.162.1` 表示版本检查成功；版本可能更新。PowerShell 可用下面的写法：

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\\.codex" | Out-Null
notepad "$env:USERPROFILE\\.codex\\config.toml"
[Environment]::SetEnvironmentVariable("XISU_API_KEY", "sk-xxxx", "User")
# 关闭 PowerShell 后重新打开
codex --version
codex
```

配置中的 `env_key` 只是告诉 Codex CLI 去哪个环境变量取密钥，密钥本身不应写进 TOML。进入交互模式后运行 `/status`，看到 `Model provider: xisu` 才能确认请求走的是本站。

### 运行任务和 Windows 沙箱

```bash
cd 你的项目目录
codex exec -s workspace-write --skip-git-repo-check "创建文件 hello.txt，内容为 hi"
```

`-s workspace-write` 允许修改当前工作目录；默认通常是只读沙箱。教程记录的 0.162.1 已移除 `--full-auto`，继续使用它会出现 `unexpected argument '--full-auto' found`。如果不在 Git 仓库中运行，加入 `--skip-git-repo-check`。

Windows 如果出现无法使用管理员权限建立沙箱的提示，可以选择 non-admin sandbox；也可以在配置中加入 `[windows]` 和 `sandbox = "unelevated"`。新版本参数可能变化，以 `codex --help` 为准。

Responses 端点不保存会话，因此不支持 `previous_response_id`。需要多轮对话时，客户端应把完整历史放进 `input`。Codex CLI 的正常使用不依赖这个字段。

## 七、Cursor 配置

本站尚未完成 Cursor 的完整实测，以下步骤依据 Cursor 官方文档整理，界面以你安装的版本为准；遇到问题请带 `request_id` 反馈。

大白话：Cursor 的自定义 OpenAI 接口入口在 Settings > Models。你需要填 OpenAI API Key、打开使用自定义密钥的开关，并启用 Override OpenAI Base URL。Cursor 官方文档说明，自定义 API Key 主要用于标准聊天模型；需要专用模型的功能，例如 Tab Completion，仍可能使用 Cursor 自己的模型。

1. 打开 Cursor Settings。
2. 进入 Models。
3. 在 OpenAI API Key 中填入 `sk-xxxx`，启用使用自己的 OpenAI API Key，并点击 Verify。
4. 启用 Override OpenAI Base URL。
5. 填写 `https://api.xisu.ai/v1`。
6. 在模型列表或自定义模型入口添加平台实际模型名，例如 `gpt-5.6-terra`。
7. 保存后重新打开一个聊天窗口，选择刚添加的模型。

Cursor 官方页面写明，API Key 配置后会直接调用对应模型提供方；自定义密钥只适用于标准聊天模型，Tab Completion 等需要专用模型的功能继续使用内置模型。具体可用能力以 Cursor 当前版本界面和官方说明为准。

Cursor 的模型名称必须与请求端点期待的模型 ID 一致。如果模型列表为空，先确认密钥有效、余额不是负数，并检查客户端是否请求了 `/v1/models`。如果填完地址仍然连接旧服务，重启 Cursor 并刷新模型列表。

## 八、三个工具怎么选模型

大白话：先按协议筛选，再按任务选择。Claude Code 先限定 Claude 系列；Codex CLI 先选择 Responses 可用的 GPT 系列；Cursor 则要确认你添加的模型能被 Chat Completions 调用。

| 任务类型 | 推荐模型 | 理由 |
| --- | --- | --- |
| 日常代码、一般重构 | `claude-sonnet-5-5` 或 `gpt-5.6-terra` | 能力和价格档位较均衡，适合作为默认模型。 |
| 难题、复杂调试、大范围重构 | `claude-opus-5-5`、`gpt-6.1-sol` | 更适合需要较强分析和规划能力的任务，价格档位相对更高。 |
| 简单问答、标题、轻量修改 | `claude-haiku-5-5`、`gemini-3.8-flash-high`、`gpt-6-luna` | 适合轻量任务和后台小请求，价格档位更适合控制日常消耗。 |
| 需要在 GPT、Claude、Gemini 之间比较 | `gpt-5.6-terra`、`claude-sonnet-5-5`、`gemini-3.8-flash-high` | 分别通过对应协议端点验证，再根据代码风格和任务表现选择。 |

具体单价不要凭记忆判断，请以模型价格表 / 模型广场为准。模型是否能走某个协议，也要结合 `/v1/models`、控制台可用模型和实际请求结果确认。

## 九、报错排查：先看状态码，再看 request_id

大白话：错误码不是“重新试一次”四个字的同义词。401 通常查认证，404 通常查路径或端点能力，429 先区分限流和余额，503 才适合按 `Retry-After` 等待重试。

| 错误 | 常见原因 | 处理顺序 |
| --- | --- | --- |
| 401 | 密钥错误、变量没生效、终端没重启、IP 白名单不匹配 | 检查密钥完整性、环境变量、来源 IP、请求头和控制台状态。 |
| 404 | Claude Code 地址多了 `/v1`、模型名拼错、模型不属于当前协议 | 检查最终路径，再确认模型与 Messages 或 Responses 的匹配关系。 |
| 429 | 限流、余额不足、密钥预算用完 | 先查看余额和预算；确定是限流后再按 `Retry-After` 退避。 |
| 400 | 参数格式或工具 schema 不被接受，WebSearch 等服务端工具不可用 | 阅读错误 message，移除不支持字段或禁用对应工具。 |
| 503 | `overloaded_error` 或暂时没有可用上游 | 读取 `Retry-After`，等待后新建请求重试。 |
| timeout | 上游排队、客户端超时设置过短或网络连接中断 | 适当增加超时；断流后新开请求，这个接口没有续传语义。 |

### 401：先确认请求到底带了哪把密钥

Windows 永久设置环境变量后必须重新打开 PowerShell；macOS/Linux 写入 shell 配置后要执行 `source ~/.zshrc` 或重新开终端。Claude Code 可以用 `/status` 检查认证变量名和 `base_url`。若密钥配置了 IP 白名单，来源地址不在名单中也会返回 401。

### 404：区分路径错误和端点不支持

Claude Code 的地址必须是 `https://api.xisu.ai`，不能写成带 `/v1` 的地址。Codex CLI 和 Cursor 则使用 `https://api.xisu.ai/v1`。如果模型名写对但把 GPT 模型发到 Anthropic Messages，可能得到 `404 not_found_error`；这不是等待重试能解决的问题，需要换 Claude 模型或改用支持该模型的 OpenAI 端点。

### 429：限流与余额不足要分开

如果确认是请求频率限制，按响应中的 `Retry-After` 退避，并使用指数退避限制重试次数。Anthropic 协议下，余额不足、配额用尽或密钥预算到顶也可能表现为 `429 rate_limit_error`；先看余额和预算，不要只等待。

### 400：工具 schema 和 WebSearch

自定义函数工具通常要符合接口允许的 schema。可以参考工具 schema 被拒排查文档。Claude Code 的 WebSearch 属于服务端工具，在该端点不支持时会返回 400；读写文件、运行命令和 WebFetch 不受这条限制影响。

更多状态码、限流和重试建议见错误码与限流文档。如果是 503 `overloaded_error` 或 Responses 的 `no_available_channel`，通常属于暂时没有可用资源，应按响应给出的等待时间退避后重新发起请求。

## 十、如何看 request_id、token 和费用

大白话：不要只看客户端最后显示的一句话。每个请求的响应头或响应体通常会带 `request_id`；拿它去控制台请求日志查，才能把模型、token、错误和费用对应起来。

1. 在客户端或调试日志中找到 `request_id`。
2. 打开控制台，在请求日志或调用明细中按 `request_id` 搜索。
3. 核对实际模型名、协议端点、输入 token、输出 token、缓存写入、缓存读取和费用。
4. 如果客户端显示失败但余额变化，优先以控制台调用明细为准，再结合请求时间和 `request_id` 排查。

Claude Code 可用 `/status` 查看当前认证、地址和模型；费用与 token 细项以控制台调用明细为准。Codex CLI 的 Responses 返回字段通常是 `input_tokens`、`output_tokens`；Chat Completions 则常见 `prompt_tokens`、`completion_tokens`。Anthropic Messages 使用其原生用量字段。

本站不在服务器端保存 Responses 对话，因此不要用 `previous_response_id` 依赖服务端自动续接。需要核对多轮费用时，把完整历史作为本次请求输入，并在控制台逐条对照。可进一步阅读用量与账单核对文档。

## 十一、安全和账单隔离建议

大白话：能共用一把密钥，不代表一定应该共用一把。开发阶段可以共用，长期使用更适合按工具或项目拆分。

- Claude Code、Codex CLI、Cursor 分别创建密钥，方便按工具看账单。
- 为每把密钥设置预算上限，避免某个后台任务持续重试。
- 密钥只放环境变量、系统密钥存储或受保护的本地配置，不要提交到 Git。
- 把包含密钥的本地配置加入 `.gitignore`，提交前搜索 `sk-`、`ANTHROPIC_AUTH_TOKEN` 和 `XISU_API_KEY`。
- 如果怀疑密钥泄露，立即在控制台停用或重建，并检查请求日志和 IP 白名单。

更完整的密钥管理建议见密钥安全文档。生产环境还应设置合理超时、有限次数重试、日志脱敏和请求关联 ID，相关上线建议见生产环境文档。

如果你准备开始配置，可以先到控制台创建一把密钥，再按 Claude Code 教程和 Codex CLI 教程逐项操作；客户端安装包可从下载页获取。建议先完成第 0 步的三个 curl 验证，再接入具体工具，这样遇到问题时更容易定位。

## 常见问题

### 一把 API Key 可以同时给 Claude Code、Codex CLI 和 Cursor 使用吗？

可以。三个工具可以共享同一把密钥和余额，但必须分别使用正确的协议端点：Claude Code 使用 Anthropic Messages，Codex CLI 使用 Responses，Cursor 使用 Chat Completions。若希望分开看账单或设置预算，建议每个工具单独创建一把。

### 为什么 Claude Code 的 base_url 不能带 /v1？

Claude Code 会自己拼接 `/v1/messages`。如果你填写 `https://api.xisu.ai/v1`，最终可能形成 `/v1/v1/messages`，从而返回 404 或 selected model 提示。Claude Code 应填写 `https://api.xisu.ai`。

### Codex CLI 为什么必须写 wire_api = "responses"？

Codex CLI 使用 OpenAI Responses 协议，请求路径是 `/v1/responses`。`wire_api = "responses"` 用来告诉客户端按 Responses 的请求和响应格式工作，而不是按 Chat Completions 格式发送。

### Cursor 里填了自定义地址，为什么模型列表还是空的？

先确认 Cursor 中已启用自己的 OpenAI API Key，并填写了 `https://api.xisu.ai/v1`。然后确认密钥有效、余额不是负数，并检查客户端是否请求了 `/v1/models`。如果填完地址仍然连接旧服务，重启 Cursor 并刷新模型列表。

## 关于 犀速-AI

犀速-AI 官网提供 API 接入、模型和端点相关文档，本文配置中的地址、协议和排查信息以官网原文为准。

官网：https://xisu.ai/