# HCME Architecture

## Status

Architecture / Research Phase.

HCME (Hermes Context & Memory Engine) is a planned subsystem for Hermes Agent that combines persistent memory, context reconstruction, selective retrieval, experiential learning, and adaptive behavior while preserving Hermes' existing architecture.

## Core principle

> The database stores durable knowledge and provenance; context is a temporary, task-specific projection of that knowledge.

The system must optimize for **relevance**, not maximum recall. A memory being important historically does not mean it belongs in the current prompt.

## Architectural boundaries

### Hermes `state.db`

Remains the authoritative store for raw session history and operational session state owned by Hermes.

### HCME `memory.db`

Stores durable memory, experience, semantic organization, relations, evidence, versions, context snapshots, and learning metadata.

### Hermes Skills

Remain the procedural/executable capability layer. HCME may record their provenance, applicability, validation history, failures, and improvements, but does not replace the existing Skill format.

## Memory scopes

Default isolation is session-first:

1. `session` — ephemeral to a logical conversation.
2. `project` — promoted knowledge relevant to a project.
3. `user` — durable user facts/preferences.
4. `global` — broadly applicable knowledge.
5. `behavior` — contextual rules describing how Hermes should adapt its approach.

Cross-session retrieval must pass explicit relevance and scope gates. Semantic similarity alone is never sufficient to leak one session's memory into another.

## Memory classes

HCME distinguishes at least:

- episodic experience — what happened;
- semantic knowledge — what is known;
- procedural knowledge — how something is done;
- behavioral memory — how Hermes should adapt in a context;
- failures and solutions — what was attempted, what failed, and what worked;
- decisions, constraints, goals, and project state.

## Semantic Atlas

Memory is organized through a dynamic hierarchy of themes plus cross-links. For example:

```text
ENTERTAINMENT
└── ANIME
    └── SHONEN
        ├── One Piece
        ├── Naruto
        └── Dragon Ball
```

The hierarchy is an indexing and routing structure, not a rigid folder system. A concept may belong to several topics through graph relations.

## Associative graph

Memory nodes are connected by typed relations such as:

- `related_to`
- `derived_from`
- `solves`
- `failed_in`
- `improves`
- `used_in`
- `belongs_to`
- `learned_from`
- `supersedes`
- `contradicts`

Relations carry contextual information such as strength, confidence, applicability, conditions, exceptions, and success/failure history.

## Context architecture

HCME separates:

- raw history;
- session state;
- durable memory;
- retrieval candidates;
- compiled context.

The model should receive only a small, clean projection of relevant information.

### Context state

A logical session maintains state such as:

- objective;
- current task/subtask;
- active project;
- active entities/topics;
- decisions;
- constraints;
- open questions;
- working assumptions;
- next actions;
- active behavioral mode.

### Context snapshots

Snapshots are durable, versioned representations of the state necessary to continue a session after compaction. They are not replacements for raw history.

### Reconstruction

After compaction, context is rebuilt from:

```text
latest turns
+ current session state
+ latest snapshot
+ relevant session/project/global memories
+ applicable behavior
+ task-specific experience
```

Compaction must not require recursive `summary -> summary -> summary` as the only source of truth.

## Retrieval pipeline

The planned retrieval pipeline is:

```text
current turn
    ↓
context understanding
    ↓
hard scope/session/project gates
    ↓
lexical + semantic + graph candidate retrieval
    ↓
contextual reranking
    ↓
conflict/deduplication checks
    ↓
memory compiler
    ↓
token-budgeted context projection
```

The retrieval layer should prefer high-utility evidence over large quantities of loosely related memories.

## Memory Compiler

The Memory Compiler converts rich stored structures into concise prompt-ready material.

Storage may contain thousands of tokens of historical evidence while the model receives only a compact projection containing the minimum useful facts, prior experience, caveats, and provenance needed for the current task.

## Experiential memory

A recurring problem should be represented as a case rather than only a textual memory:

```text
problem
  ↓
attempts
  ↓
results
  ↓
solution / failure
  ↓
lesson
```

A future problem may retrieve a related case, compare its conditions with the current situation, adapt the prior solution, test it, and record the new result.

Historical solutions are hypotheses with applicability conditions, not universal rules.

## Learning and Skills

`/learn` remains the acquisition entry point. HCME/HLE are intended to enrich the learning pipeline by extracting knowledge, procedures, experiences, and candidate capabilities before a Skill is promoted.

Future Skill Experience Factory work will generate controlled scenarios, execute or simulate them in safe environments, collect objective feedback, refine Skills, and maintain regression evidence. Reading source material alone is not considered proof of competency.

## Behavioral adaptation

Behavioral memory can influence planning and tool strategy, but only through contextual activation. A learned preference must not become a global hard rule unless explicitly promoted and validated.

Example:

```text
IF task = destructive configuration change
THEN prefer backup + validation
```

Knowledge about one technology must not force that technology in unrelated projects.

## Compatibility requirements

HCME must:

- integrate through Hermes' supported Memory/Context extension points;
- preserve existing session storage and Skill behavior;
- avoid modifying the stable system prompt on every memory write;
- remain compatible with Hermes profiles / `HERMES_HOME` isolation;
- work offline for its core functionality;
- survive Hermes upgrades through explicit migrations;
- maintain provenance and reversible history for durable memory changes.

## Planned implementation stages

1. Memory Core + SQLite schema.
2. Memory atoms, versions, events, evidence, relations.
3. Scope isolation + Semantic Atlas.
4. Session state + context snapshots.
5. Retrieval + Memory Compiler.
6. Hermes Memory Provider integration.
7. HCME ContextEngine integration.
8. Multi-compaction continuity benchmark.
9. Experiential memory and case reuse.
10. Behavioral memory.
11. Learning Engine integration.
12. Skill Experience Factory.
13. Optional semantic embeddings.
14. Optional external adapters (Obsidian, cloud, Notion, automation).

## Non-goals for the initial implementation

- replacing Hermes' entire agent loop;
- replacing `state.db`;
- replacing the existing Skill format;
- requiring a cloud service;
- requiring an external vector database;
- treating embeddings as the sole source of relevance;
- making all memories globally visible;
- claiming human-like cognition.
