# RAG-ARCHITECTURE-RESEARCH

Architecture Specification: Stateful, Stage-Conditioned Agentic Retrieval

Status: Architecture baseline
Version: 0.1
Scope: Agentic RAG, conversational memory, persistent memory, multi-stage reasoning
Design posture: Evidence-informed, deliberately conservative
Open research: Coupled read/action/write optimization, learned routing, memory governance


---

Abstract

This document specifies an architecture for a stateful LLM agent in which persistent memory, session state, execution state, conversational history, retrieval, and multi-stage reasoning operate as distinct architectural concerns.

The architecture is motivated by a recurring failure in conventional RAG systems: the system treats retrieved information as a single context stream and repeatedly exposes different cognitive operations to approximately the same information. This creates context pollution, duplicated retrieval logic, uncontrolled memory growth, and unclear mutation boundaries.

The proposed architecture introduces four explicit abstractions:

1. State — information maintained by the agent across execution boundaries.


2. Stage — an LLM-controlled cognitive operation with an independent objective.


3. Context Contract — a declaration of what information a stage may consume.


4. Context Resolver — one shared mechanism that projects agent state into stage-specific context.



State mutation and context access are deliberately separated into independent permission systems.

The initial execution model uses a fixed workflow with bounded conditional branches. Dynamic planning and learned stage routing remain replaceable future extensions rather than architectural prerequisites.

The architecture does not claim that coupled read/write/action optimization is solved. Current research identifies that as a frontier problem. The implementation therefore adopts deterministic mechanisms first and preserves interfaces through which learned policies can later replace them.


---

1. Problem Definition

Conventional RAG can be represented approximately as:

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

Agentic retrieval introduces iterative operations:

Query
  │
  ▼
Retrieve
  │
  ▼
Assess
  │
  ├── insufficient ──► Refine ──► Retrieve
  │
  └── sufficient
          │
          ▼
        Answer

A persistent agent introduces additional information:

Conversation
Memory
RAG
Tool results
Current plan
Previous attempts
Current hypotheses

The architectural problem becomes:

> Which information should each cognitive operation receive, which information may it modify, and which information should survive the current execution?



Treating all of these as one context creates an implicit and poorly controlled state-management system.


---
Yes. The objectives become much stronger if each one states **the architectural problem it exists to solve**. Otherwise they read like desirable properties rather than requirements derived from failure modes.

I would rewrite the section as follows.

# 2. Design Objectives

The architecture is designed around a central problem:

> **A stateful RAG system does more than retrieve documents. It continuously decides what information exists, what information each cognitive operation may see, what actions the system should take, and what transient observations deserve persistence.**

Those decisions have different lifetimes, authorities, and failure modes. The architecture therefore establishes the following objectives.

---

## O1 — State isolation

**Objective:** Different classes of state must have explicit lifetime and mutation semantics.

### Why

A pipeline simultaneously handles information with fundamentally different lifetimes:

* persistent knowledge that should survive sessions;
* session state that belongs to the current interaction;
* execution state that exists only while solving the current task.

Treating these as one undifferentiated state object creates ambiguous ownership.

For example, if a retrieval result is placed into the same state structure as a durable user preference, the architecture has no intrinsic answer to:

> Should this information survive the execution?

Likewise, if every stage can mutate the same state object, mutation authority becomes implicit.

The architecture therefore separates state according to **lifetime and mutation ownership**, currently:

```text
M_t       Persistent state
G_t/H_t   Session state
W_t       Working / execution state
```

The purpose is not taxonomy. The purpose is to establish enforceable boundaries.

---

## O2 — Stage-specific context

**Objective:** Each cognitive operation receives context appropriate to its objective.

### Why

Different stages solve different problems.

A query interpreter may need:

```text
conversation
current objective
relevant user/project preferences
```

A retrieval judge may need:

```text
query
retrieved evidence
retrieval history
evaluation criteria
```

A generator may need:

