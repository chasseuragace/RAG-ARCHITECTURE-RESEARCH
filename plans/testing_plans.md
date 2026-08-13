# Testing Architecture Specification

## Isolation, Contract, Integration, and Model-Dependent Testing

**Status:** Implementation Binding Document
**Version:** 0.1
**Scope:** Testing strategy for the architecture defined in the Architecture Binding Specification.

---

# D1. Purpose

The system contains components with radically different testing requirements.

Some components are ordinary deterministic software and should be tested without an LLM, vector database, network, or external service.

Some components represent boundaries around those systems and can be tested with mocks.

Other components exist specifically to evaluate model behavior. Their meaningful tests require an actual LLM, retrieval infrastructure, or persistent memory system.

The testing architecture must distinguish these cases explicitly.

> **A component should be tested at the lowest layer that can establish the property being tested.**

A model should not be involved merely because the production system uses one.

Conversely, a mock must not be treated as evidence for behavior that exists specifically because of the model.

---

# D2. Testing Model

We divide testing into five levels.

```text
                         ┌──────────────────────┐
                         │   System Evaluation  │
                         │ Real model + infra   │
                         └──────────┬───────────┘
                                    │
                         ┌──────────▼───────────┐
                         │ Integration Testing  │
                         │ Real boundaries      │
                         └──────────┬───────────┘
                                    │
                         ┌──────────▼───────────┐
                         │ Contract Testing     │
                         │ Mocks / fakes        │
                         └──────────┬───────────┘
                                    │
                         ┌──────────▼───────────┐
                         │ Component Testing    │
                         │ Deterministic        │
                         └──────────┬───────────┘
                                    │
                         ┌──────────▼───────────┐
                         │ Pure Unit Testing    │
                         │ No infrastructure   │
                         └──────────────────────┘
```

These levels answer different questions.

| Level             | Question                                                       |
| ----------------- | -------------------------------------------------------------- |
| Unit              | Does this logic satisfy its contract?                          |
| Component         | Does this architectural component behave correctly?            |
| Contract          | Do two components agree on their interface?                    |
| Integration       | Does the real infrastructure behave correctly at the boundary? |
| System evaluation | Does the complete cognitive system perform its intended task?  |

---

# D3. Testability Classification

Every architectural component receives a testing classification.

### T0 — Pure

Requires no external dependency.

Examples:

```text
ContextContract validation
Token accounting
State transition validation
Permission checks
Budget calculations
Provenance construction
Schema validation
Workflow graph validation
```

---

### T1 — Dependency-isolated

The production dependency exists, but the component's behavior can be tested through mocks/fakes.

Examples:

```text
ContextResolver
MemoryWritePolicy
PromotionPolicy
Workflow engine
Stage executor
Memory repository
Retrieval policy
```

A fake repository or deterministic retrieval source is sufficient.

---

### T2 — Infrastructure-dependent

The property being tested depends on actual infrastructure.

Examples:

```text
Vector search behavior
Embedding quality
Database indexing
Transaction semantics
Persistence/recovery
Actual retrieval latency
```

These require the relevant infrastructure.

---

### T3 — Model-dependent

The property under test is fundamentally about model behavior.

Examples:

```text
Epistemic reasoning
Query interpretation
LLM-generated retrieval refinement
Memory extraction
Contradiction detection
Structured reasoning
Prompt effectiveness
Context sufficiency
```

A mock LLM can test orchestration around these operations.

It cannot establish that the LLM itself performs them correctly.

---

### T4 — System-dependent

The property emerges from interactions among multiple cognitive components.

Examples:

```text
Does memory improve future retrieval?
Does epistemic governance improve answer quality?
Does stage-specific context improve task performance?
Does write governance prevent memory pollution?
Does the complete loop improve across sessions?
```

These require realistic model and infrastructure configurations.

---

# D4. Architectural Test Matrix

