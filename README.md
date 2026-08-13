# A Governed Cognitive Architecture for Retrieval-Augmented Agents

## State, Context, Epistemic Policy, and Controlled Memory

### Abstract

Retrieval-augmented generation has evolved from a relatively simple retrieval-and-generation pattern into systems containing persistent memory, multi-stage reasoning, tool use, corrective retrieval, context management, and autonomous state mutation. This evolution creates an architectural problem that is poorly captured by the conventional distinction between "RAG" and "agent."

The central problem is **governance of information and reasoning across an agent's execution lifecycle**.

A capable agent must determine:

1. what state exists and how long it should live;
2. what information a cognitive stage may see;
3. why that information was selected;
4. how the stage should interpret evidence;
5. what action follows from that interpretation;
6. what transient observations deserve persistence;
7. how existing memories should be corrected, superseded, or discarded;
8. and how the entire chain can remain inspectable.

This paper proposes an architecture organized around **state isolation, stage-conditioned context, centralized context resolution, explicit mutation authority, controlled persistence, replaceable policies, provenance, and epistemic governance**.

The architecture deliberately begins with deterministic mechanisms and a fixed workflow with bounded conditional branching. Learned planners, learned memory policies, and joint policy optimization remain replaceable research extensions rather than architectural prerequisites.

A further hypothesis is introduced: LLM performance may benefit from an explicit **epistemic constitution** governing the distinction between observation, evidence, hypothesis, inference, uncertainty, belief, and action. Spinoza's *Treatise on the Emendation of the Intellect* is proposed as one candidate source for such a constitution because the work explicitly develops a method concerned with modes of perception, distinguishing true, fictitious, and false ideas, doubt, memory, clear and distinct ideas, definitions, and the ordered acquisition of understanding. The Treatise is unfinished, so its use here is an engineering interpretation rather than a claim that Spinoza provides an AI architecture. ([Project Gutenberg][1])

The paper therefore defines an architecture **without assuming that the proposed policies are optimal**. It identifies known constraints, competing approaches, unresolved questions, and an experimental program capable of falsifying the central design assumptions.

---

# 1. Problem Definition

Conventional RAG can be represented approximately as:

```text
User Query
    │
    ▼
Retriever
    │
    ▼
Documents
    │
    ▼
LLM
    │
    ▼
Answer
```

This abstraction becomes inadequate once the system acquires:

```text
conversation history
persistent memory
working hypotheses
multiple cognitive stages
tool outputs
retrieval attempts
correction loops
memory mutation
```

The system instead becomes:

```text
                         ┌───────────────┐
                         │ Persistent M  │
                         └───────┬───────┘
                                 │
Conversation ──► Session State ──┼──► Working State
                                 │
                                 ▼
                         Context Resolution
                                 │
                     ┌───────────┼───────────┐
                     ▼           ▼           ▼
                 Stage A      Stage B      Stage C
                     │           │           │
                     └───────────┼───────────┘
                                 ▼
                              Decision
                                 │
                         ┌───────┴────────┐
                         ▼                ▼
                       Action          Observation
                                          │
                                          ▼
                                    Write Proposal
                                          │
                                          ▼
                                     Write Policy
                                          │
                                          ▼
                                     Persistent M
```

The architectural problem is therefore no longer merely retrieval.

It is:

> **How should information, reasoning, action, and persistent state be governed across a multi-stage LLM execution?**

---

# 2. Design Objectives

## O1 — State isolation

Different classes of state must have explicit lifetime and mutation semantics.

The architecture should distinguish durable memory, session state, and execution-local working state.

## O2 — Stage-specific context

Each cognitive operation should receive context appropriate to its objective.

A retrieval judge, query interpreter, memory manager, and answer synthesizer should not receive identical context merely because they participate in the same execution.

## O3 — Centralized context resolution

Stages should declare information requirements rather than implement independent retrieval mechanisms.

The architecture should have one context-resolution mechanism:

```text
R(stage, contract, S_t) → C_t^stage
```