```text
validated evidence
current objective
output constraints
```

Giving all three the same context creates two problems:

1. **Context pollution:** irrelevant information competes for model attention.
2. **Context leakage:** information available to one cognitive operation becomes implicitly available to another.

Therefore context should be a **function of the stage's objective**, rather than a dump of available state.

---

## O3 — Centralized context resolution

**Objective:** Stages should declare information requirements rather than implement independent retrieval mechanisms.

### Why

Without centralization, each stage eventually develops its own version of:

```text
retrieve()
searchMemory()
getRelevantDocuments()
loadHistory()
```

That produces retrieval drift.

For example:

```text
Interpreter → vector search
Retriever   → hybrid search
Judge       → another vector search
Generator   → memory API
```

After enough iterations, nobody can state precisely why each stage received its particular context.

Centralization gives us:

```text
Stage
  │
  │ ContextContract
  ▼
ContextResolver
  │
  ├── state
  ├── memory
  ├── retrieval
  ├── scoring
  └── token budget
  │
  ▼
Stage Context
```

The stage declares **what it requires**.

The resolver determines **how that information is obtained and assembled**.

This also creates the architectural seam required for replacing deterministic retrieval with learned retrieval later.

---

## O4 — Explicit mutation authority

**Objective:** Persistent memory cannot be modified merely because a stage can read it.

### Why

Read access and write access have fundamentally different consequences.

A stage may need to know:

> "The user prefers PostgreSQL."

That does not imply that the stage should be able to change:

```text
M_t.user_preferences
```

If read and write authority are coupled, any stage that receives persistent memory becomes a potential memory editor.

That makes memory corruption an emergent property of ordinary pipeline execution.

The architecture therefore separates:

```text
Read authority
     ≠
Mutation authority
```

For example:

```text
RetrievalJudge
    READ  → M_t.project_facts
    WRITE → W_t.judgement

MemoryManager
    READ  → M_t
    WRITE → M_t
```

The distinction should exist at the architecture/runtime level rather than depend on an instruction such as "please don't modify memory."

---

## O5 — Controlled persistence

**Objective:** Transient observations become persistent memories only through an explicit promotion mechanism.

### Why

An agent generates enormous amounts of information during execution:

```text
retrieval results
failed searches
hypotheses
intermediate interpretations
tool outputs
draft conclusions
temporary reasoning
```

Most of this should disappear.

If the system automatically persists interesting-looking observations, memory accumulates:

```text
useful information
+
temporary information
+
incorrect hypotheses
+
duplicates
+
obsolete information
```

The resulting memory becomes progressively less trustworthy.

Therefore:

```text
W_t
 │
 │ candidate
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
M_t
```

Persistence becomes a **decision**, rather than a side effect of execution.

---

## O6 — Replaceable policies

**Objective:** Deterministic mechanisms should be replaceable with learned policies without restructuring the state architecture.

### Why

The architecture should distinguish **where a decision happens** from **how the decision is made**.

For example:

```text
ContextResolver
      │
      └── SelectionPolicy
             ├── deterministic v1
             ├── heuristic v2
             └── learned v3
```

Likewise:

```text
WritePolicy
      │
      ├── deterministic
      ├── LLM-assisted
      └── learned/RL
```

This matters because the research frontier is moving toward learned memory management and context selection.

The architecture should therefore permit:

```text
same state
same contracts
same interfaces
different policy
```

without requiring a redesign of the system.

This also supports controlled experimentation: a learned policy can replace a deterministic baseline while the rest of the architecture remains constant.

---

## O7 — Debuggability

**Objective:** The system should expose enough intermediate state to explain:

* why something was retrieved;
* why it was presented to a stage;
* why a stage acted;
* why something became memory.

### Why

Agentic failures often produce a valid-looking final answer while the underlying decision path is wrong.

Consider:

```text
Bad answer
   ↓
wrong evidence
   ↓
wrong context selection
   ↓
wrong memory retrieved
   ↓
memory was incorrectly written three sessions earlier
```

