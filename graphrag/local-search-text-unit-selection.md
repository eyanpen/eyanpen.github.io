# GraphRAG Local Search Text Unit Selection Strategy: Design Trade-offs and Improvement Directions

## Introduction

GraphRAG's Local Search needs to select the most relevant raw text fragments (Text Units) associated with the knowledge graph to fill the LLM context window during query time. This selection strategy seems simple — sort by entity similarity, fill one by one — but in real-world scenarios it exposes a significant limitation: **popular entities can monopolize the entire Text Unit budget, causing key text from other entities to be truncated**.

This article provides an in-depth analysis of the root cause of this problem, the core problem it was designed to solve, and possible improvement directions.

---

## What Is the Current Strategy

Local Search's Text Unit selection has four steps:

1. Iterate through selected entities (ranked by vector similarity), collecting each entity's associated `text_unit_ids`
2. Deduplication: each TU is attributed only to the first entity encountered
3. Sorting: by `(entity_index, -num_relationships)` — entity order takes priority, within the same entity sorted by relationship density in descending order
4. Fill into context one by one until reaching the token limit (default 50% of total budget, approximately 6000 tokens)

Core code:

```python
for index, entity in enumerate(selected_entities):
    entity_relationships = [rel for rel in relationships if rel.source == entity.title or rel.target == entity.title]
    for text_id in entity.text_unit_ids or []:
        if text_id not in text_unit_ids_set and text_id in self.text_units:
            num_relationships = count_relationships(entity_relationships, self.text_units[text_id])
            text_unit_ids_set.add(text_id)
            unit_info_list.append((self.text_units[text_id], index, num_relationships))

unit_info_list.sort(key=lambda x: (x[1], -x[2]))
```

---

## Problem Scenario: Popular Entities Monopolize the Budget

### Concrete Example

Suppose the user asks: "What is the anti-inflammatory mechanism of chamazulene?"

Entities returned by vector search:

| Rank | Entity | Associated TU Count | Notes |
|------|--------|---------------------|-------|
| 0 | Chamomile | 50 | High-frequency entity, mentioned in almost all herbal documents |
| 1 | Chamazulene | 4 | Active component of chamomile, fewer specialized references |
| 2 | NF-κB pathway | 2 | Specific anti-inflammatory molecular mechanism |

TU attribution after deduplication:

```
index 0 "Chamomile": TU1, TU2, TU3, ..., TU50  (50 items)
index 1 "Chamazulene": TU51, TU52              (TU1, TU5 already claimed by Chamomile)
index 2 "NF-κB":  TU53                    (only 1 unclaimed)
```

Sorting result:

```
TU1(index=0, rel=5) → TU2(index=0, rel=4) → ... → TU50(index=0, rel=0)
→ TU51(index=1, rel=2) → TU52(index=1, rel=1)
→ TU53(index=2, rel=1)
```

Assuming a token budget of 6000 tokens and each TU averaging 300 tokens, only about 20 TUs can fit.

**Result**: All top 20 positions are occupied by "Chamomile" TUs. The text about "chamazulene's anti-inflammatory mechanism" that the user actually cares about (TU51, TU52, TU53) is entirely truncated. The context fed to the LLM is filled with generic introductions about "Chamomile" but contains no original text supporting chamazulene's specific molecular mechanisms.

---

## Why It Was Designed This Way: What Problem It Solves

This strategy was not designed arbitrarily — it solves a more fundamental problem: **ensuring that the most semantically relevant entities receive the most comprehensive original text support**.

### The Scenario It Addresses

Suppose the user asks: "What is the status of chamomile in European traditional medicine?"

Vector search returns:

| Rank | Entity | Associated TU Count |
|------|--------|---------------------|
| 0 | Chamomile | 50 |
| 1 | European Herbalism | 8 |
| 2 | Lavender | 30 |

In this scenario, "Chamomile" is indeed the most core entity — the user is asking about it. If a round-robin strategy were used (taking 1 TU from each entity in turn), then "Lavender's" 30 TUs would split the budget equally with "Chamomile" — but the user never asked about lavender.

The advantages of the current strategy:
- **Respects semantic ranking**: The entity with the highest vector similarity gets the most original text support, which is correct in most cases
- **Relationship density sorting ensures quality**: Among multiple TUs for the same entity, the most information-dense ones come first
- **Deduplication avoids redundancy**: The same TU won't appear repeatedly because it's associated with multiple entities

