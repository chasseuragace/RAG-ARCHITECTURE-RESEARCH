# Architecture Binding Specification

## Stateful RAG + Agentic Memory + Epistemic Governance

**Status:** Implementation Binding Document
**Version:** 0.1
**Purpose:** Define the architectural boundaries, interfaces, abstract policies, and invariants that implementation must preserve.

---

## D1. Purpose

This document defines the **interfaces and architectural abstractions** for the system we have been designing.

It deliberately separates:

* what the system **is allowed to do**
* what a particular implementation **currently does**
* what future research may replace

The implementation begins with deterministic policies wherever possible. The interfaces must remain stable enough to permit later heuristic, LLM-driven, or learned implementations.

### Binding principle

> **Defer policy sophistication, not architectural boundaries.**

An interface may exist before its advanced implementation exists.

---

# D2. Architectural Model

The system consists of five cooperating planes:

```text
                         ┌───────────────────────┐
                         │     User Objective    │
                         └───────────┬───────────┘
                                     │
                                     ▼
                         ┌───────────────────────┐
                         │   Execution Engine    │
                         │  Workflow / Routing   │
                         └───────────┬───────────┘
                                     │
                          selects / invokes
                                     │
                                     ▼
                         ┌───────────────────────┐
                         │        Stage          │
                         │                       │
                         │ Interpreter           │
                         │ Retrieval             │
                         │ Evaluation             │
                         │ Synthesis              │
                         │ etc.                  │
                         └───────────┬───────────┘
                                     │
                              requests context
                                     │
                                     ▼
                  ┌─────────────────────────────────────┐
                  │          Context Resolver           │
                  │                                     │
                  │ Contract + Policies + State         │
                  └──────────────┬──────────────────────┘
                                 │
                    reads        │        reads
                 ┌───────────────┴────────────────┐
                 ▼                                ▼
        ┌─────────────────┐              ┌─────────────────┐
        │  Agent State    │              │ External World  │
        │                 │              │                 │
        │ M  Persistent   │              │ RAG corpus      │
        │ G  Session      │              │ Tools           │
        │ W  Working     │              │ APIs            │
        └────────┬────────┘              └─────────────────┘
                 │
                 │ proposals
                 ▼
        ┌─────────────────────┐
        │    Write Policy     │
        │                     │
        │ ADD / UPDATE /      │
        │ DELETE / NOOP       │
        └─────────┬───────────┘
                  │
                  ▼
        ┌─────────────────────┐
        │ Persistent Memory   │
        └─────────────────────┘
```

---

# D3. Core Design Objectives

The implementation SHALL preserve these objectives.

### O1 — State isolation

Different state classes have explicit lifetime and mutation semantics.

### O2 — Stage-specific context

Each cognitive stage receives context appropriate to its objective.

### O3 — Centralized context resolution

Stages declare information requirements rather than implementing independent retrieval mechanisms.

### O4 — Explicit mutation authority

Read access does not imply mutation authority.

### O5 — Controlled persistence

Working observations become persistent memory only through an explicit promotion/write mechanism.

### O6 — Replaceable policies

Deterministic policies can be replaced without restructuring state or execution architecture.

### O7 — Debuggability

The system exposes sufficient provenance to explain:

* why something was retrieved
* why it was included
* why a stage acted
* why a memory mutation occurred

### O8 — Controlled implementation complexity

Advanced policies remain replaceable and inactive until evaluation demonstrates their necessity.

---

# D4. Agent State

Agent state is defined along two independent dimensions:

1. **Lifetime**
2. **Mutation authority**

The canonical state model contains three tiers.

```text
AgentState
│
├── PersistentState M_t
│
├── SessionState G_t / H_t
│
└── WorkingState W_t
```

---

## D4.1 Persistent State — `M_t`

`M_t` survives execution and session boundaries.

Examples:

* durable facts
* validated procedures
* learned experiences
* architectural decisions
* user/project preferences
* corrections
* accumulated evidence

Persistent memory MUST NOT be mutated by ordinary stage execution.

