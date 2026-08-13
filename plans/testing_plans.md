Yes. This should be treated as a **testing architecture**, rather than one giant end-to-end test suite.

The key distinction is:

> **Test the architectural guarantees independently of the intelligence that eventually operates inside the architecture.**

That gives us a large deterministic test surface before introducing an LLM, embeddings, vector databases, or external tools.

# Testing Architecture for the Governed RAG System

## 1. Purpose

The system contains two fundamentally different categories of behavior.

### Category A — Architectural behavior

These are properties that should remain correct regardless of which LLM, retriever, vector database, or policy implementation the system uses.

Examples:

* state isolation;
* authorization;
* context contracts;
* context resolution;
* token budgeting;
* persistence boundaries;
* write semantics;
* provenance;
* workflow transitions;
* policy replacement.

These should be testable with **plain deterministic fixtures and mocks**.

### Category B — Intelligence-dependent behavior

These concern whether an LLM or retrieval model makes good decisions.

Examples:

* query interpretation;
* semantic retrieval;
* evidence assessment;
* answer synthesis;
* memory extraction;
* contradiction detection;
* epistemic reasoning;
* learned context selection.

These require an actual model or a sufficiently faithful model substitute.

The test architecture should keep the two categories separate.

---

# 2. Testing Layers

```text
                         ┌──────────────────────────┐
                         │      System Tests        │
                         │                          │
                         │ LLM + Retrieval + Memory │
                         │ + Tools + Workflow       │
                         └────────────┬─────────────┘
                                      │
                         ┌────────────▼─────────────┐
                         │   Intelligence Tests     │
                         │                          │
                         │ LLM / Embedding / Rerank │
                         └────────────┬─────────────┘
                                      │
                    ┌─────────────────▼──────────────────┐
                    │       Policy / Integration Tests   │
                    │                                    │
                    │ mocked LLM + mocked retrieval      │
                    └─────────────────┬──────────────────┘
                                      │
              ┌───────────────────────▼───────────────────────┐
              │              Architecture Tests              │
              │                                               │
              │ state / contracts / resolver / permissions   │
              │ persistence / provenance / workflow          │
              └───────────────────────┬───────────────────────┘
                                      │
                         ┌────────────▼────────────┐
                         │       Unit Tests        │
                         │                         │
                         │ pure functions / types │
                         └─────────────────────────┘
```

The important property is that **higher layers should not be required to establish correctness of lower layers**.

---

# 3. Testability Classification

We can classify every component using four levels.

| Level  | Dependency                                             | Test mechanism          |
| ------ | ------------------------------------------------------ | ----------------------- |
| **T0** | No external dependency                                 | Pure deterministic test |
| **T1** | External behavior represented by interface             | Mock/fake               |
| **T2** | Actual model/database required for meaningful behavior | Integration test        |
| **T3** | Multiple real components interacting                   | End-to-end evaluation   |

This gives us a concrete rule:

> Build T0/T1 tests first. They establish architectural correctness before intelligence quality becomes a variable.

---

# 4. T0 — Pure Architectural Tests

These require:

* no LLM;
* no embeddings;
* no vector database;
* no network;
* no external service.

They should execute extremely quickly.

## 4.1 State Isolation

Test:

```text
Mₜ
Gₜ/Hₜ
Wₜ
```

### Tests

**T0-S1 — Working state disappears**

```text
create Wₜ
insert observation
terminate execution
assert Wₜ unavailable
```

**T0-S2 — Working state cannot mutate persistent memory**

Attempt:

```text
Wₜ → Mₜ
```

without the promotion mechanism.

Expected:

```text
rejected
```

**T0-S3 — Session state survives stage boundaries**

```text
Stage A
 ↓
Gₜ
 ↓
Stage B
```

The second stage sees permitted session state.

**T0-S4 — Session state does not survive session termination**

Straightforward lifecycle test.

---

