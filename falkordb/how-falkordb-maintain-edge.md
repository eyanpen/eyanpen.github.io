# How FalkorDB Stores Edges: Why Neighbor Lookup Is O(degree)

Many people have a question when they first see FalkorDB's architecture:

> It doesn't use traditional adjacency lists but maintains edges with sparse matrices — how does it efficiently find all edges of a given node?

And a follow-up question:

> If neighbor data is already stored contiguously, why is the query complexity still `O(degree)` instead of `O(1)`?


---

# 1. How Traditional Graph Databases Store Edges

Traditional graph databases (like Neo4j) typically use:

# Adjacency List

For example:

```text
A -> B
A -> C
A -> D
```

Internally it looks more like:

```text
A:
  edge1 -> edge2 -> edge3
```

That is:

* Each node maintains its own edge linked list
* To find all edges of a node:

  * Simply traverse the linked list

Therefore the complexity is:

```text
O(degree)
```

Where:

```text
degree = number of edges
```

For example:

* `out_degree`
  Number of outgoing edges

* `in_degree`
  Number of incoming edges

---

# 2. FalkorDB Is Completely Different: Sparse Matrix

FalkorDB's core design is not an adjacency list.

It is based on:

* Sparse Matrix
* GraphBLAS

to maintain the entire graph.

For example:

```text
A(id=0) -> B(id=1)
```

Internal representation:

```text
M[0,1] = edge_id
```

Meaning:

```text
source=0
target=1
```

An edge exists.

---

# 3. One Matrix Per Edge Type

For example:

```cypher
(:User)-[:FRIEND]->(:User)
(:User)-[:LIKES]->(:Post)
```

FalkorDB maintains:

```text
FRIEND matrix
LIKES matrix
```

This way during traversal:

No need to scan the entire graph.

---

# 4. How Multi-edges Are Maintained

FalkorDB supports:

```text
A -[:CALL]-> B
A -[:CALL]-> B
A -[:CALL]-> B
```

Therefore a matrix cell cannot simply be:

```text
M[0,1] = 1
```

It's more like:

```text
M[0,1] = [3,8,15]
```

That is:

```text
edge ids
```

Essentially similar to:

* sparse tensor
* compressed adjacency structure

---

# 5. How to Efficiently Find Edges?

Many people mistakenly think:

```text
0 0 0 1 0 0 1 1 0
```

Means:

> You must scan the entire row to find the 1s.

That's completely wrong.

Because:

# Sparse Matrix Doesn't Store Zeros At All

---

# 6. What Does a Sparse Matrix Actually Store?

For example:

```text
[0,0,0,1,0,0,1,1,0]
```

The actual storage looks more like:

```text
[3,6,7]
```

Meaning:

```text
index 3 has an edge
index 6 has an edge
index 7 has an edge
```

Zeros don't exist at all.

Therefore:

Finding neighbors of node A:

```text
neighbors(A) = [3,6,7]
```

Return directly.

---

# 7. CSR / CSC: Industrial-Grade Sparse Matrix Structures

Real implementations typically use:

* CSR (Compressed Sparse Row)
* CSC (Compressed Sparse Column)

For example:

Matrix:

```text
A: 0 0 0 1 0 0 1 1
B: 1 0 0 0 0 0 0 0
C: 0 1 0 0 1 0 0 0
```

CSR might store it as:

```text
indices = [3,6,7,0,1,4]
row_ptr = [0,3,4,6]
```

Explanation:

* A's data is at `indices[0:3]`
* B's data is at `indices[3:4]`
* C's data is at `indices[4:6]`

So:

Finding all edges of A:

```text
indices[0:3]
```

Gives us:

```text
[3,6,7]
```

---

# 8. Why Is the Complexity Still O(degree)?

This is the most commonly misunderstood point.

Many people ask:

> Since `[3,6,7]` is already in contiguous memory,
> isn't a direct memcpy just O(1)?

The answer:

# Locating the array is O(1)

But:

# Traversing the array is still O(k)

Where:

```text
k = degree
```

---

