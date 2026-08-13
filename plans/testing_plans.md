Yes. Based on the architecture and the testing split we established, the final plan should cover **five test layers**, with explicit boundaries around what each layer is allowed to prove.

# Test Plan — RAG / Agent Memory Architecture

**Status:** Binding implementation specification
**Purpose:** Define what must be tested, where tests run, which dependencies they may use, and what conclusions each test is allowed to support.

---

## 1. Testing Objectives

The test system must establish that the architecture satisfies the following:

| Objective                       | What must be demonstrated                                                               |
| ------------------------------- | --------------------------------------------------------------------------------------- |
| **O1 — State isolation**        | State lifetime and mutation authority are enforced                                      |
| **O2 — Stage-specific context** | Stages receive only context appropriate to their contract                               |
| **O3 — Centralized resolution** | Context assembly occurs through the resolver rather than stage-specific retrieval logic |
| **O4 — Mutation authority**     | Read access does not imply write access                                                 |
| **O5 — Controlled persistence** | Working observations reach persistent memory only through explicit promotion            |
| **O6 — Replaceable policies**   | Deterministic policies can be replaced without changing architectural boundaries        |
| **O7 — Debuggability**          | Context inclusion, decisions, and mutations have attributable provenance                |
| **O8 — Controlled complexity**  | The baseline remains understandable without requiring learned routing or memory RL      |

The test suite must also distinguish:

> **Does the architecture work?**
> **Does the infrastructure work?**
> **Does the model behave as expected?**
> **Does the complete system produce useful outcomes?**

Those are different questions.

---

# 2. Test Dependency Levels

Tests are divided into five levels.

### T0 — Pure architectural tests

No external infrastructure.

Dependencies:

* language runtime
* domain classes
* deterministic functions
* fake state
* fake policies

Examples:

* state transitions
* contract validation
* permission enforcement
* budget calculation
* provenance construction
* write-operation semantics

These tests should constitute the largest part of the suite.

---

### T1 — Dependency-isolated tests

Use mocks/fakes for external interfaces.

Possible mocked components:

* `MemoryStore`
* vector retriever
* embedding provider
* LLM
* clock
* external tools

The purpose is to test **our orchestration and policy logic** without testing the implementation of the dependency.

Example:

```text
FakeMemoryStore
       ↓
ContextResolver
       ↓
Stage
```

This establishes whether the resolver behaves correctly given known retrieval results.

---

### T2 — Infrastructure integration tests

Use real infrastructure while keeping model behavior deterministic or controlled.

Examples:

* PostgreSQL
* vector database
* embedding service
* filesystem
* cache
* serialization layer

These tests answer:

> Does our abstraction correctly operate against the infrastructure?

They do **not** establish that the LLM reasons correctly.

---

### T3 — Model-dependent tests

Require a real LLM.

These tests evaluate properties that cannot meaningfully be tested with a mock because the model's behavior is itself the subject.

Examples:

* structured interpretation
* ambiguity detection
* query formulation
* retrieval assessment
* synthesis
* epistemic evaluation
* memory extraction
* correction proposals

The test harness should constrain the model through the same interfaces used in production.

---

### T4 — End-to-end system tests

Run the complete architecture:

```text
User
 ↓
Workflow
 ↓
Stage
 ↓
ContextResolver
 ↓
Retriever / Memory
 ↓
LLM
 ↓
Decision
 ↓
WriteProposal
 ↓
WritePolicy
 ↓
Persistent State
```

These tests evaluate **system-level hypotheses**.

They are slower, more expensive, and harder to diagnose. They should therefore be a small part of the suite.

---

# 3. Dependency Rule

A test should use the **lowest dependency level capable of proving its claim**.

For example:

### Question

> Does a Judge receive excluded memory?

Use:

```text
T0
```

There is no reason to involve an LLM.

### Question

> Does PostgreSQL correctly persist a promoted memory?

Use:

```text
T2
```

### Question

> Does the LLM correctly identify a contradiction?

Use:

```text
T3
```

### Question

> Does the complete system detect the contradiction, retrieve the right evidence, correct its memory, and produce a better answer?

Use:

```text
T4
```

This prevents infrastructure and model variability from contaminating architectural tests.

---

# 4. Component Test Matrix

## 4.1 `AgentState`

**Level:** T0

Test:

* creation
* tier separation
* session boundaries
* execution boundaries
* immutable/persistent state handling
* legal transitions
* illegal mutation attempts
* promotion boundaries

Important invariant:

```text
W_t → M_t
```

must never occur implicitly.

Only:

```text
WritePolicy
    ↓
PromotionDecision
    ↓
MemoryStore
```

may produce the transition.

---

