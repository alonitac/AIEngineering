# Token Economics

Your agent works. You ship it. A month later Finance forwards you the bill: **$4,300**, for a feature used by 60 people.

Nobody wrote a bug. The agent did exactly what you asked - it just did it with a 40,000-token context, on every single turn, for every single user.

Every LLM feature you build is a metered service. This tutorial teaches you the meter.


## What Is a Token

A model never sees text. Before inference, a **tokenizer** splits your text into integers from a fixed vocabulary:

```text
"Kubernetes is unforgiving"  ->  ["Kub", "ernetes", " is", " unfor", "giving"]  ->  [63947, 34186, 374, 96632, 91098]
```

Common words become one token. Rare words, typos, code identifiers and non-English text split into many. Some rules of thumb for English:

| Content | Tokens |
| --- | --- |
| 1 token | ~4 characters, ~0.75 words |
| 1,000 words | ~1,300 tokens |
| This tutorial | ~3,500 tokens |
| A 10 kB web page | ~2,500 tokens |
| A 500 kB research PDF | ~125,000 tokens |

**Tokenizers differ between models.** Anthropic notes that Claude 4.7 and later use a newer tokenizer that produces roughly **30% more tokens for the same text** than earlier models. A cheaper per-token price does not always mean a cheaper request.

> [!NOTE]
> Images and audio are tokens too. A screenshot in a computer-use agent is worth roughly 1,000-1,600 input tokens, charged on every turn it stays in the context.


## How Cost Is Calculated

Providers bill two meters, per **million tokens** (MTok):

- **Input tokens** - everything you send: system prompt, tool definitions, conversation history, retrieved documents, tool results.
- **Output tokens** - everything the model generates, including reasoning tokens you never display.

```text
cost = (input_tokens / 1e6) * input_price  +  (output_tokens / 1e6) * output_price
```

Output costs **5x** input on every Claude model. That ratio drives most optimization decisions.

Every API response reports the meter. Read it:

```json
{
  "usage": {
    "input_tokens": 2145,
    "cache_creation_input_tokens": 0,
    "cache_read_input_tokens": 18320,
    "output_tokens": 412
  }
}
```

### Anthropic Pricing

Reference: **[claude.com/pricing#api](https://claude.com/pricing#api)** and the detailed **[Claude pricing docs](https://platform.claude.com/docs/en/about-claude/pricing)**.

A snapshot of the current models (USD per MTok):

| Model | Input | Cache write (5m) | Cache read | Output |
| --- | --- | --- | --- | --- |
| Opus 5 | $5 | $6.25 | $0.50 | $25 |
| Sonnet 5 | $2 | $2.50 | $0.20 | $10 |
| Haiku 4.5 | $1 | $1.25 | $0.10 | $5 |

Modifiers that stack on top:

- **Batch API** - 50% off input *and* output, for asynchronous jobs that tolerate delay.
- **Prompt caching** - a cache read costs 0.1x the input price; a 5-minute cache write costs 1.25x.


### A Worked Example

A support agent on Sonnet 5. Each conversation: 8 turns, ~5,000 input tokens per turn, ~300 output tokens per turn.

```text
input:   8 * 5,000 =  40,000 tokens  ->  40,000 / 1e6 * $2  = $0.080
output:  8 *   300 =   2,400 tokens  ->   2,400 / 1e6 * $10 = $0.024
                                                      total = $0.104 per conversation
```

Ten cents. Harmless - until 20,000 conversations a day turn it into **$2,080 per day**, or $62,400 a month.


## The Problem the Industry Is Fighting

Naively, cost looks linear: more users, more tokens, more dollars. It isn't. Three forces push it much harder.

### 1. Conversations Are Quadratic

The API is stateless. To give the model memory, you resend the entire conversation on every turn. A chat where each turn adds `t` tokens costs:

```text
turn 1:  t
turn 2:  2t
turn 3:  3t
...
turn n:  n*t

total input = t * n(n+1)/2   ->   O(n²)
```

Doubling the conversation length quadruples the input bill. Users experience one conversation; you pay for a triangle.

### 2. Agents Multiply Everything

A chatbot makes one model call per user message. An agent runs a loop - think, call a tool, observe, think again - and **every** tool result is appended to the context and resent on the next iteration. One `ls` of a large directory, one 200 kB API response, one screenshot, and your context explodes.

A single user request to a coding agent routinely costs 50-500x a single chat message. This is why token economics became urgent in 2025-2026: the industry moved from chat to agents, and per-request costs jumped two orders of magnitude while per-request *revenue* stayed flat.

### 3. Cost Is Non-Deterministic

Traditional infrastructure costs are predictable: a request hits a CPU for 40 ms. An LLM request costs whatever the model decides to generate. The same prompt can take 3 tool calls today and 19 tomorrow. You cannot capacity-plan a system whose unit cost has a long tail.

Put together, the industry problem is this: **AI features have a variable cost of goods sold that engineers control but cannot easily see, while products are sold at a flat monthly price.** Every subscription-priced AI product is a bet that average token spend stays below the subscription fee - and power users happily burn 50x the average. That's why usage limits, credits and "fair use" caps appeared across the entire market.


## Approaches to the Solution

### Model routing (cascading)

Match the model to the task. Classification, extraction and routing run fine on Haiku; only hard reasoning needs Opus. Route cheap first, escalate on low confidence.

```python
model = "claude-haiku-4-5" if is_simple(task) else "claude-opus-4-5"
```

Typical saving: **60-80%**, with the risk of quality regressions - measure before and after.

### Prompt caching

The largest and most stable win for agents. Your system prompt and tool definitions are identical on every turn, yet you pay full price to reprocess them. Marking them cached drops their cost to 0.1x.

Order matters: put static content (system prompt, tools, long documents) **first**, dynamic content last. The cache matches on an exact prefix - one changed character at the top invalidates everything below it.

Recomputing the earlier example with 4,000 of the 5,000 input tokens cached:

```text
uncached: 8 * 1,000 =  8,000 -> $0.016
cached:   8 * 4,000 = 32,000 -> 32,000 / 1e6 * $2 * 0.1 = $0.0064
output:                2,400 -> $0.024
                                          total = $0.046  (56% cheaper)
```

### Context engineering

Stop sending what the model doesn't need.

- **Compaction** - when the conversation crosses a threshold, summarize old turns and drop the originals. Every serious coding agent does this.
- **Truncate tool results** - cap file reads and API responses; return a handle the agent can expand on demand.
- **Subagents** - hand a noisy investigation to a subagent and return only its conclusion. The parent context never sees the 100k tokens of exploration.


