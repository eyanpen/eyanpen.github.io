# The Biggest Pitfall in GraphRAG: One Entity, Seven Identities

> You thought the hardest part of GraphRAG was "building the graph." In reality, the hardest part is "assigning entity types" — even when you've predefined a strict type schema.

## 1. A Real-World Dataset

We ran GraphRAG entity extraction on 3GPP TS 23.502 (5G Core Network signaling procedure specification). This document is about 700+ pages and one of the most critical standards in the telecom domain.

The results were painful:

- A total of **8,873 distinct entities** were extracted (deduplicated by title)
- **1,123 entities were assigned 2 or more types** — 12.7% of the total
- The most extreme case, `PMIC`, was classified into **7 different types**: `ARCHITECTURE_CONCEPT`, `DATA_TYPE`, `INFORMATION_ELEMENT`, `MANAGEMENT_ENTITY`, `NETWORK_ELEMENT`, `PROCEDURE`, `PROTOCOL`

Note that this experiment **already used a strictly predefined entity type schema**, with the prompt explicitly constraining the LLM to only use the specified type set. In other words, this isn't chaos caused by "no constraints" — it's **chaos that persists even after constraints are applied**.

What's worse, these "type conflicts" don't occur across different documents — they happen **within the same document** and even **within the same chunk**. When the LLM reads a minimal text segment, even with explicit type constraints, it still assigns different types to the same entity.

We found **63 text_unit-level overlapping conflicts** — the same entity annotated with two different types within the same text block. For example:

| Entity | Labeled as | Also labeled as |
|--------|-----------|----------------|
| AF | ORGANIZATION | NETWORK_FUNCTION |
| NRF | INTERFACE | NETWORK_FUNCTION |
| 5G SECURITY CONTEXT | SECURITY_ELEMENT | ARCHITECTURE_CONCEPT |
| HPLMN | NETWORK_FUNCTION | ORGANIZATION |
| SERVICE REQUEST | INFORMATION_ELEMENT | PROCEDURE |

This isn't the LLM making rookie mistakes, nor is the schema poorly designed. Think about it: `AF` (Application Function) genuinely is both a "network function" and an "organizational role"; `NRF` is both a "network function" and exposes "interfaces." These types are all in our predefined schema, and the LLM picks a "legal" type every time — it just picks different legal types for the same entity. **The problem isn't that the LLM judged wrong, nor that the schema isn't strict enough — it's that real-world entities are inherently not single-typed.**

## 2. Why Is This Problem So Hard?

### 2.1 Entities Are Inherently Multi-Faceted

In 3GPP specifications, the term `AMF` (Access and Mobility Management Function):

- In architecture diagrams, it's a **NETWORK_FUNCTION**
- In signaling procedures, it's a participant in a **PROCEDURE**
- In deployment descriptions, it's a **NETWORK_ELEMENT**
- In interface definitions, it's an endpoint of an **INTERFACE**

The same entity plays different roles in different contexts. This isn't a bug — it's reality.

### 2.2 LLM Type Judgment Depends on the Context Window

GraphRAG entity extraction is performed chunk by chunk. Each text_unit is roughly a few hundred tokens, and the LLM can only see that small segment.

The same entity `PDU SESSION ESTABLISHMENT`:
- In a chunk describing signaling procedures, the LLM classifies it as **PROCEDURE**
- In a chunk describing message formats, the LLM classifies it as **INFORMATION_ELEMENT**

Both judgments are correct, but they conflict when merged into the knowledge graph.

### 2.3 No Matter How Good the Schema, Type Boundaries Are Inherently Fuzzy

We already predefined a type schema, but who defines the boundary between `ARCHITECTURE_CONCEPT` and `NETWORK_FUNCTION`? In the 3GPP context, many concepts naturally span multiple categories. `POLICY CONTROL` is both a "procedure" (PROCEDURE) and an "architectural concept" (ARCHITECTURE_CONCEPT) — both types are in our schema, and the LLM isn't wrong to pick either one.

