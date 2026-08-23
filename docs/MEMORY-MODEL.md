# HCME Memory Model

## Purpose

Define the durable information model before implementation. This document is a semantic contract, not a database schema.

## Memory Atom

A Memory Atom is the smallest independently addressable durable unit of meaning.

A conceptual atom contains:

```text
id
kind
content
scope
session_id / logical_session_id
project_id
source
provenance
created_at
updated_at
confidence
importance
applicability
status
```

The content may be textual, structured, or a reference to durable evidence.

## Atom kinds

Initial taxonomy:

| Kind | Meaning |
|---|---|
| `fact` | Stable information believed to be true |
| `event` | Something that happened |
| `experience` | An event plus context and outcome |
| `decision` | A decision made during work |
| `constraint` | A condition that limits the task |
| `goal` | An intended outcome |
| `preference` | A user or project preference |
| `knowledge` | General semantic knowledge |
| `procedure` | A method or sequence of actions |
| `problem` | A recognized issue |
| `attempt` | An attempted solution/action |
| `solution` | An approach with evidence of success |
| `failure` | An approach that failed under known conditions |
| `lesson` | A generalized conclusion derived from experience |
| `behavior` | A contextual behavioral adaptation |
| `topic` | Semantic organization node |
| `entity` | A person, project, technology, place, or other identifiable concept |

The taxonomy must remain extensible. Adding a new kind must not require destructive migration of existing memories.

## Scope

Scope is not the same thing as importance.

```text
session → project → user → global
```

is a conceptual promotion path, not an automatic inheritance rule.

A memory can be highly important but remain session-scoped because its applicability is uncertain.

## Promotion

Promotion should require evidence or an explicit user instruction.

Example:

```text
session experience
      ↓ repeated / validated
project knowledge
      ↓ broadly applicable evidence
user/global knowledge
```

Promotion must retain provenance so that the system can trace where the generalized memory came from.

## Applicability

Every durable memory that can affect decisions should be able to describe:

- contexts where it applies;
- contexts where it does not apply;
- prerequisites;
- known exceptions;
- confidence;
- evidence quality.

This prevents a useful memory from becoming an unconditional rule.

## Contradictions

Contradictory memories should coexist until resolved.

The system must not silently overwrite historical evidence.

Instead it should represent:

```text
memory A
   ↕ contradicts
memory B
```

with timestamps, contexts, evidence, and confidence.

A later validated result may supersede an earlier conclusion while preserving the historical record.

## Experience Case

An Experience Case is a structured record of an attempted task:

```text
case
├── problem
├── context
├── constraints
├── attempts
│   ├── action
│   ├── expected outcome
│   ├── actual outcome
│   └── evidence
├── successful solution(s)
├── failures
├── lesson(s)
└── applicability
```

Cases are the foundation for future experience reuse.

## Relations

Relations are first-class objects, not merely foreign keys.

Conceptual fields:

```text
source_id
target_id
relation_type
strength
confidence
conditions
exceptions
success_count
failure_count
first_seen
last_seen
provenance
```

## Memory lifecycle

```text
observed
  ↓
candidate
  ↓
validated
  ↓
active
  ↓
re-evaluated
  ↓
confirmed / superseded / archived
```

A memory should not automatically become durable merely because the model generated it.

## Retrieval priority

A memory's retrieval priority should be derived from multiple signals rather than one score:

```text
relevance
+ scope
+ recency
+ task similarity
+ relationship strength
+ confidence
+ applicability
+ historical usefulness
+ contradiction risk
```

The exact scoring function is intentionally left for the implementation design phase.

## Forgetting / archival

The system should distinguish:

- active memory;
- dormant memory;
- archived memory;
- superseded memory.

Archival is not deletion. Durable evidence should remain recoverable unless explicitly deleted by the user or required by a retention policy.

## Privacy boundary

Memory scope must be enforced before semantic retrieval is allowed to rank candidates. The system must not rely on the language model to decide whether a memory is permitted to cross a scope boundary.
