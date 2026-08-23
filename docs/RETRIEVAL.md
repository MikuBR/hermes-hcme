# HCME Retrieval Contract

## Status

Architecture / Research Phase.

This document defines how HCME decides which durable memories may influence a turn. Retrieval is a safety and context-efficiency boundary, not simply a search feature.

## Core invariant

> A memory must be permitted by scope before relevance ranking can make it visible to the model.

Semantic similarity is never a permission system.

## Pipeline

```text
current user turn
      ↓
turn understanding
      ↓
scope gate
      ↓
project/session linkage
      ↓
candidate generation
      ↓
applicability filtering
      ↓
hybrid ranking
      ↓
conflict / duplicate handling
      ↓
utility + token budget
      ↓
Memory Compiler
      ↓
prompt projection
```

## 1. Turn understanding

The retrieval system should derive a lightweight query representation:

```text
goal
current task
problem type
entities
topics
project
constraints
requested technology/tools
explicit references to prior work
```

The system must distinguish an explicit recall request such as "what did we do six months ago?" from ordinary conversation.

## 2. Scope gate

Candidate access is determined before semantic ranking.

Default order:

```text
current logical session
    ↓
active project
    ↓
user-scoped memory
    ↓
global memory
```

Session memories are not promoted merely because they are old, frequently used, or semantically similar.

Project/user/global retrieval requires an applicable scope relationship or an explicit recall path.

## 3. Candidate generation

The initial implementation should use inexpensive signals first:

- exact identifiers;
- FTS5 lexical search;
- topic/entity indexes;
- project/session filters;
- explicit relation traversal.

Semantic embeddings are optional and later. They must complement, not replace, lexical and structural retrieval.

## 4. Applicability gate

A candidate may be semantically related but operationally wrong.

Evaluate:

- prerequisites;
- environment/platform;
- project/task type;
- known exclusions;
- temporal validity;
- contradiction state;
- confidence/evidence quality.

Example:

```text
Memory: "Use SQLite"
Current task: distributed high-concurrency service

Result: candidate may be related, but applicability should reject or heavily penalize it.
```

## 5. Ranking

The implementation should use a multi-signal score rather than a single similarity value.

Conceptually:

```text
utility =
    relevance
  × scope_fit
  × applicability
  × evidence_quality
  × confidence
  × relationship_strength
  × historical_usefulness
  × recency_factor
  × task_alignment
  - contradiction_penalty
  - redundancy_penalty
```

The exact equation is intentionally implementation-defined. The factors must remain separately observable so retrieval can be debugged.

## 6. Negative transfer protection

A memory can be correct and still be harmful in the current situation.

HCME must explicitly model negative transfer:

```text
correct memory
    + wrong context
    = bad retrieval
```

Therefore a memory that repeatedly causes incorrect decisions in a context should accumulate negative applicability evidence.

This evidence should lower future retrieval priority without deleting the original memory.

## 7. Experience retrieval

For a detected problem, HCME should search for prior cases in this order:

```text
same problem + same environment
        ↓
same problem class + similar environment
        ↓
similar problem + related environment
        ↓
general lesson
```

The system should present the previous solution as a conditional precedent:

```text
Prior case:
...

Worked because:
...

Current differences:
...

Risk/caveat:
...
```

This is preferable to blindly replaying the previous solution.

## 8. Contradiction handling

Contradictory memories should not both be injected into the prompt merely because they rank highly.

The retrieval layer should detect conflicts and either:

- choose the best-supported applicable record;
- present a concise conflict notice;
- retrieve the evidence needed to resolve it.

Historical contradictions remain stored.

## 9. Deduplication

Multiple atoms may express the same fact or lesson. The compiler should collapse redundant material before prompt insertion while retaining provenance references.

Deduplication must not merge records merely because their text is similar if they represent different contexts or outcomes.

## 10. Memory budget

Retrieval has two budgets:

1. candidate budget — how many records may be evaluated;
2. prompt budget — how many tokens may be projected to the model.

The prompt budget must be explicit and configurable.

The compiler should prefer:

```text
small number of high-utility memories
```

over:

```text
large number of weakly related memories
```

## 11. Explicit recall mode

When the user explicitly asks to recall prior work, retrieval may widen its normal scope, but it still must respect authorization and privacy boundaries.

Example:

```text
"Six months ago we fixed this. What did we do?"
```

should trigger historical case search using project/topic/problem anchors rather than normal conversational retrieval alone.

## 12. Retrieval provenance

Every prompt-projected memory should be traceable to:

```text
memory uid
version
scope
source/evidence
retrieval reason
ranking signals
```

The model need not see all metadata, but the system must retain it for debugging and evaluation.

## 13. Determinism

Given identical:

- database state;
- logical session state;
- retrieval configuration;
- query representation;

retrieval should be deterministic where possible. If an approximate/LLM-based stage is later introduced, its inputs and configuration must be recorded for reproducibility.

## 14. Failure behavior

If retrieval fails:

- continue with current Hermes behavior when possible;
- never fabricate a retrieved memory;
- prefer no memory over low-confidence contamination;
- expose diagnostics to logs/debug tooling;
- preserve the session even if the memory subsystem is unavailable.

HCME must be an enhancement, not a single point of failure for Hermes.

## 15. Evaluation metrics

The project should eventually measure:

- relevant-memory precision;
- relevant-memory recall;
- irrelevant-memory rate;
- cross-session leakage rate;
- negative-transfer rate;
- retrieval latency;
- prompt-token cost;
- successful prior-case reuse;
- contradiction resolution accuracy.

The most important safety metric is **irrelevant-memory leakage**, not raw recall.
