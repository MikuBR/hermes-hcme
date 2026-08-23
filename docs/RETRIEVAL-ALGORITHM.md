# HCME Retrieval Algorithm

## Status

Architecture / Research Phase. This document defines the first implementation-neutral retrieval algorithm for HCME.

## Objective

Retrieve the smallest set of memories that materially improves the current task while minimizing irrelevant historical information, cross-session leakage, contradiction risk, and token cost.

The system optimizes for **contextual utility**, not raw similarity or maximum recall.

## Core invariant

> A memory must be relevant to the current situation, permitted by scope, applicable under current conditions, and useful enough to justify its context cost before it enters the compiled context.

Semantic similarity is only one signal.

## Retrieval stages

```text
current interaction
      ↓
1. task/context understanding
      ↓
2. hard scope gate
      ↓
3. cheap candidate generation
      ↓
4. applicability filtering
      ↓
5. contextual ranking
      ↓
6. graph expansion
      ↓
7. contradiction/conflict handling
      ↓
8. deduplication/compression
      ↓
9. token-budget allocation
      ↓
10. memory compilation
      ↓
prompt-ready context
```

## 1. Task/context understanding

The retrieval request should derive a compact `RetrievalIntent` from the active context:

```text
current_goal
current_task
current_subtask
project_id
logical_session_id
active_topics
active_entities
constraints
required_memory_kinds
excluded_memory_kinds
behavioral_mode
query_terms
```

This object is a retrieval control structure, not a long natural-language summary.

## 2. Hard scope gate

Scope is a permission boundary.

Default order:

```text
current session
    ↓
current project
    ↓
user/global memory
```

Session memory should be preferred when the current problem is a continuation of that session.

Cross-session memory is eligible only when explicitly permitted by scope and relevance rules.

A memory from an unrelated session must not enter the candidate set merely because its text is semantically similar.

## 3. Candidate generation

Use several inexpensive retrieval channels:

### Session/state channel

Retrieve active memories directly referenced by the current context state, snapshot, entities, topics, decisions, constraints, or open tasks.

### Lexical channel

Use FTS5 for exact/near lexical matches, aliases, identifiers, error messages, names and technical terms.

### Semantic channel

Optional embeddings can generate approximate semantic candidates when the semantic module is enabled.

### Graph channel

Follow high-confidence relations from active entities/topics/memories.

### Experience channel

Search structured cases by problem characteristics, conditions, failures, solutions and outcomes.

Each channel returns candidates with provenance and a channel score.

## 4. Applicability filtering

Before expensive ranking, evaluate whether the memory's conditions match the current context.

Consider:

```text
technology
platform
project
scale
environment
user intent
time/state
constraints
known exceptions
```

A memory may be highly similar but inapplicable.

Example:

```text
Memory: use SQLite
Context: embedded local application
→ potentially applicable

Memory: use SQLite
Context: distributed high-write architecture
→ potentially inapplicable
```

The system should retain the memory as historical knowledge while excluding it from the active context when conditions conflict.

## 5. Contextual ranking

A candidate's final utility should be a composite score rather than one similarity value.

Conceptually:

```text
utility =
    relevance
  + scope_fit
  + applicability
  + recency_value
  + reliability
  + experience_success
  + graph_proximity
  + task_specificity
  - contradiction_risk
  - redundancy
  - context_cost
```

The exact coefficients must be calibrated by benchmark rather than hardcoded as assumed truths.

### Important distinction

`importance` and `retrieval_utility` are different.

A memory can be historically important but useless for the current task.

## 6. Graph expansion

After initial candidates are ranked, expand only from high-confidence/high-utility nodes.

Expansion should be bounded by:

```text
max hops
max nodes
max cost
relation allowlist
confidence threshold
```

Do not traverse the entire semantic graph.

## 7. Contradiction handling

Contradictory memories should not simply be averaged.

The system should identify:

```text
same subject
+ incompatible claims
+ differing conditions/time/version
```

Then prefer the claim whose applicability, provenance, recency, validation and context fit are strongest.