| Component              | T0 | T1 | T2 | T3 | T4 |
| ---------------------- | -: | -: | -: | -: | -: |
| AgentState             |  ✓ |    |    |    |    |
| State permissions      |  ✓ |    |    |    |    |
| ContextContract        |  ✓ |    |    |    |    |
| ContextResolver        |  ✓ |  ✓ |    |    |    |
| RetrievalPolicy        |  ✓ |  ✓ |  ✓ |    |    |
| RankingPolicy          |  ✓ |  ✓ |  ✓ |    |    |
| BudgetPolicy           |  ✓ |    |    |    |    |
| Provenance             |  ✓ |    |    |    |    |
| Workflow               |  ✓ |  ✓ |    |    |    |
| Stage selection        |  ✓ |  ✓ |    |  ✓ |    |
| Stage execution        |    |  ✓ |    |  ✓ |    |
| EpistemicPolicy        |  ✓ |  ✓ |    |  ✓ |  ✓ |
| EvidencePolicy         |  ✓ |  ✓ |    |  ✓ |  ✓ |
| MemoryStore            |    |  ✓ |  ✓ |    |    |
| MemoryWritePolicy      |  ✓ |  ✓ |    |  ✓ |  ✓ |
| PromotionPolicy        |  ✓ |  ✓ |    |  ✓ |  ✓ |
| ConflictResolution     |  ✓ |  ✓ |    |    |  ✓ |
| ForgettingPolicy       |  ✓ |  ✓ |    |    |  ✓ |
| LLM adapter            |    |  ✓ |    |  ✓ |    |
| Vector adapter         |    |  ✓ |  ✓ |    |    |
| Complete RAG loop      |    |    |  ✓ |  ✓ |  ✓ |
| Cross-session learning |    |    |    |    |  ✓ |

The matrix is a guide to **where evidence comes from**, rather than a requirement to test every component at every level.

---

# D5. Pure State Tests

`AgentState` should be one of the most heavily tested parts of the system.

No LLM is required.

No database is required.

No embeddings are required.

## Tests

### State lifetime

```text
Execution starts
    ↓
W_t exists
    ↓
Execution terminates
    ↓
W_t disappears
```

### Session lifetime

```text
Execution 1
    ↓
G_t persists
    ↓
Execution 2
    ↓
G_t available
    ↓
Session terminates
    ↓
G_t disappears
```

### Persistent lifetime

```text
Execution 1
    ↓
M_t mutation
    ↓
Execution terminates
    ↓
Execution 2
    ↓
M_t remains
```

These are deterministic invariants.

---

# D6. Mutation Authority Tests

Mutation permissions must be tested without an LLM.

Example:

```text
Interpreter
  read M_t ✓
  write M_t ✗

Retriever
  read M_t ✓
  write M_t ✗

MemoryManager
  read M_t ✓
  write M_t ✓
```

A test should attempt unauthorized mutations deliberately.

Expected result:

```text
PermissionDenied
```

This is an architectural security property.

It should never depend on a prompt.

---

# D7. Context Contract Tests

The closed-world property is deterministic.

Given:

```text
required:
  current_goal

optional:
  prior_failures
```

and state:

```text
current_goal
prior_failures
personal_memory
secret_state
future_field
```

the resolver must produce:

```text
current_goal
prior_failures
```

and nothing else.

Adding:

```text
future_field
```

to the state must not automatically expose it.

This test requires no LLM.

---

# D8. Context Resolver Tests

The resolver can be tested with synthetic state.

Example:

```text
State:
  100 memory records
  20 session records
  10 working records

Contract:
  required = goal
  optional = relevant_memory
  budget = 500 tokens
```

The test can verify:

* only authorized sources are searched
* required context is preserved
* optional context is ranked
* budget is respected
* excluded information is absent
* provenance is attached
* resolver does not mutate state

No real vector database is necessary.

---

# D9. Retrieval Testing