# 5. Context Contracts

**Level:** T0

Test the contract independently of retrieval.

A contract should define at minimum:

```text
required
optional
excluded
tokenBudget
```

### Required tests

#### Required field

If a required dependency is unavailable:

```text
Resolver → failure
```

The stage must not silently execute with incomplete required context.

#### Optional field

Optional context may be omitted when:

* unavailable
* over budget
* below utility threshold

#### Closed-world access

The resolver must implement:

```text
undeclared ≡ unavailable
```

This is an important security and correctness invariant.

A newly added state field must **not automatically become visible** to existing stages.

#### Budget

Verify:

```text
selected_context_tokens <= tokenBudget
```

under all combinations of candidate inputs.

---

# 6. Context Resolver

**Level:** T0 / T1

The fundamental interface is:

```text
resolve(
    stage,
    contract,
    state
) → StageContext
```

The resolver must be tested for:

* required resolution
* optional ranking
* budget fitting
* exclusion
* provenance
* deterministic ordering
* duplicate elimination
* unavailable information
* conflicting candidates

### Purity invariant

Given identical:

```text
stage
contract
state
```

the resolver must not mutate state.

Formally:

```text
R(S) = C

S_after = S_before
```

This should be a direct architectural test.

---

# 7. Retrieval

Retrieval itself should be decomposed.

```text
CandidateGeneration
        ↓
Scoring
        ↓
Filtering
        ↓
BudgetSelection
```

Each layer should have T0/T1 tests.

### Candidate generation

Test with fake stores.

### Scoring

Test deterministic scoring independently.

For example:

```text
score(candidate, query, stage)
```

must be independently testable.

### Selection

Test:

* ranking
* thresholding
* token budget
* diversity
* duplicates

### Integration

Real vector databases belong in T2.

---

# 8. Stage Tests

Each cognitive stage receives its own contract and tests.

A stage test should verify:

```text
Stage
  receives correct context
  produces valid output
  does not mutate unauthorized state
```

The stage should be testable with an injected model interface:

```text
LLM
```

and therefore support:

```text
FakeLLM
```

for architectural tests.

---

# 9. Structured LLM Outputs

The LLM boundary should be tested separately from the reasoning quality.

### T1

Fake model returns:

```json
{
  "decision": "...",
  "confidence": 0.8
}
```

Test:

* schema validation
* malformed output
* missing fields
* invalid enum
* invalid values
* retry behavior

### T3

Real model tests:

* whether the model produces useful decisions
* whether it follows the schema
* whether its reasoning policy produces the desired behavior

This separation matters because:

> JSON validity is an interface property.
> Decision quality is a model/system property.

---

# 10. Workflow

The baseline workflow is:

```text
Fixed graph
+
bounded local branching
```

Workflow tests belong primarily at T0/T1.

Test:

* correct stage ordering
* branch conditions
* retry limits
* termination
* failure propagation
* state handoff
* repeated execution
* no accidental cycles

A workflow test should be able to execute with:

```text
FakeStage
FakeResolver
FakeRetriever
FakeLLM
```

without external infrastructure.

---

# 11. Write Policy

Write operations must be independently testable.

The canonical operation set is:

```text
ADD
UPDATE
DELETE
NOOP
```

Test:

### ADD

New information creates a memory only when policy permits.

### UPDATE

Existing information is modified according to explicit conflict semantics.

### DELETE

Deletion requires explicit authority.

### NOOP

Equivalent or insufficient information produces no mutation.

NOOP must receive substantial testing because it is a critical protection against memory pollution.

---

# 12. Memory Conflict Semantics

This remains an architectural TODO and must therefore have explicit test placeholders.

The implementation must eventually specify whether conflicts produce:

```text
replace
version
supersede
coexist
reject
```

Until that policy is finalized, the tests should establish the interface boundary without pretending the semantic choice has been settled.

---

# 13. Persistence Boundary

Test:

```text
W_t
```

independently from:

```text
M_t
```

The key invariant:

```text
tool result
     ↓
W_t
```

does **not** imply:

```text
M_t
```

Only an accepted promotion does.

Test cases:

1. observation remains ephemeral
2. rejected promotion remains ephemeral
3. accepted promotion becomes persistent
4. execution termination discards unpromoted state
5. persistent state survives execution/session boundaries

---

# 14. Read Authority vs Mutation Authority

These require separate test suites.

### Read authority

Test:

```text
Stage → ContextContract → Resolver → State
```

### Mutation authority

Test:

```text
Stage → Proposal → WritePolicy → State
```

A stage may have:

```text
READ(M_t) = allowed
WRITE(M_t) = denied
```

and the architecture must enforce that distinction.