Mutation occurs through the memory write boundary.

---

## D4.2 Session State — `G_t / H_t`

Session state survives individual stage calls but terminates with the session.

Examples:

* current objective
* conversation history
* interpreted intent
* current subgoals
* session decisions
* active constraints

Session state may have more permissive mutation semantics than persistent memory.

---

## D4.3 Working State — `W_t`

Working state exists for the current execution.

Examples:

* retrieval candidates
* failed retrieval attempts
* hypotheses
* intermediate observations
* temporary rankings
* tool results
* unresolved contradictions
* intermediate reasoning artifacts

`W_t` MUST NOT automatically become persistent memory.

Explicit promotion is required.

---

# D5. State Interface

Conceptually:

```dart
abstract interface class AgentState {
  PersistentState persistent();
  SessionState session();
  WorkingState working();

  StateSnapshot snapshot();
}
```

The implementation may use another language or concrete representation.

The semantic boundary remains binding.

### Required invariant

```text
Stage
   │
   ├── read → AgentState
   │
   └── write → proposal
                    │
                    ▼
              WritePolicy
                    │
                    ▼
             State mutation
```

A stage does not directly mutate persistent memory.

---

# D6. Context Contract

A stage declares its information requirements through a `ContextContract`.

The contract is a **capability declaration**, not a retrieval implementation.

```text
ContextContract
├── required
├── optional
├── tokenBudget
└── constraints
```

The read model follows **closed-world authorization**.

> Information reaches a stage only when the contract explicitly authorizes it.

This means newly introduced state fields do not silently become visible to existing stages.

---

## D6.1 Required Context

Required information represents a hard dependency.

```text
required:
  - current_goal
  - query_interpretation
  - retrieval_results
```

The resolver must satisfy these requirements or return an explicit failure.

---

## D6.2 Optional Context

Optional information may be selected according to policy.

```text
optional:
  - relevant_memories
  - historical_conversation
  - prior_failures
  - supporting_evidence
```

Optional context competes for a token budget.

---

## D6.3 Excluded Context

`excluded` MAY be retained as an explicit defense-in-depth mechanism.

However, exclusion does not define authorization.

Authorization comes from declaration.

```text
allowed = required ∪ optional
```

Everything else is inaccessible.

---

# D7. Context Resolver

The central read abstraction is:

```text
R(stage, contract, state) → Context
```

The resolver is responsible for projecting global agent state into stage-specific context.

### Interface

```dart
abstract interface class ContextResolver {
  Future<ContextResolution> resolve(
    ContextRequest request,
  );
}
```

Conceptually:

```dart
class ContextRequest {
  StageId stage;
  ContextContract contract;
  AgentStateSnapshot state;
  ExecutionContext execution;
}
```

The resolver MUST NOT mutate agent state.

---

# D8. Context Resolution Pipeline

The resolver may internally perform:

```text
Contract
   │
   ▼
Authorized sources
   │
   ▼
Candidate generation
   │
   ▼
Filtering
   │
   ▼
Scoring
   │
   ▼
Budget fitting
   │
   ▼
Provenance attachment
   │
   ▼
Context
```

The architecture does not mandate a particular retrieval algorithm.

Possible implementations include:

```text
Deterministic
    ↓
Hybrid heuristic
    ↓
LLM-assisted
    ↓
Learned
```

The resolver interface remains constant.

---

# D9. Retrieval Policy

Retrieval is a policy inside context resolution.

```dart
abstract interface class RetrievalPolicy {
  Future<RetrievalResult> retrieve(
    RetrievalRequest request,
  );
}
```

The policy may retrieve from:

* RAG indexes
* persistent memory
* session history
* structured stores
* graph stores
* external knowledge

The policy MUST return provenance-bearing candidates.

---

# D10. Ranking Policy

Retrieval and ranking are separate concerns.

```dart
abstract interface class RankingPolicy {
  Future<RankedCandidates> rank(
    RankingRequest request,
  );
}
```

Initial implementation may use:

```text
semantic similarity
+ lexical relevance
+ entity match
+ recency
+ source quality
+ stage relevance
```

Future implementations may use decision utility or learned ranking.

---

# D11. Budget Policy

Context construction must explicitly account for the stage's context budget.

```dart
abstract interface class BudgetPolicy {
  ContextSelection fitBudget(
    Iterable<ContextCandidate> candidates,
    TokenBudget budget,
  );
}
```

The policy determines which authorized optional information survives budget fitting.

Required context has separate semantics and cannot be silently discarded.

---

# D12. Provenance

Every context item must carry provenance.

Conceptually:

```dart
class Provenance {
  SourceId source;
  RetrievalMethod method;
  MemoryId? memoryId;
  DateTime timestamp;
  double? score;

  List<Transformation> transformations;
}
```

The architecture should support answering:

```text
Why was this item available?
Why was it retrieved?
Why was it selected?
Which policy selected it?
Which stage received it?
```

---

# D13. Stage

A stage represents a cognitive operation where an LLM or equivalent policy exercises meaningful discretion.

A deterministic transformation should remain ordinary code.

```dart
abstract interface class Stage {
  StageId get id;

  ContextContract get contextContract;

  Future<StageResult> execute(
    StageExecutionRequest request,
  );
}
```

A stage owns:

* its objective
* its context contract
* its local decision
* its output schema

A stage does **not** own global retrieval or persistent-memory mutation.

---

# D14. Stage Objective

Each stage has an explicit objective.

```dart
class StageObjective {
  String description;
  SuccessCriterion criterion;
}
```

This is important because context selection should ultimately optimize against **what the stage is trying to accomplish**.

---

# D15. Stage Selection

Stage selection is an independent architectural boundary.

```dart
abstract interface class StageSelectionPolicy {
  Future<StageSelection> select(
    StageSelectionRequest request,
  );
}
```

### Initial implementation

```text
FixedWorkflowPolicy
```

with bounded conditional edges.

Example:

```text
Interpret
   ↓
Retrieve
   ↓
Evaluate
   │
   ├── insufficient evidence → Retrieve
   │
   └── sufficient → Synthesize
```

### Future implementations

```text
Planner
LLM Router
Learned Router
```

The execution architecture does not change.

---

# D16. Workflow

The workflow defines permitted execution topology.

```dart
abstract interface class Workflow {
  WorkflowId get id;

  StageGraph graph();

  ExecutionRules executionRules();
}
```

The initial system uses a fixed graph.

A future planner may choose paths within a constrained graph.

This prevents a future routing policy from gaining unrestricted authority merely because it exists.

---

# D17. Epistemic Governance

Information governance determines **what the stage sees**.

Epistemic governance determines **how the stage should reason about what it sees**.

These remain separate.

```text
Information Governance
        │
        ▼
Context Resolver
        │
        ▼
Stage Context
        │
        ▼
Epistemic Governance
        │
        ▼
Stage Decision
```

---

# D18. Epistemic Policy

The epistemic policy is a replaceable abstraction governing reasoning behavior.

```dart
abstract interface class EpistemicPolicy {
  Future<EpistemicAssessment> assess(
    EpistemicRequest request,
  );
}
```

It may evaluate:

* evidence adequacy
* source dependence
* contradiction
* causal dependencies
* inference strength
* uncertainty
* missing evidence
* provenance
* consistency

The initial implementation may be prompt-defined and deterministic at the orchestration level.

Future implementations may become learned.

---

# D19. Evidence Policy

Evidence assessment deserves an explicit boundary.

```dart
abstract interface class EvidencePolicy {
  Future<EvidenceAssessment> assess(
    EvidenceRequest request,
  );
}
```

This allows the system to distinguish:

```text
retrieved
        ↓
relevant
        ↓
supporting
        ↓
sufficient
        ↓
adequate for claim
```

These are different decisions.

---

# D20. Memory Interface

Persistent memory exposes storage semantics without exposing policy.

