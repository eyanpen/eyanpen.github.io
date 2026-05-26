# The "Ghost Clone" of Community Reports in GraphRAG: Why the Same Report Gets Created Twice

## Symptom

When querying the Top 10 nodes by `HAS_REPORT` edge count in FalkorDB, we found 4 community_report nodes each with **4** `HAS_REPORT` edges pointing to them. By design, each community should map to exactly one report — so why the one-to-many relationship?

```
Edge type: HAS_REPORT
Rank  Title                                                          Count
1     Tech Dept Core Team: Backend Architecture & System Design        4
2     Product Dept: User Growth & Monetization Strategy                4
3     Ops Dept: Service Stability & Monitoring System                  4
4     QA Dept: Quality Assurance & Test Automation                     4
```

In theory each community has one report, each report belongs to one community, and `HAS_REPORT` should be a 1:1 relationship.

---

## An Intuitive Example

### Imagine You're Managing a Company's Org Chart

Suppose your company has this department structure:

```
Tech Dept (278 people)
  └── Backend Team (253 people)
```

"Backend Team" is a sub-department of "Tech Dept." Now HR needs to write a **department brief** for each.

HR discovers that the core members of "Backend Team" heavily overlap with "Tech Dept" (the backend team IS the main force of the tech department), so the AI generates **nearly identical briefs** for both:

| Department | Brief Title | Headcount |
|---|---|---|
| Tech Dept (community 1491) | "Core Tech Team: Backend Architecture & System Design" | 278 |
| Backend Team (community 2790) | "Core Tech Team: Backend Architecture & System Design" | 253 |

The two briefs have **identical titles and content** (because they essentially describe the same group of people), differing only in "headcount" (size).

Because the content is identical, the system computes the **same ID** for both (content-based hash).

Mapping to the 4 actual problem groups we found:

| Dept Analogy | Actual community | Brief Title | Size |
|---|---|---|---|
| Tech Dept | community 1491 | "Tech Dept Core Team: Backend Architecture & System Design" | 278 |
| └── Backend Team | community 2790 | "Tech Dept Core Team: Backend Architecture & System Design" | 253 |
| Product Dept | community 200 | "Product Dept: User Growth & Monetization Strategy" | 796 |
| └── Product Team 1 | community 1100 | "Product Dept: User Growth & Monetization Strategy" | 631 |
| Ops Dept | community 1909 | "Ops Dept: Service Stability & Monitoring System" | 180 |
| └── Ops Team 1 | community 3073 | "Ops Dept: Service Stability & Monitoring System" | 178 |
| QA Dept | community 953 | "QA Dept: Quality Assurance & Test Automation" | 21 |
| └── QA Team 1 | community 2343 | "QA Dept: Quality Assurance & Test Automation" | 19 |

### Where's the Problem?

When importing this data into the graph database:

**Step 1: Create report nodes**

Taking "Tech Dept" and "Backend Team" as an example. The system sees two rows in the parquet with the same ID but different communities, and blindly creates **two nodes**:

```
Report Node A: {id: "abc123", community: 1491, size: 278}  -- Tech Dept's brief
Report Node B: {id: "abc123", community: 2790, size: 253}  -- Backend Team's brief
```

**Step 2: Create HAS_REPORT edges**

The system iterates over each report record and matches report nodes by `id`:

```cypher
-- Processing Tech Dept (community 1491)
MATCH (c:communities {community: 1491})
MATCH (r:community_reports {id: "abc123"})  -- Matches 2 nodes (A and B)!
CREATE (c)-[:HAS_REPORT]->(r)
-- Result: Tech Dept → Node A, Tech Dept → Node B (2 edges)

-- Processing Backend Team (community 2790)
MATCH (c:communities {community: 2790})
MATCH (r:community_reports {id: "abc123"})  -- Also matches 2 nodes!
CREATE (c)-[:HAS_REPORT]->(r)
-- Result: Backend Team → Node A, Backend Team → Node B (2 edges)
```

**Final result**: This report title has 4 HAS_REPORT edges (2 departments × 2 same-ID nodes = 4).

The correct result should be: Tech Dept → Tech Dept's brief (1 edge), Backend Team → Backend Team's brief (1 edge), totaling 2 edges.

---

## Root Cause Analysis

The problem is caused by two factors compounding:

### 1. Leiden Hierarchical Clustering Produces Identical Reports

GraphRAG uses the Leiden algorithm for hierarchical community detection. When a sub-community's members heavily overlap with its parent community, the LLM generates nearly identical reports for both. Since report IDs are content-based hashes, identical content → identical IDs.

Actual data verification:

| report id | communities | sizes | Hierarchy |
|---|---|---|---|
| 6516e2f4... | 2790, 1491 | 253, 278 | 2790 is a sub-community of 1491 |
| feda9fa0... | 1100, 200 | 631, 796 | 1100 is a sub-community of 200 |
| d8f25d09... | 2343, 953 | 19, 21 | 2343 is a sub-community of 953 |
| 223c76c6... | 3073, 1909 | 178, 180 | 3073 is a sub-community of 1909 |

### 2. Import Logic Lacks Deduplication and Precise Matching

In the import code:

```python
# Node creation: unconditional CREATE, no deduplication
"UNWIND $batch AS p CREATE (n:community_reports) SET n = p"

# Edge creation: matches only by id, no community condition
"MATCH (r:community_reports {id: p.rid})"  # Matches multiple same-ID nodes → Cartesian product
```

---

## Solution

### Precise Matching When Creating HAS_REPORT

When creating `HAS_REPORT` edges, match on both `id` and `community` to avoid the Cartesian product:

```python
# Before (buggy)
"MATCH (r:community_reports {id: p.rid}) "

# After (fixed)
"MATCH (r:community_reports {id: p.rid, community: p.cnum}) "
```

This way each community only matches the report node that belongs to it, creating exactly 1 edge.


**Lesson learned**: When using the `MATCH` + `CREATE` pattern to create relationships in a graph database, if the match condition isn't precise enough (target nodes have duplicates), you'll get unexpected Cartesian products. Always ensure `MATCH` conditions can uniquely locate the target node.