If uncertainty remains material, compile the conflict explicitly instead of silently choosing one.

Example:

```text
Previous solution: X worked in environment A.
Later evidence: X failed in environment B.
Current environment: B.
```

The correct retrieval is not “X works.” It is the conditional knowledge with its environment.

## 8. Deduplication

The candidate set should be collapsed when several records represent the same underlying information.

Preferred order:

```text
exact duplicate
→ version duplicate
→ semantic duplicate
→ overlapping evidence
```

Do not delete historical records merely because their compiled representations are redundant.

## 9. Token-budget allocation

The compiler receives a hard budget.

Allocate space according to expected utility:

```text
critical current state
> directly applicable experience
> active constraints/decisions
> high-confidence supporting memory
> useful historical context
> weak associations
```

Low-utility candidates should be dropped rather than allowed to consume the budget.

## 10. Memory compilation

The final result should be a compact structured projection, for example:

```text
[CURRENT STATE]
...

[RELEVANT MEMORY]
- fact
- constraint
- prior solution

[EXPERIENCE]
- previous attempt
- result
- applicability

[CAVEAT]
- conflicting evidence / exception

[SOURCE]
- memory IDs / provenance references
```

The model should not receive raw SQL rows, internal database metadata, or irrelevant provenance payloads.

## Retrieval modes

### Continuity mode

Used when reconstructing an ongoing session after compaction.

Priority:

```text
session state
→ latest snapshot
→ active decisions/constraints
→ unresolved work
→ recent relevant messages
→ supporting memories
```

### Task mode

Used for a new task.

Priority:

```text
current project
→ applicable experience
→ relevant knowledge
→ user behavioral preferences
→ broader global knowledge
```

### Troubleshooting mode

When a failure/error is detected:

```text
current failure signature
→ previous similar cases
→ successful attempts
→ failed attempts to avoid
→ applicable solutions
→ lessons
```

This is the mechanism behind the planned “same problem six months later” behavior.

### Learning mode

When Hermes is learning a new technology/skill:

```text
source knowledge
→ procedures
→ known edge cases
→ examples
→ prior learning experiences
→ validation history
```

Learning mode must not automatically promote everything learned to global behavioral memory.

## Negative transfer protection

A memory that changes recommendations must carry applicability information.

Examples of unsafe promotion:

```text
"User studied PostgreSQL"
≠
"Always use PostgreSQL"

"SQLite solved project A"
≠
"Use SQLite in every project"
```

Behavioral adaptation must therefore be conditional and scoped.

## Feedback loop

Retrieval quality should be measurable.

After an interaction, HCME may record:

```text
retrieved_memory_ids
used_memory_ids
ignored_memory_ids
helpful/not_helpful signal
result
```

Successful reuse can strengthen a relation/experience under the relevant conditions. A failed recommendation should reduce confidence for that context rather than globally deleting the memory.

## Safety against retrieval pollution

The retrieval layer must prevent:

- archived memories becoming active accidentally;
- low-confidence memories outranking validated ones without evidence;
- one session's private context leaking into another;
- contradictory memories being merged as facts;
- huge graph traversals;
- unbounded context growth;
- stale solutions being treated as universal rules.

## Deterministic baseline

The first HCME retrieval implementation should work without embeddings.

Baseline:

```text
scope filters
+
SQLite/FTS5 lexical retrieval
+
explicit topic/entity links
+
applicability metadata
+
experience matching
+
bounded ranking
```

This baseline gives us a reproducible benchmark before adding a semantic/vector layer.

## Evaluation metrics

The retrieval system should eventually measure:

- precision of retrieved memories;
- recall of intentionally relevant memories;
- cross-session leakage rate;
- negative-transfer rate;
- context tokens consumed;
- useful-memory tokens consumed;
- reconstruction fidelity after compaction;
- successful reuse of previous solutions;
- false confidence/contradiction rate;
- retrieval latency.

The most important system-level metric is not retrieval recall alone:

> **Does the agent make better decisions with HCME than without it while consuming less relevant context than naive history replay?**