# 9. What Does Algorithmic Complexity Actually Measure?

For example:

```cypher
MATCH (a)-[e]->()
RETURN e
```

The database doesn't just return:

```text
an array pointer
```

It must:

* Traverse each edge
* Decode the edge object
* Construct the result set
* Return to the client

Therefore:

```text
for edge in neighbors:
    emit(edge)
```

Must execute:

```text
degree times
```

So the overall complexity is:

```text
O(degree)
```

---

# 10. Output-sensitive Complexity

This is a classic concept:

# The size of the output itself counts toward complexity

For example:

If:

```text
A has 1 million edges
```

Even if:

```text
finding the array start
```

Only takes:

```text
O(1)
```

But:

Returning 1 million edges:

Cannot be:

```text
O(1)
```

Because:

You must at least "look at" each element.

---

# 11. Why Is FalkorDB Still Fast?

Because:

```text
[3,6,7]
```

Is:

* Contiguous memory
* Cache-friendly
* SIMD-friendly

The CPU can:

* Prefetch
* Vector load
* Branch prediction

While traditional adjacency lists:

```text
edge1 -> edge2 -> edge3
```

Involve:

# Pointer chasing

Which causes:

* Cache misses
* Memory stalls
* Branch mispredictions

Therefore:

FalkorDB has clear advantages in:

* High fan-out traversal
* Multi-hop pattern matching
* Graph analytics
* GraphRAG

scenarios.

---

# 12. Neo4j vs FalkorDB: The Essential Difference

Neo4j is more like:

```text
nodes + edge linked lists
```

Suited for:

* OLTP
* Single-hop queries
* High-frequency edge updates

---

FalkorDB is more like:

```text
a graph computation engine
```

Suited for:

* Multi-hop traversal
* Pattern matching
* Graph analytics
* Vectorized computation

For example:

```cypher
(A)-[:F]->(B)-[:F]->(C)
```

Neo4j:

```text
pointer traversal
```

FalkorDB:

```text
matrix multiply
```

That is:

```text
F × F
```

This is its biggest architectural difference.

---

# 13. Final Summary

FalkorDB's core philosophy:

> Don't store "empty"
> Only store "existing edges"

Therefore:

```text
0 0 0 1 0 0 1
```

Actually becomes:

```text
[3,6]
```

Querying all edges of a node:

* Locating adjacency data:

  * `O(1)`

* Returning all edges:

  * `O(degree)`

Where:

```text
degree = number of edges for the current node
```

Not:

```text
total number of edges in the entire graph
```

This is the core performance model of a Sparse Matrix graph database.

---

# 14. Does Splitting Edges Into Multiple Types vs. a Single Type Affect Query Speed?

A common question:

> Since locating edges is O(1) and returning edges is O(degree),
> does categorizing edges into one type vs. multiple types affect query speed?

The answer: **It depends on whether the query specifies an edge type.**

---

## When the Query Specifies an Edge Type

For example:

```cypher
MATCH (a)-[:FRIEND]->(b) RETURN b
```

FalkorDB only scans the `FRIEND` matrix.

If all edges are categorized as a single type (e.g., `:REL`), the matrix contains all edges, making the degree larger.

Multiple types = smaller matrices = less traversal = **faster**.

---

## When the Query Does Not Specify an Edge Type

For example:

```cypher
MATCH (a)-[]->(b) RETURN b
```

FalkorDB needs to merge results from multiple matrices.

In this case:

* Total traversal volume is the same (total degree)
* Multiple types have slight merge overhead
* Single type traverses one matrix directly

The difference is minimal, approximately **no impact**.

---

## Summary

| Scenario | Single Type vs. Multiple Types | Impact |
|----------|-------------------------------|--------|
| Query specifies edge type | Multiple types faster | Only scans the corresponding matrix, smaller degree |
| Query does not specify edge type | Nearly no difference | Same total degree, slight merge overhead with multiple types |

Practical modeling recommendation:

> Splitting into multiple types is the better practice.
> Most real-world queries specify a relationship type, and splitting types significantly reduces the number of edges that need to be traversed.
