# Why Does Semantic Chunking Need an Embedding API?

> Fixed-length chunking requires no external services, yet semantic chunking absolutely needs an Embedding API — why?

## The Short Answer

The core idea of semantic chunking is to **split text at semantic boundaries**. Determining whether "two pieces of text belong to the same topic" requires converting text into vectors and computing similarity — that's exactly what the Embedding API does.

## Traditional Chunking vs Semantic Chunking

| Dimension | Fixed-Length / Recursive | Semantic Chunking |
|-----------|--------------------------|-------------------|
| Split criteria | Character count, token count, delimiters | **Semantic similarity** between adjacent sentences |
| Requires Embedding | ❌ No | ✅ Yes |
| Split quality | May break in the middle of a topic | Splits at topic transitions, preserving semantic coherence |

Fixed-length chunking is like measuring paper with a ruler — regardless of content, it cuts every 500 characters. Semantic chunking is like a reader who, after finishing a paragraph, asks "is the next part still about the same thing?" If not, that's where the cut goes.

## Two Mainstream Semantic Chunking Strategies

### Strategy 1: Adjacent Similarity (Kamradt Method)

Core idea: Compute semantic distances between adjacent sentences and split where distances spike.

```
Process:
1. Split text into small sentences
2. For each sentence, concatenate buffer_size adjacent sentences as context
3. Call Embedding API to get vectors for each combined sentence
4. Compute cosine distances between adjacent combined sentences
5. Use binary search to find a threshold, split where distance exceeds it
```

Pseudocode:

```python
# Step 1: Build context windows
for i, sentence in enumerate(sentences):
    combined = ""
    for j in range(max(0, i - buffer_size), i):
        combined += sentences[j] + " "
    combined += sentence
    for j in range(i + 1, min(n, i + 1 + buffer_size)):
        combined += " " + sentences[j]
    combined_texts.append(combined)

# Step 2: Get embeddings for all combined sentences (one batch call)
embeddings = embedding_client.embed_texts(combined_texts)
embedding_matrix = np.array(embeddings)

# Step 3: Compute cosine distances only between adjacent sentences
distances = []
for i in range(len(sentences) - 1):
    similarity = dot(embedding_matrix[i], embedding_matrix[i + 1])
    distances.append(1 - similarity)  # Higher distance = greater topic difference

# Step 4: Binary search for threshold targeting total_size / avg_chunk_size cuts
threshold = binary_search_threshold(distances, target_cuts)

# Step 5: Split where distance exceeds threshold
breakpoints = [i for i, d in enumerate(distances) if d > threshold]
```

Intuition: Imagine reading an article sentence by sentence, asking yourself after each one: "Is the next sentence still about the same thing?" When you feel the topic has jumped, you cut there.

**Key characteristic: Only looks at adjacent relationships.** It only computes the distance between sentence[i] and sentence[i+1] — a **local greedy** strategy.

### Strategy 2: Cluster Optimal Segmentation (Dynamic Programming Method)

Core idea: Build a similarity matrix between all sentence pairs and use dynamic programming to find the segmentation that maximizes intra-cluster similarity.

```
Process:
1. Split text into small sentences
2. Call Embedding API to get vectors for all sentences
3. Build an N×N similarity matrix
4. Normalize the matrix by subtracting the mean (prevents degeneration into one giant cluster)
5. Use dynamic programming to find the optimal segmentation
```

Pseudocode:

```python
# Step 1: Get embeddings for all sentences (note: no buffer concatenation)
embeddings = embedding_client.embed_texts(sentences)
embedding_matrix = np.array(embeddings)

# Step 2: Build N×N similarity matrix
similarity_matrix = dot(embedding_matrix, embedding_matrix.T)

# Step 3: Mean normalization to prevent DP from putting everything in one cluster
mean_sim = mean(upper_triangle(similarity_matrix))
similarity_matrix -= mean_sim
fill_diagonal(similarity_matrix, 0)

# Step 4: Dynamic programming for optimal segmentation
# dp[i] = maximum intra-cluster similarity sum for the first i+1 sentences
for i in range(n):
    for size in range(1, i + 2):
        start = i - size + 1
        if cluster_size(start, i) > max_chunk_size and size > 1:
            break
        reward = sum(similarity_matrix[start:i+1, start:i+1])
        if start > 0:
            reward += dp[start - 1]
        dp[i] = max(dp[i], reward)

# Step 5: Backtrack to get optimal segmentation
clusters = backtrack(segmentation)
```

**Key characteristic: Globally optimal.** It considers relationships between all sentence pairs and uses DP to find the overall best segmentation.

## Deep Comparison of the Two Strategies

### Fundamental Algorithmic Differences

| Dimension | Kamradt (Adjacent Similarity) | Cluster (Dynamic Programming) |
|-----------|-------------------------------|-------------------------------|
| Scope | **Local** — only adjacent sentences | **Global** — all sentence pairs |
| Decision method | Greedy: cut when distance exceeds threshold | Optimization: maximize intra-cluster similarity |
| Threshold | Binary search for target cut count | No threshold needed, DP decides automatically |
| Context enhancement | ✅ buffer_size concatenation | ❌ Uses raw sentences directly |
| Size constraints | avg_chunk_size + max_chunk_size dual constraint | max_chunk_size hard constraint |

**The core difference in one sentence:**

- Kamradt asks: "Is there a topic transition between these two adjacent sentences?"
- Cluster asks: "Which grouping makes sentences within each group most similar to each other?"

### An Intuitive Example

Consider 6 sentences with the following topic distribution:

```
Sentence 1: Discussing Apple's earnings report
Sentence 2: Discussing Apple's new products
Sentence 3: Discussing the weather forecast
Sentence 4: Discussing tomorrow's temperature
Sentence 5: Discussing Apple's stock price
Sentence 6: Discussing Apple's competitors
```

**Kamradt's approach:** Compare adjacent pairs
- Sentence 2→3: Topic jump (Apple → weather), cut!
- Sentence 4→5: Topic jump (weather → Apple), cut!
- Result: [1,2] [3,4] [5,6]

