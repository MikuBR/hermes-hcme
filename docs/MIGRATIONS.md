# HCME Migration Contract

## Status

Architecture / Research Phase.

HCME must be able to evolve its schema without losing durable memory or leaving the database in an ambiguous state.

## Principles

1. Migrations are explicit and versioned.
2. A migration is atomic whenever SQLite permits it.
3. Backups/snapshots precede destructive or high-risk changes.
4. Existing memory provenance must survive schema changes.
5. Optional modules own their own migrations.
6. Downgrade must not be assumed safe unless explicitly implemented.
7. A failed migration must leave the previous valid schema recoverable.

## Versioning

The database has a monotonic core schema version.

```text
schema_meta
└── schema_version
```

Optional modules also have independent versions in `module_registry`.

Example:

```text
core = 4
experience = 2
semantic = 1
embeddings = 0
```

## Migration lifecycle

```text
inspect
  ↓
validate compatibility
  ↓
backup / checkpoint if required
  ↓
BEGIN
  ↓
apply migration
  ↓
validate schema + invariants
  ↓
update schema version
  ↓
COMMIT
```

If validation fails:

```text
ROLLBACK
→ restore checkpoint if necessary
→ report failure
→ do not claim migration succeeded
```

## Startup behavior

HCME should check schema compatibility before opening the database for normal writes.

Possible states:

```text
CURRENT
MIGRATION_REQUIRED
MIGRATION_IN_PROGRESS_RECOVERABLE
INCOMPATIBLE
CORRUPT
```

The agent must not silently continue normal writes against an incompatible schema.

## Migration registry

Each migration should declare:

```text
id
module
from_version
to_version
checksum
description
risk_level
requires_backup
reversible
```

Migration files should be immutable once released. Corrections require a new migration.

## Safety classes

### Low risk

Adding an index or nullable column without changing existing semantics.

### Medium risk

Data backfill, table rebuild, normalization, or large index creation.

### High risk

Dropping data, changing identifiers, changing scope semantics, or irreversible transformations.

High-risk migrations require an explicit backup/checkpoint and a recovery plan.

## Data migrations

A schema migration and a semantic data migration should be conceptually distinguishable.

For example:

```text
schema migration:
add applicability column

semantic migration:
populate applicability from legacy metadata
```

Backfills must be resumable or transactional. They must never invent provenance.

## Integrity checks

After migration, HCME should verify at minimum:

- foreign-key integrity;
- required indexes;
- schema version;
- unique identifiers;
- valid enum/status values;
- valid scope values;
- snapshot parent relationships;
- module versions;
- FTS consistency when applicable.

## Backup policy

The first production implementation should maintain rolling database backups before high-risk migrations and allow explicit manual snapshots.

Backups must be clearly distinguished from memory exports: a backup is intended for recovery; an export is intended for interoperability.

## Recovery

If a migration cannot complete safely:

1. stop normal writes;
2. preserve the failing database for diagnosis;
3. restore the last known-good backup/checkpoint when appropriate;
4. record the failed migration and reason;
5. restart only after schema compatibility is restored.

## Modular schema

Future components such as embeddings, cloud synchronization, Obsidian adapters, or experience-factory metadata must not force their tables into the core schema unless the feature is enabled.

Modules should register their migrations explicitly.

## Agent-driven schema changes

Future HCME versions may allow Hermes to propose schema extensions when it encounters a new durable memory domain.

This is deliberately constrained:

```text
Hermes proposes
      ↓
Schema Manager validates
      ↓
migration plan generated
      ↓
backup/checkpoint
      ↓
human/automation policy approval
      ↓
migration
      ↓
integrity verification
```

The language model must never execute arbitrary SQL schema mutations without passing through the migration manager and its safety policy.

## Compatibility target

The database contract should remain backward-readable for at least the supported HCME upgrade window. Exact support duration will be defined when implementation and release policy are established.
