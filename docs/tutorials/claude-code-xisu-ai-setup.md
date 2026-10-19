# Claude Code 接入犀速-AI：Windows、macOS、Linux 配置教程（2026 年 10 月）

> 本文整理自 [犀速-AI 官网原文](https://www.xisu.ai/resources/tutorials/claude-code)，原文更新于 2026-10-16。

## 结论先看

- **安装 Claude Code、设置地址、密钥和模型三个环境变量后，就能在终端里通过犀速-AI 调用 Claude 模型写代码。**
- Claude Code 2.1.300 的 npm 安装包要求 **Node.js 22 或更高版本**；也可以使用官方安装脚本，**不需要 Node.js**。
- Claude Code 每次请求会携带大约 **3 万 token** 的系统提示和工具说明；实测同一个命令连续运行两次，第二次约 3 万 token 全部命中缓存，费用只有第一次的二十几分之一。数据来自 犀速-AI 官网公布的实测（品牌方自测）。

## 为什么现在值得看

本文更新于 **2026-10-16**，并基于 **Claude Code 2.1.300** 实测，模型名也以该日期为准。近期配置 Claude Code 时，地址是否带 `/v1`、环境变量是否生效、后台小模型是否单独设置，以及 429、404、503 等常见问题，都可能直接影响调用和费用，按本文步骤配置可以更快完成接入和排查。

Claude Code 是 Anthropic 官方的命令行编程助手，能在你的项目里读代码、改文件、跑命令。犀速-AI 提供 Anthropic 原生接口，按本文设好地址和密钥后，Claude Code 直接通过犀速-AI 调用 Claude 模型，**不需要 Claude 账号**。

本文基于 Claude Code 2.1.300 实测，模型名截至 2026-10-16。数据来自 犀速-AI 官网公布的实测（品牌方自测）。

## 最短路径

1. 安装：

   ```bash
   npm i -g @anthropic-ai/claude-code
   ```

2. 设置环境变量：

   ```text
   ANTHROPIC_BASE_URL=https://api.xisu.ai
   ANTHROPIC_AUTH_TOKEN=你的密钥
   ANTHROPIC_MODEL=claude-sonnet-5-5
   ```

3. 打开新终端，进入项目目录，运行：

   ```bash
   claude
   ```

开始前请先准备好 API 密钥，还没有的话先看「新手第一步：注册、充值、创建密钥」。

## 1. 安装

两种方式任选一种。

### 方式一：npm 安装（Windows / macOS / Linux 通用）

需要先装好 Node.js。Claude Code 2.1.300 的安装包要求 **Node.js 22 或更高版本**，可以用 `node -v` 查看当前版本，版本太低就到 Node.js 官网下载 LTS 版。

```bash
npm i -g @anthropic-ai/claude-code
```

npm 下载慢的话，可以临时换成国内镜像源：

```bash
npm i -g @anthropic-ai/claude-code --registry=https://registry.npmmirror.com
```

### 方式二：官方安装脚本（不需要 Node.js）

macOS / Linux：

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

Windows（在 PowerShell 里运行）：

```powershell
irm https://claude.ai/install.ps1 | iex
```

安装脚本会从 Anthropic 的服务器下载程序，网络不通时会失败，这种情况改用方式一。

装好后检查版本：

```bash
claude --version
```

看到类似 `2.1.300 (Claude Code)` 就说明装好了。

## 2. 设置地址、密钥和模型

Claude Code 通过环境变量知道该连哪里、用哪个模型。需要设置这几项：

| 变量 | 填写内容 | 说明 |
| --- | --- | --- |
| `ANTHROPIC_BASE_URL` | `https://api.xisu.ai` | 不带 `/v1`，Claude Code 会自己拼上 `/v1/messages` |
| `ANTHROPIC_AUTH_TOKEN` | 你的密钥 | 和 OpenAI 兼容接口用的是同一把密钥 |
| `ANTHROPIC_MODEL` | `claude-sonnet-5-5` | 主模型。不设的话默认用 Opus，费用更高 |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | `claude-haiku-5-5` | 推荐设置。生成会话标题等后台小任务用它，见下文「后台小模型」 |

地址也可以从这里复制：

```text
https://api.xisu.ai
```

下面三种设置方法选一种即可。

### Windows：PowerShell 永久设置

打开 PowerShell，把密钥换成你自己的，逐行粘贴回车：

```powershell
[Environment]::SetEnvironmentVariable("ANTHROPIC_BASE_URL", "https://api.xisu.ai", "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_AUTH_TOKEN", "这里换成你的密钥", "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_MODEL", "claude-sonnet-5-5", "User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_DEFAULT_HAIKU_MODEL", "claude-haiku-5-5", "User")
```

没有输出就是成功了。这样写入的是用户环境变量，**已经打开的终端读不到**，请关掉 PowerShell 重新打开一个。

### macOS / Linux：写入 `~/.zshrc`

macOS 默认用 zsh。用 bash 的话把下面的 `~/.zshrc` 换成 `~/.bashrc`。把「你的密钥」换掉后整段粘贴：

```bash
cat >> ~/.zshrc <<'EOF'
export ANTHROPIC_BASE_URL="https://api.xisu.ai"
export ANTHROPIC_AUTH_TOKEN="你的密钥"
export ANTHROPIC_MODEL="claude-sonnet-5-5"
export ANTHROPIC_DEFAULT_HAIKU_MODEL="claude-haiku-5-5"
EOF
source ~/.zshrc
```

### 写进 Claude Code 的 `settings.json`

不想动系统环境变量的话，可以写进 Claude Code 自己的配置文件 `~/.claude/settings.json`（Windows 上是 `%USERPROFILE%\\.claude\\settings.json`）。文件不存在就新建：

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://api.xisu.ai",
    "ANTHROPIC_AUTH_TOKEN": "你的密钥",
    "ANTHROPIC_MODEL": "claude-sonnet-5-5",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "claude-haiku-5-5"
  }
}
```

文件里原来就有内容的话，把这几项合并进已有的 `"env"`，注意 JSON 的逗号。

这个文件里的密钥是明文，不要提交到 Git，也不要发给别人。更稳妥的做法是 `settings.json` 里只写地址和模型，密钥仍用环境变量设置。

## 3. 选择模型

截至 2026-10-16，Claude Code 可用的在售模型：

| 场景 | 推荐模型 | 说明 |
| --- | --- | --- |
| 日常写代码 | `claude-sonnet-5-5` | 能力和价格比较均衡，建议作为默认 |
| 难题、大范围重构 | `claude-opus-5-5` | 更强，价格明显高于 Sonnet |
| 简单任务、想省钱 | `claude-haiku-5-5` | 最便宜，速度快 |
| 最难的长任务 | `claude-fable-5-1` | 目前最强，也是最贵的一档 |

具体单价见「模型广场」。

切换模型有三种方式：

- 改 `ANTHROPIC_MODEL`，对以后所有会话生效；
- 启动时指定，只对这次生效：

  ```bash
  claude --model claude-opus-5-5
  ```

- 会话里输入 `/model` 选择。菜单里的 Opus / Sonnet / Haiku / Fable 在 2.1.300 里分别对应 `claude-opus-5-5`、`claude-sonnet-5-5`、`claude-haiku-5-5`、`claude-fable-5-1`。按 Enter 设为以后的默认，按 `s` 只用于本次会话。

`/model` 菜单里显示的每百万 token 价格是 Claude Code 内置的官方价，**不是犀速-AI 的价格**。实际扣费以「模型广场」和控制台「调用明细」为准。

## 后台小模型

除了主对话，Claude Code 还会发一些后台请求，例如根据你的第一句话自动生成会话标题。实测 2.1.300 里这类请求的模型按下面的顺序决定：

1. `ANTHROPIC_SMALL_FAST_MODEL`（旧变量名，仍然有效）；
2. `ANTHROPIC_DEFAULT_HAIKU_MODEL`；
3. 两个都没设时，直接用主模型。

所以主模型用 Opus 又没设第 2 项时，起标题这种小事也会按 Opus 计费。建议把 `ANTHROPIC_DEFAULT_HAIKU_MODEL` 设为 `claude-haiku-5-5`。

`ANTHROPIC_DEFAULT_SONNET_MODEL`、`ANTHROPIC_DEFAULT_OPUS_MODEL`、`ANTHROPIC_DEFAULT_FABLE_MODEL` 用来改 `/model` 菜单里各别名指向的模型。默认指向的都是在售模型，一般不用设；如果设了，**必须填在售的模型名**，填了下架或不存在的名字，选到它时会报 `400 model '…' is not available`。

## 4. 验证是否接通

先用非交互模式问一句：

```bash
claude -p "用一句话说明你能做什么"
```

几秒后返回一句话回答就说明接通了。

再进入项目目录启动交互模式：

```bash
cd 你的项目目录
claude
```

第一次启动会有几步引导：

- 选择配色主题，直接回车；
- 安全提示，回车继续；
- 询问是否信任当前文件夹，用方向键选 `Yes, I trust this folder` 后回车。

进入后，顶部会显示当前模型和 API Usage Billing。输入 `/status` 回车，确认这三行：

```text
Auth token: ANTHROPIC_AUTH_TOKEN
Anthropic base URL: https://api.xisu.ai
Model: 你设置的模型
```

然后就可以直接提需求了，比如「看看这个项目是做什么的」。每次调用都会出现在控制台「调用明细」里。

## 5. 费用说明

按 token 计费，输入、输出、缓存分开计价，单价见「模型广场」。没有月租。

Claude Code 每次请求都会带上很长的系统提示和工具说明，**大约 3 万 token**。会话的第一次请求会把这部分写入提示缓存，按缓存写入计费，所以第一次明显贵一些。

之后的请求会命中缓存，按便宜得多的缓存读取价计费。实测同一个命令连续跑两次，第二次约 3 万 token 全部命中缓存，费用只有第一次的二十几分之一。数据来自 犀速-AI 官网公布的实测（品牌方自测）。

缓存大约 5 分钟后过期。连续工作比隔很久问一句更省钱。

启动时偶尔会有 1～2 条请求返回 400，调用明细里显示为失败、费用为 0。这类失败不计费，Claude Code 会自动重试，不影响结果。

担心花超，可以在控制台给这把密钥设预算，用到上限就会停止调用。

调用明细里能看到每条请求的输入、输出、缓存写入、缓存读取 token 和费用。

## 6. 使用限制

### 地址不要带 `/v1`

写成 `https://api.xisu.ai/v1` 会拼出错误的路径，Claude Code 会提示：

