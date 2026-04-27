# Analysis of Alias Limitations in GraphRAG Entity Extraction

## 1. Problem Overview

During entity extraction, GraphRAG treats different aliases of the same entity as **independent entities**, causing entity fragmentation in the knowledge graph. Using "Sun Wukong" (the Monkey King) as an example:

```
Text A: "Sun Wukong wreaked havoc in Heaven"       → Entity: Sun Wukong
Text B: "Sun Xingzhe defeated the White Bone Demon" → Entity: Sun Xingzhe
Text C: "The Great Sage was trapped under Five Elements Mountain" → Entity: Great Sage
```

The resulting graph contains three independent nodes with **no connections between them**. When querying "What did Sun Wukong do?", only "wreaked havoc in Heaven" is found, while the relationship chains for "defeated the White Bone Demon" and "trapped under Five Elements Mountain" are completely broken.

---

## 2. How GraphRAG Currently Handles This

### 2.1 Entity Extraction Phase

**Source**: `packages/graphrag/graphrag/index/operations/extract_graph/graph_extractor.py`

The LLM extracts entities from each text unit. Core logic:

```python
# graph_extractor.py → _process_result()
if record_type == '"entity"' and len(record_attributes) >= 4:
    entity_name = clean_str(record_attributes[1].upper())  # Uppercase normalization
    entity_type = clean_str(record_attributes[2].upper())
    entity_description = clean_str(record_attributes[3])
    entities.append({
        "title": entity_name,
        "type": entity_type,
        "description": entity_description,
        "source_id": source_id,
    })
```

Key points:
- Entity names only undergo `clean_str()` + `upper()` processing (removing HTML escapes and control characters, converting to uppercase)
- **No alias recognition or normalization logic whatsoever**
- Whatever name the LLM extracts is recorded as-is

### 2.2 Entity Merging Phase

**Source**: `packages/graphrag/graphrag/index/operations/extract_graph/extract_graph.py`

Entities extracted from multiple text units are merged via `_merge_entities()`:

```python
def _merge_entities(entity_dfs) -> pd.DataFrame:
    all_entities = pd.concat(entity_dfs, ignore_index=True)
    return (
        all_entities
        .groupby(["title", "type"], sort=False)  # ← Groups only by title + type
        .agg(
            description=("description", list),
            text_unit_ids=("source_id", list),
            frequency=("source_id", "count"),
        )
        .reset_index()
    )
```

The merging strategy uses **exact string matching**: only entities with identical `title` (name) and `type` are merged.

This means:
- `SUN WUKONG` and `SUN XINGZHE` → **not merged** (different titles)
- `SUN WUKONG` and `MONKEY KING` → **not merged**
- `TechGlobal` and `TG` → **not merged** (abbreviation vs. full name)

### 2.3 Description Summarization Phase

**Source**: `packages/graphrag/graphrag/index/operations/summarize_descriptions/`

After merging, multiple descriptions for the same entity (same title) are summarized into one by the LLM:

```python
# description_summary_extractor.py
async def __call__(self, id, descriptions):
    if len(descriptions) == 0:
        result = ""
    elif len(descriptions) == 1:
        result = descriptions[0]  # Single description, use directly
    else:
        result = await self._summarize_descriptions(id, descriptions)  # Multiple descriptions, LLM summarization
```

The summarization prompt:

```
Given one or more entities, and a list of descriptions, all related to the same entity
or group of entities. Please concatenate all of these into a single, comprehensive
description. If the provided descriptions are contradictory, please resolve the
contradictions and provide a single, coherent summary.
```

**Problem**: This summarization step only processes descriptions that have already been merged by `_merge_entities()`. Since alias entities are never merged, their descriptions will never be summarized together.

### 2.4 Cascading Relationship Breakage

**Source**: `extract_graph.py → _merge_relationships()`

```python
def _merge_relationships(relationship_dfs) -> pd.DataFrame:
    all_relationships = pd.concat(relationship_dfs, ignore_index=False)
    return (
        all_relationships
        .groupby(["source", "target"], sort=False)  # ← Exact match on source + target
        .agg(
            description=("description", list),
            text_unit_ids=("source_id", list),
            weight=("weight", "sum"),
        )
        .reset_index()
    )
```

Relationship merging also relies on exact string matching. Suppose:

```
Text A extracts: (Sun Wukong) --master-disciple--> (Tang Seng)
Text B extracts: (Sun Xingzhe) --master-disciple--> (Tang Seng)
```

These two relationships **will not be merged** because the sources differ. The final graph contains:
- `Sun Wukong → Tang Seng` (weight=1)
- `Sun Xingzhe → Tang Seng` (weight=1)