```dart
abstract interface class MemoryStore {
  Future<List<MemoryRecord>> search(
    MemoryQuery query,
  );

  Future<MemoryRecord?> get(
    MemoryId id,
  );

  Future<MemoryMutationResult> apply(
    MemoryMutation mutation,
  );
}
```

The store does not decide **whether** a mutation should happen.

The policy decides that.

---

# D21. Memory Write Policy

Memory mutation is explicitly policy-controlled.

```dart
abstract interface class MemoryWritePolicy {
  Future<MemoryWriteDecision> decide(
    MemoryWriteRequest request,
  );
}
```

The decision space is:

```text
ADD
UPDATE
DELETE
NOOP
```

The policy receives:

```text
current memory
+
new observation
+
provenance
+
epistemic assessment
+
existing conflicts
+
promotion request
```

and produces a proposal.

---

# D22. Promotion Policy

Working observations require an explicit persistence boundary.

```dart
abstract interface class PromotionPolicy {
  Future<PromotionDecision> evaluate(
    PromotionRequest request,
  );
}
```

The conceptual flow:

```text
W_t
 │
 │ observation
 ▼
PromotionPolicy
 │
 ├── reject
 │
 └── promote
       │
       ▼
MemoryWritePolicy
       │
       ▼
M_t
```

This prevents retrieval results, hypotheses, and temporary tool outputs from becoming durable memories through accidental state persistence.

---

# D23. Conflict Resolution

Persistent memory requires explicit conflict semantics.

```dart
abstract interface class ConflictResolutionPolicy {
  Future<ConflictResolution> resolve(
    ConflictRequest request,
  );
}
```

Possible semantics:

```text
replace
supersede
version
retain-both
reject
request-more-evidence
```

The architecture does not yet mandate one.

This is an implementation TODO.

---

# D24. Forgetting

Forgetting is a first-class policy boundary.

```dart
abstract interface class ForgettingPolicy {
  Future<ForgettingDecision> evaluate(
    ForgettingRequest request,
  );
}
```

It may eventually consider:

* staleness
* redundancy
* contradiction
* access frequency
* confidence
* provenance quality
* supersession
* utility

The initial implementation can be conservative and inactive.

---

# D25. Execution Policy

Execution itself should expose policy boundaries.

```dart
abstract interface class RetryPolicy {
  RetryDecision decide(
    RetryContext context,
  );
}

abstract interface class TerminationPolicy {
  TerminationDecision decide(
    TerminationContext context,
  );
}
```

This prevents retry loops and termination logic from becoming hidden inside stages.

---

# D26. Decision Record

Every significant policy decision should generate a traceable decision record.

```dart
class DecisionRecord {
  DecisionId id;

  ActorId actor;
  PolicyId policy;

  DecisionType type;

  List<Reason> reasons;
  List<EvidenceReference> evidence;

  DateTime timestamp;

  StateVersion stateVersion;
  ContractVersion? contractVersion;
}
```

Examples:

```text
CONTEXT_INCLUDED
CONTEXT_EXCLUDED
STAGE_SELECTED
RETRIEVAL_ACCEPTED
RETRIEVAL_REJECTED
MEMORY_ADD
MEMORY_UPDATE
MEMORY_DELETE
MEMORY_NOOP
PROMOTION_ACCEPTED
PROMOTION_REJECTED
RETRY
TERMINATE
```

---

# D27. End-to-End Provenance

The complete reasoning path becomes:

```text
Memory
   │
   ▼
Candidate Generation
   │
   ▼
Retrieval Policy
   │
   ▼
Ranking Policy
   │
   ▼
Context Contract
   │
   ▼
Budget Policy
   │
   ▼
Context
   │
   ▼
Stage
   │
   ▼
Epistemic Assessment
   │
   ▼
Stage Decision
   │
   ▼
Observation
   │
   ▼
Promotion Policy
   │
   ▼
Memory Write Policy
   │
   ▼
ADD / UPDATE / DELETE / NOOP
   │
   ▼
Persistent Memory
```

Each transition should be observable.

---

# D28. Policy Abstraction

