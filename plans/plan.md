
The research result supports the architecture's **systems-engineering core**, while weakening the claim that every part of the architecture is equally novel or equally valuable. The refined paper should therefore distinguish **architectural requirements**, **implementation choices**, and **research hypotheses**.

I would structure the paper around that distinction.

# Proposed Refined Paper

## Toward a Governed Cognitive Context Architecture for Stateful RAG Systems

### Abstract

Retrieval-Augmented Generation systems have evolved from relatively simple retrieval-and-generation pipelines into stateful agentic systems containing persistent memory, intermediate reasoning, multiple cognitive operations, tool use, and iterative retrieval.

This evolution creates a systems problem that is poorly addressed by conventional RAG architectures: **the information available to an operation becomes part of the operation's effective reasoning environment**.

A system may retrieve a correct memory yet produce an incorrect result because:

* the memory was presented to the wrong operation;
* irrelevant state polluted the context;
* transient observations became persistent memory without sufficient validation;
* a persistent memory was modified by an operation that merely needed read access;
* a stage's decision cannot be traced to the information that influenced it;
* stale or contradictory memories remain retrievable;
* increasingly sophisticated orchestration compensates for deficiencies in the underlying retrieval system.

This paper proposes a governed cognitive architecture in which **state, context, cognitive stages, retrieval, and persistence are separate architectural concerns**.

The architecture introduces:

1. explicit state tiers;
2. stage-specific Context Contracts;
3. a centralized context resolver;
4. independent read and mutation authority;
5. explicit transient-to-persistent promotion;
6. provenance across context and memory decisions;
7. deterministic workflow control as the baseline;
8. replaceable policies allowing later transition toward learned mechanisms.

The architecture is deliberately conservative. It does not assume that additional agents, planners, reinforcement learning, or philosophical prompting improve performance. Those mechanisms become experimental substitutions for defined policies only after empirical failure of the deterministic baseline.

A secondary research hypothesis concerns **epistemic governance**: whether explicitly structured reasoning principles, potentially inspired by Spinoza's *Treatise on the Improvement of the Understanding*, can improve the quality, consistency, and traceability of LLM decisions when incorporated into system prompts, schemas, context contracts, and evaluation criteria.

The central proposition is therefore not that a particular prompt or memory algorithm is optimal. It is that **reasoning systems require an explicit architecture governing what information enters a cognitive operation, what that operation may change, and how its decisions become persistent system state**.

---

# 1. Problem Domain

Traditional RAG can be approximated as:

```text
query
  ↓
retrieve
  ↓
context
  ↓
LLM
  ↓
answer
```

Agentic RAG expands this:

```text
user
 ↓
interpretation
 ↓
retrieval
 ↓
evaluation
 ↓
refinement
 ↓
retrieval
 ↓
synthesis
 ↓
answer
```

Persistent memory introduces another dimension:

```text
                    ┌───────────────┐
                    │ Persistent M  │
                    └───────┬───────┘
                            │
                    retrieval / write
                            │
user → execution → stages → state
                 ↑           │
                 └───────────┘
```

The resulting system has two fundamental concerns:

### Execution

What must the system do to answer the current request?

### State management

What information should the system retain, expose, modify, promote, update, supersede, or discard?

These concerns interact, but they have different objectives and different failure modes.

The architecture therefore treats them as separate control concerns connected through explicit interfaces.

---

# 2. Design Objectives

| ID     | Objective                      | Architectural consequence                                                               |
| ------ | ------------------------------ | --------------------------------------------------------------------------------------- |
| **O1** | State isolation                | State has explicit lifetime and mutation semantics                                      |
| **O2** | Stage-specific context         | Each cognitive stage receives context appropriate to its objective                      |
| **O3** | Centralized context resolution | Stages declare information requirements instead of implementing retrieval independently |
| **O4** | Explicit mutation authority    | Read access does not imply write access                                                 |
| **O5** | Controlled persistence         | Working observations require explicit promotion before becoming durable memory          |
| **O6** | Replaceable policies           | Deterministic policies can later be replaced by learned policies                        |
| **O7** | Debuggability                  | Context, decisions, retrieval and persistence decisions remain attributable             |
| **O8** | Controlled complexity          | Additional autonomy is introduced only in response to measured failure                  |

These objectives define the architecture independently of any particular LLM, vector database, memory framework, or orchestration library.

---

# 3. State Model

The architecture uses three principal state domains.