Retrieval has three distinct questions.

### Retrieval mechanics

```text
query → candidates
```

Can be tested using a fake corpus.

### Retrieval infrastructure

```text
query → actual vector database → candidates
```

Requires the real database.

### Retrieval quality

```text
query + real corpus → useful evidence
```

Requires realistic data and evaluation.

These should remain separate.

A passing unit test proves:

> the retrieval algorithm correctly processes its inputs.

It does not prove:

> semantic retrieval is good.

---

# D10. Mock Retrieval

The testing system should provide a deterministic retrieval implementation.

```dart
abstract interface class RetrievalPolicy {
  Future<RetrievalResult> retrieve(
    RetrievalRequest request,
  );
}
```

Test implementation:

```text
FakeRetrievalPolicy
```

Example:

```text
query = "database scalability"

returns:
  memory-17
  document-42
  document-51
```

This allows us to test the entire context pipeline deterministically.

---

# D11. Ranking Tests

Ranking should be tested using known candidate sets.

Example:

```text
Candidate A:
  relevance = .9
  recency = .2
  authority = .8

Candidate B:
  relevance = .7
  recency = .9
  authority = .8
```

The test verifies the declared scoring policy.

We can test edge cases such as:

* equal scores
* missing scores
* conflicting signals
* duplicate candidates
* budget overflow

No LLM required.

---

# D12. Budget Tests

Budget fitting is deterministic.

Tests should establish:

```text
selected tokens <= budget
```

and:

```text
required tokens remain available
```

Additional tests:

* exact budget boundary
* candidate larger than remaining budget
* required context exceeding budget
* duplicate items
* provenance overhead
* truncation behavior

A failure here should be a software failure, not an LLM evaluation.

---

# D13. Stage Tests

Stages have two fundamentally different test categories.

### Structural stage tests

Use a mock LLM.

Verify:

```text
ContextContract
      ↓
ContextResolver
      ↓
Stage
      ↓
structured result
```

The mock LLM can return:

```json
{
  "interpretation": "database scalability",
  "confidence": 0.8
}
```

This tests orchestration.

### Cognitive stage tests

Use a real model.

Verify:

```text
Does the model actually interpret the query correctly?
```

That belongs to model evaluation.

---

# D14. Mock LLM

The architecture should provide a deterministic `FakeLanguageModel`.

Example behavior:

```text
Input contains "database"
→ return predefined QueryInterpretation
```

It should also support:

```text
success
malformed JSON
schema violation
timeout
empty response
refusal
unexpected fields
```

This lets us test failure handling without model variability.

---

# D15. Structured Output Tests

Structured output is a particularly valuable boundary.

Test:

```text
LLM response
    ↓
parser
    ↓
schema validator
    ↓
domain object
```

Cases:

```text
valid JSON
missing field
wrong type
extra field
invalid enum
null
truncated JSON
malformed JSON
```

These tests require no actual LLM.

---

# D16. Epistemic Policy Tests

This boundary requires more care.

We can test its **mechanics** with mocks:

```text
EvidencePolicy
    ↓
mock evidence
    ↓
expected assessment
```

Example:

```text
Evidence:
  source A supports claim
  source B contradicts claim

Expected:
  contradiction = true
```

This tests the policy implementation.

It does not prove that an LLM can reliably identify contradictions.

---

# D17. Epistemic Model Evaluation

Actual epistemic capability requires a model.

Tests should include curated cases:

```text
Case 1:
supported claim

Case 2:
unsupported claim

Case 3:
conflicting sources

Case 4:
causal leap

Case 5:
high-confidence weak evidence

Case 6:
outdated evidence

Case 7:
correct answer with misleading supporting evidence
```

Metrics can include:

* evidence attribution accuracy
* contradiction detection
* unsupported inference rate
* calibration
* provenance fidelity
* correction rate

---

# D18. Memory Store Tests

Memory storage should have infrastructure-independent tests.

