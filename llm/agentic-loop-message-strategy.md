# Don't Rush to Clear History — Understanding KV Cache Will Change How You Think About LLM Conversation Strategy


> Many people have an intuition when using LLMs: longer conversations mean more expensive tokens, so you should summarize and compress history early. When building Agent Loops, some merge multi-turn conversations into a single "stateless message" to save tokens. Both approaches seem clever but are actually anti-optimizations. This article explains from KV Cache principles why keeping the original history intact is the optimal strategy.

## The Most Common Misconception: Proactively Summarizing to Compress History

### Scenario

You've chatted with an LLM for 20 turns, using 8K out of 128K in the context window. You start worrying: "Such a long history, sending it with every request — isn't that wasteful?"

So you make an "optimization": have the LLM summarize the previous conversation into a digest, then start a new conversation with that digest.

```
Original conversation (20 turns, 8000 tokens):
  [system] [user_1] [asst_1] [user_2] [asst_2] ... [user_20] [asst_20]

"Optimized" (summary, 500 tokens):
  [system] [user: Here's a summary of the previous conversation: ...500 words...]  [user_21]
```

It looks like input dropped from 8000 tokens to 600, saving 93%?

### Why This Is an Anti-Optimization

**1. You Destroyed the KV Cache**

In the original conversation, the KV for the first 19 turns was already computed and cached in GPU memory during the last request. When the 21st turn arrives:

```
Original approach:
  [system][user_1][asst_1]...[user_20][asst_20] ← all cache hits (0 computation)
  [user_21]                                      ← only compute this one (tens of tokens)
  
Summary approach:
  [system][summary...500 tokens][user_21]        ← entirely new content, full recomputation (550 tokens)
```

The original approach only needs to compute tens of tokens (the new message), while the summary approach computes 550 tokens. **You created ten times the computational overhead to "save tokens."**

**2. The Summary Itself Is Extra Overhead**

When creating the summary, although the previous 8000 tokens are covered by cache (low compute cost), you still need the LLM to generate 500 tokens of summary output. More critically, these 500 summary tokens will be fully computed as new input in the new conversation (with zero cache). You essentially spent 500 tokens generating the summary, then another 500 tokens recomputing it — a net increase in overhead.

**3. Irreversible Information Loss**

When summarizing, you can't predict which details future conversation turns will need. The LLM might need a specific parameter from turn 3 at turn 30, but it was already lost during summarization.

### The Correct Mental Model

```
Existing history = free (covered by KV Cache, 0 computation)
Only the new tail content = actual computational cost
```

An analogy: You're reading a 200-page book and have reached page 180. Each new page only requires reading 1 page. If you tear out the first 180 pages, write a one-page summary, then claim "I only need to read 1 page of summary" — but you only needed to read 1 new page anyway! The act of tearing the book wasted time.

### When Should You Actually Summarize?

**Only when you're truly approaching the context window limit.** For example, a 128K window has used 120K, and adding new messages would overflow — then you have no choice but to compress.

But before that point (e.g., only using 10%~50%), keeping the original history intact is the optimal strategy. Don't fight against KV Cache.

### Impact on API Billing

You might say: "Even if cache hits, doesn't the API provider still charge by input token count?"

In fact, major providers already offer significant discounts for cached tokens (far more than half off):