All replaceable policies should conform to a common conceptual model.

```dart
abstract interface class Policy<I, O> {
  Future<O> evaluate(I input);
}
```

Concrete policies should carry:

```text
PolicyId
Version
Configuration
Decision trace
```

This allows:

```text
DeterministicPolicy v1
        ↓
HeuristicPolicy v2
        ↓
LLMPolicy v3
        ↓
LearnedPolicy v4
```

without changing consumers.

---

# D29. LLM Invocation

An LLM invocation is an implementation mechanism.

It should not itself define architecture.

```dart
abstract interface class LanguageModel {
  Future<ModelResponse> generate(
    ModelRequest request,
  );
}
```

A stage may use an LLM.

A policy may use an LLM.

A future learned policy may use another model.

The surrounding architectural interfaces remain independent.

---

# D30. Structured Outputs

Stages and policies should communicate through typed contracts.

```text
LLM
 ↓
structured output
 ↓
schema validation
 ↓
domain object
```

The system should avoid allowing free-form model output to cross architectural boundaries.

---

# D31. Memory and RAG Relationship

RAG and memory remain distinct concepts.

```text
RAG
 └── external knowledge retrieval

Memory
 └── persistent agent state
```

Both may participate in context resolution.

They therefore share the **read boundary**, while retaining independent storage and mutation semantics.

```text
                 ContextResolver
                /              \
               ▼                ▼
         RAG Retrieval     Memory Retrieval
               \                /
                └──────┬───────┘
                       ▼
                    Context
```

---

# D32. Conversation History

Conversation history belongs primarily to session state.

```text
G_t / H_t
   │
   └── conversation history
```

The context resolver determines which portion becomes visible to a stage.

Therefore:

```text
conversation history ≠ automatically injected context
```

The resolver decides stage-specific inclusion.

---

# D33. Memory of Reasoning

The architecture supports persistent memories about reasoning itself.

Examples:

```text
successful approach
failed approach
known ambiguity
validated assumption
previous correction
known contradiction
domain procedure
```

Such memories remain subject to the same promotion and write policies as factual memories.

---

# D34. Architectural Invariants

Implementation SHALL preserve these invariants.

### I1 — Read does not mutate

```text
ContextResolver(state) → Context
```

does not modify state.

### I2 — Stage does not directly mutate persistent memory

Stages produce observations or proposals.

### I3 — Working state is ephemeral

`W_t` is discarded at execution termination unless explicitly promoted.

### I4 — Persistent writes are policy-mediated

```text
M_t → WritePolicy → Mutation
```

### I5 — Context is capability-controlled

Undeclared state is inaccessible to a stage.

### I6 — Provenance accompanies decisions

Context inclusion and persistent mutation have attributable causes.

### I7 — Policies are replaceable

Policy implementation must not leak into state representation.

### I8 — Workflow and policy are separate

Changing routing policy must not require rewriting stages.

### I9 — Epistemic and information governance remain separate

What the system presents and how the system evaluates it are distinct concerns.

### I10 — Advanced policies are optional

The architecture remains operational with deterministic implementations.

---

# D35. Initial Policy Set

Tomorrow's implementation should begin with:

| Boundary                         | Initial implementation              |
| -------------------------------- | ----------------------------------- |
| Stage selection                  | Fixed workflow + bounded branches   |
| Retrieval                        | Deterministic hybrid retrieval      |
| Ranking                          | Deterministic scoring               |
| Context resolution               | Contract-driven resolver            |
| Budget                           | Deterministic budget fitting        |
| Epistemic assessment             | Structured LLM output               |
| Memory write                     | Deterministic policy + LLM proposal |
| Promotion                        | Explicit threshold/rule             |
| Conflict resolution              | Conservative initial rule           |
| Forgetting                       | Explicitly disabled or conservative |
| Policy learning                  | Interface only                      |
| Learned routing                  | Interface only                      |
| Joint read/action/write learning | Interface/research boundary only    |

This gives us a sophisticated architecture without requiring sophisticated policies on day one.

