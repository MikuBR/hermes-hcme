# HCME Context Model

## Objective

Allow Hermes to maintain coherent conversations across long sessions and repeated compactions without injecting irrelevant historical memory into the current task.

## Three layers

HCME separates three fundamentally different things:

### Raw history

The original session messages and tool events. Hermes' `state.db` remains authoritative for this operational history.

### Durable state

A compact structured representation of what currently matters:

```text
goal
current_task
subtask
project
constraints
decisions
open_questions
working_assumptions
next_actions
active_topics
active_entities
behavior_mode
```

### Prompt projection

The small, temporary context actually provided to the model for the current turn.

The prompt projection is disposable and can be regenerated from durable state, recent messages, and relevant memory.

## Logical session

A conversation may be represented by multiple physical Hermes sessions after compaction or other lifecycle operations.

HCME therefore distinguishes:

```text
logical_session
    ├── physical session A
    ├── physical session B
    ├── physical session C
    └── ...
```

Lineage must be preserved so the user experiences one continuous conversation.

## Context State lifecycle

```text
turn begins
   ↓
observe current request
   ↓
update task/state hypotheses
   ↓
retrieve relevant memory
   ↓
compile prompt projection
   ↓
agent execution
   ↓
observe result
   ↓
commit state/events/memory candidates
```

Writes should be asynchronous where safe, but state required for correctness must be committed before the session transitions into a new durable state.

## Relevance gates

Memory retrieval is staged.

### Gate 1 — scope

Reject memories that are not allowed by session/project/user/global scope.

### Gate 2 — task relevance

Identify the current objective, entities, topics, constraints, and problem type.

### Gate 3 — applicability

Reject memories whose conditions conflict with the current context.

### Gate 4 — evidence quality

Prefer validated experience and strong evidence over unsupported model-generated memories.

### Gate 5 — budget

Only the highest-utility material survives compilation into the prompt.

## Memory projection

A memory should normally be represented to the model in a compact form such as:

```text
Prior relevant experience:
- Situation: ...
- What worked: ...
- Caveat: ...
- Confidence: ...
```

The model should not receive raw database records unless explicitly needed for investigation.

## Context snapshots

Snapshots are immutable versions of durable context state.

A snapshot should identify:

```text
snapshot_id
logical_session_id
parent_snapshot_id
state
active_memory_refs
source_message_range
created_at
schema_version
```

Snapshots allow reconstruction without repeatedly summarizing prior summaries.

## Compaction

Compaction is a context projection operation, not data destruction.

```text
old messages
    ↓
extract state / events / durable candidates
    ↓
persist provenance
    ↓
create snapshot
    ↓
construct compact prompt projection
```

Raw history remains available through Hermes' session storage.

## Multi-compaction invariant

After any number of compactions, the following must remain reconstructable:

- current objective;
- important decisions;
- active constraints;
- unresolved problems;
- relevant prior work;
- current project identity;
- important user-provided information from the logical session;
- provenance for durable memories.

## Cache awareness

HCME must avoid unnecessary changes to stable prompt prefixes. Memory retrieval should be inserted through the appropriate Hermes context mechanisms rather than continuously mutating the system prompt.

Context selection should be deterministic where possible for a given state, memory snapshot, and retrieval configuration.

## Cross-session isolation

A new session starts with no automatic access to unrelated session memories.

Cross-session recall requires a deliberate retrieval path based on project/user/global scope or an explicit connection to the prior case.

Example:

```text
Session A: Minecraft shader debugging
Session B: SQLite project

Session B must not receive shader memories merely because both contain the word "error".
```

## Recovery strategy

When context is missing after compaction or restart:

```text
current physical session
→ logical session lineage
→ latest valid snapshot
→ relevant durable state
→ targeted historical retrieval
→ prompt projection
```

The system should recover the minimum necessary information rather than replaying the entire conversation.

## Failure handling

If HCME cannot safely reconstruct a state, it should degrade gracefully:

1. preserve the current Hermes behavior where possible;
2. report uncertainty internally;
3. avoid inventing missing memories;
4. retrieve source history when available;
5. prefer omission over false continuity.
