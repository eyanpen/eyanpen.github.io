# What Is GraphRAG Really Doing? — A Deep Dive into Microsoft's Blog Post

> Original: [GraphRAG: Unlocking LLM discovery on narrative private data - Microsoft Research](https://www.microsoft.com/en-us/research/blog/graphrag-unlocking-llm-discovery-on-narrative-private-data/)

---

In early 2024, Microsoft published a technical blog post. The core message boils down to one sentence: **Traditional RAG falls short with complex data, and GraphRAG fills the gap using knowledge graphs + graph clustering.**

This isn't an academic paper — it reads more like a "tech pitch" aimed at technical decision-makers and engineers. Let me break it down.

---

## Where Does Traditional RAG Fall Short?

To understand what GraphRAG solves, we need to start with the pain points of traditional RAG. The article highlights two scenarios where traditional RAG struggles:

### Information That Can't Be Connected

Imagine asking an AI: "What has Novorossiya done?"

Traditional RAG takes the word "Novorossiya" and runs a vector search. But among the 10 text chunks retrieved, none directly mentions that name — the answer is scattered across different documents, connected only through indirect relationships between entities. Vector search only finds text that "looks similar"; it can't handle this kind of reasoning that requires "jumping" between connections.

GraphRAG works differently: it locates the Novorossiya node in the knowledge graph, then traverses along relationship edges — actions, goals, related organizations — and assembles the complete answer.

Put simply, vector retrieval is "local matching," while real-world knowledge is often connected indirectly through chains of entity relationships.

### Can't Answer "Big Questions"

Another example: "What are the top 5 themes in this dataset?"

Traditional RAG is stumped — the word "themes" is too broad. Vector search doesn't know which direction to look, and ends up matching some irrelevant text that happens to contain the word "theme." The answer naturally goes off track.

This is fundamentally a granularity problem: vector RAG retrieves at the text chunk level, but "overall themes" require a macro-level understanding of the entire dataset. No single chunk can support that kind of answer.

GraphRAG handles this easily with pre-built community clusters and community summaries, extracting themes directly from the macro structure.

---

## How Does GraphRAG Work?

The entire process has two phases: offline indexing, then online question answering.

### Offline Indexing: Three Steps

```
Raw Documents
    │
    ▼
┌─────────────────────────────┐
│ Step 1: Entity & Relationship│  LLM processes documents chunk
│ Extraction                   │  by chunk, extracting all
│                              │  entities (people, places,
│                              │  organizations, etc.) and
│                              │  their relationships
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│ Step 2: Knowledge Graph      │  Assemble extracted entities
│ Construction                 │  and relationships into a
│                              │  complete graph structure
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│ Step 3: Community Detection  │  Perform bottom-up hierarchical
│ & Summarization              │  clustering on the graph (e.g.,
│                              │  Leiden algorithm), generate
│                              │  LLM summary reports for each
│                              │  community
└─────────────────────────────┘
```

In short: first let the LLM extract all the people, events, things, and their relationships from the documents, assemble them into a large graph, then cluster the graph into groups and write a summary for each group.

### Online Answering: Choose Strategy by Question Type

| Question Type | How to Find the Answer |
|---------|---------|
| Specific questions (e.g., "What has Novorossiya done?") | Locate entity in graph → traverse relationships → collect related text → generate answer |
| Macro questions (e.g., "Top 5 themes") | Use community summaries directly → aggregate layer by layer → generate global answer |

---

## Technical Points Worth Digging Into

### Why Use LLM for Graph Construction Instead of Traditional NLP?

The traditional approach uses NER (Named Entity Recognition) + relation extraction models, but these have hard limitations: you need to predefine entity types and relation types, they break when you switch domains, and they can't capture implicit relationships.

LLM advantages are clear:
- **Zero-shot capability** — no need to train separately for each domain
- **Can read between the lines** — for example, extracting the implicit "government attention" relationship from "the Attorney General's office reported the creation of Novorossiya"
- **Not constrained by schema** — let the LLM discover entity and relationship types on its own

The trade-off is straightforward: LLM calls are expensive, and the indexing phase needs to process the entire dataset, so computational costs are significant.

### Community Detection — GraphRAG's Killer Feature

Many approaches use knowledge graphs to enhance RAG, but what truly sets GraphRAG apart is community detection:

- Uses algorithms like Leiden to partition the knowledge graph into multi-level communities (think of them as "topic clusters")
- Pre-generates an LLM summary report for each community
- Different community levels correspond to different levels of abstraction; choose the right granularity when answering questions

This is the secret behind its ability to answer "big questions" — no need to traverse the entire graph on the fly, just look up the pre-written summaries.

When generating community reports, the LLM receives CSV tables of entities and relationships within that community: an Entities table (entity ID, name, description), a Relationships table (source, target, description, combined_degree), and an optional Claims table. Relationships are sorted by `combined_degree` in descending order, prioritizing the most important ones, with truncation when the token limit is exceeded.

### Provenance — Every Statement Is Traceable

GraphRAG places special emphasis on provenance. The complete evidence chain looks like this:

```
User Query
    → GraphRAG Answer + [Data: Entities (ID), Relationships (ID)]
        → Relationship IDs point to specific edges in the knowledge graph
            → Edges link back to specific passages in the original source documents
```

Answer → entities/relationships in the graph → original documents — fully traceable end to end. For enterprise applications, this capability is critical — you can verify every claim the AI makes.

---

## How Were the Experiments Conducted?

### Dataset

They used the VIINA dataset (violence information from news articles), chosen deliberately:

- Involves multi-party conflict with fragmented information — complex enough
- Includes news sources from both Russian and Ukrainian sides with opposing viewpoints and contradictory information
- Data from June 2023, ensuring it's not in the LLM's training set
- Thousands of articles, far exceeding context window limits — can't be handled without RAG

### Evaluation Results

Four metrics were used for scoring:

| Metric | What It Measures | How It's Evaluated |
|-----|---------|-------|
| Comprehensiveness | How complete is the answer | LLM scorer pairwise comparison |
| Human Empowerment | Does it provide sources for verification | LLM scorer pairwise comparison |
| Diversity | Does it answer from multiple perspectives | LLM scorer pairwise comparison |
| Faithfulness | Does it hallucinate | SelfCheckGPT absolute measurement |

The results are interesting: GraphRAG significantly outperforms traditional RAG on the first three metrics, but they're roughly equal on faithfulness. In other words, GraphRAG's improvement is mainly in "finding more comprehensively," not in "hallucinating less."

---

## Don't Just Look at the Strengths — Know the Limitations Too

This is a pitch piece after all, so it naturally emphasizes the positives. A few caveats to keep in mind:

**High indexing cost** — Every document chunk requires an LLM call to extract entities and relationships. For large datasets, this could take hours or even days. With GPT-4 level models, API costs are considerable.

**Incremental updates are a hard problem** — The article doesn't mention what happens when data changes. In practice, new documents require re-extraction and merging, community structures may change as a result, requiring re-clustering and re-generating summaries. There's no good engineering solution for this yet.

**Extraction quality depends on the LLM** — LLM entity and relationship extraction isn't 100% accurate. It may miss implicit entities, get relationships wrong, and different models produce varying extraction quality with inconsistent results.

**Queries will be slower** — Graph traversal + LLM generation has a longer pipeline than simple vector retrieval + LLM generation, so latency is naturally higher.

**Not every question needs it** — The article itself acknowledges that for simple factual queries (like "What is Novorossiya?"), traditional RAG is sufficient. GraphRAG's advantages are concentrated in multi-hop reasoning and global summarization scenarios.

---

## An Analogy to Build Your Intuition

Imagine you're a new employee at a company, and you want to understand "the most important project developments in the last three months."

**Traditional RAG is like searching through a filing cabinet**: You walk into the archive room and search using "project developments" as a keyword. You find dozens of files scattered across different drawers — meeting minutes, emails, reports. You have to piece the fragments together yourself.

**GraphRAG is like asking a colleague who knows everything**: They've not only read every document but also remember that "Zhang San's Project A and Li Si's Project B are actually related," and know that "last month's budget adjustment affected three departments." They can give you an organized, complete answer right away.

| | Traditional RAG | GraphRAG |
|---|---|---|
| How it works | Search keywords, find relevant passages | Build a relationship network first, then answer along relationships |
| Good at | "What is X?" "How to do X?" | "What's the relationship between X and Y?" "What's the overall picture?" |
| Analogy | A librarian helping you find books | A detective connecting clues into a complete story |
| Weakness | Fragmented, lacks global perspective | Building the relationship network takes time and compute |

---

## Key Takeaways

- **GraphRAG doesn't solve the "search more accurately" problem — it solves the "search dimension" problem** — expanding from text similarity to entity relationships and global structure.

- **The knowledge graph is the means; community clustering is the real innovation** — Many approaches use graphs to enhance RAG, but community detection + pre-summarization is GraphRAG's unique weapon for global queries.

- **Provenance is the foundation of trust** — Every assertion can be traced back to the original document. Enterprise applications can't do without this.

- **The trade-off is indexing cost** — Using LLMs to process all data for graph construction is much more expensive than simple vectorization. This must be weighed when deploying in production.

- **Not a replacement, but a complement** — Use GraphRAG for complex reasoning and global analysis, traditional RAG for simple factual queries. In real systems, combining both is the right approach.
