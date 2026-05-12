# Why Gold Answers Are Becoming Less Important in GraphRAG Systems

> Traditional RAG evaluation relies on human-annotated "standard answers," but in the GraphRAG era, this approach is losing its relevance.

## What Is a Gold Answer?

A Gold Answer is a human-annotated "standard correct answer." In traditional NLP and RAG system evaluation, the process typically goes like this:

1. Prepare a batch of test questions
2. Have humans write the "correct answer" for each question
3. Let the system answer the same questions
4. Compare system answers against Gold Answers, calculating F1, BLEU, ROUGE, and other scores

This approach has worked for years in search engines and simple Q&A systems. But in complex systems like GraphRAG, the value of Gold Answers is declining rapidly.

## Knowledge Graphs Evolve Continuously — Gold Answers Can't Keep Up

### The Core Problem

The heart of GraphRAG is the knowledge graph. Graphs aren't static — every document update, every re-extraction of entities and relationships changes the graph. Today's "correct answer" might be outdated tomorrow.

### Example

Suppose your company has an internal technical architecture document:

- **January version**: The document states "the order service uses MySQL"
- **March version**: After an architecture upgrade, it now reads "the order service uses PostgreSQL + Redis cache"

The Gold Answer you annotated in January is:

> Q: What database does the order service use?  
> A: MySQL

By March, the GraphRAG system has re-indexed the new documents and correctly answers "PostgreSQL + Redis." But if you still evaluate against the January Gold Answer, the system gets marked as "wrong."

### A More Realistic Scenario

In enterprise environments, document update frequency is much higher than most people imagine:

- API documentation changes weekly
- Organizational structures are adjusted quarterly
- Technology choices may be overhauled every six months

After each document update, you need to re-annotate Gold Answers. For an evaluation set with 500 test questions, each update might require modifying 30% of the answers — that means re-reviewing 150 answers every time.

## Human Annotation of Gold Answers Is Extremely Costly and Unreliable

### The Core Problem

The questions GraphRAG handles often involve multi-hop reasoning and cross-document correlation. For such questions, even human experts struggle to provide a single "uniquely correct" answer.

### Example

Suppose the question is:

> "Among the projects Zhang San is responsible for, which ones use EOL (End of Life) technology stacks?"

To answer this, annotators need to:

1. Find which projects Zhang San is responsible for (possibly scattered across 5 documents)
2. Find the technology stack for each project (yet more documents)
3. Determine which stacks are EOL (requires external information)
4. Synthesize all the above into an answer

Suppose the ground truth is that Zhang San is responsible for 4 projects, 3 of which use EOL tech stacks. After an hour of document review, the annotator writes this Gold Answer:

> Project A (Spring Boot 2.5), Project B (Log4j 1.x), Project C (Python 2.7)

But the annotator missed Project D — because Zhang San's responsibility for Project D was documented in meeting minutes, not in the official project assignment sheet.

Now look at the evaluation results:

| System | Answer | Score Against Gold Answer |
|--------|--------|--------------------------|
| Traditional RAG | Found Projects A, B (missed C) | Recall 2/3 = 0.67 |
| GraphRAG | Found Projects A, B, C, D (discovered D through relationship reasoning in meeting minutes) | Recall 3/3 = 1.0, but Precision 3/4 = 0.75 (D judged as "extraneous") |

The irony: GraphRAG gets penalized for being **more correct than the Gold Answer**. It discovered information through the graph's relationship chain (Zhang San → attended meeting → meeting resolution → responsible for Project D) that even the annotator missed, but in the evaluation framework, this "extra correct answer" is treated as an error.

Final F1 scores:
- Traditional RAG: F1 = 0.80
- GraphRAG: F1 = 0.86

GraphRAG clearly found more complete and accurate results, yet its score advantage is negligible — and in some evaluation settings (like strict exact matching), it might even score lower than traditional RAG. **The Gold Answer ceiling limits the ability to identify superior systems.**

### The Cost Calculation

Annotating a single complex GraphRAG test question might take a domain expert 30-60 minutes (requiring cross-referencing multiple documents). If you need 200 test questions, that's 100-200 hours of expert time. And these answers might only remain valid for a few months (see the first point above).

## GraphRAG Answers Are Inherently Diverse in Form

### The Core Problem

Traditional RAG typically answers factual questions ("What is X?"), where answers are relatively fixed. But GraphRAG excels at relationship reasoning and comprehensive analysis — questions where the "correct answer" naturally has multiple valid expressions.

### Example

Question:

> "In our microservices architecture, which services have circular dependencies?"

GraphRAG might answer:

> **Answer A**: Service A → Service B → Service C → Service A forms a cycle; Service D and Service E call each other.