A final-output log cannot identify the failure.

The architecture therefore needs provenance across the entire path:

```text
Memory
   ↓
candidate generation
   ↓
retrieval score
   ↓
ContextContract
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

This makes the system inspectable at the **decision boundary**, rather than merely observable at the API boundary.

A useful invariant is:

> Every persistent mutation and every context inclusion should have an attributable reason.

---

## O8 — Controlled complexity

**Objective:** The architecture should not introduce a planner, learned router, memory-RL system, or additional LLM invocation until an observed failure justifies it.

### Why

Each additional intelligent component introduces another source of:

* latency;
* token consumption;
* nondeterminism;
* coordination failure;
* debugging complexity;
* evaluation burden.

There is a strong architectural temptation to solve uncertainty by adding another agent:

```text
Retriever
    ↓
Judge
    ↓
Planner
    ↓
Memory Agent
    ↓
Context Agent
    ↓
Router
```

That can make the architecture appear more capable while making causal attribution harder.

The baseline therefore uses:

```text
fixed workflow
+
local conditional branches
+
deterministic policies
```

A more sophisticated mechanism enters only after an observed failure establishes a need for it.

For example:

```text
Fixed workflow
      │
      ├── works → keep
      │
      └── observed failure
              ↓
        identify missing capability
              ↓
        introduce smallest mechanism
              ↓
        evaluate against baseline
```

This makes complexity **evidence-driven**.

---

# C: The objectives form a dependency structure

There is also a useful hierarchy here.

```text
                    O1
              State isolation
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
         O4                    O5
 Mutation authority       Controlled persistence
          │                     │
          └──────────┬──────────┘
                     ▼
                    O3
          Centralized resolution
                     │
                     ▼
                    O2
          Stage-specific context
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
         O6                    O7
 Replaceable policies      Debuggability
          │                     │
          └──────────┬──────────┘
                     ▼
                    O8
          Controlled complexity