## O4 — Explicit mutation authority

The ability to read persistent memory must not imply the ability to modify it.

Read authority and mutation authority are separate architectural concerns.

## O5 — Controlled persistence

Transient observations should become persistent memory only through an explicit promotion mechanism.

## O6 — Replaceable policies

Deterministic mechanisms should be replaceable with learned policies without restructuring the state architecture.

## O7 — Debuggability and provenance

The system should expose enough intermediate state to explain:

* why something was retrieved;
* why it was presented;
* why a stage acted;
* why an observation became memory;
* what evidence supported a conclusion;
* what transformed one representation into another.

Recent work on agent provenance independently identifies evidence tracing across retrieval, memory, tool outputs, intermediate claims, actions, and final answers as an emerging requirement for trustworthy agent systems. ([arXiv][2])

## O8 — Controlled complexity

The architecture should not introduce a planner, learned router, memory-RL system, or additional LLM invocation until an observed failure justifies it.

## O9 — Epistemic coherence

The system should provide an explicit framework governing how stages distinguish:

```text
observation
evidence
interpretation
hypothesis
inference
belief
uncertainty
contradiction
decision
```

The purpose is not philosophical ornamentation. It is to establish a stable method for interpreting information and producing traceable decisions.

---

# 3. Agent State

The state model uses two independent dimensions:

1. **lifetime**;
2. **mutation authority**.

This avoids creating an unnecessarily elaborate memory taxonomy.

### 3.1 Persistent state — Mₜ

```text
Mₜ
```

Cross-session information that the system deliberately preserves.

Examples:

```text
validated facts
durable user/project information
validated procedures
learned operational experience
established relationships
```

Mutation requires the write policy.

---

### 3.2 Session state — Gₜ / Hₜ

Current conversational and objective state.

Examples:

```text
current goal
subgoals
conversation history
current interpretation
active constraints
session decisions
```

Selected cognitive stages may mutate this state.

---

### 3.3 Working state — Wₜ

Execution-local information.

Examples:

```text
retrieval candidates
failed searches
temporary hypotheses
intermediate tool results
candidate interpretations
unconfirmed observations
```

The default lifecycle is:

```text
execution begins
      ↓
Wₜ created
      ↓
stages operate
      ↓
execution terminates
      ↓
Wₜ discarded
```

Promotion requires an explicit write decision:

```text
Wₜ
 ↓
Write Proposal
 ↓
Epistemic Assessment
 ↓
Write Policy
 ↓
Mₜ
```

This creates an explicit persistence boundary.

---

# 4. What Constitutes a Stage?

A stage is not merely a pipeline function.

A **stage is a point at which an LLM exercises meaningful discretion under a distinct success criterion**.

Deterministic operations should remain code.

For example:

```text
JSON parsing       → code
schema validation  → code
sorting            → code
deduplication      → code
```

Whereas:

```text
query interpretation
retrieval sufficiency judgment
hypothesis generation
evidence assessment
answer synthesis
memory evaluation
```

may constitute cognitive stages.

A useful operational criterion is:

> Two adjacent operations constitute separate stages when different context materially changes their performance or when their success criteria can be independently optimized.

This provides a mechanism for preventing uncontrolled proliferation of LLM stages.

---

# 5. Context Contracts

Each cognitive stage declares a **Context Contract**.

```text
ContextContract
{
    required
    optional
    tokenBudget
}
```

The architecture uses a closed-world interpretation.

Anything not declared is inaccessible.

```text
required ∪ optional = permitted context
everything else = denied
```

This is stronger than an exclusion list because newly introduced state does not automatically become visible to existing stages.

For example:

```text
RetrievalJudgeContract

required:
    current_query
    retrieved_documents
    retrieval_scores

optional:
    previous_attempts
    relevant_memory
    query_interpretation

tokenBudget:
    6000
```

A memory-management stage could receive:

