# Multiple Independent Questions: Batch Into One Request or Split Into Many? — An Analysis of LLM Concurrent Processing

> When you have 5 unrelated questions, should you pack them into one message to the LLM, or send 5 requests simultaneously? Which is faster?

## The Short Answer

**Splitting into multiple independent parallel requests is almost always faster.**

This isn't a gut feeling — it's determined by the underlying inference mechanism of LLMs. Let's walk through the reasoning from first principles.

## 1. How LLMs Generate Text: Autoregressive Decoding

To understand this problem, you first need to know how LLMs "write."

LLMs (GPT-4, Claude, etc.) use **autoregressive generation**: they produce one token at a time, append that token back to the input, then generate the next token. This repeats until generation is complete.

The key insight: **Generating N tokens requires N forward passes.**

This means:
- A 100-token answer requires 100 inference steps
- A 500-token answer requires 500 inference steps
- **Total output length directly determines total latency**

## 2. Batched Request: Output Volumes Stack, Latency Grows Linearly

Suppose you have 5 independent questions, each requiring ~200 tokens to answer.

**Approach A: Combine into one request**

You stuff all 5 questions into a single message:

```
Please answer the following questions separately:
1. xxx
2. xxx
3. xxx
4. xxx
5. xxx
```

The LLM needs to generate total output ≈ 5 × 200 = 1000 tokens. Due to autoregressive decoding, these 1000 tokens are generated **sequentially** — token #201 must wait for the first 200 to finish.

**Total latency ≈ 1000 × per-token generation time**

Plus additional overhead:
- The LLM must maintain context switches between answers ("now answering question 3")
- Longer KV Cache means increasing attention computation at each step
- Actual output often exceeds 1000 tokens (formatting, transition phrases, etc.)

## 3. Split Requests: Parallel Inference, Latency Equals the Slowest One

**Approach B: Split 5 questions into 5 independent requests, sent simultaneously**

Each request independently generates ~200 tokens. If the server has sufficient concurrent processing capacity (all modern LLM services do), these 5 requests are **processed in parallel**.

**Total latency ≈ max(individual request latencies) ≈ 200 × per-token generation time**

Comparison:

| Approach | Total output tokens | Actual latency (relative) |
|----------|-------------------|--------------------------|
| Combined request | ~1000+ | ~1000 steps (sequential) |
| Split into 5 requests | ~200 each | ~200 steps (parallel) |

**Theoretical speedup ≈ 5x** (equals the number of questions).

## 4. Why Does Parallelism Work? — Server-Side Continuous Batching

You might ask: doesn't the LLM server have capacity limits? Won't 5 simultaneous requests queue up?

Modern LLM inference engines (vLLM, TensorRT-LLM, TGI, etc.) all implement **Continuous Batching**:

1. **Multiple requests share the same GPU matrix operation**: GPUs excel at parallel computation. Combining tokens from 5 requests into one batch allows a single forward pass to generate one token for each request simultaneously.
2. **Dynamic scheduling**: Different requests have different output lengths. Shorter ones finish first, and their slots are immediately given to new requests.
3. **Throughput vs. latency decoupling**: Larger batches mean higher GPU utilization and more total tokens processed per unit time.

From the server's perspective:
- 5 short parallel requests → GPU does 5-way batched inference, producing 5 tokens per step
- 1 long request → GPU does single-sequence inference, producing 1 token per step

**The GPU's parallel computing power is wasted when requests are combined.**

## 5. The Prefill Phase Difference

LLM inference has two phases:

1. **Prefill**: Process the input prompt, computing KV Cache for all input tokens. This step can process all input tokens in parallel, with latency roughly linear to input length.
2. **Decode**: Generate output token by token. This step is sequential.

With combined requests:
- Prefill phase: Longer input (all 5 questions concatenated), longer prefill time
- Decode phase: Longer output, longer decode time

With split requests:
- Each request's prefill is shorter, and all 5 prefills can run in parallel or pipelined
- Each request's decode is shorter, and they run in parallel

**Both phases favor splitting.**

## 6. An Often-Overlooked Factor: Quality

Beyond speed, combining requests carries quality risks:

- **Attention dilution**: When an LLM processes multiple unrelated tasks in one generation, its "focus" on each task decreases. Research shows that more irrelevant information in the prompt leads to lower answer quality (the "Lost in the Middle" phenomenon).
- **Format confusion**: Answers to 5 questions easily suffer from numbering errors, omissions, or mismatched responses.
- **Error propagation**: If the answer to question 2 goes wrong, the LLM may be influenced in subsequent answers (autoregressive "inertia").

Split requests completely isolate context, giving each question the LLM's "full attention."

## 7. When Is Combining Actually Better?

To be fair, there are a few scenarios where combining may be more appropriate:

1. **Hidden correlations between questions**: Even if you think they're independent, the LLM might give more consistent answers seeing the full picture (e.g., different sections of the same report).
2. **Strict API rate limits**: If your API quota is 3 requests per minute, you have no choice but to combine 5 questions.
3. **Network latency far exceeds generation time**: If each API call has 2 seconds of network round-trip but generation only takes 0.5 seconds, splitting 5 times (5 × 2s = 10s network overhead) might exceed the combined generation time. But this is rare in practice — modern API network latency is typically 100-300ms, far less than generation time.
4. **Extremely short answers**: If each question only needs a word or two, prefill overhead dominates, and combining can reduce redundant prefill costs.

## 8. How to Verify This Yourself

If you want to test this empirically:

```python
import asyncio
import time
import aiohttp

async def ask_single(session, question):
    start = time.time()
    # Call LLM API
    resp = await session.post(API_URL, json={"prompt": question})
    result = await resp.json()
    return time.time() - start

async def benchmark():
    questions = ["Question 1", "Question 2", "Question 3", "Question 4", "Question 5"]

    async with aiohttp.ClientSession() as session:
        # Approach A: Combined
        start = time.time()
        combined = "Please answer separately:\n" + "\n".join(questions)
        await ask_single(session, combined)
        time_combined = time.time() - start

        # Approach B: Parallel
        start = time.time()
        await asyncio.gather(*[ask_single(session, q) for q in questions])
        time_parallel = time.time() - start

    print(f"Combined: {time_combined:.2f}s")
    print(f"Parallel: {time_parallel:.2f}s")
    print(f"Speedup: {time_combined / time_parallel:.1f}x")
```

In practice, 5 moderately complex independent questions typically achieve 3-5x speedup with parallel requests.

## Summary

| Dimension | Combined request | Split parallel requests |
|-----------|-----------------|----------------------|
| Generation speed | Slow (sequential output of all answers) | Fast (parallel generation, latency = slowest) |
| GPU utilization | Low (single-sequence inference) | High (batched parallel inference) |
| Answer quality | May degrade (attention dilution) | Better (isolated context) |
| API calls | 1 | N |
| Best for | Rate-limited / extremely short answers | Independent questions needing detailed answers |

**Core principle in one sentence: LLM's autoregressive mechanism means output is sequential; combining requests = forcing all outputs into a single serial stream; splitting requests = leveraging server-side parallelism to generate multiple outputs simultaneously. Splitting independent questions is the classic strategy of trading space (concurrent slots) for time.**