```text
There's an issue with the selected model (claude-sonnet-5-5). It may not exist or you may not have access to it. Run --model to pick a different model.
```

这时模型名其实没写错，把地址改成 `https://api.xisu.ai` 即可。

### 余额不足时报 429

Anthropic 协议没有「余额不足」这种错误类型，所以余额不足、密钥预算用完时返回 `429 rate_limit_error`，看起来像限流。遇到 429 先去控制台看余额和密钥预算，不要只是等待重试。

### 404 和 503 处理方法相反

- `404 not_found_error`：这个模型不能通过 Anthropic 接口调用，例如把 `ANTHROPIC_MODEL` 设成了 GPT 系列。重试没用，换成 Claude 系列模型。
- `503 overloaded_error`：模型暂时没有可用资源，属于临时情况。Claude Code 会自动重试，一般稍等即可恢复。

### 不支持 WebSearch 等服务端工具

Claude Code 的 WebSearch 工具会报：

```text
API Error: 400 server-side tool 'web_search_20250305' is not enabled on this endpoint …
```

读写文件、运行命令、WebFetch（在你本机抓取网页）都不受影响。想让 Claude Code 别再尝试 WebSearch，可以在 `settings.json` 里禁用它：

```json
{
  "permissions": {
    "deny": ["WebSearch"]
  }
}
```