```text
MemoryManagerContract

required:
    write_proposal
    evidence
    provenance

optional:
    related_memories
    historical_conflicts

tokenBudget:
    4000
```

The two stages therefore operate over the same system state while receiving materially different projections of that state.

---

# 6. Context Resolution

Context assembly becomes a centralized operation:

```text
R(stage, contract, Sₜ) → Cₜˢ
```

The resolver:

1. obtains required information;
2. retrieves optional candidates;
3. scores candidates against stage utility;
4. fits candidates to the token budget;
5. attaches provenance;
6. produces the stage context.

The resolver is pure:

```text
read state
    ↓
compute projection
    ↓
return context
```

It does not mutate memory.

This separation is deliberate:

```text
READ
R(stage, contract, Sₜ)
       │
       ▼
    context

WRITE
proposal
       │
       ▼
write policy
       │
       ▼
persistent state
```

Current agent-context research increasingly treats relevance, sufficiency, isolation, economy, and provenance as separate properties of context rather than treating context as an undifferentiated prompt payload. ([arXiv][3])

---

# 7. Context Selection

The initial resolver should use deterministic selection.

A candidate context item can be evaluated through a function such as:

```text
U(x | stage, objective, state)
```

with signals such as:

```text
semantic relevance
task relevance
recency
source reliability
epistemic status
dependency relevance
historical usefulness
token cost
```

Then:

```text
required → always included
optional → ranked under budget
undeclared → inaccessible
```

A learned resolver can later replace the deterministic scorer:

```text
Deterministic Resolver
        ↓
      interface
        ↓
Learned Resolver
```

without changing the state model or stage contracts.

---

# 8. Workflow Control

The initial architecture uses:

### Fixed workflow + bounded conditional branching

For example:

```text
Interpret
   ↓
Retrieve
   ↓
Judge
   │
   ├── sufficient ───────► Generate
   │
   └── insufficient ────► Refine → Retrieve
```

The system does not initially give an unconstrained planner authority to select arbitrary stages.

This has several advantages:

```text
predictability
debuggability
bounded execution
stable provenance
controlled cost
```

A planner can later replace the fixed controller without changing:

```text
State
ContextContract
ContextResolver
Stage interface
Write interface
```

This preserves O6.

---

# 9. Epistemic Governance

This is the principal extension of the architecture.

The architecture separates:

```text
Information governance
```

from:

```text
Epistemic governance
```

Information governance asks:

```text
What may this stage see?
What may it remember?
What may it modify?
```

Epistemic governance asks:

```text
What does this information constitute?
How strong is the evidence?
What follows from it?
What remains uncertain?
When should a conclusion be revised?
```

---

# 10. The Epistemic Constitution

The term **epistemic constitution** refers to a stable set of instructions governing how the model interprets information.

A constitution might establish:

```text
Observation ≠ inference

Retrieved information ≠ established truth

Hypothesis ≠ belief

Confidence must track evidence

Contradictions remain explicit until resolved

Uncertainty must survive into downstream reasoning

Persistent memory requires stronger justification
than transient working hypotheses

Claims should retain their evidential provenance

New evidence may require revision of prior conclusions
```

This can be implemented initially as system-level instructions.

It could later become a structured policy object.

---

# 11. Why Spinoza Is a Candidate

Spinoza's *Treatise on the Emendation of the Intellect* is unusually relevant because it is itself a methodological project concerning the improvement of understanding.

The work explicitly discusses:

```text
modes of perception
true / fictitious / false ideas
doubt
memory and forgetfulness
clear and distinct ideas
definitions
the method of understanding
```

and distinguishes the method from reasoning itself: the method concerns the route by which understanding acquires and orders ideas. ([Project Gutenberg][1])

The work is also unfinished, so it should not be treated as a complete computational specification. Cambridge's recent overview explicitly characterizes the Treatise as an early, unfinished text and places it within Spinoza's broader methodological development. ([Cambridge University Press][4])