```

This exposes something important.

**O8 is not merely another feature. It governs how the other objectives evolve.**

The architecture establishes strong boundaries first:

```text
state
authority
context
persistence
```

Then it allows increasingly sophisticated policies to inhabit those boundaries.

That gives us a useful architectural principle:

> **Increase decision intelligence without increasing architectural entanglement.**

That, I think, is a better statement of what the architecture is trying to achieve than "build a better RAG pipeline."


---

3. Conceptual Model

The system state is:

S_t = {M_t, G_t, H_t, W_t}

where:

M_t = persistent cross-session memory;

G_t = current goal/subgoal state;

H_t = session conversation/history state;

W_t = execution-scoped working state.


For practical implementation, G_t and H_t may share a session store while remaining conceptually distinct.


---

4. State Model

4.1 Persistent Memory — M_t

M_t survives sessions.

Examples:

durable facts
validated preferences
procedures
validated experiences
project knowledge
learned constraints

The defining property is controlled mutation.

Stage
  │
  │ read
  ▼
M_t

Stage
  │
  │ write request
  ▼
Write Policy
  │
  ├── ADD
  ├── UPDATE
  ├── DELETE
  └── NOOP
  │
  ▼
M_t+1

A stage does not directly mutate M_t.


---

4.2 Session State — G_t / H_t

This state survives individual stage invocations but expires with the session.

G_t

Current task state:

goal
subgoals
interpretation
plan
constraints
progress

H_t

Conversation state:

messages
conversation references
session events
interaction history

Mutation is primarily controlled by interpretation/planning/session-management operations.


---

4.3 Working State — W_t

W_t exists for one execution.

Examples:

current hypotheses
retrieval attempts
failed queries
candidate documents
tool results
intermediate evidence
temporary assessments
unresolved contradictions

At execution termination:

W_t → discard

unless the write policy explicitly promotes information:

W_t
 │
 ▼
Write Policy
 │
 ▼
M_t+1

This prevents every retrieved document or intermediate hypothesis from becoming long-term memory.


---

5. Mutation Authority

State lifetime and state mutation authority are separate concepts.

The architecture therefore defines:

MutationAuthority

independently from:

ReadContract

For example:

Retriever
    read: M, G, H
    write: W

Judge
    read: M, G, H, W
    write: W

Memory Manager
    read: M, G, H, W
    write: M

A component's ability to read persistent memory therefore does not imply an ability to modify it.


---

6. Read Authority

Read authority belongs to the stage context contract.

This is deliberately separate from mutation authority.

Read authority
    ↓
Context Contract / Resolver

Mutation authority
    ↓
State layer / Write Policy

These systems should have independent configuration and versioning.

A stage's prompt or context requirements may change frequently.

Its persistent-memory mutation authority should change much less frequently.


---

7. Stage Definition

A stage is an LLM-controlled decision point with an independently meaningful success criterion.

Deterministic transformations remain code.

For example:

JSON parsing        → code
schema validation   → code
sorting             → code
deduplication       → code

query interpretation → stage
retrieval strategy   → stage
evidence assessment  → stage
answer synthesis     → stage
memory decision      → stage/policy

The architecture avoids decomposing every logical operation into an LLM invocation.

A useful empirical criterion is:

> Two adjacent operations should become separate stages when different context or independent optimization measurably changes their performance.



This provides an experimental basis for stage decomposition.


---

8. Context Contract

Each stage declares its information requirements.

ContextContract {
    required
    optional
    excluded
    tokenBudget
}

The intended semantics are:

Required

The stage depends on this information.

missing → stage cannot execute

Optional

The information can improve the stage but is subject to selection and budget.

candidate → utility scoring → inclusion decision

Excluded

The information is explicitly prohibited.

excluded → resolver veto


---

9. Closed-World Context Policy

A correction is required here.

The architecture should not treat unspecified fields as optional.

The default must be:

> Deny unless declared.



Therefore:

required ∪ optional
        ↓
candidate information

everything else
        ↓
unavailable

excluded becomes a defense-in-depth mechanism rather than the primary security boundary.

For example:

JudgeContract {

    required:
        interpreted_query
        retrieved_evidence

    optional:
        retrieval_history
        source_reliability

    excluded:
        pending_memory_mutations
}

A newly added state field does not automatically become visible to the Judge.

The contract must explicitly grant access.

This prevents silent context leakage caused by future schema expansion.


---

10. Context Resolver

The architecture uses one shared resolver:

R(stage, contract, S_t) → C_t^s

where:

stage identifies the cognitive operation;

contract specifies its information requirements;

S_t contains available state;

C_t^s is the resulting stage context.


The resolver is pure:

R(...) does not mutate S_t


---

11. Context Resolution Algorithm

Conceptually:

resolve(stage, contract, state):

    required = resolveRequired(
        contract.required,
        state
    )

    candidates = resolveOptional(
        contract.optional,
        state
    )

    ranked = score(
        candidates,
        stage.goal
    )

    selected = fitBudget(
        ranked,
        contract.tokenBudget
    )

    return ContextBundle(
        required,
        selected
    )

Excluded information never enters the candidate set.


---

12. Initial Selection Policy

The first implementation should use deterministic scoring.

Possible inputs include:

task relevance
stage relevance
recency
source quality
provenance
dependency relationship
prior utility
token cost

A simplified utility function could be:

U(x | s) =
    α relevance
  + β recency
  + γ provenance
  + δ priorUtility
  - λ tokenCost

This is an implementation starting point, not an architectural commitment.

A learned selector can later replace:

score(...)

without changing the contract or state interfaces.


---

13. Stage Execution Model

A stage execution becomes:

S_t
 │
 ▼
Context Resolver
 │
 ▼
C_t^s
 │
 ▼
LLM Stage
 │
 ▼
Observation / Result
 │
 ▼
Working State W_t

The stage does not directly modify persistent memory.


---

14. Write Architecture

Writing is explicitly separated from reading.

Observation
     │
     ▼
Write Policy
     │
 ┌───┼──────────────┐
 ▼   ▼              ▼
ADD UPDATE        DELETE
 │     │             │
 └─────┼─────────────┘
       ▼
      M_t+1

The initial policy may be deterministic or LLM-assisted.

The architecture should support:

NOOP
ADD
UPDATE
DELETE

rather than a generic:

WRITE

because these operations have different risk profiles.


---

15. Persistence Boundary

The critical transition is:

temporary information
        │
        ▼
 candidate memory
        │
        ▼
 write policy
        │
        ├── NOOP
        ├── ADD
        ├── UPDATE
        └── DELETE

Therefore:

> Retrieval does not imply memory.



> Observation does not imply memory.



> LLM output does not imply memory.



Persistence requires an explicit state transition.


---

16. RAG Position

RAG is treated as a retrieval capability, not as the overall memory architecture.

The broader system may contain:

Context Resolver
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
        Memory Store   RAG Store    Session State
             │            │            │
             └────────────┼────────────┘
                          ▼
                     Stage Context

This allows a stage to request:

persistent memory
domain knowledge
conversation history
retrieval evidence
working state

through one context interface.

The resolver decides how those sources are assembled.


---

17. Retrieval Pipeline

A concrete initial RAG execution could therefore be:

User
 │
 ▼
Interpretation Stage
 │
 ▼
Session Goal G_t
 │
 ▼
Query Construction Stage
 │
 ▼
W_t.query
 │
 ▼
Retrieval Stage
 │
 ├──── RAG
 ├──── memory retrieval
 └──── previous retrieval attempts
 │
 ▼
W_t.evidence
 │
 ▼
Assessment Stage
 │
 ├── insufficient
 │       │
 │       ▼
 │    Query Refinement
 │       │
 │       ▼
 │    Retrieval
 │
 └── sufficient
          │
          ▼
       Synthesis
          │
          ▼
        Answer
          │
          ▼
      Write Policy
          │
          ▼
        M_t+1


---

18. Execution Control

The initial architecture uses:

> Fixed workflow + bounded local branching.



For example:

Retrieve
   │
   ▼
Judge
   │
   ├── sufficient ──► Synthesize
   │
   └── insufficient ──► Refine
                           │
                           ▼
                       Retrieve

The graph declares permitted transitions.

There is no unrestricted planner deciding:

"Which of the 27 stages should I invoke next?"

This reduces coordination complexity and makes failures inspectable.


---

19. Why Stage Selection Remains Replaceable

The execution interface should be:

StageSelection
      │
      ▼
R(stage, contract, S_t)
      │
      ▼
Stage

Today:

Fixed Graph

Later:

Planner

Potentially later:

Learned Router

The state and context architecture remains unchanged.

This makes stage-selection policy an intentionally replaceable component.


---

20. Full Architecture

USER
                           │
                           ▼
                 ┌──────────────────┐
                 │ Session State    │
                 │ G_t / H_t        │
                 └────────┬─────────┘
                          │
                          ▼
                    Fixed Workflow
                          │
                          ▼
              ┌────────────────────────┐
              │   Context Resolver     │
              │ R(stage, contract,S_t) │
              └────────────┬───────────┘
                           │
                           ▼
                    Context Bundle
                           │
                           ▼
                    ┌────────────┐
                    │ LLM Stage  │
                    └─────┬──────┘
                          │
                          ▼
                    Stage Result
                          │
                          ▼
                    Working State
                         W_t
                          │
               ┌──────────┴──────────┐
               │                     │
               ▼                     ▼
         Workflow Branch        Write Policy
               │                     │
               │             ┌───────┼────────┐
               │             ▼       ▼        ▼
               │            ADD    UPDATE   DELETE
               │             │       │        │
               │             └───────┼────────┘
               │                     ▼
               │                    M_t
               │
               └──────────► Next Stage

The architecture therefore has four fundamental abstractions:

State
Stage
Context Contract
Context Resolver

And two independent policy domains:

Read / Context Policy
Mutation / Write Policy

The workflow controls execution order.


---

21. Architectural Invariants

The following should become implementation-level invariants.

I1 — Context isolation

A stage receives only information authorized by its contract.

I2 — Closed-world access

Unlisted state is unavailable.

I3 — Read purity

Context resolution cannot mutate state.

I4 — Persistent write mediation

No component directly mutates M_t outside the write policy.

I5 — Working-state ephemerality

W_t disappears unless explicitly promoted.

I6 — Retrieval/persistence separation

Retrieving information does not automatically persist it.

I7 — Workflow boundedness

Initial execution transitions are explicitly declared.

I8 — Policy replaceability

Stage-selection and context-ranking mechanisms can evolve without changing state semantics.

I9 — Permission independence

Read authorization and mutation authorization remain separate configurations.

I10 — Deterministic baseline

The initial system must have a deterministic mode that does not require learned policy.


---

22. Known Limitations

L1 — The state ontology remains a design choice

The three tiers are useful for implementation, but the literature does not establish them as a universal ontology.

A future system may require additional distinctions.

The architecture should therefore treat the tiers as interfaces, not metaphysical categories.


---

L2 — Stage boundaries are partly empirical

The definition of a stage is operational, but deciding whether two operations deserve separate LLM invocations remains a system-specific optimization.

Over-decomposition can increase:

latency;

token consumption;

orchestration complexity;

failure surface.



---

L3 — Deterministic context scoring may plateau

The initial resolver may fail to recognize complex dependencies between information items.

A learned selector may eventually outperform it.

The architecture deliberately postpones that decision.


---

L4 — Read/write coupling remains unresolved

Current research supports separate read and write mechanisms and has formalized their coupling.

The architecture does not claim to solve joint optimization of:

read → action → write

This remains an experimental research direction.


---

L5 — Memory correctness is unresolved

A write policy can still create:

false memories
stale memories
contradictory memories
self-reinforcing errors

Persistent memory therefore requires provenance and validation research.


---

L6 — Forgetting remains underspecified

The architecture supports deletion and update but does not yet define an optimal forgetting policy.

Questions remain around:

staleness
supersession
confidence decay
contradiction
privacy
memory utility decay


---

L7 — Token budgeting remains empirical

There is evidence for saturation and for stage-specific routing, but no universal budget formula.

Budget allocation must be measured against the target workload.


---

L8 — Shared state introduces concurrency concerns

If multiple stages eventually execute concurrently, state transitions require:

versioning
conflict detection
transaction semantics
write ordering

The current architecture assumes sequential state transitions.


---

L9 — Context provenance is not fully specified

The context bundle should eventually preserve enough metadata to determine:

source
retrieval mechanism
timestamp
memory identifier
confidence
transformation history

The exact provenance model remains a TODO.


---

23. Explicitly Deferred Research

The following should not enter the initial implementation unless experiments justify them.

R1 — Learned context selection

Replace deterministic ranking with learned decision utility.

R2 — Learned stage routing

Replace the fixed graph with a planner/router.

R3 — Joint read/action/write optimization

Investigate:

π_R + π_A + π_U

as a coupled policy.

R4 — Learned memory write policy

Compare deterministic, prompted, and RL-based memory management.

R5 — Uncertainty-gated persistent writes

Introduce value/uncertainty arbitration for irreversible memory operations.


---

24. TODO: Architecture Validation

The architecture needs empirical validation before further abstraction.

Experiment A — Global vs stage-specific context

Compare:

Global context
vs.
Stage-conditioned context

Control for:

task set;

model;

token budget;

retrieval corpus;

number of inference calls.


Measure:

accuracy
context tokens
latency
irrelevant-context rate
failure rate


---

Experiment B — Closed-world contracts

Compare:

default allow
vs.
default deny

Measure information leakage and maintenance failures after adding new state fields.


---

Experiment C — Context ranking

Compare:

random
recency
semantic similarity
deterministic utility
learned utility

under equal token budgets.


---

Experiment D — Stage granularity

Compare:

coarse stages
vs.
fine stages

Measure whether additional stages produce meaningful gains relative to their coordination cost.


---

Experiment E — Memory writes

Compare:

no memory
append-only memory
ADD/UPDATE/DELETE
learned memory policy

Measure long-horizon performance and memory quality.


---

Experiment F — Memory vs long context

Hold available token budget constant.

Compare:

long conversation context
vs.
persistent memory + selective retrieval

Measure decision quality rather than recall alone.


---

Experiment G — Read/write coupling

Eventually compare:

Independent read/write
vs.
coupled read/write
vs.
coupled read/action/write

This is the strongest research experiment in the program.


---

25. TODO: State Schema

Define concrete schemas for:

M_t
G_t
H_t
W_t

including:

identity
version
timestamp
provenance
confidence
validity
source
relationships

without prematurely forcing a graph, vector database, or document store.


---

26. TODO: Memory Representation

Benchmark:

vector records
structured records
claim-oriented records
graph representation
trajectory/filesystem representation
hybrid representation

against the actual workload.

The architecture should not hard-code the storage substrate.


---

27. TODO: Write Semantics

Define formal semantics for:

ADD
UPDATE
DELETE
NOOP

including conflict handling.

Example:

Existing:
"user prefers X"

New evidence:
"user now prefers Y"

UPDATE:
replace?

Version:
preserve both?

Supersession:
mark X obsolete?

Conflict:
retain both with provenance?

This is a major part of making persistent memory reliable.


---

28. TODO: Context Provenance

Every context item should eventually be traceable:

Context Item
     │
     ├── state source
     ├── retrieval method
     ├── memory ID
     ├── timestamp
     ├── relevance score
     └── transformation history

This allows debugging:

> "Why did this LLM receive this information?"



That question should have a deterministic answer.


---

29. TODO: Evaluation Harness

Every stage invocation should produce an execution trace containing at minimum:

execution_id
stage_id
contract_version
state_version
selected_context
omitted_candidates
model
prompt/version
tool calls
output
transition
state mutations
latency
token usage

This turns architecture evaluation into an observable system rather than a qualitative prompt comparison.


---

30. Final Architectural Position

The current architecture can be summarized as:

STATE
                    │
                    │
          ┌─────────▼─────────┐
          │ Context Contract  │
          └─────────┬─────────┘
                    │
                    ▼
             Context Resolver
             R(stage,C,S)
                    │
                    ▼
              Cognitive Stage
                    │
                    ▼
                 W_t
                    │
            ┌───────┴────────┐
            │                │
        Workflow          Write Policy
            │                │
            ▼                ▼
       Next Stage           M_t

The central architectural principle is:

> The agent maintains state; stages declare what they need; one resolver constructs stage-specific context; stages produce observations; a separate write policy controls persistence; the workflow controls which cognitive operation executes next.



The architecture intentionally leaves three dimensions replaceable:

Context selection
Stage selection
Memory write policy

The baseline uses deterministic implementations for all three.

That gives us a system that can be built and measured before attempting learned control.

The research frontier then becomes an experimental extension rather than an architectural dependency:

┌──────────────────────────┐
               │      Current System       │
               │                           │
               │ deterministic R           │
               │ fixed workflow            │
               │ governed U                │
               └────────────┬──────────────┘
                            │
                     empirical evidence
                            │
                            ▼
               ┌──────────────────────────┐
               │     Research System      │
               │                           │
               │ learned R                │
               │ learned routing          │
               │ joint R → A → U          │
               └──────────────────────────┘

That separation is important. The architecture is now stable enough to implement; the policies remain experimentally replaceable.