The two permission systems must remain independently configurable and testable.

---

# 15. Provenance

Every context inclusion should produce provenance.

Minimum chain:

```text
source
 ↓
candidate
 ↓
score
 ↓
selection
 ↓
stage context
```

Every persistent mutation should produce:

```text
observation
 ↓
proposal
 ↓
policy decision
 ↓
mutation
```

Tests must establish that the chain remains reconstructible.

A provenance record should answer:

> Why was this information available here?

and:

> Why did this information become persistent?

---

# 16. Epistemic / Reasoning Policy

The reasoning constitution should have its own tests.

These should remain separate from context authorization.

### Information governance

Answers:

```text
What may the stage see?
```

### Epistemic governance

Answers:

```text
How should the stage evaluate what it sees?
```

This separation should be reflected in the test architecture.

A stage may have identical context while receiving different epistemic policies.

Conversely, the same epistemic policy may operate over different stage contexts.

---

# 17. Memory Quality Tests

These require T3/T4 because model behavior becomes part of the subject.

Evaluate:

* factuality
* provenance retention
* contradiction detection
* stale-memory detection
* unnecessary memory creation
* memory correction
* duplicate suppression
* contextual validity
* unsupported inference

Particular attention should go to:

```text
high-confidence / poorly-supported memory
```

because scalar confidence alone does not establish epistemic adequacy.

---

# 18. Cross-Execution Tests

These are system-level tests.

Example:

### Execution 1

```text
observe
→ reason
→ produce answer
→ create memory proposal
→ promote
```

### Execution 2

```text
retrieve memory
→ reason with memory
→ discover contradiction
→ update memory
```

### Execution 3

```text
retrieve corrected memory
→ improved reasoning
```

The test asks whether memory produces measurable improvement across executions.

This is one of the strongest tests of the architecture's intended purpose.

---

# 19. Model Evaluation

Model-dependent tests should use controlled fixtures.

A test fixture should define:

```text
task
initial state
available evidence
expected behavioral property
evaluation criteria
```

Avoid tests whose expected result is merely:

```text
exact string equality
```

for reasoning tasks.

Prefer properties such as:

```text
must cite evidence
must identify contradiction
must not invent evidence
must distinguish observation from inference
must produce valid structured output
```

---

# 20. Infrastructure Tests

Each external dependency gets its own T2 suite.

Examples:

```text
MemoryStore
VectorStore
EmbeddingProvider
LLMProvider
Cache
Filesystem
```

The architectural suite should never depend on these being available.

Infrastructure tests establish that the adapters correctly implement the architecture's interfaces.

---

# 21. End-to-End Tests

The E2E suite should remain intentionally small.

Its purpose is to validate interactions that isolated tests cannot establish.

Example:

```text
User Query
    ↓
Interpretation
    ↓
Query Construction
    ↓
Retrieval
    ↓
Context Resolution
    ↓
RAG Generation
    ↓
Evaluation
    ↓
Memory Proposal
    ↓
Write Policy
    ↓
Persistent Memory
```

The E2E suite should test representative architectural scenarios rather than exhaustively test individual branches.

---

# 22. Failure-Injection Tests

The architecture should deliberately test failure.

Inject:

* unavailable memory
* empty retrieval
* contradictory documents
* malformed LLM output
* invalid write proposal
* vector-store failure
* timeout
* context-budget overflow
* missing required context
* unauthorized mutation
* stale memory
* duplicate memory
* model refusal/failure

The objective is to establish **controlled degradation**.

---

# 23. Determinism Tests

Deterministic components should produce reproducible results.

Given:

```text
S
contract
candidate set
budget
policy configuration
```

the resolver should produce the same result.

Likewise:

```text
same WriteProposal
+
same memory state
+
same policy
```

should produce the same mutation decision.

This provides a stable foundation underneath inherently stochastic LLM behavior.

---

# 24. Observability Tests

Observability itself requires testing.

A trace should permit reconstruction of:

```text
Input
 ↓
State snapshot
 ↓
Context contract
 ↓
Candidates
 ↓
Scores
 ↓
Selected context
 ↓
Stage invocation
 ↓
Stage output
 ↓
Decision
 ↓
Write proposal
 ↓
Write decision
 ↓
Mutation
```

Test that required trace information survives failure paths as well as successful paths.

---

# 25. Test Pyramid

The resulting distribution should look approximately like:

```text
                    ┌───────────────┐
                    │     T4        │
                    │  End-to-End   │
                    └───────────────┘
                 ┌─────────────────────┐
                 │        T3           │
                 │   Model Behavior   │
                 └─────────────────────┘
              ┌───────────────────────────┐
              │            T2             │
              │ Infrastructure Adapters  │
              └───────────────────────────┘
          ┌───────────────────────────────────┐
          │                T1                 │
          │     Mocked Integration Tests      │
          └───────────────────────────────────┘
 ┌─────────────────────────────────────────────────┐
 │                       T0                        │
 │          Pure Architectural / Policy Tests     │
 └─────────────────────────────────────────────────┘
```

The base should be large.

The top should remain small.

---

# 26. What Each Test Level Can Claim

| Level  | Can establish                                    | Cannot establish                                                   |
| ------ | ------------------------------------------------ | ------------------------------------------------------------------ |
| **T0** | Architecture, invariants, deterministic policies | LLM quality                                                        |
| **T1** | Component interaction and orchestration          | Real infrastructure behavior                                       |
| **T2** | Adapter/infrastructure correctness               | Model reasoning                                                    |
| **T3** | Model behavior under controlled conditions       | Complete system behavior                                           |
| **T4** | System-level behavior                            | Which internal component caused an outcome without instrumentation |

This distinction should be preserved in test reports.

---

# 27. Required Test Fixtures

The implementation should maintain reusable fixtures for:

### State

```text
EmptyState
NormalState
ConflictingState
StaleState
LargeState
```

### Memory

```text
NoMemory
RelevantMemory
IrrelevantMemory
DuplicateMemory
ConflictingMemory
StaleMemory
```

### Retrieval

```text
NoResults
PerfectResults
MixedResults
MisleadingResults
OverBudgetResults
```

### LLM

```text
ValidResponse
MalformedResponse
LowConfidenceResponse
ContradictoryResponse
UnsupportedClaimResponse
```

These fixtures allow T0–T3 tests to reuse identical scenarios.

---

# 28. Required Acceptance Gates

Implementation should proceed through gates.

### Gate 1 — Architectural core

Must pass:

* state isolation
* contract enforcement
* resolver purity
* mutation authority
* persistence boundary
* provenance

No LLM or vector database required.

### Gate 2 — Policy layer

Must pass:

* retrieval policies
* budget policies
* write operations
* conflict handling
* workflow branching

Mocks sufficient.

### Gate 3 — Infrastructure

Must pass:

* memory persistence
* vector retrieval
* embeddings
* adapter contracts

### Gate 4 — Model integration

Must pass:

* structured outputs
* stage behavior
* reasoning policies
* retrieval assessment
* memory extraction

### Gate 5 — System behavior

Must demonstrate:

* useful retrieval
* appropriate context
* correct persistence
* correction of memory
* provenance
* cross-execution behavior

---

# 29. Research Experiments vs Product Tests

These should not be mixed.

## Product correctness

Tests whether the architecture does what it specifies.

Examples:

```text
Does excluded context remain inaccessible?
Does W_t disappear?
Can an unauthorized stage mutate M_t?
Does the resolver respect tokenBudget?
```

## Research experiments

Test whether an architectural choice improves performance.

Examples:

```text
Does stage-conditioned context outperform global context?
Does coupled read/write improve memory quality?
Does epistemic governance improve calibration?
Does memory produce compounding improvement?
Does learned selection outperform deterministic selection?
```

A research hypothesis failing does **not** necessarily mean the implementation is broken.

---

# 30. Deferred Research Tests

The architecture deliberately leaves room for later experiments involving:

* learned context selection
* learned memory management
* planner-based routing
* joint read/action/write policies
* uncertainty-gated writes
* richer causal memory
* alternative memory representations
* adaptive stage decomposition

These must enter the test suite as **comparative experiments**, rather than becoming prerequisites for the baseline implementation.

---

# 31. Final Architectural Testing Principle

The test architecture should mirror the architecture itself:

```text
                 SYSTEM
                    │
             ┌──────┴──────┐
             │             │
          POLICY        STATE
             │             │
        ┌────┴────┐    ┌───┴────┐
        │         │    │        │
      READ      WRITE   G/H/W   M
        │         │
        └────┬────┘
             │
          STAGES
             │
       CONTEXT RESOLVER
             │
        EXTERNAL WORLD
        /      |       \
     LLM    Vector    Tools
```

The corresponding testing rule is:

> **Test each architectural boundary without its downstream dependencies first. Then test the boundary against the real dependency. Finally test the composition.**

That gives us a clean progression:

```text
Architecture correctness
        ↓
Policy correctness
        ↓
Adapter correctness
        ↓
Model behavior
        ↓
System behavior
        ↓
Research evaluation
```

This is the appropriate final testing strategy for the implementation: **the architecture can be built and substantially validated before an LLM or vector database is involved, while the parts whose hypothesis is inherently about model behavior are explicitly deferred to model-dependent tests.**