A particularly relevant feature is the relationship between **adequate ideas** and the ordered construction of understanding. Scholarship on the Treatise emphasizes its concern with definitions, the organization of ideas, and the way understanding can reproduce the structure of what it investigates. ([Revistas USC][5])

That provides a possible basis for an agent epistemic policy.

The proposal is therefore:

> **Do not encode "Spinozism" as the agent's personality. Translate selected epistemic principles from the Treatise into operational constraints and test whether they improve agent behavior.**

---

# 12. From Spinozist Method to Agent Operations

A preliminary translation might look like this:

| Epistemic concern                          | Agent operation                                                               |
| ------------------------------------------ | ----------------------------------------------------------------------------- |
| Mode of perception                         | classify provenance of information                                            |
| Distinguishing true/fictitious/false ideas | classify claim status                                                         |
| Doubt                                      | preserve unresolved uncertainty                                               |
| Memory                                     | distinguish remembered proposition from current evidence                      |
| Clear/distinct ideas                       | require explicit representation of important propositions                     |
| Definition                                 | establish semantic contracts before reasoning                                 |
| Ordered understanding                      | maintain dependency/provenance chains                                         |
| Adequacy                                   | evaluate whether the available representation sufficiently explains the claim |

This table is an **engineering hypothesis**, not a claim that Spinoza himself proposed these computational operations.

That distinction should remain explicit throughout the research.

---

# 13. Epistemic State

The architecture can therefore represent a proposition as more than text:

```text
Proposition
 ├── content
 ├── epistemicType
 ├── evidence
 ├── provenance
 ├── confidence
 ├── temporalValidity
 ├── derivedFrom
 ├── contradicts
 └── supersedes
```

For example:

```json
{
  "content": "The retrieval index fails after prolonged inactivity.",
  "epistemicType": "hypothesis",
  "confidence": 0.61,
  "evidence": [
    "execution-184",
    "execution-201"
  ],
  "status": "unconfirmed"
}
```

Later:

```text
new evidence
     ↓
re-evaluation
     ↓
hypothesis strengthened
     ↓
validated experience
```

or:

```text
new evidence
     ↓
contradiction
     ↓
hypothesis weakened
     ↓
superseded / discarded
```

This gives memory a **lifecycle of understanding**, rather than a CRUD lifecycle alone.

---

# 14. Memory Write Pipeline

The write path becomes:

```text
Stage output
     │
     ▼
Observation / Claim extraction
     │
     ▼
Epistemic assessment
     │
     ├── observation
     ├── hypothesis
     ├── inference
     ├── established knowledge
     └── contradiction
     │
     ▼
Write Proposal
     │
     ▼
Write Policy
     │
     ├── ADD
     ├── UPDATE
     ├── SUPERSEDE
     ├── DELETE
     └── NOOP
     │
     ▼
Mₜ
```

The key architectural invariant becomes:

> **A stage may produce information without thereby acquiring authority to make that information persistent.**

---

# 15. Provenance

The provenance graph spans the complete execution:

```text
Source
  ↓
Observation
  ↓
Candidate generation
  ↓
Retrieval
  ↓
Retrieval score
  ↓
Context selection
  ↓
ContextContract
  ↓
Stage invocation
  ↓
Interpretation
  ↓
Decision
  ↓
Action
  ↓
Observation
  ↓
Write proposal
  ↓
Write policy
  ↓
Persistent mutation
```

Each significant artifact should carry an identifier.

For example:

```text
context-item-431
retrieved-from: memory-91
retrieval-method: hybrid
score: 0.82
selected-for: RetrievalJudge
contract: judge-v3
reason: high decision utility
```

Then:

```text
memory-91
      ↓
context-item-431
      ↓
judge-decision-77
      ↓
write-proposal-14
      ↓
memory-update-204
```

This makes the system inspectable at **decision boundaries**, rather than merely observable at API boundaries.

The need for this kind of evidence/execution lineage is independently emerging in current agent research. ([arXiv][2])

---

# 16. Complete Architecture