# 5. T0 — Mutation Authority

This is one of the most important test groups.

Given:

```text
Stage A
read: Mₜ.foo
write: Gₜ.bar
```

attempt:

```text
Stage A → Mₜ.foo = newValue
```

must fail.

Test matrix:

| Stage          | Read M | Write M | Read G | Write G | Read W | Write W |
| -------------- | -----: | ------: | -----: | ------: | -----: | ------: |
| Interpreter    |      ✓ |       ✗ |      ✓ |       ✓ |      ✓ |       ✓ |
| Retriever      |      ✓ |       ✗ |      ✓ |       ✗ |      ✓ |       ✓ |
| Judge          |      ✓ |       ✗ |      ✓ |       ✗ |      ✓ |       ✓ |
| Memory Manager |      ✓ |       ✓ |      ✓ |       ✓ |      ✓ |       ✓ |

The actual permissions can change.

The invariant does not:

> **Permission must be enforced by the architecture rather than by the LLM prompt.**

This requires no LLM.

---

# 6. T0 — Context Contract

A contract can be tested entirely using synthetic state.

Example:

```text
ContextContract {
    required = [
        currentQuery,
        retrievedEvidence
    ]

    optional = [
        conversationSummary
    ]

    tokenBudget = 4000
}
```

Synthetic state:

```text
M:
  userPreference
  projectFact
  secretFact

G:
  currentQuery
  conversationSummary

W:
  retrievedEvidence
  failedRetrieval
```

Expected context:

```text
currentQuery
retrievedEvidence
conversationSummary
```

and:

```text
userPreference
projectFact
secretFact
failedRetrieval
```

remain inaccessible unless explicitly declared.

No LLM is necessary.

---

# 7. T0 — Default-Deny Regression Test

This deserves its own test.

Suppose version 1 contains:

```text
M.userPreference
M.projectFact
```

The Judge contract does not request either.

Later version 2 adds:

```text
M.privateNote
```

The resolver must continue producing:

```text
no privateNote
```

without anyone modifying the Judge contract.

This tests the architectural property we identified earlier:

> New state must not automatically become new context.

This is one of the strongest tests for the architecture.

---

# 8. T0 — Context Resolver

The resolver should be as close to a pure function as practical:

```text
R(stage, contract, state) → context
```

Tests should verify:

### Determinism

Same:

```text
stage + contract + state
```

produces equivalent context.

### Isolation

Resolver does not mutate state.

### Authorization

Resolver never returns undeclared fields.

### Budget compliance

Returned context remains within budget.

### Ordering

Candidate ordering follows the configured policy.

### Provenance

Every selected item receives its source metadata.

---

# 9. T0 — Token Budget

This does not require an LLM.

Create synthetic context items:

```text
A = 500 tokens
B = 1000
C = 1500
D = 2000
```

Budget:

```text
2500
```

Test:

```text
required = A
optional = B,C,D
```

Expected:

```text
A + selected optional items <= 2500
```

Then test edge cases:

```text
required > budget
```

The system needs an explicit policy.

Possible result:

```text
ContextBudgetExceeded
```

or controlled truncation.

The important thing is that the behavior is defined and tested.

---

# 10. T0 — Memory Write Semantics

The write policy can initially operate on structured fake memories.

Example:

```text
existing:
  "User uses Dart"

observation:
  "User uses Dart"
```

Expected:

```text
NOOP
```

Another:

```text
existing:
  "User uses Flutter 3.30"

observation:
  "User uses Flutter 3.32"
```

Expected:

```text
UPDATE
```

or:

```text
SUPERSEDE
```

depending on configured semantics.

The test does not need to know how an LLM generated the observation.

---

# 11. T0 — Persistence Boundary

Test:

```text
retrieval result
      ↓
Wₜ
```

and ensure:

```text
Wₜ ≠ Mₜ
```

Then explicitly submit:

```text
PromotionRequest
```

