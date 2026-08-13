# Focused Test Plan: Agentic RAG Retrieval Loop

**Status:** Binding companion test plan
**Scope:** Agentic retrieval execution only
**Relationship:** Extends the architecture and testing plan already defined. It does not introduce a second state, memory, context, or orchestration model.

---

## 1. Purpose

This document defines how we test the **agentic RAG execution path**:

```text
User Request
    ↓
Query Interpretation
    ↓
Retrieval Objective
    ↓
Query Construction / Rewrite
    ↓
Candidate Retrieval
    ↓
Evidence Evaluation
    ↓
    ├── sufficient → Evidence Set
    │
    └── insufficient → Refine → Retrieve
    ↓
Synthesis
```

The test plan focuses on the behavior of this loop while remaining consistent with the existing architecture:

* `M_t` remains persistent memory.
* `G_t / H_t` remains session state.
* `W_t` remains execution-local working state.
* Stages obtain context through the centralized resolver.
* Stages do not implement independent memory retrieval.
* Persistent mutation remains exclusively governed by the write policy.
* The baseline workflow remains deterministic and bounded.
* LLMs and vector databases are introduced only in tests that specifically require them.

The purpose is to establish whether the retrieval loop behaves correctly **before** we optimize it or make it more autonomous.

---

# 2. Architectural Boundary

The retrieval subsystem sits inside the broader execution architecture.

```text
                    ┌─────────────────────────┐
                    │      User Request       │
                    └────────────┬────────────┘
                                 ↓
                    ┌─────────────────────────┐
                    │ Query Interpretation    │
                    │        Stage            │
                    └────────────┬────────────┘
                                 ↓
                    ┌─────────────────────────┐
                    │ Retrieval Workflow      │
                    │                         │
                    │  Rewrite                │
                    │     ↓                   │
                    │  Retrieve               │
                    │     ↓                   │
                    │  Evaluate               │
                    │     ↓                   │
                    │  Branch                 │
                    │   ↙     ↘               │
                    │ accept   refine         │
                    │   ↓       │             │
                    │ synthesize←             │
                    └────────────┬────────────┘
                                 ↓
                    ┌─────────────────────────┐
                    │      Synthesis Stage    │
                    └────────────┬────────────┘
                                 ↓
                    ┌─────────────────────────┐
                    │   Memory Evaluation     │
                    │    / Write Policy       │
                    └─────────────────────────┘
```

The retrieval workflow **uses** the state/context architecture. It does not redefine it.

---

# 3. What Counts as a Retrieval Stage

A retrieval operation becomes a cognitive stage when an LLM exercises meaningful discretion.

Examples:

| Operation                  |        Stage? | Reason                         |
| -------------------------- | ------------: | ------------------------------ |
| Query string normalization |            No | Deterministic transform        |
| Embedding generation       |            No | Infrastructure operation       |
| Vector search              |            No | Retrieval mechanism            |
| Keyword search             |            No | Retrieval mechanism            |
| Result merging             |    Usually no | Deterministic operation        |
| Query interpretation       |           Yes | Determines retrieval objective |
| Query rewriting            |           Yes | Chooses reformulation          |
| Evidence evaluation        |           Yes | Determines sufficiency         |
| Conflict analysis          |           Yes | Requires judgment              |
| Answer synthesis           |           Yes | Produces final reasoning       |
| JSON validation            |            No | Deterministic validation       |
| Loop termination           | No, initially | Policy/workflow decision       |
| Dynamic stage selection    | No, initially | Deferred complexity            |

The existing architectural test remains applicable:

> Two operations should be separate stages when different context materially changes their success criteria or behavior.

---

# 4. Retrieval Execution State

The retrieval loop uses `W_t`.

```text
W_t
└── RetrievalExecutionState
    ├── originalQuery
    ├── interpretedObjective
    ├── currentQuery
    ├── iteration
    ├── retrievalAttempts
    ├── candidateEvidence
    ├── selectedEvidence
    ├── assessments
    ├── unresolvedEvidence
    ├── failedQueries
    └── termination
```

This state is **execution-local**.

A retrieval failure therefore produces:

```text
failedQueries += Q
```

rather than:

```text
M_t += "Q failed"
```

unless the write policy later determines that the failure represents durable knowledge.

That distinction must remain enforced by the existing persistence boundary.

---

# 5. Core Interfaces Under Test

The retrieval test suite should test against interfaces rather than concrete implementations.

