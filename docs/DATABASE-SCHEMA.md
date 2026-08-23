# HCME Database Schema Contract

## Status

Architecture / Research Phase. This document defines the logical schema contract for HCME `memory.db`. It is intentionally independent of a final SQL migration so the model can evolve before implementation.

## Design goals

The database must be:

- local-first and offline-capable;
- profile-scoped by `HERMES_HOME`;
- append-aware and provenance-preserving;
- modular enough to add memory domains later;
- efficient for session-scoped retrieval;
- safe to migrate across HCME versions;
- separate from Hermes' authoritative raw session store (`state.db`);
- able to represent both simple facts and multi-step experiences.

## Storage boundaries

```text
Hermes state.db
    └── raw operational session history

HCME memory.db
    ├── durable memory
    ├── logical session state
    ├── context snapshots
    ├── semantic organization
    ├── experiences and outcomes
    ├── relations and provenance
    └── retrieval indexes

Hermes skills/
    └── executable/procedural artifacts
```

HCME should reference Hermes history rather than duplicate full message bodies unless a future migration explicitly requires otherwise.

## Core entities

### `memories`

Canonical durable memory records.

Conceptual fields:

```text
id                 INTEGER PK
uid                TEXT UNIQUE
kind               TEXT
scope              TEXT
content            TEXT
summary            TEXT NULL
logical_session_id TEXT NULL
project_id         TEXT NULL
source_type        TEXT NULL
source_id          TEXT NULL
importance         REAL
confidence         REAL
status             TEXT
created_at         INTEGER
updated_at         INTEGER
last_accessed_at   INTEGER NULL
applicability      JSON/TEXT NULL
metadata           JSON/TEXT NULL
content_hash       TEXT NULL
```

`kind` remains extensible. The current taxonomy is defined in `MEMORY-MODEL.md`.

`scope` is a hard access boundary, not merely a ranking feature.

### `memory_versions`

Immutable revisions of canonical memory content.

```text
id
memory_id
version
content
summary
metadata
change_type
reason
created_at
created_by
```

A memory may have one current representation while retaining all historical versions.

### `memory_events`

Append-oriented lifecycle/audit events.

```text
id
memory_id
event_type
actor_type
actor_id
source_type
source_id
payload
created_at
```

Examples include `created`, `updated`, `promoted`, `validated`, `superseded`, `archived`, `restored`, `merged`, `split`, and contradiction events.

### `evidence`

Provenance pointing back to durable sources.

```text
id
memory_id
source_kind
source_id
locator
excerpt_hash
confidence
created_at
metadata
```

For Hermes session evidence, `source_id` should identify the relevant session/message/event in `state.db` rather than copying the entire source into `memory.db`.

## Session/context entities

### `logical_sessions`

Represents one continuous conversation across physical Hermes session records.

```text
id
uid
root_session_id
project_id
source_platform
started_at
last_activity_at
status
metadata
```

### `session_bindings`

Maps logical HCME sessions to physical Hermes sessions.

```text
id
logical_session_id
physical_session_id
parent_physical_session_id
bound_at
unbound_at
reason
```

### `context_states`

Current durable state for a logical session.

```text
logical_session_id PK
version
goal
current_task
current_subtask
project_id
constraints
working_assumptions
decisions
open_questions
next_actions
active_topics
active_entities
behavior_mode
updated_at
```

The state is mutable, but every meaningful transition should produce an event or snapshot reference.

### `context_snapshots`

Immutable reconstructible checkpoints.

```text
id
logical_session_id
parent_snapshot_id
state_payload
active_memory_refs
source_message_range
created_at
schema_version
snapshot_hash
```

Snapshots are recovery material, not replacements for raw history.

## Semantic organization

### `topics`

Nodes in the Semantic Atlas.

```text
id
uid
name
parent_topic_id
kind
confidence
importance
status
created_at
updated_at
metadata
```

A topic hierarchy is advisory/indexing structure. Many-to-many assignments are handled by relations.

### `entities`

Canonical concepts/persons/projects/technologies/etc.

```text
id
uid
name
canonical_name
entity_type
confidence
status
created_at
updated_at
metadata
```