更多协议细节见「Anthropic 原生端点」，错误码见「错误码与限流」。

## 常见问题

### 运行后很久没有输出怎么办？

多半是密钥错误。密钥无效时 Claude Code 会反复重试，实测 `claude -p` 超过 3 分钟还没结束。数据来自 犀速-AI 官网公布的实测（品牌方自测）。按 `Ctrl+C` 退出，检查密钥是否完整、有没有多余空格，或者加上 `--debug` 运行，日志里能看到 `401 authentication_error`。

### 设置了环境变量但没生效怎么办？

Windows 上写入用户环境变量后要重新打开终端；macOS / Linux 上确认执行过 `source ~/.zshrc`。在 Claude Code 里输入 `/status`，看 Anthropic base URL 是不是 `https://api.xisu.ai`。

### 提示模型有问题怎么办？

如果出现 `There's an issue with the selected model`，依次检查地址是否多了 `/v1`、模型名是否拼错、是否用了非 Claude 模型。在售模型名见「模型广场」。

### `/status` 里出现「Can't auto-update」是什么意思？

这是 Claude Code 自动更新的提示，和犀速-AI 无关。需要更新时重新执行一次安装命令即可。

### 想换回 Claude 官方账号怎么做？

删掉上面设置的几个环境变量，或删掉 `settings.json` 里对应的 `env` 项，然后重新打开终端。

其他客户端怎么配：见「犀速-AI 客户端下载与配置大全」。

## 关于 犀速-AI

犀速-AI 提供 Anthropic 原生接口，可按本文配置让 Claude Code 通过犀速-AI 调用 Claude 模型。

官网：https://xisu.ai/