```text
                         ┌───────────────────────┐
                         │     PERSISTENT Mₜ     │
                         │ facts / procedures /  │
                         │ validated experience  │
                         └───────────┬───────────┘
                                     │
                                     │
                         ┌───────────▼───────────┐
                         │     SESSION Gₜ/Hₜ     │
                         │ goal / history /      │
                         │ constraints / plan    │
                         └───────────┬───────────┘
                                     │
                                     │
                         ┌───────────▼───────────┐
                         │       WORKING Wₜ      │
                         │ hypotheses / tools /  │
                         │ retrieval attempts    │
                         └───────────┬───────────┘
                                     │
                                     ▼
                    ┌──────────────────────────────┐
                    │      CONTEXT RESOLVER R      │
                    │                              │
                    │ stage + contract + state     │
                    │          ↓                   │
                    │ stage-specific context       │
                    └──────────────┬───────────────┘
                                   │
                                   ▼
                       ┌────────────────────────┐
                       │    EPISTEMIC POLICY    │
                       │                        │
                       │ evidence               │
                       │ uncertainty            │
                       │ inference              │
                       │ contradiction          │
                       │ belief formation       │
                       └───────────┬────────────┘
                                   │
                                   ▼
                              ┌──────────┐
                              │   LLM    │
                              │  Stage   │
                              └────┬─────┘
                                   │
                         ┌─────────┴─────────┐
                         ▼                   ▼
                     Decision           Proposition
                         │                   │
                         ▼                   ▼
                      Action          Write Proposal
                         │                   │
                         ▼                   ▼
                    Observation       Write Policy
                         │                   │
                         └─────────┬─────────┘
                                   ▼
                                Wₜ / Gₜ
                                   │
                                   │ promotion
                                   ▼
                                  Mₜ
```

The central architectural boundary is:

```text
                 INFORMATION
                     │
              Context Resolver
                     │
                     ▼
                 COGNITION
                     │
              Epistemic Policy
                     │
                     ▼
                  ACTION
                     │
                     ▼
                 OBSERVATION
                     │
                     ▼
                  MEMORY
```

---

# 17. What the Architecture Claims

The architecture makes **architectural claims**, not performance claims.

### Claim A

State lifetime and mutation authority should be explicit.

### Claim B

Cognitive stages benefit from stage-specific context rather than indiscriminate state exposure.

### Claim C

Context resolution should be centralized.

### Claim D

Read authorization and write authorization should remain separate.

### Claim E

Transient execution state should have an explicit persistence boundary.

### Claim F

A fixed workflow with bounded branching provides a useful baseline before introducing autonomous planning.

### Claim G

Reasoning provenance should extend from retrieval through persistent mutation.

### Hypothesis H

An explicit epistemic constitution can improve reasoning consistency, calibration, correction, and memory quality.

### Hypothesis H-Spinoza

A constitution derived from Spinoza's methodological treatment of understanding may produce measurable improvements over generic reasoning instructions.

H-Spinoza is **the most speculative component** of the architecture.

---

# 18. What We Are Deliberately Not Claiming

This paper does not claim that:

* Spinoza provides a complete theory of machine reasoning;
* philosophical prompts automatically improve LLM reasoning;
* explicit epistemic policies outperform ordinary prompting;
* centralized context resolution is universally superior;
* three state tiers are theoretically canonical;
* deterministic policies will outperform learned policies;
* memory necessarily requires graphs;
* every cognitive operation requires an LLM;
* the architecture is superior to LangGraph, LlamaIndex, Mem0, or other production frameworks.

The architecture is a **testable design hypothesis**.

---

# 19. Known Limitations

## L1 — LLM compliance

The model may interpret epistemic instructions inconsistently.

A constitution expressed in natural language remains a probabilistic instruction.

## L2 — Epistemic policy ambiguity

Terms such as:

```text
evidence
adequacy
belief
certainty
understanding
```

require operational definitions before evaluation.

## L3 — Provenance does not guarantee correctness

A complete provenance chain can explain why a system reached a conclusion while the conclusion remains wrong.