The correct result should be a single relationship with weight=2. More critically, if certain relationships only appear in alias contexts, querying by the primary name will miss them entirely.

### 2.5 Impact on Query Phase

**Source**: `packages/graphrag/graphrag/query/context_builder/entity_extraction.py`

Queries match entities via embedding vector similarity:

```python
def map_query_to_entities(query, text_embedding_vectorstore, text_embedder, ...):
    search_results = text_embedding_vectorstore.similarity_search_by_text(
        text=query,
        text_embedder=lambda t: text_embedder.embedding(input=[t]).first_embedding,
        k=k * oversample_scaler,
    )
```

When querying "Sun Wukong", embedding similarity may match the "Sun Wukong" node, but the "Sun Xingzhe" and "Great Sage" nodes may have distant embeddings and fall outside the top-k results. Even if matched, as independent nodes, their respective relationship subgraphs remain fragmented.

---

## 3. Root Cause Summary

| Stage | Current Behavior | Problem |
|-------|-----------------|---------|
| LLM Extraction | Extracts names as they appear in text | Different aliases produce different entity titles |
| Entity Merging | `groupby(["title", "type"])` exact match | Alias entities cannot be merged |
| Description Summarization | Only summarizes descriptions of merged entities | Alias entity descriptions remain permanently separated |
| Relationship Merging | `groupby(["source", "target"])` exact match | Aliases cause relationship fragmentation |
| Query Matching | Embedding similarity search | Alias nodes may not appear in top-k |
| Prompt | No alias recognition instructions | LLM is not guided to unify aliases |

---

## 4. Solution: LLM Alias Discovery + External Knowledge Base Deterministic Merging

Overall approach: **Two layers of protection**. The LLM discovers alias relationships during extraction, covering most cases; an external knowledge base provides deterministic fallback for entities of particular concern, ensuring critical entities are not missed due to LLM inconsistency.

### 4.1 Overall Flow

```
                    ┌──────────────────────────┐
                    │  External Alias KB        │
                    │  (JSON/DB, manually maintained) │
                    └───────────┬──────────────┘
                                │ Load
                                ▼
text units ──→ LLM extraction(with aliases) ──→ Alias normalization ──→ _merge_entities ──→ Downstream
                                                  ▲
                                                  │
                               ┌──────────────────┴──────────────────┐
                               │ 1. External KB takes priority        │
                               │ 2. LLM aliases supplement            │
                               └─────────────────────────────────────┘
```

### 4.2 External Alias Knowledge Base

Users maintain an alias mapping file defining canonical names and all known aliases for entities of particular concern:

```json
// alias_kb.json
[
  {
    "canonical": "Sun Wukong",
    "aliases": ["Sun Xingzhe", "Great Sage Equal to Heaven", "Handsome Monkey King", "Victorious Fighting Buddha"]
  },
  {
    "canonical": "Zhu Bajie",
    "aliases": ["Marshal Tianpeng", "Zhu Wuneng", "Zhu Ganglie", "Idiot", "Second Brother"]
  }
]
```

Characteristics:
- **Deterministic**: Mappings in the knowledge base are hard rules, independent of LLM judgment, guaranteeing 100% merge
- **Controllable**: Only needs to cover entities of business importance, no need to enumerate all entities
- **Incrementally maintainable**: When a missed merge is discovered, simply add a record

### 4.3 Adding an Aliases Field to LLM Extraction

Modify the extraction prompt (`prompts/index/extract_graph.py`) to have the LLM output aliases alongside entities:

```
1. Identify all entities. For each identified entity, extract the following information:
- entity_name: Name of the entity, capitalized
- entity_type: One of the following types: [{entity_types}]
- entity_description: Comprehensive description of the entity's attributes and activities
- aliases: Other names, abbreviations, or titles for this entity found in the text.
  If none, leave empty.
Format each entity as ("entity"<|><entity_name><|><entity_type><|><entity_description><|><aliases>)
```

Parse aliases in `graph_extractor.py`'s `_process_result()`:

```python
if record_type == '"entity"' and len(record_attributes) >= 4:
    entity_name = clean_str(record_attributes[1].upper())
    entity_type = clean_str(record_attributes[2].upper())
    entity_description = clean_str(record_attributes[3])
    aliases = []
    if len(record_attributes) >= 5:
        aliases = [clean_str(a.upper()) for a in record_attributes[4].split(",") if a.strip()]
    entities.append({
        "title": entity_name,
        "type": entity_type,
        "description": entity_description,
        "source_id": source_id,
        "aliases": aliases,
    })
```

