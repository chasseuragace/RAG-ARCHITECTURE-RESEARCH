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

2. Design Objectives

The architecture should provide:

O1 — State isolation

Different classes of state must have explicit lifetime and mutation semantics.

O2 — Stage-specific context

Each cognitive operation receives a context appropriate to its objective.

O3 — Centralized context resolution

Stages should declare information requirements rather than implement independent retrieval mechanisms.

O4 — Explicit mutation authority

Persistent memory cannot be modified merely because a stage can read it.

O5 — Controlled persistence

Transient observations become persistent memories only through an explicit promotion mechanism.

O6 — Replaceable policies

Deterministic mechanisms should be replaceable with learned policies without restructuring the state architecture.

O7 — Debuggability

The system should expose enough intermediate state to explain:

why something was retrieved
why it was presented
why a stage acted
why something became memory

O8 — Controlled complexity

The architecture should not introduce a planner, learned router, memory RL system, or additional LLM invocation until an observed failure justifies it.


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