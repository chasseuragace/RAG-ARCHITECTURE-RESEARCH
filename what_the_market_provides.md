D: What is worth stealing from the market

I went back to the actual framework designs rather than relying on the earlier market summary. The useful conclusion is that we should steal mechanisms, while keeping our architectural boundaries intact.

The strongest candidates are below.

Priority	Mechanism	Source	Decision

P0	Checkpointed execution + replay	LangGraph	Steal
P0	Memory blocks / typed memory units	Letta	Steal concept
P0	Temporal validity + supersession	Zep/Graphiti	Steal
P0	Multi-signal retrieval	Mem0 ecosystem	Steal
P0	Full execution tracing	Haystack/OpenTelemetry	Steal
P1	Dynamic attach/detach of memory	Letta	Adapt
P1	Graph + episodic + semantic representations	Zep/Graphiti	Adapt
P1	Delta checkpoint storage	LangGraph	Adapt
P1	Memory scopes	Mem0	Adapt
P2	Agent-autonomous memory editing	Letta	Do not adopt directly
P2	Fully dynamic orchestration	agent frameworks	Keep out of baseline



---

D1. Steal LangGraph's checkpoint model

This is probably the most important thing to steal.

LangGraph treats execution state as a sequence of checkpoints, rather than merely "the current state." Each graph step produces a checkpoint, enabling:

recovery after failure

replay

time travel

human intervention

branching from previous states

debugging historical execution




That fits our architecture extremely well.

Our state should therefore conceptually become:

S_t
 │
 ├── M_t       persistent memory
 ├── G_t/H_t   session state
 └── W_t       execution state

but execution produces:

S_0
 ↓
checkpoint_0
 ↓
stage A
 ↓
checkpoint_1
 ↓
stage B
 ↓
checkpoint_2
 ↓
stage C
 ↓
checkpoint_3

Why this matters

Our current architecture talks about W_t as disposable.

That is correct for logical lifetime.

It does not mean the system should physically throw it away immediately.

We should distinguish:

logical lifetime
        ≠
operational observability

W_t can remain available in an execution trace/checkpoint store without becoming persistent agent memory.

That is a very useful distinction.


---

D2. Steal Letta's memory blocks

Letta has a particularly useful abstraction: memory blocks.

A block has:

label
description
value
limit
read_only

and can be attached to agents dynamically. Blocks can also be shared or made read-only. 

This maps surprisingly well onto our Context Contract idea.

Instead of treating M_t as one giant memory database:

M_t

we can expose it conceptually as typed resources:

M_t
├── user_preferences
├── project_facts
├── validated_procedures
├── prior_failures
├── domain_knowledge
└── ...

Then a stage contract can say:

QueryInterpreter:

required:
    conversation
    current_goal

optional:
    user_preferences
    prior_query_patterns

forbidden:
    retrieval_failures
    generator_drafts

The resolver decides which blocks become actual context.

Important modification

We should not copy Letta's assumption that memory blocks are necessarily always visible.

Letta's blocks are explicitly designed to sit in the context window continuously. 

Our architecture is stronger if:

Memory Block
     ↓
Context Resolver
     ↓
stage-specific projection

rather than:

Memory Block
     ↓
always injected

So we steal the unit of organization, not the injection policy.


---

D3. Steal read-only memory

This one is directly useful.

Letta supports read-only memory blocks so an agent can consume organizational information without modifying it. 

Our architecture already separates:

read authority

from:

mutation authority

This gives us a concrete implementation precedent.

For example:

Stage: RetrievalJudge

READ:
    M.project_facts
    M.validated_procedures
    G.current_goal
    W.retrieval_results

WRITE:
    W.judgement

CANNOT WRITE:
    M.*

Then:

MemoryWritePolicy
        ↓
      M_t

remains the only route into persistent memory.

This makes the architecture enforceable rather than prompt-dependent.


---

D4. Steal Zep/Graphiti's temporal memory

This is probably the most important thing to steal from the data model side.

Zep/Graphiti represents:

entities

relationships/facts

episodic information

temporal relationships


and explicitly preserves historical changes rather than treating the latest value as the entire truth. 

Their architecture is particularly relevant because agent memory has a fundamental problem:

Fact A was true
        ↓
Fact B became true
        ↓
Fact A is now obsolete

A vector database tends to leave you with:

A
B

and retrieval has to figure out which one wins.

A temporal representation can encode:

A
valid: t1 → t2

B
valid: t2 → ∞

That should influence our M_t schema.


---

D5. Add supersession to our write semantics

Our current:

ADD
UPDATE
DELETE
NOOP

is too crude for durable memory.

We should probably investigate:

ADD
UPDATE
SUPERSEDE
DELETE
NOOP

with the possibility that UPDATE means modify the same logical memory, while SUPERSEDE means:

> preserve the old assertion because it was historically valid, but make the new assertion authoritative for the current state.



Example:

Memory 17
User prefers PostgreSQL
valid_until = 2026-04-01