> **Answer B**: Two groups of circular dependencies exist: (1) A-B-C triangular cycle (2) D-E bidirectional dependency. Recommend prioritizing decoupling the A-B-C cycle as it involves the core transaction path.

> **Answer C**: Circular dependency path detected: A→B→C→A. Additionally, D↔E has bidirectional calls, but since they use asynchronous messaging, the actual impact is minimal.

All three answers are "correct," but with different emphases. Using any single one as the Gold Answer would unfairly penalize other equally correct responses.

### Traditional Metrics Fail

Comparing the three answers above using ROUGE scores:

- Answer A vs Answer B: ROUGE-L might be only 0.3 (completely different wording)
- Answer A vs Answer C: ROUGE-L might be 0.5 (some overlap)

But from an information correctness perspective, all three should receive full marks. The Gold Answer + text similarity metric combination completely fails here.

## LLM-as-Judge Is Replacing Gold Answers

### The Core Problem

Given all these issues with Gold Answers, the industry is shifting toward a new evaluation paradigm: using LLMs as judges (LLM-as-Judge), directly evaluating answer quality rather than comparing against "standard answers."

### Example

Traditional approach:

```
System answer: "PostgreSQL + Redis"
Gold Answer: "MySQL"
ROUGE score: 0.0  → Judged as incorrect ❌
```

LLM-as-Judge approach:

```
Question: "What database does the order service use?"
System answer: "PostgreSQL + Redis"
Reference document: [Latest architecture doc, clearly states PostgreSQL + Redis]

LLM judgment: Answer is consistent with documentation, information is accurate, score 5/5 ✅
```

Advantages of LLM-as-Judge:

| Dimension | Gold Answer | LLM-as-Judge |
|-----------|-------------|--------------|
| Requires human annotation | Extensive manual work | Not needed |
| Adapts to document updates | Requires re-annotation | Automatically adapts (references latest docs) |
| Handles multiple valid expressions | Cannot | Can (understands semantic equivalence) |
| Evaluation cost | High (manual) | Low (API calls) |
| Evaluation speed | Slow (days/weeks) | Fast (minutes) |

## GraphRAG Evaluation Dimensions Far Exceed "Answer Correctness"

### The Core Problem

Gold Answers can only evaluate one dimension: whether the answer content is correct. But GraphRAG system quality depends on many other factors that Gold Answers simply cannot measure.

### Example

For the same question, two GraphRAG systems both give correct answers, but the quality differs dramatically:

**System A's response**:
> Zhang San is responsible for Project X, which uses Spring Boot 2.5 (EOL).

**System B's response**:
> Zhang San is responsible for Project X, which uses Spring Boot 2.5 (maintenance ended November 2023). Additionally, the project depends on Log4j 1.x (EOL since 2015, with known security vulnerability CVE-2019-17571). Recommend referring to the internal migration guide [link] for upgrading.

Both answers might score identically against the Gold Answer, but System B is clearly more valuable — it provides more complete information, security risk alerts, and actionable recommendations.

### The Dimensions That Actually Matter

For GraphRAG systems, we should focus on:

- **Graph coverage**: Are entities and relationships being completely extracted?
- **Reasoning path explainability**: Which nodes and edges did the system traverse to reach its conclusion?
- **Information completeness**: Are important related details being missed?
- **Timeliness**: Is the referenced information current?
- **Actionability**: Does the answer provide executable recommendations?

None of these dimensions can be evaluated by Gold Answers.

## How Should We Evaluate GraphRAG Then?

Since Gold Answers are no longer a silver bullet, here are evaluation strategies better suited for GraphRAG:

1. **LLM-as-Judge + dimension decomposition**: Have LLMs score separately on accuracy, completeness, relevance, and other dimensions
2. **Source document fact-checking**: Verify whether each fact in the answer can be traced back to source documents
3. **Graph quality metrics**: Directly evaluate knowledge graph entity coverage and relationship accuracy
4. **End-to-end user satisfaction**: Have real users evaluate whether answers solved their problems
5. **Regression testing over absolute scoring**: Focus on quality changes before and after system updates, rather than pursuing absolute scores

## Final Thoughts

Gold Answers aren't entirely worthless — for simple factual Q&A and system cold-start phases, they remain a useful baseline. But in complex systems like GraphRAG, over-reliance on Gold Answers introduces three risks:

1. **False sense of security**: High Gold Answer scores don't mean the system is actually useful
2. **Maintenance burden**: The cost of continuously updating Gold Answers may exceed the value they provide
3. **Evaluation blind spots**: Gold Answers cannot cover GraphRAG's most important quality dimensions

Rather than spending enormous effort maintaining a set of "standard answers" destined to become outdated, invest that energy into more modern, comprehensive evaluation systems. GraphRAG evaluation should be like GraphRAG itself — dynamic, multi-dimensional, and based on understanding rather than rote memorization.