Use:

```text
FakeMemoryStore
```

to test:

```text
ADD
UPDATE
DELETE
NOOP
```

and:

* identity
* versioning
* provenance
* timestamps
* retrieval
* conflict representation

Then run a separate integration suite against the actual database.

---

# D19. Memory Write Policy Tests

Memory write policy can be tested without persistent infrastructure.

Input:

```text
Existing memory
+
New observation
+
Evidence
```

Output:

```text
ADD
UPDATE
DELETE
NOOP
```

Example:

```text
Existing:
"Postgres is used for the application."

Observation:
"Postgres is used for the application."

Expected:
NOOP
```

Another:

```text
Existing:
"Application uses PostgreSQL 15."

Observation:
"Application migrated to PostgreSQL 17."

Expected:
UPDATE / SUPERSEDE
```

The exact conflict semantics remain a TODO.

---

# D20. Promotion Tests

Promotion is one of the most important isolated tests.

Given:

```text
W_t:
  failed retrieval
  hypothesis
  validated observation
  tool result
```

the policy should determine:

```text
promote?
```

The test does not need a database.

It only needs a deterministic policy and synthetic observations.

---

# D21. Persistence Boundary Test

A critical system invariant:

```text
Working Observation
       │
       ├── no promotion
       ▼
Execution ends
       │
       ▼
Observation absent from M_t
```

and:

```text
Working Observation
       │
       ├── promotion approved
       ▼
Write Policy
       │
       ▼
M_t
```

This test should exist at both unit and integration levels.

---

# D22. Provenance Tests

Provenance can be tested entirely without an LLM.

Given:

```text
document-17
retrieval-policy-v2
score=.83
stage=retriever
timestamp=T
```

the final context item should preserve the expected chain.

Likewise:

```text
Observation
 → promotion
 → write proposal
 → memory mutation
```

must remain traceable.

A provenance test should fail if an operation loses its origin.

---

# D23. Workflow Tests

The initial fixed workflow is highly testable without models.

Given:

```text
Interpret
 → Retrieve
 → Evaluate
 → Synthesize
```

test:

```text
normal path

Evaluate = insufficient
→ Retrieve

Evaluate = sufficient
→ Synthesize

Retry limit exceeded
→ Terminate
```

These tests should execute with mock stages.

---

# D24. Stage Selection Tests

The same interface should later support a learned router.

For v1:

```text
FixedWorkflowStageSelectionPolicy
```

Test the policy using synthetic stage results.

For example:

```text
retrieval_quality = .2
```

must trigger the declared retry edge.

No LLM is needed.

---

# D25. End-to-End Mock System

Before introducing an actual model, the entire architecture should run using:

```text
FakeLanguageModel
FakeRetrievalPolicy
FakeMemoryStore
DeterministicRankingPolicy
DeterministicBudgetPolicy
DeterministicWritePolicy
FixedWorkflow
```

This creates a **fully deterministic cognitive-system simulator**.

Its purpose is extremely important.

We can test:

```text
state
→ context
→ stage
→ decision
→ observation
→ promotion
→ memory
→ next execution
```

without any probabilistic component.

---

# D26. Real Infrastructure Integration

Once the mock system passes, replace boundaries individually.

### Stage 1

```text
FakeMemoryStore
       ↓
Real database
```

### Stage 2

```text
FakeRetrievalPolicy
       ↓
Real vector / hybrid retrieval
```

### Stage 3

```text
FakeLanguageModel
       ↓
Real LLM
```

### Stage 4

```text
all real
```

This gives us controlled fault localization.

---

# D27. Real LLM Tests

Real-model testing should be isolated from deterministic CI.

A model evaluation suite should record:

```text
model
model version
system prompt
stage
context contract
context
temperature/configuration
tool availability
output
decision
evaluation
```

The same task set should be repeatable.

---

# D28. Model Evaluation Must Not Become Unit Testing