### `memory_topics`

Connects memories to multiple topic nodes.

```text
memory_id
topic_id
strength
confidence
source
created_at
```

### `memory_entities`

Connects memories to active entities.

```text
memory_id
entity_id
role
strength
confidence
created_at
```

### `relations`

First-class semantic/experiential graph edges.

```text
id
source_type
source_id
target_type
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
metadata
```

Relations must support both semantic links (`related_to`, `belongs_to`) and experiential links (`solves`, `failed_in`, `derived_from`, `improves`, `supersedes`).

## Experience entities

### `cases`

Structured problem-solving episodes.

```text
id
uid
problem_memory_id
logical_session_id
project_id
title
context_payload
status
confidence
created_at
updated_at
```

### `attempts`

Ordered actions within a case.

```text
id
case_id
sequence
action
expected_outcome
actual_outcome
status
evidence_id
created_at
```

### `solutions`

Candidate or validated approaches attached to cases.

```text
id
case_id
content
status
confidence
success_count
failure_count
applicability
created_at
updated_at
```

A solution is never a universal rule. Its conditions and exceptions are part of the record.

### `lessons`

Generalized conclusions derived from one or more experiences.

```text
id
uid
content
confidence
source_case_count
applicability
created_at
updated_at
status
```

## Behavioral entities

### `behavior_rules`

Contextual rules that can alter Hermes' strategy.

```text
id
uid
content
trigger
scope
confidence
priority
activation_conditions
inhibiting_conditions
source_memory_id
status
created_at
updated_at
```

Behavior rules are recommendations for planning/retrieval, not unconditional system instructions.

## Retrieval indexes

### `memory_fts`

FTS5 index over searchable memory representations. The final schema should prefer an external-content or content-synchronized design where practical so canonical text exists in one authoritative place.

Vector/embedding indexes are intentionally not part of the initial mandatory schema. Semantic retrieval is a later module.

## System metadata

### `schema_meta`

```text
key
value
updated_at
```

Required keys include schema version and migration state.

### `module_registry`

Tracks optional HCME modules and their schema versions.

```text
module_name
module_version
status
installed_at
updated_at
metadata
```

This enables modular schema growth without coupling the core to every future feature.

## Identifiers

Use an integer primary key for internal SQLite joins when appropriate and a stable text `uid` for external references, APIs, logs, exports, and graph edges that need a durable identifier across migrations.

A future implementation may standardize `uid` on UUIDv7/ULID if available without changing the logical contract.

## Data types and SQLite policy

Initial implementation should prefer:

- `STRICT` tables for core typed records;
- `INTEGER` Unix timestamps for efficient ordering;
- `REAL` normalized confidence/importance values;
- `TEXT` for identifiers and canonical strings;
- JSON/TEXT only for genuinely variable payloads;
- foreign keys enabled for every connection;
- WAL journaling for the local primary database.

Frequently filtered values should be normal columns, not hidden inside JSON.

## Deletion and retention

Default lifecycle is non-destructive:

```text
active → dormant → archived / superseded
```

Physical deletion should require an explicit retention operation. Tombstones may be needed to prevent stale synchronization or imports from resurrecting deleted records.

## Indexing strategy

Do not index every column. Initial indexes should favor:

- active memories by scope;
- logical-session state;
- project-scoped memory;
- memory kind/status;
- recently updated/used records;
- relation lookups by source and target;
- FTS5 retrieval.

Partial indexes should be preferred where historical/archive rows greatly outnumber active rows.

## Transaction boundaries

Operations that change canonical memory state should be atomic with their associated lifecycle event whenever feasible.

A memory promotion should not be committed without its provenance/event record.

A migration must not partially apply a module schema.

A context snapshot must be internally self-consistent before being marked current.

## Modular extension rule

A new module must declare:

```text
module name
version
required core schema version
tables/indexes introduced
migrations
rollback/recovery notes
optional dependencies
```

Modules may add tables/indexes but must not silently reinterpret existing core columns.

## Explicit non-goals

The initial database is not a graph database replacement, vector database replacement, or raw transcript archive. Those concerns may be represented through modules/adapters later.