Memory 42
User prefers SQLite
valid_from = 2026-04-01
supersedes = 17

That is substantially better than:

UPDATE 17 → SQLite

because historical reasoning can still ask:

> What did the system believe previously?



This is one of the places where temporal graph memory is worth stealing. Zep explicitly models changing facts and preserves historical relationships. 


---

D6. Steal Mem0's retrieval fusion

Our resolver currently has something conceptually like:

score(x | stage, goal, state)

We should avoid making that synonymous with:

embedding_similarity(x, query)

Mem0's current retrieval architecture uses multiple signals, including semantic similarity, keyword matching and entity matching.

That gives us a useful baseline:

U(x | s)
 =
 α semantic
 + β lexical
 + γ entity
 + δ recency
 + ε temporal_validity
 + ζ stage_utility

The exact equation remains an implementation decision.

The important architectural decision is:

> The resolver receives a candidate set and evaluates candidates using multiple signals.



That preserves the resolver abstraction while allowing the scoring implementation to evolve.


---

D7. Steal memory scopes from Mem0

Mem0's scope model is also useful.

Rather than treating memory as one global namespace, memory can belong to different scopes such as user, agent, run, and application.

That maps naturally onto ours:

M_t
├── application
├── domain
├── agent
├── user
├── session
└── execution

Although I would not copy that exact hierarchy.

Our state model should retain the lifetime/authority distinction:

M_t = durable
G_t/H_t = session
W_t = execution

Scope then becomes metadata:

Memory {
    lifetime
    owner
    scope
    type
    validity
    provenance
}

That is cleaner than making scope itself define the memory ontology.


---

D8. Steal Haystack's tracing discipline

Haystack's tracing model is less intellectually interesting than Graphiti, but operationally it is extremely important.

It instruments pipeline components and exports traces through OpenTelemetry or other tracing backends, allowing developers to see execution order and where time is spent. 

Our system needs to go further.

We should trace context construction itself.

For every LLM invocation:

InvocationTrace
├── stage
├── contract_version
├── state_checkpoint
├── candidate_memories
├── selected_memories
├── rejected_memories
├── exclusion_reason
├── retrieval_scores
├── token_budget
├── actual_tokens
├── prompt/context hash
├── model
├── output
└── latency

That gives us something critical:

> We can inspect why the model saw what it saw.



That is much more valuable than ordinary LLM tracing.


---

D9. Steal LangGraph's failure/recovery semantics

This also fits our fixed workflow decision.

Suppose:

QueryInterpreter
        ↓
Retriever
        ↓
Judge
        ↓
Generator

The Retriever fails.

With checkpointing:

checkpoint after interpreter
        ↓
retrieval attempt
        ↓
failure
        ↓
retry

The system does not need to recompute interpretation.

Likewise:

Judge says insufficient evidence
        ↓
bounded retrieval branch
        ↓
new retrieval
        ↓
Judge again

This reinforces our earlier decision:

fixed graph
+
bounded conditional edges
+
checkpointing

rather than:

LLM chooses arbitrary next stage

Checkpointing makes the conservative architecture much more capable operationally. LangGraph explicitly uses checkpoints for fault tolerance and resumption. 


---

D10. Steal Graphiti's distinction between episodes and derived knowledge

This is especially relevant to our W_t → M_t promotion boundary.

Graphiti/Zep maintains both:

episodic nodes

and:

entity/fact structures



That suggests our persistent memory should not necessarily contain only "facts."

We could model:

M_t
├── Facts
├── Procedures
├── Experiences
├── Preferences
├── Episodes
└── Derived knowledge

Then the write policy can decide:

raw episode
    ↓
retain?
    ↓
derive fact?
    ↓
update existing fact?
    ↓
discard?

This is much better than:

everything interesting → vector DB


---

D11. Steal the idea of memory evolution

The graph-memory literature increasingly treats memory as a lifecycle:

extract
   ↓
store
   ↓
retrieve
   ↓
reason
   ↓
evolve

rather than:

write once
   ↓
retrieve forever

The 2026 graph-memory survey explicitly organizes the field around extraction, storage, retrieval and evolution. 

This fits our architecture extremely well.

Our memory subsystem should therefore be thought of as:

┌───────────────┐
             │   M_t         │
             └───────┬───────┘
                     │
             ┌───────▼───────┐
             │ Read Resolver  │
             └───────┬───────┘
                     │
                     ▼
                 Stage LLM
                     │
                     ▼
               W_t / G_t
                     │
             ┌───────▼───────┐
             │ Write Policy   │
             └───────┬───────┘
                     │
                     ▼
                  M_(t+1)

That is the architectural loop we have been converging toward.


---

D12. A particularly interesting new thing: immutable/versioned memory

There is a newer research direction worth watching here.

WorldDB proposes content-addressed immutable memory nodes with write-time reconciliation and explicit semantics for operations such as supersession and contradiction. Its reported LongMemEval-s results are strong, although this is a very recent research system and should not be treated as established evidence yet. 