```text
┌──────────────────────────────────────────────┐
│              Persistent State Mₜ             │
│                                              │
│ durable facts                                │
│ validated experience                         │
│ procedures                                   │
│ user/domain knowledge                         │
└──────────────────────┬───────────────────────┘
                       │
                controlled read/write
                       │
┌──────────────────────▼───────────────────────┐
│             Session State Gₜ / Hₜ             │
│                                              │
│ current goal                                 │
│ conversation history                         │
│ task decomposition                            │
│ session decisions                             │
└──────────────────────┬───────────────────────┘
                       │
                     read
                       │
┌──────────────────────▼───────────────────────┐
│              Working State Wₜ                │
│                                              │
│ hypotheses                                   │
│ retrieval attempts                           │
│ candidate evidence                           │
│ intermediate observations                     │
│ failed approaches                            │
└──────────────────────────────────────────────┘
```

The important property is the **persistence boundary**.

A retrieval result entering `Wₜ` does not become memory merely because an LLM saw it.

Promotion requires an explicit decision:

```text
Wₜ
 │
 │ candidate memory
 ▼
Write Policy
 │
 ├── NOOP
 ├── ADD
 ├── UPDATE
 ├── SUPERSEDE
 └── DELETE
 │
 ▼
Mₜ₊₁
```

This separates **experience during execution** from **knowledge retained across executions**.

---

# 4. State Authority and Context Authority

Two permission systems should remain separate.

### Mutation authority

Determines:

> Who may change this state?

### Read authority

Determines:

> What information may this stage see?

Therefore:

```text
             Mutation Authority
                    │
                    ▼
             State Repository
                    ▲
                    │
             Context Resolver
                    ▲
                    │
             Read Contract
                    ▲
                    │
                  Stage
```

A stage may therefore have:

```text
read:    Mₜ.user_preferences
         Mₜ.validated_procedures

write:   Wₜ.hypotheses
         Gₜ.subgoal

deny:    Mₜ.*
```

This distinction matters because **information access and information authority have different security and lifecycle requirements**.

---

# 5. Stage Model

A stage is a cognitive operation with:

1. an objective;
2. a success criterion;
3. a context requirement;
4. an output schema;
5. potentially a state mutation proposal.

Examples:

```text
Query Interpreter
Retrieval Planner
Evidence Judge
Answer Synthesizer
Memory Manager
```

A deterministic operation such as JSON parsing remains ordinary program logic.

The architecture therefore does not equate:

```text
pipeline node = LLM stage
```

Instead:

```text
pipeline
 ├── deterministic operations
 ├── retrieval operations
 ├── validation operations
 └── cognitive stages
```

This is important for O8.

An LLM invocation must earn its existence through a measurable decision problem.

---

# 6. Context Contracts

Each cognitive stage declares a contract.

```text
ContextContract {
    required
    optional
    excluded
    tokenBudget
}
```

However, the earlier research exposed an important correction.

The contract should operate under a **closed-world / default-deny model**.

```text
declared required  → permitted
declared optional  → permitted
undeclared         → denied
```

`excluded` can remain as a defense-in-depth mechanism, but omission already denies access.

This prevents a new state field from silently becoming visible to every stage.

For example:

```text
EvidenceJudge:

required:
  current_query
  retrieved_evidence

optional:
  prior_answer
  retrieval_history

excluded:
  personal_preferences
  unrelated_session_state
```

The judge does not receive the entire agent state and then rely on the LLM to ignore irrelevant information.

---

# 7. Central Context Resolution

The architecture introduces one common projection mechanism:

```text
R(stage, contract, Sₜ) → Cₜˢ
```

where:

* `stage` identifies the cognitive operation;
* `contract` defines its information requirements;
* `Sₜ` represents current system state;
* `Cₜˢ` is the context presented to that stage.

Conceptually:

```text
                 State
                   │
                   ▼
        ┌─────────────────────┐
        │ Context Resolver R  │
        └──────────┬──────────┘
                   │
          ┌────────┼────────┐
          ▼        ▼        ▼
       Stage A   Stage B   Stage C
       Context  Context   Context
```

The resolver can perform:

```text
1. authorization
2. candidate generation
3. retrieval
4. scoring
5. filtering
6. transformation
7. budget fitting
8. provenance attachment
```

The stage therefore specifies **what it needs**, rather than **how to find it**.

That creates a stable architectural boundary.

---

# 8. Retrieval Is a Policy

The initial implementation should use deterministic retrieval/context selection.