**Cluster's approach:** The global similarity matrix shows sentences 1,2,5,6 are highly similar to each other
- But since DP requires contiguous segmentation (can't skip around), it can only cut contiguous spans
- Result is likely also [1,2] [3,4] [5,6], but the reasoning is different

**The key difference emerges when boundaries are fuzzy:**

Consider an article that gradually transitions from "EV technology" to "energy policy":

```
Sentence 1: Tesla released a new generation of battery technology
Sentence 2: The new battery's energy density improved by 50%
Sentence 3: Higher energy density means longer driving range
Sentence 4: Range anxiety has been a barrier for consumers buying EVs
Sentence 5: The government introduced charging station subsidies to address this
Sentence 6: Subsidies cover both residential and commercial charging facilities
Sentence 7: Commercial charging uses time-of-use electricity pricing
Sentence 8: Time-of-use pricing is a key component of electricity market reform
```

**What Kamradt sees (adjacent distances):**

```
1→2: 0.08  (both about batteries)
2→3: 0.10  (battery → range, very close)
3→4: 0.12  (range → range anxiety, very close)
4→5: 0.15  (consumers → government policy, slightly far but not outstanding)
5→6: 0.09  (both about subsidies)
6→7: 0.13  (subsidies → pricing, somewhat far)
7→8: 0.11  (both about pricing)
```

No single distance clearly "spikes" — the topic slides gradually. Kamradt's binary search struggles to find a reasonable threshold, potentially producing suboptimal splits like [1-4][5-8] or [1-3][4-6][7-8].

**What Cluster sees (global similarity matrix summary):**

```
        S1    S2    S3    S4    S5    S6    S7    S8
S1      --   0.9   0.7   0.4   0.2   0.1   0.1   0.05
S2           --    0.8   0.5   0.2   0.15  0.1   0.05
S3                 --    0.6   0.3   0.2   0.15  0.1
S4                       --    0.5   0.4   0.3   0.2
S5                             --    0.8   0.6   0.4
S6                                   --    0.7   0.5
S7                                         --    0.8
S8                                               --
```

The global view clearly shows: sentences 1-3 are highly similar to each other (battery/range technology), sentences 5-8 are highly similar to each other (policy/pricing), and sentence 4 is a transition. DP optimization discovers that [1-3][4-8] or [1-4][5-8] maximizes intra-cluster similarity, producing a more reasonable split.

**The essential difference:** Kamradt only looks at "the gap between adjacent sentences" — in a gradual transition, each step's gap is small, like the boiling frog metaphor. Cluster looks at "the overall similarity within each group" — even when the transition is smooth, it can still detect that sentence 1 and sentence 8 are essentially unrelated.

### Embedding Cost Comparison

This is one of the most important practical differences between the two strategies:

| Dimension | Kamradt | Cluster |
|-----------|---------|---------|
| Embedding input | combined_sentence (with buffer context) | Raw sentences (no buffer) |
| Embedding call count | N texts, 1 batch call | N texts, 1 batch call |
| Average text length | Longer (~7 sentences, buffer_size=3) | Shorter (1 sentence) |
| Total token consumption | **Higher** (buffer causes input inflation) | **Lower** (no redundancy) |
| Post-embedding computation | O(N) — only adjacent distances | **O(N²)** — full similarity matrix |
| DP computation | None | O(N × max_cluster_size) |

#### Concrete Numbers (1000 sentences, ~30 tokens each)

**Kamradt:**
- Embedding input: 1000 combined_sentences, each ~7×30 = 210 tokens
- Total token consumption: 1000 × 210 = **210,000 tokens**
- Distance computation: 999 dot products → negligible
- Memory: 1000 × embedding_dim matrix

**Cluster:**
- Embedding input: 1000 raw sentences, each ~30 tokens
- Total token consumption: 1000 × 30 = **30,000 tokens**
- Similarity matrix: 1000 × 1000 = **1 million floats** (~8MB)
- DP computation: O(1000 × max_cluster_size) iterations

**Conclusions:**
- **Embedding API cost**: Kamradt consumes ~7x more tokens (due to buffer concatenation), higher API cost
- **Compute resources**: Cluster's O(N²) matrix and DP are more expensive on local CPU/memory
- **Network latency**: Same for both (both use 1 batch call, or multiple calls based on batch_size)

#### Large-Scale Scenario (100,000 sentences)

| Metric | Kamradt | Cluster |
|--------|---------|---------|
| Total embedding tokens | ~21 million tokens | ~3 million tokens |
| API calls (batch_size=500) | 200 | 200 |
| Similarity computation | 99,999 dot products | 10 billion dot products (N² matrix) |
| Memory usage | ~400MB (embedding matrix) | ~40GB (N² similarity matrix) ⚠️ |

**At 100K sentences, Cluster's N² matrix will blow up memory** — this is its hard limitation. In practice, Cluster is better suited for medium-length documents (hundreds to thousands of sentences), while Kamradt can handle any length.

### Split Quality Comparison

| Scenario | Kamradt | Cluster |
|----------|---------|---------|
| Clear topic boundaries | ✅ Excellent, obvious distance spikes | ✅ Excellent |
| Gradual topic transitions | ⚠️ May fail to find split points | ✅ Global optimization still finds best split |
| Short documents (<50 sentences) | ✅ Fast | ✅ Higher quality |
| Long documents (>10K sentences) | ✅ Linear scaling | ❌ Memory explosion |
| Very short sentences | ⚠️ Needs buffer for context | ⚠️ Short sentence embeddings are low quality |

### How to Choose?

| Your Scenario | Recommended | Reason |
|---------------|-------------|--------|
| Unknown document length, need general solution | Kamradt | Linear complexity, won't blow memory |
| Short documents (<2000 sentences), want optimal splits | Cluster | Globally optimal, higher quality |
| Embedding API charges per token | Cluster | No buffer inflation, 7x fewer tokens |
| Limited local compute resources | Kamradt | O(N) computation, memory-friendly |
| Fuzzy topic boundaries, need precise splits | Cluster | DP global optimization is more robust |

## Why Can't Other Methods Replace Embedding?

| Alternative | Problem |
|-------------|---------|
| Keyword overlap / TF-IDF | Cannot capture synonyms or contextual semantics ("automobile" and "vehicle" would be considered unrelated) |
| Rule-based delimiters (paragraphs, periods) | One paragraph may contain multiple topics; different paragraphs may discuss the same topic |
| LLM direct judgment | Too expensive, high latency, unsuitable for batch processing tens of thousands of sentences |

Embedding maps text into a high-dimensional semantic space where semantically similar texts have small vector distances and dissimilar texts have large distances. This is currently the optimal approach for semantic similarity measurement, balancing **cost, speed, and quality**.

## buffer_size: The Role of the Context Window

Semantic chunking has a key parameter `buffer_size` (default: 3) that determines how much context is concatenated when generating embeddings for each sentence.

```python
# Concatenation logic
for each sentence[i]:
    combined = sentence[i-3] + sentence[i-2] + sentence[i-1]  # 3 before
              + sentence[i]                                     # current
              + sentence[i+1] + sentence[i+2] + sentence[i+3]  # 3 after
```

**Key point: buffer_size does not affect the number of Embedding calls — only the length of each input text.**

With 10 sentences, whether buffer_size is 1 or 10, you still embed 10 combined_sentences. The difference is how much context each text contains:

| buffer_size | Avg sentences per text | Effect |
|-------------|----------------------|--------|
| 1 | ~3 | Less context, may misjudge |
| 3 (default) | ~7 | Balance point |
| 10 | ~21 | Rich context, but may exceed model token limit |

Note: Embedding models have input length limits (e.g., BGE-M3 max 8192 tokens). If buffer_size is too large, texts get truncated, potentially losing the current sentence's information.

## Performance at Scale

Suppose a long document is split into 100,000 sentences:

- Texts to embed = **100,000**
- With batch_size of 500, actual API calls = 100,000 ÷ 500 = **200 HTTP requests**

The performance bottleneck is API call count (determined by total sentences and batch_size), independent of buffer_size.

## Fallback Strategy: What If Embedding Is Unavailable?

Good system design should account for Embedding service unavailability. The common approach: when Embedding calls fail, automatically fall back to recursive chunking (pure rule-based splitting, no Embedding needed).

This means semantic chunking is an **enhancement**, not a **dependency** — the system still works without the Embedding service, just with lower split quality.

## Summary

| Question | Answer |
|----------|--------|
| Why is Embedding needed? | Judging semantic similarity requires vector representations |
| Can rules replace it? | No, rules cannot capture semantics |
| Can LLM replace it? | Theoretically yes, but cost and latency are unacceptable |
| Kamradt vs Cluster core difference? | Local adjacent comparison vs global optimal segmentation |
| Which has higher Embedding cost? | Kamradt: higher token consumption (buffer inflation); Cluster: higher compute cost (N² matrix) |
| Which for large documents? | Kamradt — linear complexity, won't blow memory |
| Which for optimal splits? | Cluster — global DP optimization, but limited to medium-length documents |
| What if service is unavailable? | Both fall back to rule-based chunking |

The Embedding API is the "eyes" of semantic chunking — without it, the chunking algorithm is a blind person cutting a cake. The two strategies "see" text differently: Kamradt is like a line-by-line scanner, Cluster is like an editor with a bird's-eye view. Which to choose depends on your document scale and split quality requirements.
