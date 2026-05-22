# Known Pitfall in DeepEval Faithfulness Metric: "idk" Verdicts Don't Penalize the Score

## Background

While using DeepEval to evaluate a GraphRAG system in a no-reference setting, we discovered that `FaithfulnessMetric` can produce misleading perfect scores under certain conditions.

## Observed Behavior

We asked GraphRAG a complex question about the 5GC PDU Session establishment procedure. The system returned a detailed technical answer (covering specific responsibilities of AMF, SMF, UPF, PCF, etc.), but the retrieved context contained only the **table of contents** from 3GPP documents, such as:

```
The document contains a section '5.6 Session Management' with several sub-subsections.
The document contains a section '5.2 Network Access Control' with several sub-subsections.
```

The context contained no substantive technical content, yet the Faithfulness score was **1.00 (perfect)**.

## Root Cause Analysis

The Faithfulness metric evaluation consists of 4 steps:

| Step | Purpose |
|------|---------|
| 1. Truths extraction | Extract factual statements from retrieval_context |
| 2. Claims extraction | Extract claims from actual_output |
| 3. Verdicts | Compare each claim against context, assign `yes`/`no`/`idk` |
| 4. Score calculation | Compute final score from verdicts |

The key lies in Step 3's verdict rules:

- **`yes`** — claim is consistent with context
- **`no`** — claim **directly contradicts** context
- **`idk`** — context contains no relevant information to judge

And Step 4's **default scoring formula**:

```python
score = (total - no_count) / total
```

**`idk` does not count as a penalty.** Only explicit contradictions (`no`) reduce the score.

## Real-World Example

In our evaluation, the LLM judge (after switching to a stricter model) assigned `idk` to all 20 claims:

```json
{
  "verdicts": [
    {"verdict": "idk"},
    {"verdict": "idk"},
    ...  // 20 total, all idk
  ]
}
```

Score calculation: `score = (20 - 0) / 20 = 1.00`

The final reason output:
> "The score is 1.00 because there are no contradictions; the actual output fully aligns with the retrieval context."

This is clearly misleading — none of the claims in the answer are **supported by the context**, but since none are "contradicted" either, the score is perfect.

## The Fundamental Issue

Faithfulness measures **"is there a contradiction with the context"**, not **"is the answer supported by the context"**.

These are entirely different dimensions:

| Scenario | Faithfulness | Groundedness |
|----------|-------------|-------------|
| Answer fully based on context | High | High |
| Answer correct but context irrelevant | High (no contradiction) | Low (no support) |
| Answer contradicts context | Low | Low |

When retrieval context contains only table-of-contents or summary-level information, it's nearly impossible for any specific claim to "directly contradict" it, so Faithfulness will always be perfect.

## Solutions

### Solution 1: Enable `penalize_ambiguous_claims`

DeepEval provides a built-in parameter:

```python
FaithfulnessMetric(model=model, threshold=0.5, penalize_ambiguous_claims=True)
```

With this enabled, the scoring formula becomes:

```python
score = (total - no_count - idk_count) / total
```

Now 20 claims all judged `idk` yields: `(20 - 0 - 20) / 20 = 0.00`, which more accurately reflects how well the context supports the answer.

### Solution 2: Add a Groundedness Metric

Use GEval to define a custom Groundedness metric that directly evaluates whether the answer is supported by context:

```python
GEval(
    name="Groundedness",
    criteria="Determine whether the actual output is fully supported and grounded by the retrieval context. "
             "Penalize claims in the output that cannot be traced back to specific information in the retrieval context.",
    evaluation_params=[SingleTurnParams.INPUT, SingleTurnParams.ACTUAL_OUTPUT, SingleTurnParams.RETRIEVAL_CONTEXT],
    model=model,
    threshold=0.5,
)
```

### Recommendation

Use both solutions together:
- Keep Faithfulness (with `penalize_ambiguous_claims` enabled) to detect contradictions and unsupported claims
- Add Groundedness to positively evaluate support coverage
- Note Faithfulness limitations in reports to avoid misinterpretation

## Additional Pitfall: Summary Claims Misjudged as "idk"

Even when the context contains specific detailed information, if the actual output summarizes those details, the judge may still assign `idk`.

### Real-World Example

The context contained specific procedural details about PDU Session establishment (AMF handling registration, SMF selecting UPF, N4 session setup, etc.), while the actual output included a summary claim:

> "From the UE attempting to access a specific DNN to achieving effective user plane forwarding, the entire process involves close cooperation among multiple core network elements, each playing an indispensable role."

The judge's verdict:

```json
{
  "verdict": "idk",
  "reason": "The claim is a summary statement; the context provides specific procedural details but does not directly confirm this overall description."
}
```

### Cause

The Faithfulness prompt imposes strict constraints on the judge:

> "Only use 'no' if retrieval context DIRECTLY CONTRADICTS the claim — never use prior knowledge."  
> "Use 'idk' for claims not backed up by context — do not assume your knowledge."

The judge is required to perform **literal-level matching**, not **semantic-level reasoning**. Even though the context details fully support the summary through logical inference, since the context doesn't "directly confirm" the statement, the judge can only assign `idk`.

### Impact

For RAG systems, answers are expected to synthesize and summarize context — this is normal and desired behavior. However, Faithfulness's literal-level matching treats such reasonable summaries as "unsupported," causing scores to drop when `penalize_ambiguous_claims` is enabled.

### Possible Improvement

DeepEval's `FaithfulnessMetric` supports an `evaluation_template` parameter. You can inherit from `FaithfulnessTemplate` and modify the verdict guidelines to include "summaries that can be reasonably inferred from context details" in the `yes` category. However, this changes the semantics of the evaluation criteria and should be used cautiously.

## Conclusion

The Faithfulness metric was designed to detect hallucination — whether the model fabricates information that contradicts the context. However, it has limitations on two levels:

1. **"idk" doesn't penalize by default** — always perfect when context is irrelevant (solved with `penalize_ambiguous_claims=True`)
2. **Literal-level matching is too strict** — reasonable summaries are judged as unsupported (requires custom templates or supplementary Groundedness metrics)

When evaluating RAG systems, both Faithfulness and Groundedness dimensions must be considered to comprehensively assess answer quality.