For candidate `x` and stage `s`:

```text
U(x | s, Gₜ, Mₜ, Wₜ)
```

can combine signals such as:

```text
semantic relevance
lexical relevance
entity relevance
recency
temporal validity
source reliability
stage relevance
duplication
contradiction
```

The architecture should allow this later:

```text
Deterministic Resolver
        ↓
Learned Resolver
        ↓
Joint Read/Action Policy
```

without changing the state model or stage interfaces.

This satisfies O6.

---

# 9. Provenance

The architecture's strongest operational invariant becomes:

> Every persistent mutation and every context inclusion must have an attributable reason.

The complete path becomes:

```text
Memory
   ↓
candidate generation
   ↓
retrieval score
   ↓
Context Contract
   ↓
context selection
   ↓
stage invocation
   ↓
stage decision
   ↓
write proposal
   ↓
write policy
   ↓
persistent mutation
```

Each transition should emit provenance.

For context:

```text
ContextItem {
    sourceId
    sourceType
    memoryId
    retrievalMethod
    retrievalScore
    contractField
    transformation
    timestamp
}
```

For persistence:

```text
WriteDecision {
    operation
    target
    evidence
    reason
    confidence
    policyVersion
    sourceStage
}
```

This changes observability from:

```text
"the LLM returned this"
```

to:

```text
"this stage received these items,
because these items satisfied this contract,
under this resolver policy,
and produced this decision,
which generated this persistence proposal."
```

That is substantially more useful for debugging.

---

# 10. Memory Management

The write path should initially remain deterministic.

```text
candidate observation
       ↓
memory comparison
       ↓
conflict detection
       ↓
operation selection
       ↓
ADD / UPDATE / SUPERSEDE / DELETE / NOOP
       ↓
persistent store
```

The architecture should explicitly distinguish:

### ADD

No equivalent durable memory exists.

### UPDATE

Existing memory remains conceptually the same entity but its content changes.

### SUPERSEDE

A new proposition replaces an older proposition while preserving history.

### DELETE

The memory should no longer participate in future retrieval.

### NOOP

The observation provides insufficient durable value.

`NOOP` deserves explicit representation because retaining information has a cost.

---

# 11. Temporal and Contradictory Memory

Persistent memory cannot be modeled purely as:

```text
fact → embedding
```

A mature memory representation needs to accommodate:

```text
fact
source
timestamp
validity
confidence
supersession
contradiction
provenance
```

For example:

```text
M1:
User prefers X
valid: 2025–2026

M2:
User prefers Y
valid: 2026–
supersedes: M1
```

This permits retrieval to answer:

> What was true?

and:

> What is currently believed to be true?

as different questions.

The architecture therefore leaves room for temporal/graph-based implementations without requiring a graph database in v1.

---

# 12. Conversation History

Conversation history belongs primarily to session state:

```text
Hₜ
```

It should not automatically become persistent memory.

The architecture instead creates:

```text
conversation
    ↓
working interpretation
    ↓
candidate durable observation
    ↓
memory policy
    ↓
Mₜ₊₁
```

This prevents the common conflation:

```text
"the model saw it"
        =
"the system knows it"
```

Those are different states.

---

# 13. Workflow Control

The initial controller should be:

```text
fixed workflow
+
bounded conditional branches
```

For example:

```text
Query Interpretation
        ↓
Retrieval
        ↓
Evidence Judge
        │
        ├── sufficient → synthesis
        │
        └── insufficient → bounded retrieval refinement
```

A planner capable of freely selecting stages is deferred.

A learned router is deferred.

A learned memory controller is deferred.

The interface remains compatible with all three.

This gives:

```text
v1
Fixed Workflow
     ↓
v2
Planner
     ↓
v3
Learned Router
```

without forcing the architecture to become increasingly autonomous merely because the technology permits it.

---

# 14. The LLM as a Governed Cognitive Component

This is where your **belief-system/system-prompt observation** becomes relevant.

An LLM does not operate from architecture alone.

Its behavior is shaped by:

```text
system instructions
+
schemas
+
examples
+
context
+
available tools
+
interaction history
+
evaluation feedback
```

Modern systems increasingly formalize these as:

* skills;
* policies;
* context engineering;
* tool contracts;
* structured output schemas;
* role-specific instructions.

Therefore a stage should not merely be:

```text
LLM(prompt)
```

It should conceptually be:

```text
Stage =
    Objective
  + Epistemic Policy
  + Context Contract
  + Output Contract
  + Tool Authority
  + Mutation Authority
  + Evaluation Criteria
```

This gives us a more precise place for the Spinoza hypothesis.

---

# 15. The Spinoza Hypothesis

Spinoza's *Treatise on the Improvement of the Understanding* is interesting here because it provides something closer to an **epistemic operating procedure** than a persona.

The relevant idea is not:

> "Tell the model to behave like Spinoza."

That would reduce the hypothesis to persona prompting.

The more interesting question is:

> Can an explicit epistemic discipline governing how claims are formed, examined, distinguished, connected, and revised improve LLM reasoning?

That can be tested.

For example, a stage could be instructed to maintain distinctions between:

```text
observation
interpretation
hypothesis
inference
established proposition
uncertainty
contradiction
```

rather than allowing them to collapse into one undifferentiated textual context.

This fits the architecture because the epistemic policy operates **inside the cognitive stage**, while the Context Resolver governs **what information reaches the stage**.

They are therefore orthogonal:

```text
                 System State
                     │
                     ▼
             Context Resolver
                     │
                     ▼
          ┌────────────────────┐
          │ Cognitive Stage    │
          │                    │
          │ Objective          │
          │ Epistemic Policy   │
          │ Context            │
          │ Output Contract    │
          └─────────┬──────────┘
                    │
                    ▼
               Decision
```

That is the correct place to test the hypothesis.

It should **not** become an architectural assumption.

---

# 16. Research Hypotheses

The architecture itself should remain relatively theory-neutral.

The experiments can then test:

### H1 — Context

Stage-conditioned context improves decision quality relative to globally supplied context under equivalent token budgets.

### H2 — Governance

Closed-world context authorization reduces irrelevant-context exposure and cross-stage contamination.

### H3 — Persistence

Explicit promotion reduces long-term memory pollution relative to automatic persistence.

### H4 — Provenance

Decision-level provenance reduces diagnosis time for retrieval and memory failures.

### H5 — Memory policy

Explicit ADD/UPDATE/SUPERSEDE/DELETE/NOOP semantics improve long-term memory quality relative to append-only memory.

### H6 — Control

Fixed workflows with bounded branching achieve a better reliability/complexity tradeoff than unconstrained planning for the target workload.

### H7 — Epistemic policy

A structured epistemic policy improves reasoning quality or correction behavior relative to an otherwise identical stage without that policy.

### H8 — Coupled optimization

Joint optimization of context selection, stage action, and memory mutation may outperform independently optimized subsystems.

H8 should remain a **research question**, rather than a v1 implementation requirement.

---

# 17. What the Literature Currently Supports

The evidence should be represented honestly.

| Component                          | Evidence                                          | Status                    |
| ---------------------------------- | ------------------------------------------------- | ------------------------- |
| Stage-specific context             | Strong and growing                                | **Build**                 |
| Centralized context selection      | Strong architectural rationale                    | **Build**                 |
| Closed-world context authorization | Limited direct evidence, strong systems rationale | **Test**                  |
| Explicit memory mutation           | Strong emerging evidence                          | **Build**                 |
| Controlled promotion               | Strong rationale, weaker direct evidence          | **Build/Test**            |
| Temporal memory                    | Established direction                             | **Build incrementally**   |
| Provenance                         | Strong engineering rationale                      | **Build**                 |
| Fixed workflow baseline            | Strong practitioner evidence                      | **Build**                 |
| Learned memory control             | Active research                                   | **Later experiment**      |
| Joint read/action/write policy     | Open research problem                             | **Research**              |
| Spinoza-inspired epistemic policy  | No direct evidence                                | **Controlled experiment** |

This is substantially stronger than claiming that the entire architecture represents an already-established consensus.

---

# 18. What Should Be Stolen From the Market

The architecture should **reuse proven mechanisms** wherever the abstraction boundary permits it.

### Retrieval

Steal:

```text
semantic + lexical + entity + metadata signals
```

rather than building a single similarity score.

### Memory

Steal:

```text
ADD / UPDATE / DELETE / NOOP
```

and extend it with:

```text
SUPERSEDE
```

where temporal history matters.

### Temporal knowledge

Steal:

```text
validity intervals
source provenance
supersession
```

from temporal-memory systems.

### Context management

Steal:

```text
priority
token budgeting
context compaction
```

while keeping the resolver as our architectural boundary.

### Orchestration

Steal:

```text
explicit graph execution
checkpointing
observability
bounded branching
```

from mature orchestration systems.

### Evaluation

Steal:

```text
long-horizon memory tests
multi-session tests
context-budget sweeps
retrieval ablations
```

rather than evaluating only final answer quality.

---

# 19. What We Should Deliberately Not Steal Yet

The architecture should resist importing complexity merely because the market contains it.

Deferred mechanisms include:

```text
autonomous planner
learned stage router
RL memory manager
joint read/write/action training
graph memory as a mandatory substrate
multi-agent orchestration
```

Each can later occupy a policy slot.

None should determine the underlying architecture.

---

# 20. Known Limitations

### L1 — Retrieval quality remains foundational

Context governance cannot rescue a fundamentally poor retriever.

The architecture therefore assumes a competent retrieval baseline.

### L2 — Context contracts require maintenance

A closed-world contract creates a deliberate failure mode:

```text
new state field
    ↓
undeclared
    ↓
invisible
```

This is safe, but operationally inconvenient.

### L3 — Provenance does not establish causality

Knowing that a memory entered a context does not prove that the LLM used it.

The architecture provides attribution of **availability and decisions**, not perfect causal attribution of neural computation.

### L4 — LLM reasoning remains probabilistic

Structured prompts and epistemic policies constrain behavior; they do not guarantee logical consistency.

### L5 — Memory conflicts remain difficult

ADD/UPDATE/SUPERSEDE/DELETE defines operations, but deciding whether two propositions genuinely conflict remains a semantic problem.

### L6 — Stage decomposition is itself uncertain

Too few stages cause context and objective conflation.

Too many stages introduce coordination overhead and additional failure surfaces.

### L7 — The architecture adds infrastructure

Contracts, provenance, state isolation, and policy boundaries create implementation cost.

The system therefore needs empirical evidence that these mechanisms repay that cost.

### L8 — Spinoza remains an unvalidated hypothesis

There is currently no basis for claiming that a Spinoza-derived epistemic policy improves general LLM reasoning.

It belongs in the experimental layer.

---

# 21. Implementation TODO

## Phase I — Instrument the baseline

```text
[ ] existing RAG pipeline
[ ] stage execution traces
[ ] retrieval candidates
[ ] retrieval scores
[ ] complete stage contexts
[ ] final outputs
```

Establish the baseline before changing architecture.

## Phase II — Introduce state boundaries

```text
[ ] Mₜ
[ ] Gₜ/Hₜ
[ ] Wₜ
[ ] explicit promotion
[ ] memory operation semantics
```

## Phase III — Introduce Context Contracts

```text
[ ] ContextContract schema
[ ] default-deny authorization
[ ] centralized resolver
[ ] token budgeting
[ ] provenance
```

## Phase IV — Evaluate

Run controlled comparisons:

```text
global context
vs.
stage-conditioned context

automatic memory
vs.
governed promotion

append-only memory
vs.
mutation policy

no provenance
vs.
full provenance
```

## Phase V — Epistemic experiment

Hold the architecture constant.

Compare:

```text
Baseline system prompt
        vs.
Structured epistemic policy
        vs.
Spinoza-derived epistemic policy
```

Measure:

```text
accuracy
calibration
contradiction handling
correction rate
memory quality
retrieval refinement
reasoning trace consistency
```

## Phase VI — Learned policies

Only after deterministic policy limitations are measured:

```text
learned context scoring
learned write policy
learned routing
joint read/action/write optimization
```

---

# 22. Central Architectural Principle

The paper should ultimately reduce to this:

> **An agent should not receive information merely because the information exists, nor retain information merely because the agent encountered it.**

Information should pass through explicit architectural boundaries:

```text
              STATE
                │
                │
        ┌───────▼────────┐
        │ Context Policy │
        └───────┬────────┘
                │
                ▼
          COGNITIVE STAGE
                │
                │
          ┌─────▼─────┐
          │ Decision  │
          └─────┬─────┘
                │
          ┌─────▼─────┐
          │ Write     │
          │ Policy    │
          └─────┬─────┘
                │
                ▼
             STATE
```

The resulting architecture is therefore less about "better RAG" and more about **governing the information lifecycle of an LLM-based cognitive system**.

That is the version of the paper I would carry forward. The literature supports enough of the underlying engineering decisions to justify building it, while the genuinely uncertain pieces—joint policy optimization and the epistemic/Spinoza layer—remain explicitly testable rather than being smuggled in as conclusions.