---

# D36. Deferred Components

The following interfaces should exist before implementation, while their sophisticated implementations remain deferred:

```text
StageSelectionPolicy
RetrievalPolicy
RankingPolicy
BudgetPolicy
EpistemicPolicy
EvidencePolicy
MemoryWritePolicy
PromotionPolicy
ConflictResolutionPolicy
ForgettingPolicy
RetryPolicy
TerminationPolicy
PolicyOptimizer
```

The existence of these interfaces does **not** commit the implementation to LLM agents, RL, or learned policies.

---

# D37. Future Learning Boundary

The architecture should eventually permit:

```text
                  Agent State
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
       Read π_R     Action π_A   Write π_U
          │           │           │
          └───────────┼───────────┘
                      ▼
                 State transition
```

The architecture does not assume that these policies should eventually be jointly learned.

It merely ensures that such an experiment can be conducted without redesigning the system.

---

# D38. Implementation Rule

The implementation team should follow this rule:

> **Concrete classes implement policies; they do not redefine architectural responsibilities.**

For example:

```text
Bad:

RetrievalService
 ├── retrieves documents
 ├── decides context
 ├── updates memory
 ├── decides retry
 └── invokes generator
```

Preferred:

```text
RetrievalPolicy
ContextResolver
MemoryWritePolicy
RetryPolicy
Stage
Workflow
```

Each boundary has one architectural responsibility.

---

# D39. What This Document Does Not Decide

The following remain deliberate research/implementation decisions:

1. Exact memory representation: vector, graph, relational, hybrid.
2. Exact conflict-resolution semantics.
3. Memory forgetting algorithm.
4. Learned versus prompted epistemic governance.
5. Learned versus deterministic context ranking.
6. Whether read/action/write policies should eventually be jointly optimized.
7. Whether stage selection should eventually become planner-driven.
8. Exact provenance schema.
9. Exact adequacy representation.
10. Exact Spinoza-derived epistemic formulation.
11. Optimal token-budget allocation.
12. Whether additional cognitive stages improve performance enough to justify their complexity.

These decisions must be evaluated against evidence rather than embedded as architectural assumptions.

---

# D40. Implementation Acceptance Test

An implementation conforms to this specification if the following experiment can be performed without architectural restructuring:

```text
v1
Deterministic retrieval
Deterministic routing
Prompted epistemic assessment
Rule-based memory writes
```

can become:

```text
v2
Decision-aware retrieval
Prompted routing
LLM memory management
```

and eventually:

```text
v3
Learned retrieval
Learned stage selection
Learned epistemic policy
Joint read/action/write experiments
```

while preserving:

```text
AgentState
ContextContract
ContextResolver
Stage
Workflow
MemoryStore
Policy interfaces
Provenance
```

as the stable architectural substrate.

---

# C1. Binding Summary

The system is therefore defined around **state, context, stages, policies, and provenance**.

```text
                         AGENT
                           │
          ┌────────────────┼────────────────┐
          │                │                │
        STATE           WORKFLOW         POLICIES
          │                │                │
      M / G / W       stage selection    read/write/
          │                │             epistemic/etc.
          │                ▼                │
          │              STAGE              │
          │                │                │
          └───────────────►│◄───────────────┘
                           │
                           ▼
                    CONTEXT RESOLVER
                           │
                           ▼
                       DECISION
                           │
                           ▼
                       OBSERVATION
                           │
                           ▼
                     PROMOTION
                           │
                           ▼
                    MEMORY WRITE
                           │
                           ▼
                          M_t
```

The central architectural boundary is:

> **Stages request context. They do not construct their own world.**

The second is:

> **Stages produce decisions and observations. They do not decide what becomes durable memory.**

The third is:

> **Policies may become increasingly intelligent without changing the state and execution architecture beneath them.**

And the fourth is the research boundary:

> **The architecture defines where intelligence may evolve; experiments determine whether greater intelligence at that boundary is warranted.**

**This document should be treated as the binding contract for tomorrow's implementation.**
