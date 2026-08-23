# HCME SQLite Design

## Status

Architecture / Research Phase. This document converts the logical schema into implementation-level SQLite rules without yet shipping production migrations.

## Database location and ownership

The canonical HCME database is:

```text
$HERMES_HOME/memory/memory.db
```

`$HERMES_HOME` must be obtained from Hermes' active profile/environment and never hardcoded to `~/.hermes`.

The database is owned by HCME. Hermes' `state.db` remains authoritative for raw session history and operational session metadata.

## Connection policy

Every connection must establish the HCME invariants before use:

```sql
PRAGMA foreign_keys = ON;
PRAGMA journal_mode = WAL;
```

The application should also configure a finite busy timeout and use short write transactions. Long-running retrieval, embedding, or LLM work must never hold a write transaction open.

WAL is chosen because HCME is a local agent workload with many reads and comparatively short writes. SQLite documents WAL as allowing readers and a writer to proceed concurrently, while retaining a single-writer boundary. The WAL and its associated `-shm` state are part of the database's persistent state and must be handled together during backups/copies.

## Type discipline

Core tables should use `STRICT` mode. Frequently queried scalar values should use real SQLite types:

- `INTEGER` for internal IDs and Unix timestamps;
- `TEXT` for stable UIDs, canonical text, enums, and hashes;
- `REAL` for normalized scores/confidence values;
- `BLOB` only for intentionally binary data;
- `ANY` only when preserving an intentionally heterogeneous payload is required.

`STRICT` is available in SQLite 3.37.0+ and adds datatype validation while preserving normal SQLite constraints/index behavior.

## Identifier policy

Use two identifiers where needed:

```text
INTEGER PRIMARY KEY
    fast local relational key

TEXT uid
    durable external/API/export identifier
```

Do not use randomly generated long text IDs as the primary join key for every relationship when an integer rowid-backed key is sufficient.

## Canonical vs derived data

Canonical tables hold authoritative records.

Derived structures must be rebuildable:

```text
canonical memory
    ↓
FTS index
retrieval caches
embedding index
topic statistics
```

A rebuild of a derived index must never destroy canonical memory.

## FTS5 design

HCME will use FTS5 for lexical candidate generation. SQLite FTS5 supports column filters, tokenizers, prefix/NEAR queries, and external-content tables.

The preferred design is an external-content FTS5 table or equivalent synchronized index so canonical memory text is not unnecessarily stored twice.

Initial indexed representation should prioritize:

- canonical content;
- compact summary;
- searchable aliases/keywords;
- optionally generated retrieval text.

Metadata such as scope, project, status, kind, and session must remain ordinary columns and be applied as SQL filters before expensive ranking.

FTS5 is a candidate generator, not the final relevance decision.

## Initial index strategy

Only add indexes that correspond to actual retrieval/update paths.

Expected early indexes include:

```text
memories(scope, status)
memories(logical_session_id, status)
memories(project_id, status)
memories(kind, status)
memories(updated_at)
memories(last_accessed_at)
relations(source_type, source_id)
relations(target_type, target_id)
cases(project_id, status)
attempts(case_id, sequence)
```

Where archived/history rows greatly outnumber active rows, prefer partial indexes such as an active-memory index rather than indexing every historical row.

## Normalization policy

Store frequently filtered dimensions relationally.

Use JSON/TEXT payloads only for variable structures such as:

```text
applicability conditions
provider-specific metadata
tool payload summaries
experimental module metadata
```

Do not hide scope, status, project ID, kind, timestamps, or confidence inside JSON.

## Constraints

Core tables should use:

- `NOT NULL` for required invariants;
- `CHECK` constraints for bounded values;
- `UNIQUE` constraints for canonical UIDs and natural keys where appropriate;
- foreign keys for internal relationships;
- explicit `ON DELETE` behavior;
- hash fields where deduplication/integrity depends on stable content.

A memory record should not become active without the required provenance/status invariants being satisfied.

## Transactions

### Memory creation

The following should commit atomically where possible:

```text
memory row
+
initial version
+
creation event
+
provenance/evidence links
```

### Promotion

Promotion should atomically record:

```text
scope change
+
promotion event
+
source/provenance
+
version
```

### Experience result

A completed attempt should atomically record its outcome and evidence before its result is eligible to strengthen a solution or lesson.

### Snapshot creation

A snapshot becomes current only after its payload, memory references, source range and integrity metadata are internally consistent.

## Concurrency

Expected pattern:

```text
many short readers
+ occasional short writer
```

The implementation should avoid application-level global locks around retrieval. Write serialization should be delegated to SQLite's WAL behavior plus short transactions.

For operations that absolutely require cross-process mutual exclusion, use a dedicated lock strategy rather than holding an SQLite write transaction while doing LLM/tool work.

## Durability and backups

A normal backup must treat the SQLite database and any active WAL state consistently. Prefer SQLite-aware backup APIs or a safe checkpoint/backup procedure instead of blindly copying `memory.db` while a write workload is active.

HCME should eventually provide:

```text
memory backup
memory verify
memory restore
memory export
```

A backup should include schema version and HCME version metadata.

## Integrity checks

At startup and during maintenance, HCME should be able to run:

```sql
PRAGMA quick_check;
PRAGMA foreign_key_check;
```

For strict tables, SQLite's integrity checks also validate type correctness.

Integrity failures should place HCME into a safe/degraded mode rather than allowing silent writes to a corrupted schema.

## Checkpoint policy

Do not blindly force frequent full checkpoints. WAL already has automatic checkpointing behavior. HCME should measure actual WAL growth and only add maintenance policy when benchmarks show a need.

## Vacuum and cleanup

Do not run `VACUUM` as part of normal turn processing.

Archival, FTS maintenance, checkpointing and vacuum are maintenance operations and should be decoupled from latency-sensitive inference.

## Migration policy

Every schema change is a numbered migration:

```text
001_initial
002_context_snapshots
003_semantic_atlas
004_experience_cases
...
```

Each migration must be:

- deterministic;
- idempotency-aware at the migration runner level;
- transactionally applied where SQLite permits;
- recorded in `schema_meta`;
- accompanied by integrity validation;
- tested on a copy of a previous-version database.

A migration must never silently reinterpret an existing column's meaning.

## Modular schema

Optional modules register through `module_registry` and own their additional tables/indexes/migrations.

Examples for later modules:

```text
embeddings
external_sync
skill_experience
observation_analytics
```

The core must remain functional with all optional modules disabled.

## Performance principles

1. Filter by hard scope before semantic ranking.
2. Query IDs and compact fields before loading large payloads.
3. Do not retrieve full historical records during initial candidate generation.
4. Keep write transactions short.
5. Cache stable derived metadata where profiling proves value.
6. Treat embeddings as an optional second-stage signal, not the primary index.
7. Compile memories into small prompt projections rather than passing database records directly to the model.

## Future vector/semantic layer

A future semantic module may store embeddings locally and join semantic candidates back into SQLite. The implementation must keep semantic retrieval optional so the base HCME remains deterministic, local, and useful without a vector backend.

## References

- SQLite WAL: https://sqlite.org/wal.html
- SQLite STRICT tables: https://sqlite.org/stricttables.html
- SQLite FTS5: https://sqlite.org/fts5.html
- SQLite partial indexes: https://sqlite.org/partialindex.html