A test such as:

> "The LLM should return X."

is not an ordinary unit test.

It belongs to model evaluation because output distributions can change with:

* model version
* prompt
* context
* temperature
* provider
* quantization
* inference engine

Therefore:

```text
Unit test:
  contract is satisfied.

Model evaluation:
  model performs desired cognitive behavior.
```

---

# D29. Vector Database Evaluation

A vector database introduces its own test domain.

### Infrastructure tests

```text
insert
update
delete
query
filter
index
persistence
recovery
```

### Retrieval tests

```text
known query → expected candidate set
```

### Quality evaluation

```text
Recall@k
Precision@k
MRR
NDCG
```

The third category requires representative data.

---

# D30. RAG Evaluation

RAG evaluation must distinguish:

```text
retrieval quality
        ↓
context quality
        ↓
reasoning quality
        ↓
answer quality
```

A correct answer does not prove good retrieval.

A good retrieval result does not prove good reasoning.

This separation should appear in the evaluation harness.

---

# D31. Architecture-Level Experiments

These tests evaluate the architectural hypotheses themselves.

Examples:

### Context contracts

```text
Global context
vs.
stage-conditioned context
```

### Retrieval

```text
semantic-only
vs.
hybrid
vs.
decision-aware
```

### Memory

```text
no memory
vs.
retrieval memory
vs.
governed memory
```

### Persistence

```text
automatic persistence
vs.
promotion-gated persistence
```

### Epistemic governance

```text
ordinary prompting
vs.
structured epistemic policy
```

These are experiments rather than regression tests.

---

# D32. Cross-Session Evaluation

The architecture's long-term claim requires a longitudinal test.

```text
Session 1
   ↓
Observation
   ↓
Memory
   ↓
Session 2
   ↓
Memory retrieval
   ↓
Decision
   ↓
Correction
   ↓
Updated memory
   ↓
Session 3
```

Measure:

```text
memory precision
memory recall
memory contradiction rate
correction rate
future-task performance
```

The key question:

> Does governed persistence improve future execution?

---

# D33. Regression Categories

Every discovered failure should be classified.

```text
STATE
CONTRACT
RETRIEVAL
RANKING
BUDGET
STAGE
MODEL
EPISTEMIC
MEMORY
PROMOTION
PROVENANCE
WORKFLOW
INFRASTRUCTURE
```

This prevents "the model produced a bad answer" from becoming the generic failure category.

---

# D34. Test Doubles

The implementation should provide explicit test doubles.

```text
FakeLanguageModel
FakeRetrievalPolicy
FakeMemoryStore
FakeEmbeddingProvider
FakeClock
FakeStage
FakeWorkflow
FakePolicy
```

They should be deterministic and inspectable.

A fake should expose the calls it received.

For example:

```text
FakeContextResolver.calls
FakeMemoryStore.mutations
FakeLLM.requests
```

This makes architectural behavior observable.

---

# D35. Contract Tests

Each interface should have a contract test suite.

For example:

```text
MemoryStoreContractTests
```

must be executable against:

```text
InMemoryMemoryStore
PostgresMemoryStore
VectorMemoryStore
GraphMemoryStore
```

Each implementation must satisfy the same behavioral contract.

Likewise:

```text
RetrievalPolicyContractTests
```

can run against:

```text
FakeRetrievalPolicy
HybridRetrievalPolicy
VectorRetrievalPolicy
LearnedRetrievalPolicy
```

This is critical for the replaceable-policy objective.

---

# D36. Determinism

The deterministic test suite should guarantee:

```text
same input
+
same state
+
same configuration
=
same result
```

Where randomness is required, the test must control the seed.

Real LLM evaluation naturally operates outside this guarantee.

---

# D37. Observability as a Testing Dependency

The architecture's provenance requirement means observability is not merely operational tooling.

It is part of the test surface.

A test should be able to inspect:

```text
ExecutionTrace
    │
    ├── state version
    ├── stage
    ├── contract
    ├── candidates
    ├── ranking
    ├── selected context
    ├── model invocation
    ├── stage decision
    ├── promotion decision
    ├── write proposal
    └── final mutation
```

This allows assertions about **why**, rather than only **what**.

---

# D38. CI Structure

The test suite should be divided into execution classes.

```text
tests/
├── unit/
│   ├── state/
│   ├── contracts/
│   ├── policies/
│   ├── provenance/
│   └── schemas/
│
├── component/
│   ├── context/
│   ├── workflow/
│   ├── memory/
│   └── stages/
│
├── contract/
│   ├── memory_store/
│   ├── retrieval/
│   └── model/
│
├── integration/
│   ├── database/
│   ├── retrieval/
│   └── model/
│
└── evaluation/
    ├── rag/
    ├── memory/
    ├── epistemic/
    └── longitudinal/
```

---

# D39. CI Tiers

### Tier 0 — Every commit

```text
pure unit
schema
state
contracts
permissions
workflow
provenance
```

No network.

No model.

No database.

Fast.

---

### Tier 1 — Pull request

```text
component tests
fake integrations
contract tests
in-memory end-to-end
```

Still deterministic.

---

### Tier 2 — Integration

```text
real database
real vector store
real embedding service
```

Executed in controlled environments.

---

### Tier 3 — Model evaluation

```text
real LLM
real prompts
real context
representative tasks
```

Results are recorded rather than treated as ordinary deterministic assertions.

---

### Tier 4 — Research evaluation

```text
architecture comparisons
ablation studies
longitudinal memory experiments
model comparisons
policy comparisons
```

These produce experimental datasets.

---

# D40. What Must Be Tested Before Tomorrow's Implementation Is Considered Valid

The initial implementation should not require an LLM to establish that the architecture works.

At minimum:

### State

* lifetime semantics
* mutation authority
* persistence boundary

### Context

* closed-world contracts
* required/optional handling
* budget enforcement
* provenance

### Workflow

* fixed graph
* bounded branching
* retry
* termination

### Memory

* CRUD semantics
* promotion boundary
* conflict representation
* provenance

### Policies

* policy replacement
* deterministic behavior
* policy decision tracing

### End-to-end

A complete execution should be able to run with:

```text
FakeLLM
FakeRetriever
FakeMemoryStore
```

and demonstrate:

```text
User objective
      ↓
workflow
      ↓
stage
      ↓
context resolution
      ↓
decision
      ↓
working observation
      ↓
promotion
      ↓
memory mutation
      ↓
next execution
```

---

# D41. What We Must **Not** Claim From Those Tests

Passing the deterministic suite does **not** establish:

* that retrieval is semantically good
* that the LLM reasons correctly
* that epistemic governance improves reasoning
* that memory improves future performance
* that stage-specific context improves task performance
* that persistent memory remains correct over long periods

Those require empirical evaluation.

This distinction should remain explicit in the project.

---

# D42. Testing Principle

The complete strategy can be summarized as:

```text
                   "Does the code obey its contract?"
                                │
                                ▼
                         Unit / Component
                                │
                                ▼
                   "Do boundaries interact correctly?"
                                │
                                ▼
                         Contract / Integration
                                │
                                ▼
                   "Does the model perform the task?"
                                │
                                ▼
                         Model Evaluation
                                │
                                ▼
                  "Does the architecture improve
                    the cognitive system?"
                                │
                                ▼
                       Research Evaluation
```

## C1 — Binding Rule

> **Use mocks to test architecture. Use real infrastructure to test infrastructure. Use real models to test model behavior. Use complete experimental systems to test architectural hypotheses.**

That separation should prevent the project from falling into two common traps: building an expensive LLM-dependent test suite for ordinary software, and mistaking successful orchestration tests for evidence that the cognitive architecture itself works.