Traceability and truth are separate properties.

## L4 — Context contracts can become maintenance overhead

Closed-world authorization prevents accidental information leakage but increases contract-maintenance cost.

## L5 — Centralization can become a bottleneck

A universal resolver could become:

```text
single performance bottleneck
single architectural dependency
single source of policy complexity
```

This must be tested rather than assumed away.

## L6 — Memory write policies face delayed credit assignment

The value of a memory mutation may become apparent much later.

A memory that appears useless now may become valuable in a future task, while an apparently useful memory may later cause systematic error.

## L7 — Epistemic representations can become expensive

Representing evidence, relationships, contradictions, and derivations creates additional tokens, storage, and processing.

## L8 — Philosophical translation is lossy

Mapping a historical philosophical text into computational primitives inevitably introduces interpretation.

The resulting system would be **Spinoza-inspired**, not "Spinoza implemented."

## L9 — Architecture complexity

The architecture deliberately introduces additional boundaries:

```text
state
contract
resolver
epistemic policy
stage
write policy
provenance
```

The system therefore needs evidence that each boundary produces practical value.

---

# 20. Research Questions

### RQ1 — Context

Does stage-conditioned context improve task performance compared with global context at equivalent token budgets?

### RQ2 — Resolver

Does centralized context resolution reduce irrelevant context and improve debugging compared with stage-local retrieval?

### RQ3 — Memory

Does explicit promotion from `Wₜ → Mₜ` improve long-term memory quality?

### RQ4 — Mutation

Does explicit ADD/UPDATE/SUPERSEDE/DELETE/NOOP policy reduce memory corruption?

### RQ5 — Provenance

Does decision-level provenance improve error diagnosis compared with ordinary execution tracing?

### RQ6 — Epistemic policy

Does an explicit epistemic constitution improve:

```text
calibration
contradiction handling
unsupported inference
memory correction
```

?

### RQ7 — Spinoza

Does a Spinoza-derived epistemic constitution outperform:

```text
generic system prompt
structured reasoning instructions
domain-specific reasoning policy
```

under controlled conditions?

### RQ8 — Compounding effects

Does improved epistemic memory produce greater performance gains over multiple sessions than it produces on individual tasks?

### RQ9 — Complexity

At what point does the architectural machinery cost more than it contributes?

---

# 21. Experimental Program

The experiments should proceed incrementally.

## Phase I — Information governance

Compare:

```text
Global context
vs.
Stage-specific contracts
```

Hold the model, task, retrieval system, and token budget constant.

Measure:

```text
answer quality
retrieval precision
token consumption
context pollution
latency
```

---

## Phase II — Persistence governance

Compare:

```text
automatic memory accumulation
vs.
explicit promotion policy
```

Measure:

```text
memory precision
memory recall
stale-memory rate
contradiction rate
future-task utility
```

---

## Phase III — Provenance

Compare:

```text
API/execution traces
vs.
decision-level provenance
```

Measure:

```text
failure localization
debugging time
incorrect attribution
recovery success
```

---

## Phase IV — Epistemic constitution

Four controlled conditions:

```text
A — Baseline
B — Structured reasoning policy
C — General epistemic constitution
D — Spinoza-derived constitution
```

Same:

```text
model
tools
retrieval
workflow
context budget
tasks
```

Measure:

```text
accuracy
calibration
contradiction resolution
uncertainty preservation
unsupported claims
memory quality
memory correction
provenance quality
```

---

## Phase V — Longitudinal compounding

This is arguably the most important experiment.

Run repeated sessions:

```text
Session 1
   ↓
Memory
   ↓
Session 2
   ↓
Memory correction
   ↓
Session 3
   ↓
...
   ↓
Session N
```

Compare the rate at which each architecture accumulates:

```text
useful knowledge
incorrect knowledge
obsolete knowledge
contradictions
unsupported beliefs
```

A memory architecture should be evaluated as a **dynamic system**, not merely by whether it answers an isolated question correctly.

