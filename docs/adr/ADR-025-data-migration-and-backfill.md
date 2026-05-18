# ADR-025: Data Migration and Backfill Strategy

## Status
**ACCEPTED** - 2026-03-17

## Context
Schema changes and new read models require consistent migration and backfill
to keep services reliable and reduce operational risk.

## Decision
### 1. Migration Ownership
- Each service owns its own schema migrations.
- Migrations are versioned and applied in CI/CD.

### 2. Backfill Strategy
- Backfill is mandatory when introducing new read models or indexes.
- Backfill can be batch jobs or streaming consumers.

### 3. Compatibility Window
During migration:
- Old and new schema versions must coexist.
- Application must tolerate both shapes until cutover.

### 4. Observability
Backfill jobs must emit:
- Progress metrics
- Failure counts
- Lag indicators

## Consequences
- Safer schema evolution
- Predictable rollout and rollback paths

## References
- ADR-017: Event Schema Versioning
- ADR-016: DLQ and Error Handling
