# A FalkorDB Vector Search Gotcha: Why Won't db.idx.vector.queryNodes Work?

When using FalkorDB (a Redis-protocol-compatible graph database) for GraphRAG or semantic search, we often want to tap into its built-in native vector search capability, namely this API:

```cypher
CALL db.idx.vector.queryNodes('Entity', 'embedding', 10, vecf32($query_vec))
```

The dream is beautiful: a single Cypher statement fetches "the 10 nodes most similar to the query vector," backed by efficient Approximate Nearest Neighbor (ANN) search.

But many people find, on their first attempt, that it either throws an error, returns empty results, or degrades into an absurdly slow full scan. The data is clearly written in — so why won't it work?

In this article we'll spell out the **two necessary conditions** for `db.idx.vector.queryNodes` to work properly, then break down a few of the easiest traps to fall into.

---

# 1. The Conclusion First: Both Conditions Are Required

For native vector search to actually take effect, two things must be true at the same time:

1. The embedding data is stored as a **native vector type** (a vector converted through a function like `vecf32()`).
2. A **vector index** has been created on the corresponding property.

These two are an "AND" relationship, not an "OR." Miss either one, and `db.idx.vector.queryNodes` won't behave the way we expect.

Here's an analogy:

- Condition one (the vector type) is like "the content of the book really is arranged in alphabetical order."
- Condition two (the vector index) is like "the book has an alphabetical table of contents up front."

Only when the content itself is ordered *and* there's an index can we flip to the index and locate things quickly. If the content isn't actually ordered alphabetically, the index is a lie; if it's ordered but there's no index, we still have to flip through page by page. Miss either one, and "fast lookup" is off the table.

Let's walk through both conditions in detail, and why neither can be skipped.

---

# 2. Condition One: The Data Must Be a Native Vector Type

There's a crucial but easily overlooked distinction in FalkorDB: **"a string of numbers" and "a vector" are completely different things at the storage level.**

## What Actually Counts as a Vector Type

When writing, we must use `vecf32()` to explicitly convert the array into a vector type:

```cypher
CREATE (:Entity {name: 'Alice', embedding: vecf32([0.1, 0.2, 0.3, 0.4])})
```

Note the `vecf32(...)` here. It converts a plain array into FalkorDB's internal 32-bit floating-point vector type. Only after this step is the property a "real vector" that the vector index and ANN search recognize.

## Pitfall One: The embedding Is a Plain List, Not a Vector Type

This is the most common trap. A lot of write code looks like this:

```python
# Anti-pattern: write the 4096-dim array straight in
graph.query(
    "MATCH (n:entities {id: $id}) SET n.embedding = $vec",
    {"id": doc_id, "vec": embedding_list},  # embedding_list is list[float]
)
```

`embedding_list` is a 4096-dimensional Python `list`. Once it's passed in through Redis / Cypher, FalkorDB stores it as a **native List type**.

The problem is:

- The List looks like it holds all the floats fine, and functionally "there's no error";
- But the vector index **will not include List-type properties**;
- So `db.idx.vector.queryNodes` either returns empty, or fails to find the target node because there's no entry for it in the index.

**The correct approach** is to wrap it in `vecf32()` inside the Cypher:

```python
# Correct
graph.query(
    "MATCH (n:entities {id: $id}) SET n.embedding = vecf32($vec)",
    {"id": doc_id, "vec": embedding_list},
)
```

> Quick check: use `RETURN typeof(n.embedding)` to inspect the property type. If it returns something other than a vector type — an array type instead — then we've fallen into this trap.

## Pitfall Two: The embedding Is a String, Not a Vector Type

The second common problem: the vector gets serialized into a **string** before being stored. This happens especially easily during cross-system transfer or JSON serialization:

```python
# Anti-pattern: JSON-serialize the vector into a string for storage
import json
graph.query(
    "MATCH (n:entities {id: $id}) SET n.embedding = $vec",
    {"id": doc_id, "vec": json.dumps(embedding_list)},  # becomes "[0.1, 0.2, ...]"
)
```

At this point `n.embedding` is a `string` whose content is `"[0.1, 0.2, ...]"`.

The consequences are similar to pitfall one, but even more insidious:

- A string simply cannot be recognized by the vector index;
- If later code needs to read the vector back for manual similarity computation, it has to `json.loads()` and deserialize first — an extra layer of overhead;
- Worse still, once some data is a string and some is a vector, the problem becomes very hard to diagnose.

**The root cause** is usually this: the data got JSON-serialized somewhere along the way (passing through some API, a caching layer, or a misconfigured ORM mapping), and by the time it's written to the database, the deserialization + `vecf32()` was forgotten.

**The correct approach** is to ensure that what's passed into Cypher is the raw float array, and to convert it with `vecf32()`:

```python
# Correct: make sure it's an array first, then vecf32()
vec = json.loads(raw) if isinstance(raw, str) else raw
graph.query(
    "MATCH (n:entities {id: $id}) SET n.embedding = vecf32($vec)",
    {"id": doc_id, "vec": vec},
)
```

## How to Confirm You Stored It Correctly

The key to telling real from fake is to look at the **type**, not the **appearance**. We can use Cypher to print out the property's type and confirm:

```cypher
MATCH (n:Entity {name: 'Alice'})
RETURN n.embedding, typeof(n.embedding)
```

If the returned type is `Vectorf32`, it's stored correctly; if it's `Array` (List) or `String`, then we've fallen into one of the traps above.

Here's a point worth emphasizing: **a plain List and a vector print out almost identically** — both look like `[0.1, 0.2, ...]`. So eyeballing the data won't fool anyone but ourselves; we have to look at the type. A lot of people spend ages troubleshooting with no clue precisely because they keep staring at the "value" instead of checking the "type."

---

# 3. Condition Two: A Vector Index Must Be Created on the Property

Suppose we've already stored the embedding correctly as a vector type. Can we query now? Not yet. We still need to explicitly create a vector index on this property:

```cypher
CREATE VECTOR INDEX FOR (n:Entity) ON (n.embedding)
OPTIONS {dimension: 4096, similarityFunction: 'cosine'}
```

A few parameters here deserve special attention:

- `dimension`: it must match the dimension of the vectors we actually write in **exactly**. If our model outputs 4096 dimensions, this has to be 4096. If the dimension doesn't match, the index either fails to build or fails to match at query time.
- `similarityFunction`: the similarity function, commonly `cosine` or `euclidean` (Euclidean distance). This has to be consistent with the semantics we use at retrieval time — if the embedding was trained for cosine similarity, we should use `cosine`.

## Why It Seems to "Work" Without an Index — but Is Useless

There's a phenomenon here that's especially easy to misjudge: even without a vector index, some query styles **won't throw an error outright**, and may even return results. This can trick us into thinking "everything's fine."

But the truth is: without a vector index, this native ANN entry point `db.idx.vector.queryNodes` simply can't be used; even if we switch to some other method (like manually computing distances and sorting) to scrape by, it goes through a **full linear scan** — pulling out every node's vector, computing the distance for each, then sorting to take the Top-K.

On a toy dataset of a few hundred nodes, this full scan doesn't feel slow. But once the data grows to hundreds of thousands or millions of nodes, every query having to traverse all vectors makes latency explode. The ANN advantage we were counting on — "approximate nearest neighbor, sublinear complexity" — is nowhere to be enjoyed.

So "returns results" and "vector search is working" are two different things. The real sign it's working is that `db.idx.vector.queryNodes` can go through the index and enjoy the ANN speedup.

---

# 4. Stringing the Two Conditions Together: One Complete, Correct Flow

Let's walk through the entire correct pipeline end to end, for easy cross-checking:

Step one, create the index (you can create it first, or after the data is written):

```cypher
CREATE VECTOR INDEX FOR (n:Entity) ON (n.embedding)
OPTIONS {dimension: 4096, similarityFunction: 'cosine'}
```

Step two, use `vecf32()` to convert to a vector type when writing data:

```cypher
CREATE (:Entity {name: 'Alice', embedding: vecf32($vec_4096)})
```

Step three, use the native API to search:

```cypher
CALL db.idx.vector.queryNodes('Entity', 'embedding', 10, vecf32($query_vec))
YIELD node, score
RETURN node.name, score
ORDER BY score
```

Note that the query vector itself must also be wrapped in `vecf32()` — the type on the query side and the storage side must line up.

As long as all three steps are right, we get to enjoy true native ANN search.

---

# 5. A Troubleshooting Checklist: When queryNodes Won't Work

If search misbehaves, we can go through the items below in order, which will pinpoint the vast majority of cases:

1. **Check the type, not the value.** Use `typeof(n.embedding)` to confirm whether the property is `Vectorf32`. If it's `Array` or `String`, that means `vecf32()` wasn't used on write, or the data got serialized into something else during import.
2. **Confirm the index really was created.** Use `db.indexes` or the corresponding command to list all indexes, and check whether there really is a vector index on the target property.
3. **Verify the dimension.** The index's declared `dimension` must match the dimension of the vectors actually written. A 4096-dim vector paired with a 1536-dim index definitely won't match.
4. **Verify the similarity function.** The retrieval semantics must be consistent with `similarityFunction` — don't do cosine search against a Euclidean-distance index.
5. **Confirm the query vector was converted too.** The vector passed in on the query side must also go through `vecf32()`.

Of these five steps, step 1 is the most frequent trap. Because a plain List, a string, and a vector print out almost identically, only looking at the type can pierce the disguise.

---

# 6. Summary

For FalkorDB's native vector search `db.idx.vector.queryNodes` to work, it comes down to two necessary conditions, neither of which can be skipped:

- The data is a **true vector type** (converted through `vecf32()`), not a plain List or string that merely looks like a vector.
- A **vector index** is built on the property, with dimension and similarity function both matching up.

The easiest place to trip up is the illusion that "the data looks fine": List, string, and vector print out nearly indistinguishably, so when we troubleshoot we must always **look at the type, not the value**. Also remember that "the query returns results" doesn't equal "the vector index is working" — only ANN search that goes through the index can truly run fast at scale.

Keep these two conditions and these few pitfalls firmly in mind, and we'll dodge a lot of traps when doing vector search on FalkorDB.

---

If you found this article helpful, please **like, bookmark, and follow**. I'll keep sharing more valuable content. Your support is my greatest motivation to keep creating!