This isn't a problem of poorly written prompts or imprecise schema definitions — it's **a fundamental tension between the granularity of type systems and the complexity of the real world**. You can make the schema more fine-grained, but a finer schema only creates more boundary issues, not fewer.

### 2.4 Scale Amplifies the Problem

Our data shows that among entities with multiple types, the top 20 entities average 4–7 types and are associated with 10–200 descriptions. A core entity like `AF` has 209 descriptions, 192 text_unit references, and 4 types.

When a knowledge graph contains thousands of such "multi-faceted entities," downstream community detection, relationship reasoning, and summary generation are all affected — because the graph structure is polluted by type noise.

## 3. How Does the Industry Currently Address This?

### Approach 1: Predefined Strict Type System (Schema-First) ⚠️ We Already Tried This

**Method**: Before extraction, manually define a strict entity type schema and explicitly constrain the LLM in the prompt to only use these types.

**Representatives**: Microsoft GraphRAG's default configuration, most enterprise knowledge graph projects.

**Our actual results**: All the data at the beginning of this article was produced under Schema-First mode. We predefined the type set and explicitly constrained it in the prompt — yet 1,123 entities still had multi-type conflicts, and 63 text_unit-level overlapping conflicts persisted.

**Why it's not enough**:
- Schema can constrain the LLM to "only pick from these types," but **can't constrain it to "pick only one for the same entity"**
- Domain concepts are inherently multi-faceted; `AF` in the 3GPP context genuinely is both NETWORK_FUNCTION and ORGANIZATION — no schema, however strict, changes this fact
- Requires domain experts to design the schema — high cost, and you need to redesign for each new domain
- Being too strict loses information — forcing `AF` to be `NETWORK_FUNCTION` discards its semantics as `ORGANIZATION`

**Conclusion**: Schema-First is a necessary condition but not a sufficient one. It reduces the "random naming" problem but doesn't solve the fundamental contradiction of "one entity, multiple identities."

### Approach 2: Allow Multi-Types, Post-Processing Merge (Multi-Label + Post-Processing)

**Method**: Don't limit the number of types during extraction; allow an entity to have multiple types, then merge, deduplicate, and select a primary type through rules or models in post-processing.

**Representatives**: LlamaIndex's PropertyGraphIndex, some academic research.

**Pros**:
- Preserves multi-faceted entity information
- No information loss during extraction

**Cons**:
- Post-processing logic is complex; rules are hard to enumerate exhaustively
- "Selecting a primary type" itself requires domain knowledge
- Graph complexity increases; query performance degrades

**Suitable for**: Exploratory analysis, early stages where domain boundaries are uncertain.

### Approach 3: Hierarchical Typing

**Method**: Build a hierarchical type system where, for example, `NETWORK_FUNCTION` is a subtype of `ARCHITECTURE_CONCEPT`. Extract at the finest granularity; aggregate by hierarchy during queries.

**Representatives**: Wikidata's type system, YAGO knowledge base.

**Pros**:
- Balances precision and flexibility
- Supports queries at different granularities

**Cons**:
- Designing the hierarchy itself is a major undertaking
- LLMs struggle to accurately determine hierarchical relationships during extraction
- Cross-domain hierarchies are hard to unify

**Suitable for**: Large-scale, long-term knowledge graph projects.

### Approach 4: Abandon Explicit Types, Use Embeddings (Type-Free + Embedding)

**Method**: Don't assign discrete type labels to entities; instead, use vector embeddings to represent semantic features. Similar entities naturally cluster in vector space.

**Representatives**: Some recent research, such as GNN-based entity representation learning.

**Pros**:
- Completely avoids the type conflict problem
- Captures subtle semantic differences between entities

**Cons**:
- Loses interpretability — you can't tell users "this is a network function"
- Downstream community detection and summary generation need redesign
- Difficult to debug

**Suitable for**: Research projects, scenarios with low interpretability requirements.

### Approach 5: Context-Aware Dynamic Typing

**Method**: Don't fix types during extraction; instead, dynamically determine entity types based on query context. For example, when a user asks about architecture, `AF` is treated as `NETWORK_FUNCTION`; when asking about organization, it's treated as `ORGANIZATION`.