### Core Trade-off

This is a classic **relevance depth vs. coverage breadth** trade-off:

- The current strategy chooses **depth**: ensuring the most relevant entity has sufficient original text evidence
- The cost is **breadth**: secondary entities may have no original text support at all

For most "questions about a specific entity" (the design target of Local Search), depth-first is reasonable. The problem emerges when queries involve cross-entity relationships.

---

## The Essence of the Problem: A Single Sorting Dimension Cannot Express Multi-Objective Optimization

Text Unit selection is fundamentally a multi-objective optimization problem:

1. **Relevance**: The semantic relevance of a TU to the query (expressed indirectly through entity ranking)
2. **Information density**: The number of relationships contained in a TU
3. **Coverage**: Ensuring every selected entity has original text support
4. **Diversity**: Avoiding homogeneous content flooding the context

The current strategy uses a single tuple `(entity_index, -num_relationships)` attempting to optimize the first two objectives simultaneously, but completely ignores the latter two.

---

## Improvement Directions

### Approach 1: Per-Entity Cap

The simplest improvement — set a TU contribution cap for each entity:

```python
MAX_TU_PER_ENTITY = 5

for index, entity in enumerate(selected_entities):
    count = 0
    for text_id in entity.text_unit_ids or []:
        if count >= MAX_TU_PER_ENTITY:
            break
        if text_id not in text_unit_ids_set and text_id in self.text_units:
            # ... addition logic unchanged
            count += 1
```

**Pros**: Simple to implement, guarantees each entity at least has a chance to contribute TUs
**Cons**: Cap value is hard to determine; if an entity genuinely needs extensive original text support, it gets artificially limited

### Approach 2: Round-Robin

Each round takes 1 TU from each entity (selecting the best by relationship density), cycling until the budget is exhausted:

```python
entity_queues = {i: sorted_tus_for_entity_i for i in range(len(selected_entities))}
result = []
while budget > 0 and any(entity_queues.values()):
    for i in range(len(selected_entities)):
        if entity_queues[i]:
            tu = entity_queues[i].pop(0)
            result.append(tu)
            budget -= token_count(tu)
```

**Pros**: Guarantees coverage, every entity has original text support
**Cons**: Depth of the most relevant entity is diluted; lower-ranked irrelevant entities also receive equal budget

### Approach 3: Weighted Quota Allocation

Allocate TU quotas based on entity vector similarity scores:

```python
# Assuming similarity scores: [0.95, 0.82, 0.71]
scores = [0.95, 0.82, 0.71]
total = sum(scores)
quotas = [int(max_tus * s / total) for s in scores]
# quotas ≈ [15, 13, 11] (assuming max_tus=39)
```

**Pros**: Balances depth and breadth; higher-relevance entities get more quota without monopolizing
**Cons**: Increased implementation complexity; requires preserving similarity scores from vector search results (not retained in current code)

### Approach 4: Minimum Guarantee + Remaining Competition

Guarantee each entity at least N TUs (e.g., 2), with remaining budget competed for using the current strategy:

```python
# Phase 1: Guarantee 2 best TUs per entity
for entity in selected_entities:
    guaranteed_tus = top_2_by_relationship_density(entity)
    result.extend(guaranteed_tus)

# Phase 2: Fill remaining budget using original sorting strategy
remaining = all_tus - guaranteed_tus
remaining.sort(key=lambda x: (x.entity_index, -x.num_relationships))
fill_until_budget(remaining)
```

**Pros**: Guarantees coverage while preserving the depth advantage of the original strategy
**Cons**: If many entities are selected, the guarantee phase may consume significant budget

---

## Summary

| Dimension | Current Strategy | Issue |
|-----------|-----------------|-------|
| Relevance depth | ✅ Excellent | — |
| Information density | ✅ Excellent | — |
| Coverage breadth | ❌ Missing | Popular entities monopolize budget |
| Content diversity | ❌ Missing | Homogenization risk |

GraphRAG's current Text Unit selection strategy is a "depth-first" design that performs well for "questions about a single entity" scenarios, but exposes insufficient coverage when queries involve multi-entity cross-relationships.

The most pragmatic improvement is **Approach 4 (Minimum Guarantee + Remaining Competition)** — it guarantees that every selected entity has at least some original text support with minimal code changes, without breaking the original strategy's advantages in mainstream scenarios.