LLM-discovered aliases cover long-tail cases not in the external knowledge base. For example, if text mentions "Monkey Bro" referring to Sun Wukong, the knowledge base may not include it, but the LLM can recognize and output `aliases: Monkey Bro`.

### 4.4 Alias Normalization: Two-Layer Merging

Before `_merge_entities()`, build a unified alias → canonical mapping with external knowledge base taking priority:

```python
def _build_alias_map(entity_dfs, alias_kb_path=None):
    """Build alias → canonical mapping. External KB takes priority, LLM aliases supplement."""
    alias_to_canonical = {}

    # Layer 1: External knowledge base (deterministic, highest priority)
    if alias_kb_path:
        import json
        with open(alias_kb_path) as f:
            kb_entries = json.load(f)
        for entry in kb_entries:
            canonical = entry["canonical"].upper()
            for alias in entry["aliases"]:
                alias_to_canonical[alias.upper()] = canonical

    # Layer 2: LLM-extracted aliases (supplement what KB doesn't cover)
    all_entities = pd.concat(entity_dfs, ignore_index=True)
    name_freq = all_entities["title"].value_counts()

    for _, row in all_entities.iterrows():
        title = row["title"]
        for alias in row.get("aliases", []):
            if not alias or alias == title:
                continue
            # Don't override existing KB mappings
            if alias in alias_to_canonical or title in alias_to_canonical:
                continue
            # LLM aliases: higher frequency name becomes canonical
            if name_freq.get(alias, 0) > name_freq.get(title, 0):
                alias_to_canonical[title] = alias
            else:
                alias_to_canonical[alias] = title

    # Transitive closure: A→B, B→C then A→C
    def resolve(name):
        visited = set()
        while name in alias_to_canonical and name not in visited:
            visited.add(name)
            name = alias_to_canonical[name]
        return name

    return resolve
```

In `extract_graph()`, uniformly rewrite names in entities and relationships before merging:

```python
async def extract_graph(...) -> tuple[pd.DataFrame, pd.DataFrame]:
    # ... LLM extraction ...
    results = await derive_from_rows(...)

    entity_dfs = [r[0] for r in results if r]
    relationship_dfs = [r[1] for r in results if r]

    # Alias normalization (new)
    resolve = _build_alias_map(entity_dfs, alias_kb_path=config.alias_kb_path)
    for df in entity_dfs:
        df["title"] = df["title"].map(resolve)
    for df in relationship_dfs:
        df["source"] = df["source"].map(resolve)
        df["target"] = df["target"].map(resolve)

    # Original merging logic (now correctly merges alias entities)
    entities = _merge_entities(entity_dfs)
    relationships = _merge_relationships(relationship_dfs)
    relationships = filter_orphan_relationships(relationships, entities)

    return (entities, relationships)
```

### 4.5 Comparison of Results

Using "Sun Wukong" as an example:

| Scenario | Current Behavior | After Improvement |
|----------|-----------------|-------------------|
| Text A mentions "Sun Wukong", Text B mentions "Sun Xingzhe" | Two independent nodes, broken relationships | External KB match, unified as "Sun Wukong" |
| Text C mentions "Monkey Bro" (not in KB) | Independent node | LLM aliases discovery, normalized to "Sun Wukong" |
| Text D mentions "Marshal Tianpeng" (in KB) | Independent node | External KB match, unified as "Zhu Bajie" |
| Text E mentions an obscure abbreviation (not covered by either layer) | Independent node | Still independent; add to KB when discovered |

---

## 6. Source File Index

| File | Purpose |
|------|---------|
| `prompts/index/extract_graph.py` | Entity extraction prompt definition |
| `index/operations/extract_graph/graph_extractor.py` | LLM extraction result parsing, entity/relationship construction |
| `index/operations/extract_graph/extract_graph.py` | Entity/relationship merging logic (`_merge_entities`, `_merge_relationships`) |
| `index/operations/extract_graph/utils.py` | Orphan relationship filtering |
| `index/operations/summarize_descriptions/` | Description summarization (only processes merged entities) |
| `index/workflows/extract_graph.py` | Extraction workflow orchestration |
| `index/workflows/finalize_graph.py` | Graph finalization (degree calculation, deduplication) |
| `index/operations/finalize_entities.py` | Entity finalization (deduplication by title) |
| `query/context_builder/entity_extraction.py` | Entity matching during queries |
| `index/utils/string.py` | `clean_str()` string cleaning |
| `prompt_tune/template/extract_graph.py` | Tunable extraction prompt template |