**Representatives**: Currently mostly in the academic exploration stage.

**Pros**:
- Most aligned with reality — an entity's "identity" truly depends on context
- No difficult type decisions needed during extraction

**Cons**:
- Extremely high engineering complexity
- Graph structure can't be determined during offline graph building; community detection algorithms are hard to apply
- Increased query latency

**Suitable for**: A research direction for next-generation GraphRAG systems.

## 4. My Recommendation: Schema-First Foundation + Layered Types + Primary Type Voting + Context Preservation

Our experiments have proven that Schema-First is a necessary starting point — without it, types become even more chaotic. But it alone isn't enough. Based on our hands-on experience with 3GPP documents, I recommend layering a **pragmatic post-processing approach** on top of Schema-First:

### Layer 0: Keep Schema-First (Already in Place)

Continue using the predefined type schema to constrain the LLM. This step is already done; its value lies in keeping types within a finite set, preventing the LLM from freely inventing meaningless types like `THINGY` or `STUFF`.

### Layer 1: Preserve All Types During Extraction

On top of Schema-First, don't force a single type during extraction. If the LLM picks multiple types from the predefined set, keep them all. Preserve every (entity, type, text_unit) triple. This is the raw signal — once lost, it can't be recovered.

### Layer 2: Statistical Voting for Primary Type

For each entity, count how many times it's annotated as each type across all text_units, and select the most frequent as the **primary type**.

Taking `AF` as an example:
- NETWORK_FUNCTION: 150 occurrences → **primary type**
- ORGANIZATION: 30 occurrences
- ARCHITECTURE_CONCEPT: 20 occurrences
- NETWORK_ELEMENT: 9 occurrences

The primary type is used for the knowledge graph's main structure, community detection, and default queries.

### Layer 3: Preserve Alternative Types as Properties

Other types aren't discarded — they're stored as the entity's `alternative_types` property, available for use during queries as needed.

```json
{
  "title": "AF",
  "primary_type": "NETWORK_FUNCTION",
  "alternative_types": ["ORGANIZATION", "ARCHITECTURE_CONCEPT", "NETWORK_ELEMENT"],
  "type_distribution": {
    "NETWORK_FUNCTION": 150,
    "ORGANIZATION": 30,
    "ARCHITECTURE_CONCEPT": 20,
    "NETWORK_ELEMENT": 9
  }
}
```

### Layer 4: Type Conflict Detection and Manual Review

For text_unit-level overlapping conflicts (same entity labeled as different types within the same chunk), flag them as candidates for review. These 63 conflicts are the most worth manually checking — they often reveal blind spots in the type system design.

### What's the Cost?

1. **Increased storage**: Each entity stores multiple types and distribution info; graph data volume increases by roughly 20–30%.
2. **No change to extraction**: No need to modify prompts or extraction pipelines; no additional cost.
3. **Post-processing development needed**: The voting, merging, and conflict detection pipeline requires additional development — roughly 2–3 days of engineering effort.
4. **Slightly more complex queries**: The query layer needs to decide whether to use the primary type or all types, but this logic can be encapsulated.
5. **Can't be fully automated**: Text_unit-level conflicts still require human judgment, but the volume is manageable (only 63 in our case).

## 5. Final Thoughts

GraphRAG papers and blog posts always focus on the flashy capabilities like "community detection" and "global queries," but when it comes to real-world deployment, **entity type chaos is the first roadblock**.

One TS 23.502 document, 8,873 entities, 1,123 with multi-type conflicts — and this is **after applying Schema-First constraints**. This isn't an edge case; it's the norm for all complex domain documents. Predefined type schemas are necessary but far from sufficient.

There's no silver bullet for this problem. But at least we can: **build on Schema-First, avoid losing information during post-processing, use statistical methods to select primary types, preserve multi-faceted nature for downstream use, and keep the conflicts that truly need human judgment within a manageable scope.**

This is the gap between "running a demo" and "going to production" in GraphRAG — and it's the most important one to fill.