| Provider | Model | New Input Token | Cached Input Token | Cache Discount |
|---|---|---:|---:|---:|
| [OpenAI](https://openai.com/api/pricing/) | GPT-5 Series | $1.25 | $0.125 | 90% |
| [OpenAI](https://openai.com/api/pricing/) | GPT-4.1 | $2.00 | $0.50 | 75% |
| [OpenAI](https://openai.com/api/pricing/) | GPT-4.1 Mini | $0.40 | $0.10 | 75% |
| [Anthropic](https://www.anthropic.com/api) | Claude Sonnet 4.x | $3.00 | $0.30 | 90% |
| [Anthropic](https://www.anthropic.com/api) | Claude Opus 4.x | $15.00 | $1.50 | 90% |
| [Anthropic](https://www.anthropic.com/api) | Claude Haiku | $0.80 | $0.08 | 90% |
| [Google AI Studio](https://ai.google.dev/pricing) | Gemini 2.5 Pro | $1.25 | $0.125 | 90% |
| [Google AI Studio](https://ai.google.dev/pricing) | Gemini 2.5 Flash | $0.15 | $0.015 | 90% |
| [Google AI Studio](https://ai.google.dev/pricing) | Gemini 2.0 Flash | $0.10 | $0.025 | 75% |

Chinese providers typically offer even more aggressive cache discounts, especially the DeepSeek series (cached token prices as low as 1/10 or even lower than new tokens).

**This means:** At the API billing level, keeping the original history intact is equally economical. Suppose you have 8000 tokens of history:

- Keep as-is: 8000 × cached price (10~25% of full price) + new message × full price
- Replace with summary: 500 × full price (summary is new content, no cache) + new message × full price + summary generation output cost

On the surface 8000 → 500 seems like savings, but 8000 tokens at 10% pricing = equivalent to 800 tokens at full price. Adding summary output costs and information loss, the benefit is minimal or even negative.

For **self-deployed models (vLLM/TGI)**: There's no per-token billing; overhead purely depends on GPU computation. Here the advantage of keeping original history is overwhelming — cache hit = zero extra computation.

---

## The Same Problem in Agentic Loops

The above misconception has a variant in Agent Loop design: merging multi-turn tool call history into a single "stateless message" to "save tokens." Let's analyze this with a concrete example.

### Background

In an Agentic RAG iterative search scenario, the Agent calls LLM each round to decide the next action (search, discard, finish). The LLM needs to know:

1. The user's original question
2. Which tool calls were previously executed
3. What evidence has been collected so far

The question is: **How do you pass this information to the LLM?** This is fundamentally the same question as "should you compress history."

## Two Approaches

### Approach A: Full Merge (Stateless Merge)

Each time calling the LLM, compress all history into one or two user messages:

```python
def build_messages():
    msgs = [
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": query},
    ]
    # Merge all traces into one text
    msgs.append({"role": "user", "content": f"[Executed tool calls]\n{trace_text}"})
    # Merge all evidence into one JSON
    msgs.append({"role": "user", "content": f"[Current evidence]\n{evidence_json}"})
    return msgs
```

**Motivation**: Fewer messages, simpler structure, and omits the LLM's assistant replies from history (which may include verbose thinking/reasoning) — intuitively saving tokens.

### Approach B: Standard Multi-Turn Conversation (Stateful Messages)

Maintain the complete conversation structure, appending assistant tool_call + tool result each round:

```python
messages = [
    {"role": "system", "content": SYSTEM_PROMPT},
    {"role": "user", "content": query},
]

for each iteration:
    response = llm.chat(messages, tools=...)
    messages.append(response.message)  # assistant with tool_calls
    result = execute_tool(response.tool_call)
    messages.append({"role": "tool", "content": result, "tool_call_id": ...})
```

## A Concrete Example

Suppose the Agent runs 3 rounds, each tool returning ~500 tokens of evidence, with ~200 tokens of LLM reasoning per round.

### Approach A: Input Tokens Across 3 Rounds

```
Round 1: system(100) + user(50)                                     = 150
Round 2: system(100) + user(50) + trace(30) + evidence(500)         = 680
Round 3: system(100) + user(50) + trace(60) + evidence(1000)        = 1210
                                                        Total input = 2040
```

Every time it's entirely new content → **KV Cache hit rate ≈ 0%** → all 2040 tokens require full GPU computation from scratch.

### Approach B: Input Tokens Across 3 Rounds

```
Round 1: system(100) + user(50)                                     = 150
Round 2: system(100) + user(50) + asst_1(200) + tool_1(500)         = 850
Round 3: system(100) + user(50) + asst_1(200) + tool_1(500)
         + asst_2(200) + tool_2(500)                                = 1550
                                                        Total input = 2550
```

More assistant messages (+400 tokens), but the key difference:

- Round 2's first 150 tokens are identical to Round 1 → **cache hit**
- Round 3's first 850 tokens are identical to Round 2 → **cache hit**

Tokens actually needing computation:

```
Round 1: 150 (full computation)
Round 2: 700 (first 150 cache hit, only compute new 700)
Round 3: 700 (first 850 cache hit, only compute new 700)
                                    Actual computation = 1550
```

### Comparison Table

| Metric | Approach A (Full Merge) | Approach B (Standard Multi-Turn) |
|---|---|---|
| Total input tokens | 2040 | 2550 |
| KV Cache hit rate | 0% | ~60% |
| **Actual GPU computation** | **2040** | **1550** |
| LLM comprehension difficulty | Higher (non-standard format) | Low (native training format) |

**Conclusion: Approach A appears to have fewer tokens but actually requires more computation.**

## Deep Dive: Prefill, Decode, and KV Cache

### The Two Phases of LLM Inference

You've surely noticed: after the LLM receives input, the first token comes out slowly, but subsequent tokens stream quickly. This reflects the two phases:

**1. Prefill**: Process all input tokens, computing Key and Value vectors for each token at every Transformer layer, storing them in the KV Cache. This is **compute-intensive** — requiring full attention matrix operations on N tokens, with complexity O(N²).

**2. Decode**: Generate output tokens one by one. For each new token generated, only its Query needs attention against existing Keys in the KV Cache, with complexity O(N). Then the new token's K and V are appended to the cache for the next token.

An analogy:
- Prefill = Reading an entire book and taking notes (time-consuming, corresponds to slow TTFT)
- Decode = Writing answers based on notes (relatively easy, corresponds to fast subsequent tokens)

So the "pause then stream" you experience is the Prefill → Decode boundary.

### What Is KV Cache?

The Self-Attention computation at each Transformer layer:

```
Attention(Q, K, V) = softmax(Q × K^T / √d) × V
```

For a model with 32 layers, Key dimension 128, and 32 attention heads (similar to LLaMA-7B), the KV Cache size for 1000 tokens:

```
32 layers × 2(K and V) × 32 heads × 1000 tokens × 128 dims × 2 bytes(fp16)
≈ 512 MB
```

Once computed, these K and V vectors can be repeatedly reused during the Decode phase when generating subsequent tokens — no need to recompute for historical tokens. This is the core value of KV Cache.

### Decode Phase: Only One Token Computed Per Step

During Decode, each step always computes Q/K/V for **exactly 1 new token**. The new token's KV is directly appended to the next slot in the cache:

```
Block5 (capacity 16):
  slot 0: token_a's KV  ← already computed
  slot 1: token_b's KV  ← already computed
  slot 2: token_c's KV  ← new token, only compute this one, write here
  slot 3~15: empty
```

A Block is the **storage management** unit for KV Cache (similar to memory paging), not a computation unit. When a block isn't full, the new token's KV is directly written to the next slot in the same block without affecting existing values or requiring the entire block to be recomputed.

### Cross-Request Prefix Caching

Key insight: **If two requests share the same prefix, the KV vectors for the prefix are identical and don't need recomputation.**

#### Example: Standard Multi-Turn Conversation in Agent Loop

Assume system prompt = "You are a search assistant", user question = "What is GraphRAG?"

**Round 1 request:**
```
[system: You are a search assistant] [user: What is GraphRAG?]
 ←────────── 150 tokens ───────────→
```
Prefill computes KV for 150 tokens → stored in cache, key = hash("You are a search assistant|What is GraphRAG?")

LLM returns: call search({"query": "GraphRAG"})

**Round 2 request:**
```
[system: You are a search assistant] [user: What is GraphRAG?] [asst: search(...)] [tool: Result A]
 ←──── identical to Round 1 ────→ ←────── new 700 tokens ──────→
 ←────────────────────── 850 tokens ──────────────────────────→
```

The inference engine discovers: the hash of the first 150 tokens matches the cache!

```
Cached: KV for tokens 1~150 (directly reused, 0 computation)
To compute: KV for tokens 151~850 (only compute new 700 tokens)
```

**Round 3 request:**
```
[same 850 tokens above] [asst: search(...)] [tool: Result B]
 ←─ cache hit ─→ ←── new 700 ──→
```

Cache hits 850 tokens, only need to compute 700 tokens.

#### With the Full Merge Approach

**Round 2 request:**
```
[system: You are a search assistant] [user: What is GraphRAG?] [user: [Executed tools]\n search→5 results] [user: [evidence]\n{...500 chars...}]
 ←──── same as Round 1 ────→ ←─────────── entirely new content ───────────────→
```

First 150 tokens match, the remaining 530 tokens are new content.

**Round 3 request:**
```
[system: You are a search assistant] [user: What is GraphRAG?] [user: [Executed tools]\n search→5 results\n search→3 results] [user: [evidence]\n{...1000 chars...}]
 ←──── same as Round 1 ────→ ←───────── content changed! ─────────────────→
```

The third message's content changed from `"search→5 results"` to `"search→5 results\n search→3 results"` — **cache is fully invalidated from this point**:

```
Cache hit: 150 tokens (only system + user query)
To compute: 1060 tokens
```

Compare with Approach B which only needs to compute 700 tokens in the same round. The gap accelerates with more iterations.

### Strict Sequential Nature of Prefix Matching

Prefix caching is **sequentially matched block by block from the beginning**. The reason is positional encoding in the attention mechanism — the same token at position 0 and position 16 has different KV values.

This means: **If new tokens are inserted at the beginning, the entire cache is invalidated and everything must be recomputed.**

```
In cache:    [block0][block1][block2][block3][block4]
New request: [new_block][block0'][block1'][block2'][block5][block6]
                ✗ → first block doesn't match, subsequent blocks can't be reused even if content is identical
```

You cannot skip ahead to match later blocks — positions changed, so KV values changed.

This also explains why **placing the system prompt at the very beginning is beneficial** — it's the fixed prefix shared by all requests, ensuring the beginning portion always has cache hits.

### Prefix Caching Implementation Mechanism (vLLM)

1. **Block hashing**: Divide the token sequence into fixed-size blocks (e.g., 16 tokens), compute hash for each block's content
2. **Sequential block matching**: When a new request arrives, compare hashes block by block from the start to find the longest matching prefix
3. **Reuse KV Blocks**: Matched blocks directly reference cached KV data in GPU memory
4. **Only compute the tail**: Start prefill from the first non-matching block

```
Cached request:  [block0][block1][block2][block3][block4]
New request:     [block0][block1][block2][block5][block6]
                    ✓       ✓       ✓      ✗ → start computing from here
```

### Visual Comparison

```
Approach B (Standard Multi-Turn) — only compute new tail content each round

Round 1: [████████]                    compute 150
Round 2: [--------][██████████████]    compute 700  (first 150 cache hit)
Round 3: [--------------------][████]  compute 700  (first 850 cache hit)
                              Total computation = 1550

Approach A (Full Merge) — content changes from the 3rd message each round

Round 1: [████████]                    compute 150
Round 2: [--------][██████████████]    compute 530  (first 150 cache hit)
Round 3: [--------][████████████████]  compute 1060 (first 150 cache hit, rest all changed)
                              Total computation = 1740
```

As rounds increase, Approach A's disadvantage accelerates.

## Hidden Costs of Approach A

### 1. Decreased Model Comprehension

The tool use format LLMs see during training is:

```
assistant: I'll search for... [tool_call: search({query: "..."})]
tool: [results...]
assistant: Based on results, I'll now... [tool_call: ...]
```

Simulating this with plain text:

```
user: [Executed tool calls]
  [0] search({"query": "..."}) → 5 results
  [1] search({"query": "..."}) → 3 results
```

The model needs extra "cognitive overhead" to understand this non-standard format, potentially leading to:
- Repeating already-executed tool calls (because the structure isn't as clear as native format)
- Inability to correctly distinguish which information comes from tools vs. from the user

### 2. Cannot Express Tool Failures

In the standard approach, tool failures can be explicitly returned:

```json
{"role": "tool", "content": "Error: timeout after 10s", "tool_call_id": "..."}
```

The LLM sees this and adjusts its strategy. In Approach A, you can only write `→ 0 results`, and the LLM can't distinguish "no results found" from "search error."

### 3. Loss of Parallel Tool Call Capability

The standard format supports returning multiple tool_calls at once, and the inference engine knows they're parallel calls from the same round. Approach A's flat trace text cannot express this structure.

## When Does Approach A Have an Advantage?

To be fair, there are a few scenarios where Approach A makes more sense:

1. **The inference engine doesn't support prefix caching** (rare — mainstream engines all support it)
2. **Each round's assistant reasoning is extremely long** (e.g., DeepSeek's thinking often exceeds 2000+ tokens), and you're certain this reasoning doesn't help subsequent decisions
3. **Cross-session recovery needed** — stateless design allows recovery from any intermediate state without depending on complete conversation history

For point 2, a better approach is: maintain the standard multi-turn format, but **truncate the reasoning portion** when appending historical assistant messages, keeping only the tool_call structure. This saves tokens while preserving cache and format advantages.

## Recommended Implementation

```python
messages = [
    {"role": "system", "content": SYSTEM_PROMPT},
    {"role": "user", "content": query},
]

for i in range(max_iterations):
    response = await llm.chat.completions.create(
        model=model, messages=messages, tools=tools_schema
    )
    assistant_msg = response.choices[0].message

    if not assistant_msg.tool_calls:
        break

    # Append assistant message (optional: truncate reasoning to save tokens)
    messages.append(assistant_msg.model_dump())

    # Execute tools and append results
    for tool_call in assistant_msg.tool_calls:
        result = await execute(tool_call)
        messages.append({
            "role": "tool",
            "tool_call_id": tool_call.id,
            "content": json.dumps(result, ensure_ascii=False),
        })
```

Simple, standard, cache-friendly.

## Summary

| | Full Merge | Standard Multi-Turn |
|---|---|---|
| Token count | Slightly fewer | Slightly more |
| Actual inference cost | **Higher** (no cache) | **Lower** (high cache hit rate) |
| Model comprehension accuracy | Lower | Good (native format) |
| Engineering complexity | Manual serialization needed | Framework-native support |
| Observability | Poor (lost structure) | Good (each round is clear) |

**Don't sacrifice the enormous advantages of KV Cache and native format to save a few hundred tokens.** The apparent "optimization" is actually an anti-optimization — like disabling CPU cache to save memory, the cost far outweighs the benefit.

## One-Line Summary

> **Existing history is free; only new content costs. Don't destroy the cache yourself.**