## 5.1 Query Interpreter

```text
QueryInterpreter.interpret(
    request,
    context
) → RetrievalObjective
```

Expected output:

```text
RetrievalObjective
├── intent
├── entities
├── constraints
├── requiredEvidence
└── ambiguities
```

The implementation may initially use an LLM.

The tests should not require one.

---

## 5.2 Query Rewriter

```text
QueryRewriter.rewrite(
    objective,
    previousQuery,
    assessment,
    context
) → RetrievalQuery
```

The rewrite operation should be capable of using:

```text
current query
+
missing evidence
+
failed retrieval attempts
+
retrieval objective
```

It should not independently access persistent memory.

The resolver supplies its permitted context.

---

## 5.3 Candidate Retriever

```text
Retriever.retrieve(
    query
) → RetrievalResult
```

This interface intentionally hides the retrieval technology.

The implementation may be:

```text
MockRetriever
KeywordRetriever
BM25Retriever
VectorRetriever
HybridRetriever
```

The agentic loop should not care.

---

## 5.4 Evidence Evaluator

```text
EvidenceEvaluator.evaluate(
    objective,
    evidence,
    context
) → RetrievalAssessment
```

Example:

```text
RetrievalAssessment
├── sufficient
├── quality
├── completeness
├── consistency
├── missingEvidence
├── contradictions
└── recommendation
```

Possible recommendation:

```text
ACCEPT
REFINE
RETRIEVE_MORE
TERMINATE
```

The initial implementation should keep this bounded.

---

## 5.5 Retrieval Workflow

```text
RetrievalWorkflow.execute(
    objective,
    executionState
) → EvidenceSet
```

The workflow owns:

* iteration count
* stage ordering
* conditional branches
* termination
* retry limits

It does **not** own:

* persistent memory
* context construction
* embedding infrastructure
* memory writes

---

# 6. The Most Important Test Boundary

The retrieval loop must be testable with a completely deterministic fake environment.

For example:

```text
FakeRetriever:
    "Postgres scalability"
        → [document A, document B]

FakeEvaluator:
    A + B
        → insufficient
        → missing: "benchmark data"

FakeRewriter:
    missing "benchmark data"
        → "Postgres scalability benchmark results"

FakeRetriever:
    "Postgres scalability benchmark results"
        → [document C, document D]

FakeEvaluator:
    C + D
        → sufficient
```

The entire loop can then be tested without:

* an LLM
* embeddings
* a vector database
* network access

This is the **T0/T1 foundation** for the subsystem.

---

# 7. Test Levels

The previous testing architecture should be retained.

## T0 — Pure unit tests

No external infrastructure.

Test:

* query objects
* retrieval state
* assessments
* policies
* termination rules
* branch conditions
* context contracts
* provenance records
* deterministic ranking
* evidence aggregation

These should be the majority of the retrieval test suite.

---

## T1 — Component tests with mocks

Still no real LLM or vector database.

Use:

```text
MockQueryInterpreter
MockQueryRewriter
MockRetriever
MockEvaluator
MockContextResolver
```

Test interactions between components.

Examples:

```text
Interpreter → Workflow
Rewriter → Workflow
Retriever → Evaluator
Evaluator → Branch
Resolver → Stage
```

---

## T2 — Infrastructure integration

Introduce actual retrieval infrastructure.

Examples:

```text
BM25
Vector DB
Hybrid retrieval
Embedding model
Reranker
```

The purpose is to establish:

```text
query → candidate generation → ranking → evidence
```

correctness.

The agentic control loop remains deterministic.

---

## T3 — LLM stage tests

Introduce the actual model for cognitive stages:

```text
QueryInterpreter
QueryRewriter
EvidenceEvaluator
Synthesis
```

Hold retrieval infrastructure fixed.

This isolates the contribution of LLM reasoning.

---

## T4 — End-to-end agentic RAG

Use:

```text
real LLM
+
real retrieval
+
real context resolver
+
real memory
+
real workflow
```

This tests the complete system.

---

# 8. Core Test Groups

## A. Query Interpretation

### A1 — Intent extraction

Given:

```text
"Compare PostgreSQL and MongoDB for a system with heavy relational queries."
```

verify that the interpreter identifies the relevant retrieval objective.

### A2 — Constraint preservation

Input constraints must survive interpretation.

For example:

```text
"using PostgreSQL 16"
```

must not disappear from the retrieval objective.