---

# 22. Falsification Criteria

The architecture should be considered unsuccessful if controlled experiments show that:

### F1

Global context performs as well as stage-conditioned context at equal cost.

### F2

Stage-local retrieval performs as well as centralized resolution without producing greater debugging complexity.

### F3

Automatic memory accumulation performs as well as explicit promotion.

### F4

Provenance does not improve failure diagnosis.

### F5

Epistemic policies do not improve decision quality or calibration.

### F6

The additional architecture produces less benefit than its computational and engineering overhead.

### F7

The Spinoza-derived constitution provides no measurable advantage over simpler epistemic policies.

That last result would be useful.

The architecture should survive even if the philosophical hypothesis fails.

---

# 23. Implementation Roadmap

### V1 — Deterministic architecture

Implement:

```text
Mₜ
Gₜ/Hₜ
Wₜ

ContextContract
ContextResolver

fixed workflow
bounded branches

write proposals
ADD / UPDATE / SUPERSEDE / DELETE / NOOP

provenance
```

No learned policies.

---

### V2 — Epistemic representation

Introduce:

```text
Observation
Evidence
Hypothesis
Inference
Belief
Contradiction
```

without changing the workflow.

---

### V3 — Epistemic constitution

Introduce a structured system-level epistemic policy.

Test generic versus Spinoza-derived variants.

---

### V4 — Learned policies

Only after deterministic baselines establish their failure modes:

```text
learned context ranking
learned memory write policy
learned routing
```

---

### V5 — Joint optimization

Only if evidence warrants it:

```text
π_R
π_A
π_U
```

could eventually be optimized jointly.

This remains a research direction rather than a V1 architectural requirement.

---

# 24. Open TODOs

## Architecture

* Formalize the state schema.
* Define stage lifecycle.
* Define contract schema.
* Define resolver interface.
* Define mutation capabilities.
* Define persistence boundaries.
* Define provenance schema.

## Epistemic layer

* Extract the operational principles from the Treatise.
* Separate textual interpretation from engineering interpretation.
* Define an epistemic state machine.
* Determine whether "adequacy" can receive a useful computational definition.
* Define evidence strength without collapsing it into arbitrary confidence scores.
* Define contradiction semantics.
* Define belief revision rules.

## Memory

* Define conflict resolution.
* Define supersession.
* Define temporal validity.
* Define provenance retention.
* Define forgetting.
* Define memory consolidation.

## Evaluation

* Build longitudinal tasks.
* Construct contradiction tasks.
* Construct memory-corruption tasks.
* Construct provenance-debugging tasks.
* Measure token cost.
* Measure latency.
* Measure context pollution.
* Measure cumulative memory quality.

## Research

* Compare philosophical constitutions.
* Compare natural-language versus structured epistemic policies.
* Compare one global constitution versus stage-specific epistemic policies.
* Test whether epistemic policy benefits survive across models.
* Test whether benefits persist after prompt optimization.

---

# 25. What Is Worth Stealing From Existing Systems

The architecture should remain compatible with proven ideas from the current ecosystem rather than rebuilding everything.

### Memory

Borrow:

```text
multi-signal retrieval
priority-based context fitting
temporal validity
explicit CRUD semantics
```

Systems such as Mem0, LlamaIndex Memory, and temporal-memory systems provide useful implementation precedents.

### Context

Borrow:

```text
token budgeting
structured context
progressive retrieval
context compaction
```

Recent agent-context work increasingly treats context as a managed lifecycle rather than a static prompt. ([arXiv][6])

### Provenance

Borrow:

```text
execution tracing
evidence lineage
claim attribution
memory lineage
```

This aligns with current evidence-tracing research. ([arXiv][2])

### Orchestration

Borrow:

```text
graph execution
checkpointing
bounded branching
typed state
```

The architecture does not need to replace existing orchestration frameworks.

It needs to impose stronger contracts around them.

---

# 26. Architectural Thesis

