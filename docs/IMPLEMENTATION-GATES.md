# HCME Implementation Gates

## Purpose

HCME should not move from architecture to implementation merely because the design is ambitious. Each layer must earn implementation status by passing explicit gates.

## Gate 0 — Architecture

Required:

- memory model defined;
- context model defined;
- database contract defined;
- retrieval algorithm defined;
- scope boundaries defined;
- non-goals documented.

Status: **passed**.

## Gate 1 — SQLite core

Implement only:

- database initialization;
- migrations;
- core memory tables;
- provenance/events;
- integrity checks;
- basic CRUD;
- backup/restore primitives.

No embeddings. No autonomous schema mutation. No complex ranking.

Exit criteria:

- fresh database initializes correctly;
- migrations are deterministic;
- foreign keys pass;
- integrity checks pass;
- backup restores to an equivalent database;
- concurrent readers/writers behave within expected limits.

## Gate 2 — Session/context continuity

Implement:

- logical sessions;
- physical-session bindings;
- context state;
- snapshots;
- reconstruction.

Exit criteria:

A controlled conversation remains semantically continuous after repeated simulated compactions without requiring the entire historical transcript in the active prompt.

## Gate 3 — Baseline retrieval

Implement:

- hard scope gates;
- FTS5 candidate generation;
- topic/entity retrieval;
- applicability filtering;
- bounded ranking;
- deduplication;
- context compilation.

Exit criteria:

- unrelated session memories are not retrieved;
- directly relevant memories are retrieved;
- context stays within a configured budget;
- retrieval latency is measurable;
- retrieval is reproducible without embeddings.

## Gate 4 — Experience reuse

Implement:

- cases;
- attempts;
- solutions;
- lessons;
- troubleshooting retrieval;
- feedback recording.

Exit criteria:

A previously solved problem can be recognized, the old solution can be considered under current conditions, and a failed old solution can be avoided.

## Gate 5 — Behavioral adaptation

Implement:

- conditional behavior rules;
- activation/inhibition conditions;
- scoped preferences;
- promotion/demotion based on evidence.

Exit criteria:

Behavior changes when context warrants it and does not become an unconditional global rule from a single observation.

## Gate 6 — Hermes integration

Integrate through supported Hermes extension points.

Exit criteria:

- HCME can operate as an extension without modifying Hermes core unnecessarily;
- existing session storage remains valid;
- Hermes Skills continue to work unchanged;
- HCME can be disabled cleanly;
- failures degrade safely instead of blocking the agent.

## Gate 7 — Semantic retrieval

Only after the deterministic baseline is benchmarked should embeddings/vector retrieval be added.

Exit criteria:

Semantic retrieval must improve benchmark results enough to justify its storage, latency, and operational complexity.

## Gate 8 — Learning integration

Connect HCME to `/learn` and Skill metadata only after memory/retrieval correctness is established.

Learning must distinguish:

```text
source knowledge
procedure
experience
validation
promotion
```

Reading a repository is not equivalent to validating a capability.

## Gate 9 — Experience Factory

Only after the learning layer is stable.

The Experience Factory should generate controlled scenarios, execute or simulate capabilities in a safe environment, collect objective results, and maintain regression evidence.

## Gate 10 — External adapters

Optional integrations such as Obsidian, Notion, cloud storage, or n8n events come last.

The HCME core must remain useful without them.

## Universal rule

A later layer must never become a hidden dependency of an earlier layer.

```text
Core DB
  ↓
Context
  ↓
Retrieval
  ↓
Experience
  ↓
Behavior
  ↓
Hermes integration
  ↓
Semantic layer
  ↓
Learning
  ↓
Experience Factory
  ↓
External adapters
```

Each stage must be independently testable.
