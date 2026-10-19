# 2026年10月 API 中转账单与用量核对教程：固定请求、逐笔对账

> 本文整理自 [犀速-AI 官网原文](https://www.xisu.ai/resources/articles/relay-billing-check)，原文更新于 2026-10-16。

## 结论先看

- **单价不能只看输入价**：输入、输出、缓存读取、缓存写入、计价单位、充值换算和单价以外的费用，都要放在一起核对。
- **最稳妥的用量核对方式**：固定请求，三组各跑 3 次，记录每一笔的 `request_id`、`usage` 和应扣费用，再与调用明细逐笔比对。
- **犀速-AI 官网实测**：9 笔按公式算出的费用，向下取整后都和调用明细一致；余额差为 11,964 积分，按公式精确计算的费用为 11,963.62，差异来自取整和同一密钥的其他调用。

## 为什么现在值得看

本文更新于 2026-10-16，内容覆盖近期核对低价 API 中转时容易遇到的缓存计费、流式响应 `usage`、GPT 系列 `max_tokens` 参数未生效，以及调用明细保留时间等问题。现在按本文方法测试，可以把单价、接口返回、账单明细和余额变化放在同一套证据链里核对，而不是凭感觉判断。

实用文章

低价不一定不可靠。价格差异可能来自官方折扣、缓存、批量采购或供应商策略。本文不评价任何具体服务商，只给可以复现的核对方法；判断以接口响应、账单记录、隐私政策和稳定性数据为准，包括对我们自己。

账单对不对，凭感觉判断不了，要逐笔对账。本文分两步：先把单价看全，再用一组固定请求核对用量和扣费。文中的脚本都可以直接运行，最后附上我们在犀速-AI 上跑出来的实际结果。五类常见风险的总览见「低价 API 中转怎么辨别」。

### 测试之前

只用虚构内容，不要把真实客户资料、API Key 或源代码发给任何中转做测试，包括我们（我们怎么处理请求内容，见「隐私政策」）。最好单独建一把测试密钥并设好预算（见「数据安全与密钥安全说明」），这样账单里可以按密钥筛出只属于这次测试的记录。

## 倍率虚标：「单价看着低，账单比官方还狠」

这类落差多半来自只看了一个数。核对价格时，把下面几项都找出来：

| 看什么 | 为什么 |
|---|---|
| 输入、输出单价 | 输出通常比输入贵好几倍，只看输入价会低估账单 |
| 缓存读取、缓存写入单价 | 长上下文、多轮对话里缓存命中便宜很多；缓存写入可能比普通输入还贵 |
| 计价单位 | 统一换算成「每 100 万 token 多少钱」。「X 折」「倍率」「分组系数」都要换算回来再比 |
| 充值换算 | 充进去的是额度、点数还是美元，1 元到底能买多少 |
| 单价以外的费用 | 按次收费、最低消费、充值手续费；图片模型一般按张计价，要单独看各档价格 |

举个假设的例子：某模型输入价 1 元/百万 token，输出价 5 元/百万 token，一次调用输入 2,000、输出 1,000 token。只按输入价估，3,000 token 是 0.003 元；分开算是 2,000×1 + 1,000×5 = 7,000，也就是 0.007 元，是前者的 2 倍多。写作、代码生成、推理模型这类输出多的任务，差得更多。

价格看全之后，最终要落到一件事：**用平台公开的单价，能不能算出你实际被扣的钱。** 这一步放在下面的对账里一起做。

## 用量虚标：「调一千扣两三千」

先排除一个常见误会：token 数不能按字数换算。各家分词器不同，同一段中文在不同模型上的 token 数可以差很多，所以「我只发了 1000 字，凭什么算 1500 token」不能当证据。能当证据的是两件事：同一个请求重复发送时数字是否稳定，以及账单是否和接口返回的 `usage` 一致。

## 方法：固定请求，三组各跑 3 次

| 组 | 请求 | 主要查什么 |
|---|---|---|
| 短进短出 | 一句话翻译，`max_tokens` 100 | 小请求有没有被多算 |
| 长进短出 | 400 行虚构订单，只问条数 | 输入计数；缓存命中是否按缓存价算 |
| 短进长出 | 写 1000 字的虚构小镇介绍，`max_tokens` 600 | 输出计数；输出上限 |

### 步骤

1. 记下余额和页面上的数据时间；
2. 运行下面的脚本。它关掉了 SDK 的自动重试（`max_retries=0`），否则一次失败的调用可能被悄悄重发、扣两次，而你只看到一个结果；
3. 脚本会打印每一笔的 `request_id`（响应头 `x-request-id`）、`usage` 和按单价算出的应扣费用；
4. 等一两分钟，再记一次余额。

下面的脚本用于固定请求对照：三组 × 每组 3 次，打印每一笔的 `request_id` 和 token 用量。填入页面公布的单价后，会算出每一笔的「应扣费用」，拿去和账单逐笔比。

环境变量：

- `API_KEY`：你的密钥（必填）
- `BASE_URL`：接口地址，例如 `https://api.xisu.ai/v1`（必填）
- `MODEL`：模型名，例如 `claude-haiku-5-5`（必填）

单价（可选，每 100 万 token；单位用平台账单的单位，元、美元、积分都行）：

- `PRICE_INPUT`
- `PRICE_OUTPUT`
- `PRICE_CACHE_READ`

```python
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["API_KEY"],
    base_url=os.environ["BASE_URL"],
    max_retries=0,  # 关掉自动重试，否则一次调用可能被悄悄发两遍、扣两次
    timeout=120,
)

MODEL = os.environ["MODEL"]

def price(name):
    v = os.environ.get(name)
    return float(v) if v else None

P_IN, P_OUT, P_CACHE = price("PRICE_INPUT"), price("PRICE_OUTPUT"), price("PRICE_CACHE_READ")

# 测试内容全部是虚构的。不要换成真实客户资料、密钥或源代码。
LONG_DOC = "\n".join(
    f"虚构订单 {i:04d}：城市 {'甲乙丙丁戊'[i % 5]}市，金额 {i * 37 % 500} 元，状态 {'已发货' if i % 3 else '待发货'}。"
    for i in range(1, 401)
)

GROUPS = [  # (组名, 提示词, max_tokens)
    ("短进短出", "把这句话翻译成英文：青柠咖啡馆每周一休息。", 100),
    ("长进短出", LONG_DOC + "\n\n上面一共有多少条订单？只回答数字。", 100),
    ("短进长出", "写一篇 1000 字左右的虚构小镇介绍，小镇叫青石镇。", 600),
]

rows = []
for name, prompt, max_tokens in GROUPS:
    for n in range(1, 4):
        raw = client.chat.completions.with_raw_response.create(
            model=MODEL,
            messages=[{"role": "user", "content": prompt}],
            temperature=0,
            max_tokens=max_tokens,
        )
        resp = raw.parse()
        u = resp.usage
        details = u.prompt_tokens_details
        rows.append({
            "group": name,
            "n": n,
            "max": max_tokens,
            "rid": raw.headers.get("x-request-id", "(没有返回)"),
            "prompt": u.prompt_tokens,
            "cached": (details.cached_tokens or 0) if details else 0,
            "completion": u.completion_tokens,
            "finish": resp.choices[0].finish_reason,
        })

def cost(r):
    if P_IN is None or P_OUT is None:
        return None
    cache_price = P_CACHE if P_CACHE is not None else P_IN
    return ((r["prompt"] - r["cached"]) * P_IN
            + r["cached"] * cache_price
            + r["completion"] * P_OUT) / 1_000_000

print(f"模型 {MODEL}")
print(f"{'组':<6}{'次':>2} {'request_id':<38}{'prompt':>8}{'cached':>8}{'compl.':>8} {'finish':<8}{'应扣':>10}")
for r in rows:
    c = cost(r)
    print(f"{r['group']:<6}{r['n']:>2} {r['rid']:<38}{r['prompt']:>8}{r['cached']:>8}{r['completion']:>8}"
          f" {r['finish'] or '':<8}{'' if c is None else f'{c:.2f}':>10}")

print()
for name, _, max_tokens in GROUPS:
    g = [r for r in rows if r["group"] == name]
    same = len({r["prompt"] for r in g}) == 1
    over = [r["rid"] for r in g if r["completion"] > max_tokens]
    print(f"{name}：prompt_tokens 三次{'一致' if same else '不一致，需要排查'}；"
          f"completion 超过 max_tokens({max_tokens}) 的：{over or '无'}")

costs = [cost(r) for r in rows]
if None not in costs:
    print(f"\n9 笔应扣合计：{sum(costs):.2f}（单位与你填的单价相同）")

print("下一步：拿上面的 request_id 去账单 / 调用明细里逐笔核对 token 和费用。")
```

### 运行方式

先安装依赖：

```bash
pip install openai
```

PowerShell：

```powershell
$env:API_KEY = "sk-你的密钥"
$env:BASE_URL = "https://api.xisu.ai/v1"
$env:MODEL = "claude-haiku-5-5"
# 单价可选：填平台页面公布的「每 100 万 token」价格，单位用账单的单位
$env:PRICE_INPUT = "108000"; $env:PRICE_OUTPUT = "540000"; $env:PRICE_CACHE_READ = "10800"
python check_usage.py
```

macOS / Linux 把 `$env:X = "…"` 换成 `export X="…"`。上面的单价是 `claude-haiku-5-5` 在 2026-10-16 的积分价，测别的模型或别的平台时换成对应页面上的数字。

如果你的业务用流式输出，也要单独验一次。流式响应默认不带 `usage`，要加 `stream_options={"include_usage": True}`，用量在最后一块里：

```python
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["API_KEY"],
    base_url=os.environ["BASE_URL"],
    max_retries=0,
)

stream = client.chat.completions.create(
    model=os.environ["MODEL"],
    messages=[{"role": "user", "content": "用三句话介绍一座虚构的海边小城。"}],
    max_tokens=200,
    stream=True,
    stream_options={"include_usage": True},  # 不加这一项，很多平台的流式响应里没有 usage
)

usage = None
for chunk in stream:
    if chunk.choices and chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="", flush=True)
    if chunk.usage:  # 最后一块：choices 为空，只带 usage
        usage = chunk.usage

print("\n\nusage:", usage.prompt_tokens, usage.completion_tokens)
```

## 怎么判断

### `prompt_tokens` 三次完全一致

同一个请求，输入 token 数不应该变。第 2、3 次可能命中缓存，命中的部分包含在 `prompt_tokens` 里，总数仍然一样。

### `completion_tokens` 不超过 `max_tokens`

`temperature=0` 也不保证每次输出相同，所以输出长短不一是正常的，但不应超过你设的上限；被截断时 `finish_reason` 是 `length`。推理模型的推理 token 也计入 `completion_tokens`。超过上限时，要么参数没有生效，要么计数有问题，两种情况你都在为超出的部分付费，应当要求服务商解释。

### 公式能复现账单

每一笔应扣：

```text
[(prompt − 缓存命中) × 输入价 + 缓存命中 × 缓存读取价 + completion × 输出价] ÷ 1,000,000
```

平台单列缓存写入时，再加上缓存写入 × 缓存写入价。和账单逐笔比，只允许显示取整带来的差。

### 条数对得上

**9 个 `request_id` 在账单里各对应一条，不多不少。**

### 余额差约等于费用合计

前后两次余额相减，应当接近这几笔费用之和。

### 对不上时，先排除这些情况

在下「虚标」的结论前，先排除下面几种情况：

- **重试**：多出来的记录，先看是不是 SDK 或你自己的代码重发了；
- **缓存**：缓存命中按缓存价算，缓存写入可能单独计价，不能都按输入价算；
- **快照时间**：余额页面一般标有数据时间，调用结束后等一会儿再取；
- **同账户的其他调用**：别的程序或同事用着同一账户，会混进余额差，所以要按密钥筛选；
- **取整**：页面上的数字可能是取整后的。

排除之后仍然对不上，带上 `request_id` 和你的计算过程找服务商核实。

## 在犀速-AI 上实测

> 数据来自 犀速-AI 官网公布的实测（品牌方自测）。

我们的价格同样低于官方，所以欢迎你用本文的方法来测我们。下面是 2026-10-16 用上面的脚本跑出的结果。

先说几个对账时用得到的规则：犀速-AI 的账单单位是积分，¥1 = 100,000 积分，页面上的积分显示为整数；模型广场公开每个模型的输入、输出、缓存读取、缓存写入四项单价；调用结束才结算，不预扣，失败的请求不计费。调用明细可以按时间范围、密钥、状态和请求编号筛选，请求编号就是响应头里的 `x-request-id`；字段显示「—」表示没有取到数据，「0」是真的零。详细说明见「用量与账单怎么核对」。

### `claude-haiku-5-5`

OpenAI 兼容接口；单价为输入 108,000、输出 540,000、缓存读取 10,800 积分/百万 token：

| 组 | `prompt_tokens` | 缓存命中 | `completion_tokens` | `finish_reason` | 明细费用（积分） |
|---|---:|---:|---:|---|---:|
| 短进短出 | 159 / 159 / 159 | 0 / 0 / 0 | 65 / 83 / 51 | stop | 52 / 61 / 44 |
| 长进短出 | 12,550 / 12,550 / 12,550 | 0 / 12,546 / 12,546 | 49 / 49 / 46 | stop | 1,381 / 162 / 160 |
| 短进长出 | 165 / 165 / 165 | 0 / 0 / 0 | 600 / 600 / 600 | length | 341 / 341 / 341 |

每组 `prompt_tokens` 三次一致；短进长出三次都正好停在 600，没有超过上限。

9 笔按公式算出的费用，向下取整后都和调用明细一致。例如长进短出第 2 笔：

```text
(4×108,000 + 12,546×10,800 + 49×540,000) ÷ 1,000,000 = 162.39
```

明细显示 162。

同样的长输入，命中缓存后从 1,381 积分降到 162 积分。

有一处要单独说明：长进短出第 1 笔，接口响应报告了 12,546 个缓存写入 token，调用明细里没有单列缓存写入，这一笔按普通输入价计费（1,381 积分，低于按缓存写入价算出的结果）。这个现象我们只观察到这一次，还不能确定是否总是如此，核对时以调用明细里的分项为准。

### `gpt-6-luna`

`gpt-6-luna` 同样跑了三组：每组 `prompt_tokens` 三次一致（323、9,167、323），9 笔费用都能用公式复现。缓存命中同样按缓存读取价计费：9,167 token 的长输入，第一次 66 积分，命中 8,960 token 缓存后是 8 积分。

但短进长出那组的输出是 1,225～1,395 token，超过了设定的 600 上限，并按实际输出计费：GPT 系列目前 `max_tokens` 这类输出上限参数没有生效，这是已知问题，修复前请不要靠它控制 GPT 模型的费用（见「低价 API 中转怎么辨别」第 7 节）。

### Anthropic 原生接口

Anthropic 原生接口的 `usage` 口径不一样：`input_tokens` 不含缓存部分，缓存读取和缓存写入分别在 `cache_read_input_tokens` 和 `cache_creation_input_tokens` 里。实测调用明细的「输入」等于 `input_tokens` 加 `cache_read_input_tokens`，`cache_creation_input_tokens` 显示为「缓存写入」，按缓存写入价单独计费。

下图是按请求编号搜出的一笔：响应里 `input_tokens` 为 2、`cache_read_input_tokens` 为 17,197，明细显示输入 17,199、其中缓存命中 17,197，费用 188 积分。

### 余额核对

测试前余额 1,356,425 积分，测试后 1,344,461 积分，差 11,964。这段时间里这把密钥共有 48 笔调用，明细显示的费用加起来是 11,939，按公式精确计算是 11,963.62，和余额差对得上。两者差的 25 积分来自取整：明细每笔显示向下取整的整数，余额按精确值扣。

另外，这 48 笔里有 352 积分是另一个测试程序同时段用同一把密钥产生的，这也是建议你单独建测试密钥的原因。

还有两点：调用明细保留 40 天，之后只留按天汇总，逐笔对账要在 40 天内做；发现重复扣费或计费错误，带上 `request_id` 开工单，核实后退还多扣的部分，见「充值与退款」。最低充值 10 元，余额不过期，可以先充 10 元把本文的测试跑一遍。

账单对得上，只能说明按实际用量计费。调用到的是不是你付钱的那个模型，要用另一套方法测，见「如何确认调用到的是真模型？」。

## 常见问题

### API 中转的账单怎么核对？

先把输入、输出、缓存读取、缓存写入等单价和计价单位看全，再用固定请求测试，记录每笔的 `request_id`、接口返回的 `usage` 和按单价算出的应扣费用，最后与调用明细逐笔比对。

### token 用量虚标怎么判断？

不要用字数换算 token 作为证据。应检查同一个请求重复发送时 `prompt_tokens` 是否稳定，以及账单是否和接口返回的 `usage` 一致；同一个请求的 `prompt_tokens` 三次完全一致，是重要的核对结果。

### 为什么账单费用和公式算出来的不完全一样？

先排查重试、缓存读取或写入、余额页面的快照时间、同一账户的其他调用，以及页面取整。排除这些情况后，和账单逐笔比对时只允许出现显示取整带来的差。

### 流式输出没有 usage 怎么办？

流式响应默认不带 `usage`，要加 `stream_options={"include_usage": True}`；用量在最后一块里，最后一块的 `choices` 为空，只带 `usage`。

### 为什么 GPT 模型输出超过 max_tokens？

实测中，GPT 系列目前 `max_tokens` 这类输出上限参数没有生效，短进长出那组输出为 1,225～1,395 token，超过了设定的 600 上限，并按实际输出计费。修复前不要靠它控制 GPT 模型的费用。

## 关于 犀速-AI

犀速-AI 提供 API 中转服务，并公开模型价格、调用明细与用量核对相关说明。

官网：https://xisu.ai/