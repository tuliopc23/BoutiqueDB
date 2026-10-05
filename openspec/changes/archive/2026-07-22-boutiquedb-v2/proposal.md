> **ARCHIVED 2026-07-22** — Boutiquedb-v2 implementation cycle complete. Refinement: `BoutiqueDB-Refinement-Tasks.md`.

# BoutiqueDB v2 — OpenSpec Proposal

## Why

The current `BoutiqueDB-Swift` framework works but is stuck on a pre-2024 concurrency/observation model: writes are synchronous and `LiveQuery`/`LiveQueryOne` poll the CDC table on a 100 ms timer. To become a production-grade, SQLiteData-style Swift persistence framework for Turso, it needs modern async writes, `AsyncStream`-based observation, macro-driven schema/index generation, full Turso feature exposure, hardened CloudKit sync, and a release pipeline. The v2 framework spec defines *what* to build; this change tracks turning that spec into shipping code in a single, ordered implementation cycle.

## What Changes

- **BREAKING**: `BoutiqueDB.read` / `BoutiqueDB.write` / `BoutiqueDB.transaction` become `async throws` and run on a dedicated `DatabaseActor`.
- Replace `Timer` polling in `LiveQuery` / `LiveQueryOne` with an `AsyncStream<ChangeEvent>` backed by Turso CDC.
- Add a new `BoutiqueDBMacros` `.macro` target with `@BoutiqueTable`, `@FTSIndex`, `@VectorIndex`, and `@MaterializedView` macros.
- Expose the full Turso feature matrix via typed Swift APIs: MVCC (`BEGIN CONCURRENT`), FTS, vector search, materialized views, encryption at rest, multi-process WAL, and extensions.
- Introduce a `SyncAdapter` protocol; keep `CloudKitSyncAdapter` as the default, with a pluggable seam for future Turso Cloud sync.
- Integrate `swift-dependencies`, `swift-perception`, `swift-testing`, and `swift-snapshot-testing`.
- Add migration/schema helpers, a sample app, CI, SPI packaging, and release automation.

## Capabilities

### New Capabilities
- `observation-live-query`: `AsyncStream` CDC observation, modern `LiveQuery`/`LiveQueryOne` property wrappers, `Perception` backport.
- `async-writes-mvcc`: `DatabaseActor`, `async` read/write/transaction, concurrent transaction API with `BEGIN CONCURRENT` and retry/backoff.
- `macro-layer`: `BoutiqueDBMacros` package, `@BoutiqueTable`, `@FTSIndex`, `@VectorIndex`, `@MaterializedView`, schema snapshot testing.
- `turso-features`: Typed DSL + runtime APIs for FTS, vector search, materialized views, encryption, multi-process WAL, extensions, and custom types.
- `cloudkit-sync`: `SyncAdapter` protocol, hardened `CloudKitSyncAdapter`, batching, conflict resolution, account change handling.
- `developer-experience`: `BoutiqueMigrationPlan`, sample app, docs, CI, SPI `.xcframework` target, release pipeline.

### Modified Capabilities
- None at the spec level; this is a greenfield v2 implementation.

## Impact

- `BoutiqueDB-Swift/Package.swift`: new dependencies and a `.macro` target.
- `Sources/BoutiqueDB/`: container and connection APIs become `async`; new `DatabaseActor`.
- `Sources/TursoObservation/`: `TursoStore` gets `AsyncStream` and `invalidate()`; property wrappers rewritten.
- `Sources/TursoCKSync/`: `SyncAdapter` seam and hardened engine.
- `Sources/BoutiqueDBMacros/`: new compiler-plugin macro target.
- `BoutiqueDB-Framework-Spec-v2.md` becomes the source of truth; stale `.cursor/plans/BoutiqueDB-Implementation-v2.md` is removed and replaced by this OpenSpec change.