The central thesis can now be stated more precisely:

> **A persistent LLM agent should be treated as a governed cognitive system rather than a retrieval pipeline.**

Its architecture has four interacting dimensions:

```text
                 ┌───────────────────┐
                 │       STATE       │
                 │ What exists?      │
                 │ How long?         │
                 │ Who can mutate?   │
                 └─────────┬─────────┘
                           │
                           ▼
                 ┌───────────────────┐
                 │      CONTEXT      │
                 │ What may this     │
                 │ stage see?        │
                 └─────────┬─────────┘
                           │
                           ▼
                 ┌───────────────────┐
                 │     EPISTEMIC     │
                 │ How should it     │
                 │ understand it?   │
                 └─────────┬─────────┘
                           │
                           ▼
                 ┌───────────────────┐
                 │      ACTION       │
                 │ What should it    │
                 │ do?               │
                 └─────────┬─────────┘
                           │
                           ▼
                 ┌───────────────────┐
                 │      MEMORY       │
                 │ What should       │
                 │ persist?          │
                 └───────────────────┘
```

The provenance layer cuts across all four:

```text
                    PROVENANCE
                       │
       ┌───────────────┼────────────────┐
       ▼               ▼                ▼
     STATE          CONTEXT          EPISTEMIC
       │               │                │
       └───────────────┼────────────────┘
                       ▼
                     ACTION
                       │
                       ▼
                     MEMORY
```

That is the refined architecture.

The **Context Contract + Context Resolver** governs information access.

The **mutation authority + Write Policy** governs persistence.

The **epistemic constitution** governs interpretation.

The **provenance system** makes the resulting decisions inspectable.

The **fixed workflow** keeps the initial system tractable.

And the entire policy layer remains replaceable.

---

# 27. Final Research Position

The strongest version of the proposal is therefore **not**:

> "Build a RAG system based on Spinoza."

It is:

> **Build a governed cognitive architecture in which state, context, epistemic interpretation, action, and memory are explicit and independently controllable; then investigate whether an explicit epistemic constitution, including a Spinoza-derived formulation, improves the system's ability to reason, correct itself, and accumulate reliable knowledge over time.**

That framing protects the architectural work from the philosophical hypothesis.

If the Spinoza experiment fails, we still have:

```text
State isolation
+
Context contracts
+
Central resolver
+
Mutation authority
+
Controlled persistence
+
Provenance
```

If it succeeds, however, the implications are substantially larger:

**the system prompt stops being merely an instruction layer and becomes part of the agent's epistemic architecture.**

That is the hypothesis I would take to other frontier models for adversarial review.

[1]: https://www.gutenberg.org/files/1016/1016-h/1016-h.htm?utm_source=chatgpt.com "The Project Gutenberg E-text of On the Improvement of the Understanding, by Baruch Spinoza"
[2]: https://arxiv.org/abs/2606.04990?utm_source=chatgpt.com "From Agent Traces to Trust: Evidence Tracing and Execution Provenance in LLM Agents"
[3]: https://arxiv.org/abs/2603.09619?utm_source=chatgpt.com "Context Engineering: From Prompts to Corporate Multi-Agent Architecture"
[4]: https://www.cambridge.org/core/books/abs/cambridge-spinoza-lexicon/treatise-on-the-emendation-of-the-intellect/E931822ABFB48CF29805A8E4E18486F8?utm_source=chatgpt.com "Treatise on the Emendation of the Intellect (185.) - The Cambridge Spinoza Lexicon"
[5]: https://revistas.usc.gal/index.php/agora/article/view/8240/12853?utm_source=chatgpt.com "Adequate idea and order of philosophizing in Spinoza’s Treatise on the emendation of the intellect: prolegomena to the ontology of the Ethics | Agora. Papeles de Filosofía"
[6]: https://arxiv.org/abs/2607.21503?utm_source=chatgpt.com "Agentic Context Management: Solving Agent Memory and Cost by Treating Them as Lifecycle and Architecture Problems"