### A3 — Ambiguity preservation

The interpreter should represent unresolved ambiguity rather than silently inventing an interpretation.

---

# 9. Query Rewrite Tests

## Q1 — No rewrite when evidence is sufficient

```text
assessment.sufficient = true
```

must not trigger another rewrite.

## Q2 — Rewrite from missing evidence

```text
missingEvidence = ["benchmark results"]
```

must influence the next query.

## Q3 — Avoid repeating failed queries

Given:

```text
failedQueries = [Q1]
```

the refinement policy should not produce the same query indefinitely.

## Q4 — Bounded rewriting

A rewrite cycle must eventually terminate.

```text
Q1 → Q2 → Q3 → ... → maxIterations
```

must produce a defined terminal state.

---

# 10. Retrieval Tests

## R1 — Retriever isolation

The workflow depends only on:

```text
Retriever.retrieve(query)
```

and does not depend on vector-database implementation details.

## R2 — Empty results

```text
retrieve(Q) → []
```

must produce a defined assessment.

## R3 — Duplicate evidence

Repeated retrievals must not silently inflate the evidence set.

## R4 — Conflicting evidence

The evidence evaluator must receive contradictory documents as distinct evidence rather than having the retrieval layer silently discard one.

## R5 — Provenance preservation

Every retrieved item must retain its source identity.

---

# 11. Evidence Evaluation Tests

This is a major part of the agentic loop.

## E1 — Sufficient evidence

```text
evidence → sufficient
```

must terminate retrieval.

## E2 — Insufficient evidence

```text
evidence → insufficient
```

must produce actionable missing evidence.

## E3 — Unsupported conclusion

The evaluator should distinguish:

```text
"I found no evidence"
```

from:

```text
"The evidence contradicts the claim"
```

These produce different retrieval behavior.

## E4 — Contradiction

Given:

```text
Source A → X
Source B → not-X
```

the assessment should expose the contradiction.

The workflow should not automatically choose one source merely because it has a higher similarity score.

---

# 12. Loop-Control Tests

The retrieval loop needs especially strong deterministic tests.

### L1 — Accept path

```text
Interpret
→ Retrieve
→ Evaluate(sufficient)
→ Finish
```

### L2 — Refinement path

```text
Interpret
→ Retrieve
→ Evaluate(insufficient)
→ Rewrite
→ Retrieve
→ Evaluate(sufficient)
→ Finish
```

### L3 — Repeated failure

```text
Retrieve
→ insufficient
→ Rewrite
→ insufficient
→ Rewrite
→ ...
```

must terminate according to the configured bound.

### L4 — No-progress detection

If:

```text
Q1 → Q2
```

produces effectively identical retrieval behavior, the system should recognize the lack of progress rather than consume the entire retry budget blindly.

### L5 — Retrieval failure

Infrastructure failure must produce a distinct state from epistemic insufficiency.

```text
RETRIEVAL_ERROR
```

is different from:

```text
INSUFFICIENT_EVIDENCE
```

That distinction matters for retries and provenance.

---

# 13. Context Contract Tests

Every LLM stage in the retrieval loop must use the existing resolver.

Example:

```text
QueryRewriter
    ↓
ContextContract
    ↓
ContextResolver
    ↓
ContextBundle
    ↓
LLM
```

Test:

### C1 — Required context

Missing required information prevents invocation.

### C2 — Optional context

Optional information is included according to the resolver policy.

### C3 — Closed-world access

Undeclared state does not enter the context.

### C4 — Read/write separation

The stage can read permitted persistent memory while possessing no mutation authority.

### C5 — Context provenance

Every included context item identifies its origin and transformation path.

These tests belong here because the retrieval loop is one of the principal consumers of the context architecture.

---

# 14. Provenance Tests

The system should produce an execution trace resembling:

```text
Execution
│
├── QueryInterpretation
│     └── objective O1
│
├── RetrievalIteration 1
│     ├── query Q1
│     ├── candidates C1...Cn
│     ├── selected evidence E1...Em
│     └── assessment A1
│
├── QueryRewrite
│     ├── input A1
│     └── query Q2
│
├── RetrievalIteration 2
│     ├── query Q2
│     ├── evidence E2
│     └── assessment A2
│
└── Synthesis
```

The test suite should verify causal links:

```text
Q2
  ← rewrite policy
  ← A1
  ← missingEvidence(E1)
```

and:

```text
context item X
  ← ContextResolver
  ← ContextContract
  ← state source
```

---

# 15. Memory Boundary Tests

Agentic retrieval creates many observations.

The test suite must establish that these observations remain in `W_t` unless promoted.

For example:

```text
RetrievedDocument
FailedQuery
IntermediateHypothesis
RetrievalAssessment
```

should remain execution state.

A test must verify:

```text
execution terminates
        ↓
W_t discarded
        ↓
M_t unchanged
```

unless:

```text
WriteProposal
        ↓
WritePolicy
        ↓
ADD / UPDATE / DELETE
```

occurs.

This prevents the retrieval loop from becoming an accidental memory writer.

---

# 16. Synthesis Tests

Synthesis should consume the **accepted evidence set**, rather than reaching into the retrieval infrastructure itself.

Test:

```text
EvidenceSet → Synthesis
```

and verify:

* unsupported claims are detectable
* source attribution survives
* conflicting evidence remains visible
* irrelevant retrieval candidates do not leak into the synthesis context

The synthesis stage should use its own `ContextContract`.

That contract can differ from the evaluator's.

---

# 17. Evaluation Metrics

The first implementation should measure the retrieval loop before optimizing it.

### Retrieval quality

```text
Recall
Precision
MRR / nDCG where applicable
Evidence completeness
```

### Agentic behavior

```text
First-pass sufficiency
Average iterations
Rewrite rate
Useful rewrite rate
No-progress rate
Premature termination rate
Unnecessary retrieval rate
```

### Cost

```text
retrieval calls / task
LLM calls / task
tokens / stage
total tokens
latency
```

### Epistemic behavior

```text
unsupported synthesis rate
contradiction detection
source attribution
missing-evidence identification
```

### Memory effects

```text
persistent writes / execution
false promotion rate
useful promotion rate
```

---

# 18. Required Baselines

We should not evaluate the agentic loop in isolation.

At minimum:

### B0 — Single retrieval

```text
Query
→ Retrieve
→ Synthesize
```

### B1 — Fixed multi-retrieval

```text
Query
→ Retrieve
→ Retrieve more
→ Synthesize
```

### B2 — Agentic retrieval

```text
Query
→ Retrieve
→ Evaluate
→ Rewrite
→ Retrieve
→ ...
→ Synthesize
```

This establishes whether the evaluator/rewrite loop provides value beyond simply retrieving more information.

Later:

### B3 — Agentic retrieval + stage-conditioned context

This tests the interaction between the retrieval architecture and the broader context architecture.

---

# 19. What We Deliberately Do Not Test Yet

Consistent with **O8 — Controlled Complexity**, this test plan does not assume:

* learned query rewriting
* learned retrieval policy
* RL memory management
* autonomous planner
* dynamic stage selection
* unrestricted tool selection
* joint optimization of read/action/write
* autonomous loop-length selection

Those can become later experimental conditions.

The baseline must first establish whether the deterministic architecture has identifiable failure modes.

---

# 20. The Key Experimental Question

The most important comparison eventually becomes:

```text
                Same model
                   │
          Same retrieval corpus
                   │
          Same token allowance
                   │
       ┌───────────┴───────────┐
       │                       │
 Single-pass RAG       Agentic RAG
       │                       │
       ▼                       ▼
    Answer                  Answer
```

Then:

```text
Does the additional
Retrieve → Evaluate → Refine
loop produce enough improvement
to justify its cost?
```

After that:

```text
Agentic RAG
      ↓
+ stage-conditioned context
      ↓
+ governed memory
```

can be evaluated incrementally.

That sequencing matters. Otherwise an improvement in the final system becomes impossible to attribute.

---

# 21. Binding Test Principle

The retrieval subsystem should therefore obey the same principle as the broader architecture:

> **Every additional cognitive operation must earn its complexity through an observable improvement in system behavior.**

The first implementation consequently establishes:

```text
deterministic workflow
+
bounded retrieval loop
+
explicit evaluation
+
explicit refinement
+
centralized context resolution
+
execution-local retrieval state
+
provenance
+
controlled persistence
```

Only after those pieces are measurable should we ask whether a learned policy, planner, additional LLM call, or more sophisticated retrieval strategy improves the system.

That keeps this document **additive to the existing test plan rather than contradictory to it**: the previous plan establishes the infrastructure and architectural invariants; this plan exercises those invariants through the concrete **agentic RAG execution loop**.