The interesting part for us isn't necessarily its graph structure.

It is this idea:

memory modification
        ↓
new version
        ↓
old version remains auditable

That fits our checkpoint/provenance philosophy.

I would put this in the architecture's research TODO, rather than immediately adopting it.


---

D13. What I would not steal

Several popular patterns would actively conflict with the architecture we've established.

Autonomous unrestricted memory editing

Letta gives agents powerful memory-management tools and allows blocks to be agent-managed. 

Useful capability.

Wrong default for us.

Our boundary should remain:

LLM proposes
     ↓
Write Policy evaluates
     ↓
mutation authority
     ↓
M_t

rather than:

LLM → database


---

Always-visible memory

Useful for tiny core memory blocks.

Poor default for a large persistent store.

Our resolver should control exposure.


---

Direct state access from every stage

LangGraph's State model makes this convenient.

Our architecture should deliberately avoid:

stage.state.memory

because that bypasses:

ContextContract
       ↓
ContextResolver

and destroys our central guarantee.


---

Dynamic agent routing as the baseline

Frameworks increasingly encourage planners and dynamic tool selection.

Our fixed graph plus bounded branching is easier to test, replay and attribute.

Dynamic routing should remain an extension point.


---

D14. The architecture after stealing the best pieces

I would now visualize it like this:

USER
                           │
                           ▼
                    ┌─────────────┐
                    │   Session   │
                    │  G_t / H_t  │
                    └──────┬──────┘
                           │
                           ▼
                  ┌──────────────────┐
                  │ Fixed Workflow   │
                  │ + bounded edges  │
                  └────────┬─────────┘
                           │
             ┌─────────────▼─────────────┐
             │      Stage Execution       │
             │                            │
             │  Stage + ContextContract  │
             └─────────────┬─────────────┘
                           │
                           ▼
                ┌────────────────────┐
                │ Context Resolver R │
                │                    │
                │ contract           │
                │ state              │
                │ budget             │
                │ retrieval signals  │
                └─────────┬──────────┘
                          │
            ┌─────────────┼─────────────┐
            ▼             ▼             ▼
          G/H            M_t           W_t
       session        persistent      working
                          │
                  ┌───────┴────────┐
                  │ memory indexes │
                  │ vector/text    │
                  │ entity/graph   │
                  │ temporal       │
                  └────────────────┘
                          │
                          ▼
                     Stage LLM
                          │
                          ▼
                    Stage Result
                          │
             ┌────────────┴────────────┐
             ▼                         ▼
       update W/G                 Write candidate
                                       │
                                       ▼
                               ┌──────────────┐
                               │ Write Policy │
                               │              │
                               │ ADD          │
                               │ UPDATE       │
                               │ SUPERSEDE    │
                               │ DELETE       │
                               │ NOOP         │
                               └──────┬───────┘
                                      │
                                      ▼
                                    M_t+1

And around the whole thing:

┌───────────────────────────┐
                 │      Execution Trace       │
                 │                           │
                 │ checkpoints               │
                 │ context provenance        │
                 │ retrieval decisions       │
                 │ memory mutations          │
                 │ model calls               │
                 │ token/cost/latency        │
                 └───────────────────────────┘

That last box is important enough that I would make it a first-class architectural subsystem, not observability bolted on afterward.


---

D15. The actual "steal list"

If we are trying to keep the architecture lean, I would take these:

Core implementation

1. LangGraph

checkpoint semantics

replay

recovery

execution snapshots 



2. Letta

typed memory blocks

read-only blocks

attach/detach semantics 



3. Zep/Graphiti

temporal validity

episodic → semantic transformation

entity/fact relationships

historical state 



4. Mem0

multi-signal retrieval

memory CRUD lifecycle

scoped memory



5. Haystack/OpenTelemetry

component-level tracing

execution observability 




Research extensions

6. WorldDB

immutable memory versions

write-time reconciliation

explicit supersession/contradiction semantics 



7. Learned read/write control

later research track

not baseline architecture





---

C: The important architectural consequence

The market research actually strengthens one particular decision:

Our novelty should not be "we invented agent memory."

The useful architecture is the governance layer between memory and cognition:

MEMORY
                       │
                       │
                 ┌─────▼─────┐
                 │   POLICY  │
                 └─────┬─────┘
                       │
            ┌──────────┴──────────┐
            │                     │
        READ POLICY          WRITE POLICY
            │                     │
            ▼                     ▼
     Context Resolver        M_t mutation
            │
            ▼
      Stage-specific
         context
            │
            ▼
          LLM

And the key invariant becomes:

> The LLM never receives raw agent state and never directly owns persistent memory. It receives a policy-governed projection of state, and its proposed persistent changes pass through a separate mutation policy.



That is the architectural boundary worth protecting.

The market has strong implementations for the pieces around that boundary. The opportunity is to combine those mechanisms without collapsing the boundaries themselves.