and test:

```text
PromotionRequest
      ↓
WritePolicy
      ↓
Mₜ
```

This establishes one of the most important system invariants:

> **Encountering information does not imply remembering information.**

---

# 12. T0 — Provenance

Create a fake pipeline:

```text
memory-17
 ↓
candidate
 ↓
score = 0.82
 ↓
contract field = optional.evidence
 ↓
context
 ↓
stage
 ↓
write proposal
```

Then assert that the resulting trace contains all required links.

For example:

```text
Trace {
    sourceId
    candidateId
    retrievalPolicy
    retrievalScore
    contractVersion
    contextItemId
    stageId
    stageInvocationId
    decisionId
    writeProposalId
    writePolicyVersion
    mutationId
}
```

Again, no LLM.

---

# 13. T0 — Workflow

The fixed workflow itself can be tested with deterministic stage doubles.

```text
Interpreter
    ↓
Retriever
    ↓
Judge
    ↓
Synthesizer
```

Replace every LLM stage with:

```text
FakeStage
```

that returns predefined outputs.

Then test:

```text
Judge = insufficient
```

causes:

```text
Retriever retry
```

and:

```text
Judge = sufficient
```

causes:

```text
Synthesizer
```

This establishes workflow correctness independently of model quality.

---

# 14. T1 — Mocked Cognitive System

Now introduce interfaces:

```text
LLM
Retriever
Reranker
MemoryStore
EmbeddingProvider
```

but replace implementations with deterministic fakes.

For example:

```text
FakeLLM
```

can implement:

```text
input → predefined structured response
```

This allows us to test the **orchestration between intelligence components**.

Example:

```text
FakeInterpreter
     ↓
query = "Dart architecture"
     ↓
FakeRetriever
     ↓
documents
     ↓
FakeJudge
     ↓
sufficient
     ↓
FakeSynthesizer
```

The real LLM remains completely absent.

---

# 15. T1 — Failure Injection

This is where mocks become especially valuable.

The fake components should deliberately produce failures.

### Retrieval failure

```text
Retriever → zero results
```

Test:

```text
bounded retry
```

### Contradictory evidence

```text
Retriever → A + contradictory B
```

Test:

```text
Judge detects conflict
```

### Invalid structured output

```text
LLM → malformed JSON
```

Test:

```text
validation → retry/failure
```

### Memory write conflict

```text
WritePolicy → conflicting mutations
```

Test:

```text
mutation arbitration
```

### Context overflow

```text
Resolver → budget exceeded
```

Test:

```text
budget policy
```

These tests give us much more control than attempting to induce the same situations through a real LLM.

---

# 16. T1 — Contract Mutation Tests

This is important for long-term architecture maintenance.

Given:

```text
StageContract v1
```

modify:

```text
required
optional
tokenBudget
```

and verify that:

* only intended context changes;
* provenance identifies the new contract version;
* stage behavior can be compared across versions.

This enables architectural regression testing.

---

# 17. T2 — Real LLM Tests

Only now do we introduce an actual LLM.

These tests answer questions that deterministic tests cannot.

For example:

### Query interpretation

Does the LLM correctly convert:

```text
"What happens if the retrieval keeps finding contradictory docs?"
```

into the intended structured query representation?

### Evidence judgment

Can the model distinguish:

```text
supporting evidence
contradictory evidence
irrelevant evidence
missing evidence
```

?

### Memory extraction

Can it distinguish:

```text
temporary observation
```

from:

```text
durable fact
```

?

### Memory correction

Can it recognize:

```text
old fact
+
new contradictory evidence
```

as:

```text
SUPERSEDE
```

rather than:

```text
ADD
```

?

These tests evaluate the intelligence itself.

---

# 18. T2 — Prompt / Epistemic Policy Tests

This is where the Spinoza experiment belongs.

Hold everything else constant:

```text
same model
same documents
same state
same ContextContract
same token budget
same task
```

Change only:

```text
Epistemic Policy
```

Compare:

```text
P0 — ordinary system instruction

P1 — structured epistemic policy

P2 — Spinoza-derived epistemic policy
```

Then evaluate:

```text
accuracy
uncertainty calibration
contradiction handling
self-correction
unsupported claims
memory decisions
```

This turns the philosophical idea into an actual experimental variable.

---

# 19. T2 — Retrieval Tests

A real retrieval backend becomes necessary here.

Compare:

```text
vector retrieval
hybrid retrieval
reranking
metadata filtering
temporal filtering
stage-conditioned retrieval
```

The architecture should remain unchanged.

Only the retrieval policy changes.

That is exactly what O6 enables.

---

# 20. T3 — Full System Tests

Finally:

```text
real LLM
+
real embedding model
+
real vector/structured store
+
real memory
+
real workflow
+
real tools
```

Now test emergent behavior.

Examples:

```text
multi-turn conversation
      ↓
memory formation
      ↓
later retrieval
      ↓
contradictory new information
      ↓
memory correction
      ↓
future answer
```

This is where the complete system can demonstrate value.

---

# 21. Evaluation Matrix

The whole testing strategy can therefore be represented as:

| Component                 |  T0 | T1 Mock | T2 Real | T3 E2E |
| ------------------------- | :-: | :-----: | :-----: | :----: |
| State lifecycle           |  ✓  |         |         |        |
| State isolation           |  ✓  |         |         |        |
| Mutation authority        |  ✓  |         |         |        |
| Context Contract          |  ✓  |         |         |        |
| Default-deny              |  ✓  |         |         |        |
| Context Resolver          |  ✓  |    ✓    |         |        |
| Token budget              |  ✓  |         |         |        |
| Provenance                |  ✓  |    ✓    |         |        |
| Workflow                  |  ✓  |    ✓    |         |        |
| Branching                 |  ✓  |    ✓    |         |        |
| Memory CRUD semantics     |  ✓  |    ✓    |         |        |
| Promotion                 |  ✓  |    ✓    |         |        |
| LLM output validation     |     |    ✓    |    ✓    |        |
| Query interpretation      |     |    ✓    |    ✓    |    ✓   |
| Evidence judgment         |     |    ✓    |    ✓    |    ✓   |
| Memory extraction         |     |    ✓    |    ✓    |    ✓   |
| Semantic retrieval        |     |    ✓    |    ✓    |    ✓   |
| Embedding quality         |     |         |    ✓    |    ✓   |
| Prompt policy             |     |         |    ✓    |    ✓   |
| Spinoza hypothesis        |     |         |    ✓    |    ✓   |
| Long-term memory quality  |     |         |         |    ✓   |
| Multi-session behavior    |     |         |         |    ✓   |
| End-to-end answer quality |     |         |         |    ✓   |

---

# 22. Test Fixtures

The architecture should maintain a reusable synthetic world.

For example:

```text
TestWorld
 ├── Memory
 │    ├── M001
 │    ├── M002
 │    └── M003
 │
 ├── Session
 │    ├── goal
 │    ├── history
 │    └── subgoals
 │
 ├── WorkingState
 │    ├── hypotheses
 │    ├── retrieval attempts
 │    └── evidence
 │
 ├── Documents
 │    ├── D001
 │    ├── D002
 │    └── D003
 │
 └── StageContracts
      ├── Interpreter
      ├── Retriever
      ├── Judge
      └── Synthesizer
```

This lets us reproduce entire scenarios without external infrastructure.

---

# 23. Golden Scenarios

We should maintain a small number of deterministic scenarios that exercise the architecture.

### Scenario A — Context isolation

A memory exists that is highly semantically relevant but unauthorized.

Expected:

```text
never presented
```

### Scenario B — New state field

A new persistent field is introduced.

Expected:

```text
existing stages cannot see it
```

### Scenario C — Transient observation

A retrieval result looks useful but is not promoted.

Expected:

```text
available during execution
absent next session
```

### Scenario D — Memory update

New evidence supersedes old evidence.

Expected:

```text
old memory preserved as history
new memory becomes current
```

### Scenario E — Contradiction

Two documents disagree.

Expected:

```text
Judge receives both
provenance identifies both
write policy does not silently choose one
```

### Scenario F — Retrieval failure

No sufficient evidence.

Expected:

```text
bounded retry
```

### Scenario G — Context overflow

Required context exceeds budget.

Expected:

```text
explicit failure
```

rather than silent truncation.

---

# 24. What We Can Build Before Any LLM

This is the practical payoff.

We can implement and test a substantial portion of the architecture immediately:

```text
State
StateRepository
StateLifecycle
MutationAuthority
ContextContract
ContextAuthorization
ContextResolver
TokenBudget
CandidateScoring
MemoryOperations
PromotionPolicy
Provenance
Trace
Workflow
Branching
Policy interfaces
FakeLLM
FakeRetriever
FakeMemoryStore
```

The architecture can therefore reach a substantial degree of maturity **before downloading a model or configuring a vector database**.

That is useful because architectural bugs become dramatically harder to diagnose once model behavior and retrieval uncertainty enter the system.

---

# 25. What Explicitly Requires Real Infrastructure

Some questions cannot be meaningfully answered with mocks.

### Requires a real LLM

```text
reasoning quality
query interpretation quality
evidence judgment quality
memory extraction quality
epistemic policy effectiveness
prompt sensitivity
self-correction
```

### Requires real embeddings/retrieval

```text
semantic retrieval quality
embedding representation quality
hybrid retrieval effectiveness
retrieval recall
reranking effectiveness
```

### Requires persistent infrastructure

```text
concurrent memory mutation
durability
transaction behavior
index consistency
temporal queries
large-scale retrieval
```

### Requires the complete system

```text
long-horizon memory quality
multi-session behavior
memory pollution
end-to-end retrieval/refinement loops
real-world answer quality
```

---

# 26. The Testing Principle

The architecture should follow one strict rule:

> **Do not use an LLM to test a property that can be tested deterministically.**

If the question is:

> "Does a Judge stage have access to a field it is forbidden to read?"

use a deterministic test.

If the question is:

> "Can the Judge correctly identify contradictory evidence?"

use an LLM evaluation.

Likewise:

> "Does the resolver respect the token budget?"

is deterministic.

> "Does the resolver select context that improves the Judge's decision?"

requires a model.

That separation prevents us from confusing **architectural correctness** with **model intelligence**.

---

# 27. Recommended Development Order

```text
Phase 0
Pure domain model
        ↓
Phase 1
State + authority
        ↓
Phase 2
Context Contracts + Resolver
        ↓
Phase 3
Memory promotion + write semantics
        ↓
Phase 4
Provenance + tracing
        ↓
Phase 5
Deterministic workflow
        ↓
Phase 6
Fake LLM / fake retriever
        ↓
Phase 7
Real LLM
        ↓
Phase 8
Real retrieval
        ↓
Phase 9
Full-system evaluation
        ↓
Phase 10
Learned policy experiments
```

The important consequence is that **the first six phases can be developed as an ordinary deterministic software system**.

Only after those invariants pass do we introduce the probabilistic components.

That gives us a clean experimental boundary:

```text
                    ARCHITECTURE
                         │
          ┌──────────────┴──────────────┐
          │                             │
   Deterministic correctness       Intelligence
          │                             │
       T0 / T1                      T2 / T3
          │                             │
          └──────────────┬──────────────┘
                         │
                  System evaluation
```

This also gives the research program a much cleaner structure: **first prove that the machine obeys its architecture; then determine whether the cognitive policies operating inside that machine make it better.**
