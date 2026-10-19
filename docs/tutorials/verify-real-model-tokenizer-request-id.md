# 2026年10月如何确认调用到的是真模型？分词器、原厂功能与 request_id 核验指南

> 本文整理自 [犀速-AI 官网原文](https://www.xisu.ai/library/verify-model)，原文更新于 2026-10-16。

## 结论先看

- **不要问模型“你是谁”**：系统提示词可以让任何模型自称任何名字，模型自报身份不能作为证据。
- **用分词器指纹和原厂功能核对**：GPT 看分词器的 5 段是否全部与 OpenAI 官方一致，Claude 看第二次请求是否命中提示词缓存。
- **用 request_id、usage 和费用逐笔对账**：响应头里的 `x-request-id`、响应体里的 `usage` 与控制台「调用明细」应当对应同一笔调用，模型、token 和费用可以按公式核对回来。

## 为什么现在值得看

本文整理自犀速-AI 官网 2026-10-16 的更新内容。使用中转服务时，用户可能会关心自己付费购买的 A 模型是否实际变成了更便宜的 B 模型；尤其在晚高峰或遇到近期常见的模型替换、降级疑问时，单靠模型自称或文风判断并不可靠。本文提供可直接运行的检测脚本，以及通过 `request_id`、token 和费用逐笔对账的方法，适合现在就自行核验。

## 模型与选型

## 如何确认调用到的是真模型？

问它是谁不算数。用分词器指纹、原厂功能和 `request_id` 逐笔对账来核对，附可直接运行的检测脚本。

用中转服务时，很多人都会担心：我付的是 A 模型的钱，拿到的会不会是更便宜的 B？

这一篇教你**自己动手验证**，不用相信任何人的保证（包括我们的）。

## 3 步快速核对

赶时间的话，按这三步做就够了：

1. **跑检测脚本（约 5 分钟）。** 复制下文「方法一」里的 `check_model.py` 运行：GPT 看分词器的 5 段是否全部和 OpenAI 官方一致，Claude 看第二次请求是否命中提示词缓存。
2. **抽一笔调用对账。** 记下响应头里的 `x-request-id` 和响应体里的 `usage`，到控制台「调用明细」按请求编号搜出同一笔：模型和各项 token 应完全一致，费用按下文的公式能算回来。做法见下文「用 request_id、token 和费用核对每一笔」。
3. **怀疑被换成小模型时，对照信号表复核。** 见下文「怎么判断发生了模型替换或降级」。

全程都不需要问模型「你是谁」，它的回答不能当证据，原因下面会讲。

## 先说结论

| 方法 | 能看出什么 | 难度 |
|---|---|---|
| 跑下面的检测脚本（分词器 + 原厂功能） | **换了厂商**：比如拿别家模型冒充 Claude 或 GPT | ⭐ 复制粘贴即可 |
| 拿同一组难题和官方并排对比 | **同厂降级**：比如拿小模型冒充大模型 | ⭐⭐ 需要官方账号对照 |
| 用 `request_id` 对账（响应 `usage` ↔ 调用明细 ↔ 单价） | **记账对不上**：记录的模型、token 数或费用和你实际拿到的不一致 | ⭐ 控制台里搜一下 |
| 问它「你是谁」、看文风 | 几乎什么都看不出 | 别用 |

## 为什么「问它是谁」不算数

模型对自己是谁的回答**非常不可靠**：

- 一句系统提示词就能让任何模型自称任何名字；
- 很多模型的训练数据里混着别家模型的对话，**真模型也经常认错自己**；
- 知识截止日期也一样，模型自己报的往往不准。

所以「它说自己是 Claude」既证明不了真，「它说自己是 GPT」也证明不了假。

要看的是**它装不出来的东西**。

## 方法一：5 分钟跑一个检测脚本（推荐）

### 第 1 步：准备

- 装 Python（3.8 或更新）。Windows 去 python.org 下载安装，安装时勾上 `Add Python to PATH`；macOS 和 Linux 一般自带。
- 打开终端（Windows 叫「命令提示符」或 PowerShell），运行：

```bash
pip install tiktoken
```

这是 OpenAI 官方的分词器，脚本拿它当「标准答案」。

准备三样东西，运行时脚本会依次问你。接入地址是这一串：

```text
https://api.xisu.ai/v1
```

- **API Key**：在控制台「API 密钥」里创建；
- **模型名**：去模型广场点一下复制，不要手打。

### 第 2 步：保存脚本

把下面整段复制，保存成一个文件，名字叫 `check_model.py`（比如放在桌面）：

```python
#!/usr/bin/env python3
# 模型真伪自检。只用 Python 自带的库；检测 GPT 时需要先 pip install tiktoken。
import getpass
import json
import time
import urllib.error
import urllib.request
import uuid

SEGMENTS = [
    "鱻 龘 齉 —— 这是一段用来比对分词器的测试文本，量子纠缠。",
    "The quick brown fox jumps over the lazy dog, 1234567890 times.",
    "🙂🚀🧪 emoji 也算进来。🙂🚀🧪",
    "def tokenize(text):\n return [ord(c) for c in text] # 缩进与空格",
    "Pneumonoultramicroscopicsilicovolcanoconiosis, Ünïcödé, ½ ≈ 0.5",
]
SHORT_TEXT = "x"

def ask(prompt):
    value = ""
    while not value:
        value = input(prompt).strip()
    return value

def post(url, headers, body):
    try:
        req = urllib.request.Request(
            url,
            data=json.dumps(body).encode("utf-8"),
            headers={"content-type": "application/json", **headers},
            method="POST",
        )
        with urllib.request.urlopen(req, timeout=120) as resp:
            return resp.status, json.loads(resp.read().decode("utf-8"))
    except urllib.error.HTTPError as e:
        raw = e.read().decode("utf-8", "replace")
        try:
            return e.code, json.loads(raw)
        except ValueError:
            return e.code, {"raw": raw[:500]}
    except (urllib.error.URLError, TimeoutError, ValueError) as e:
        raise RuntimeError(f"连不上 {url}：{e}（检查接入地址是否抄对）")

def error_text(data):
    err = data.get("error")
    if isinstance(err, dict):
        return str(err.get("message", err))
    return str(err or data)[:300]

def load_tiktoken():
    try:
        import tiktoken
    except ImportError:
        print("⚠ 没装 tiktoken，无法和 OpenAI 分词器对照。先运行：pip install tiktoken")
        return None
    return tiktoken.get_encoding("o200k_base")

class Client:
    def __init__(self, base, key, model):
        self.base = base.rstrip("/")
        self.root = self.base[:-3] if self.base.endswith("/v1") else self.base
        self.key = key
        self.model = model

    def openai_chat(self, content, **extra):
        body = {"model": self.model, "messages": [{"role": "user", "content": content}]}
        body.update(extra)
        body.setdefault("max_tokens", 16)
        status, data = post(
            self.base + "/chat/completions",
            {"authorization": "Bearer " + self.key},
            body,
        )
        if status == 400 and "max_completion_tokens" in error_text(data):
            body["max_completion_tokens"] = body.pop("max_tokens")
            status, data = post(
                self.base + "/chat/completions",
                {"authorization": "Bearer " + self.key},
                body,
            )
        return status, data

    def anthropic_messages(self, content, system=None):
        body = {
            "model": self.model,
            "max_tokens": 16,
            "messages": [{"role": "user", "content": content}],
        }
        if system is not None:
            body["system"] = system
        return post(
            self.root + "/v1/messages",
            {"x-api-key": self.key, "anthropic-version": "2023-06-01"},
            body,
        )

    def input_tokens(self, vendor, content):
        if vendor == "claude":
            status, data = self.anthropic_messages(content)
            field = ("usage", "input_tokens")
        else:
            status, data = self.openai_chat(content)
            field = ("usage", "prompt_tokens")
        if status != 200:
            raise RuntimeError(f"请求失败（HTTP {status}）：{error_text(data)}")
        value = (data.get(field[0]) or {}).get(field[1])
        if not isinstance(value, int):
            raise RuntimeError(f"返回里没有 {field[0]}.{field[1]}，无法比对")
        if vendor == "claude":
            usage = data.get("usage") or {}
            value += (usage.get("cache_creation_input_tokens") or 0)
            value += (usage.get("cache_read_input_tokens") or 0)
        return value

def test_tokenizer(client, vendor, results):
    print("\n【分词器指纹】")
    base = client.input_tokens(vendor, SHORT_TEXT)
    diffs = [client.input_tokens(vendor, text) - base for text in SEGMENTS]
    print(f"这家逐段的 token 数：{diffs}")
    if min(diffs) <= 0:
        results.append(("分词器", "可疑", f"有一段文本的 token 数竟不比单个字母多（{diffs}），返回的计数不可信"))
        return
    enc = load_tiktoken()
    expected = None
    if enc is not None:
        expected = [len(enc.encode(text)) - len(enc.encode(SHORT_TEXT)) for text in SEGMENTS]
    print(f"OpenAI 分词器逐段算出：{expected}")
    if vendor == "gpt":
        if expected is None:
            results.append(("分词器", "跳过", "没装 tiktoken"))
        elif diffs == expected:
            results.append(("分词器", "通过", f"{len(SEGMENTS)} 段全部与 OpenAI 官方分词器一致"))
        else:
            results.append(("分词器", "可疑", f"OpenAI 官方应为 {expected}，这里是 {diffs}"))
        return
    if expected is not None and diffs == expected:
        results.append(("分词器", "可疑", "号称不是 GPT，逐段分词结果却与 OpenAI 完全一致"))
    elif expected is not None:
        differ = sum(1 for a, b in zip(diffs, expected) if a != b)
        results.append(("分词器", "通过", f"{len(SEGMENTS)} 段里有 {differ} 段与 OpenAI 不同，符合预期"))

def test_logprobs(client, results):
    print("\n【逐 token 概率 logprobs】")
    status, data = client.openai_chat("用一句话介绍一下大熊猫。", logprobs=True, max_tokens=30)
    items = None
    if status == 200:
        items = (((data.get("choices") or [{}])[0].get("logprobs")) or {}).get("content")
    if not items:
        why = error_text(data) if status != 200 else "响应里没有 logprobs"
        print(f"未拿到：{why}")
        results.append(("logprobs", "参考", "没拿到。推理型 GPT 官方本身就不支持；普通对话型 GPT 拿不到则可疑"))
        return
    enc = load_tiktoken()
    if enc is None:
        results.append(("logprobs", "跳过", "没装 tiktoken"))
        return
    bad = 0
    for item in items:
        raw = bytes(item["bytes"]) if item.get("bytes") else item["token"].encode("utf-8")
        try:
            enc.encode_single_token(raw)
        except KeyError:
            bad += 1
    print(f"共 {len(items)} 个 token，不在 OpenAI 词表里的：{bad} 个")
    if bad == 0:
        results.append(("logprobs", "通过", f"{len(items)} 个 token 全部在 OpenAI 词表里"))
    else:
        results.append(("logprobs", "可疑", f"有 {bad} 个 token 不在 OpenAI 词表里"))

def test_prompt_cache(client, results):
    print("\n【提示词缓存】")
    if input("这一项会发两次各一万多 token 的请求（第二次大部分按缓存价计），费用约等于一次长对话。继续？[Y/n] ").strip().lower() == "n":
        results.append(("提示词缓存", "跳过", "已手动跳过"))
        return
    lines = [f"Reference line {i:04d}: the quick brown fox jumps over the lazy dog." for i in range(500)]
    system = [{
        "type": "text",
        "text": f"[{uuid.uuid4()}]\n" + "\n".join(lines),
        "cache_control": {"type": "ephemeral"},
    }]
    rounds = []
    for n in (1, 2):
        start = time.time()
        status, data = client.anthropic_messages("只回复 OK。", system=system)
        elapsed = time.time() - start
        if status != 200:
            print(f"第 {n} 次请求失败（HTTP {status}）：{error_text(data)}")
            results.append(("提示词缓存", "跳过", "请求失败"))
            return
        usage = data.get("usage") or {}
        write = usage.get("cache_creation_input_tokens") or 0
        read = usage.get("cache_read_input_tokens") or 0
        print(f"第 {n} 次：写入缓存 {write}，读取缓存 {read}，耗时 {elapsed:.1f} 秒")
        rounds.append((write, read))
        time.sleep(1)
    if rounds[1][1] > 0:
        results.append(("提示词缓存", "通过", f"第二次命中缓存 {rounds[1][1]} token"))
    else:
        results.append(("提示词缓存", "可疑", "两次都没命中缓存，原厂 Claude 应当命中"))

def main():
    print("模型真伪自检\n")
    base = ask("接入地址（文档里那一串，以 /v1 结尾）：")
    key = getpass.getpass("API Key（输入时不显示，粘贴后回车）：").strip()
    model = ask("要检测的模型名（从模型广场复制）：")
    name = model.lower()
    if "claude" in name:
        vendor = "claude"
    elif "gpt" in name or name.startswith(("o1", "o3", "o4")):
        vendor = "gpt"
    else:
        vendor = "other"
    label = {"claude": "Claude", "gpt": "GPT", "other": "其它厂商"}[vendor]
    print(f"\n按「{label}」的方式检测。")
    client = Client(base, key, model)
    results = []
    try:
        test_tokenizer(client, vendor, results)
        if vendor == "gpt":
            test_logprobs(client, results)
        if vendor == "claude":
            test_prompt_cache(client, results)
    except RuntimeError as e:
        print(f"\n✖ {e}")
        return
    print("\n================ 结果 ================")
    marks = {"通过": "✅", "可疑": "❌", "参考": "ℹ️", "跳过": "⏭"}
    for item, verdict, note in results:
        print(f"{marks[verdict]} {item}：{verdict} —— {note}")
    if any(v == "可疑" for _, v, _ in results):
        print("\n有可疑项。建议换几个时间段重测；结果稳定的话，可以把这段输出发给服务商要解释。")
    else:
        print("\n没有发现可疑项。")

if __name__ == "__main__":
    main()
```

### 第 3 步：运行

在终端里进入文件所在的目录，运行：

```bash
python check_model.py
```

（macOS / Linux 上如果提示找不到 `python`，换成 `python3`。）

按提示粘贴接入地址、Key、模型名。Key 输入时屏幕上**不会显示**，粘贴后直接回车即可。

整个过程只发几次很短的请求；检测 Claude 时多一项会花掉大约一次长对话的费用，运行前会先问你。

### 第 4 步：看结果

最后会打印一张结果表，每一项是 ✅ 通过、❌ 可疑、ℹ️ 仅供参考或 ⏭ 跳过：

| 检测项 | 测的是哪类模型 | ✅ 的意思 | ❌ 的意思 |
|---|---|---|---|
| 分词器 | 所有模型 | GPT：token 数和 OpenAI 官方分词器**分毫不差**；其他模型：和 OpenAI 不一样（本来就该不一样） | GPT 对不上官方；或者号称不是 GPT，却和 OpenAI 分得一模一样 |
| `logprobs` | GPT | 模型吐出的每个 token 都在 OpenAI 的词表里 | 有 token 不在 OpenAI 词表里，说明背后不是 OpenAI 的模型 |
| 提示词缓存 | Claude | 第二次请求命中了缓存 | 两次都没命中，原厂 Claude 不会这样 |

出现 ❌ 先别急着下结论。隔一段时间再跑两三次；结果稳定的话，把整段输出发给服务商，要一个解释。

## 原理：为什么这几项装不出来

不想看原理可以跳过这一节。

### 分词器是模型的「指纹」

模型不是按字读文本的，而是先把文本切成一个个 token，再按 token 计费。

每家用的切法（分词器）都不一样：同一段中英混排加 emoji 的文本，OpenAI 切出来是一个数，Claude 是另一个数，Gemini、DeepSeek、通义千问又各不相同。即便是同一家，换一代模型也可能换分词器。

脚本的做法是：准备 5 段专挑难处的文本（生僻字、emoji、代码缩进、超长单词……），每段单独发一次，再和只发一个字母 `x` 的请求相减。消息格式本身带来的固定开销每次都一样，相减就抵掉了，剩下的就是那段文本的 token 数。

之所以**逐段**比、不比总数：别家分词器在某一段、甚至几段加起来的总数上和 OpenAI 碰巧相等并不稀奇，5 段全都相等就不是巧合了。

**GPT**：OpenAI 的分词器是公开的（就是刚才装的 `tiktoken`），GPT-4o 及之后的模型用的是 `o200k_base`，所以每一段都必须**完全相等**。

如果 OpenAI 将来给新模型换了分词器，这一项会对不上——那时以 `tiktoken` 的官方说明为准。

**Claude**：Anthropic 没有公开离线分词器，所以脚本只能确认它「不是 OpenAI 的切法」。如果你有 Anthropic 官方的 API Key，脚本会顺便调用官方的计数接口（`count_tokens`，免费），得到**精确的标准答案**再比。

⚠️ 拿来当裁判的必须是**官方**接口——任何中转服务（包括我们）自己提供的计数接口都不能用来验证它自己。

这一项有一个前提：返回的 token 数要是**模型实际报出的**。如果某个服务商的 token 数是自己估算出来的，这一项测到的就是它的估算器，而不是模型——但那本身也说明它的计费不是按真实用量来的，同样值得追问。

### `logprobs`：每个 token 都要在词表里

OpenAI 的普通对话型模型可以返回「每个输出 token 的概率」（`logprobs`）。

真的 GPT 吐出的**每一个 token**，都必然是 OpenAI 词表里的一个词条；换成别家模型，切出来的 token 对不上 OpenAI 的词表，很难伪造得滴水不漏。

推理型的 GPT 模型官方本身就不支持 `logprobs`，拿不到是正常的，脚本会标成「仅供参考」。

### 提示词缓存：Claude 的原厂功能

Claude 支持「提示词缓存」：同一段很长的前缀连续发两次，第二次会直接从缓存读取，返回里的 `cache_read_input_tokens` 大于 0。脚本每次都在前缀开头放一个随机串，保证第一次一定是新写入，第二次命中才算数。

走 OpenAI 兼容接口（`/v1/chat/completions`）调 Claude 时，这类 Claude 专有的能力不会生效——那是两种协议之间转换造成的，不代表模型被换了。

脚本测 Claude 时自动走 Anthropic 原生接口 `/v1/messages`，详见「Anthropic 原生端点」那一篇。

## 用 request_id、token 和费用核对每一笔

检测脚本回答的是「是不是这个模型」，这一节回答「这一笔记的账对不对」。要对上的有三样：响应里的 `usage`、控制台「调用明细」里的同一笔、按单价算出来的费用。

> **数据来自 犀速-AI 官网公布的实测（品牌方自测）。**

### 第 1 步：从响应里拿到 request_id 和 usage

每个对话响应都带响应头 `x-request-id`，它就是这一笔调用的编号；响应体里的 `usage` 是这一笔的 token 分项。先把密钥放进环境变量 `XISU_API_KEY`（PowerShell：`$env:XISU_API_KEY="你的密钥"`；macOS / Linux：`export XISU_API_KEY="你的密钥"`），再运行下面的代码。

OpenAI 兼容接口，用官方 SDK（`pip install openai`）：

```python
import os
from openai import OpenAI

client = OpenAI(base_url="https://api.xisu.ai/v1", api_key=os.environ["XISU_API_KEY"])
raw = client.chat.completions.with_raw_response.create(
    model="gpt-6-luna",
    messages=[{"role": "user", "content": "用一句话介绍大熊猫。"}],
    max_tokens=50,
)
print("x-request-id:", raw.headers.get("x-request-id"))
print(raw.parse().usage)
```

Anthropic 原生接口 `/v1/messages`，只用 Python 自带的库：

```python
import json, os, urllib.request

req = urllib.request.Request(
    "https://api.xisu.ai/v1/messages",
    data=json.dumps({
        "model": "claude-haiku-5-5",
        "max_tokens": 50,
        "messages": [{"role": "user", "content": "用一句话介绍大熊猫。"}],
    }).encode(),
    headers={
        "x-api-key": os.environ["XISU_API_KEY"],
        "anthropic-version": "2023-06-01",
        "content-type": "application/json",
    },
)
with urllib.request.urlopen(req) as resp:
    print("x-request-id:", resp.headers.get("x-request-id"))
    print(json.dumps(json.load(resp)["usage"], indent=2, ensure_ascii=False))
```

用 `curl` 的话加 `-i`，响应头会和响应体一起打印出来，找 `x-request-id` 那一行。注意响应体里的 `id` 字段是另一个东西，对账请用响应头里的编号。

`usage` 里的字段和「调用明细」的 Token 分项这样对应：

| 调用明细 | OpenAI 兼容接口的 usage | Anthropic 原生接口的 usage |
|---|---|---|
| 输入 | `prompt_tokens` | `input_tokens + cache_read_input_tokens` |
| 其中缓存命中 | `prompt_tokens_details.cached_tokens` | `cache_read_input_tokens` |
| 缓存写入 | `prompt_tokens_details.cache_write_tokens`（有则看） | `cache_creation_input_tokens` |
| 输出 | `completion_tokens` | `output_tokens` |
| 其中推理 | `completion_tokens_details.reasoning_tokens` | — |

两条「其中」已经包含在上一行里，不要重复相加；缓存写入不在「输入」里，单独一行。Anthropic 协议的 `input_tokens` 只算没命中缓存的那部分，所以对「输入」时要把 `cache_read_input_tokens` 加回去。

## 常见问题

### 问模型“你是谁”能确认真模型吗？

不能。系统提示词可以让任何模型自称任何名字，很多模型也会认错自己。模型自报身份不能作为证据。

### 怎么判断中转服务有没有把 GPT 换成别的模型？

运行检测脚本，检查分词器和 `logprobs`。GPT 的 token 数应当与 OpenAI 官方分词器分毫不差；模型吐出的每个 token 都应在 OpenAI 的词表里。

### Claude 没有命中提示词缓存，是不是一定被换模型了？

不要马上下结论。隔一段时间再跑两三次；结果稳定的话，把整段输出发给服务商，要一个解释。走 OpenAI 兼容接口调 Claude 时，Claude 专有的提示词缓存能力不会生效，这不代表模型被换了。

### `request_id` 和响应体里的 `id` 是同一个东西吗？

不是。对账请使用响应头里的 `x-request-id`，响应体里的 `id` 字段是另一个东西。

### 怎么核对一笔调用的 token 和费用？

把响应头里的 `x-request-id`、响应体里的 `usage`，与控制台「调用明细」里的同一笔调用对应起来，再按单价计算费用。模型、各项 token 和费用应当一致。

## 关于 犀速-AI

犀速-AI 官网提供模型调用与相关文档，本文内容整理自其模型核验说明。

官网：https://xisu.ai/
