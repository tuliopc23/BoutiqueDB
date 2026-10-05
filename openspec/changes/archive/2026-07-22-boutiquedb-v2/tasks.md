> **ARCHIVED 2026-07-22** — Implementation cycle complete.

# BoutiqueDB v2 — Implementation Tasks

> Derived from `BoutiqueDB-Framework-Spec-v2.md` and the OpenSpec design/specs in this change. Each task is one session-sized unit. Order matters: finish a phase before starting dependent tasks in the next phase.

---

## 1. Foundation & Modern Observation

- [x] 1.1 Add `swift-dependencies`, `swift-perception`, `swift-testing`, and `swift-snapshot-testing` to `Package.swift`.
- [x] 1.2 Introduce `DatabaseActor` and make `BoutiqueDB.read` / `write` / `transaction` `async throws`.
- [x] 1.3 Rewrite `TursoStore` to expose `changes: AsyncStream<ChangeEvent>` and `invalidate()`.
- [x] 1.4 Implement CDC listener task (`SELECT change_id FROM turso_cdc` loop with cooperative await).
- [x] 1.5 Rewrite `LiveQuery` and `LiveQueryOne` to use `AsyncStream` + `withObservationTracking` (or `Perception` backport).
- [x] 1.6 Add `BoutiqueError` cases for async/runtime failures (`cdcMutuallyExclusiveWithMVCC`, `encryptionUnavailable`, etc.).
- [x] 1.7 Write Swift Testing unit tests for async read/write and `LiveQuery` refresh (< 500 ms).
- [x] 1.8 Write SwiftUI integration test that writes a row and asserts `@LiveQuery` updates within 1 s.

**Phase deliverable:** `swift build` + `swift test` green; modern observation works end-to-end. ✅

---

## 2. Macro Layer & Schema Generation

- [x] 2.1 Add `.macro` target `BoutiqueDBMacros` with `SwiftSyntaxMacros` and `SwiftCompilerPlugin` dependencies.
- [x] 2.2 Implement `@BoutiqueTable` peer macro (wraps `@Table`, adds `withoutRowid`/`strict`/generated column support).
- [x] 2.3 Implement `@FTSIndex` macro (generates `CREATE INDEX ... USING fts` and typed `FTSConfig`).
- [x] 2.4 Implement `@VectorIndex` macro (generates `CREATE INDEX ... USING vector` and validates metric/column type).
- [x] 2.5 Implement `@MaterializedView` macro (generates `CREATE MATERIALIZED VIEW ... AS <source>` and validates source expression).
- [x] 2.6 Add macro validation errors for unsupported tokenizers, metrics, nested views, and invalid column types.
- [x] 2.7 Add snapshot tests for all macro-generated SQL outputs.
- [x] 2.8 Wire macros into `BoutiqueDB` so `db.create(Note.self)` and `db.create(CustomerTotals.self)` work.

**Phase deliverable:** Macros compile, generate valid DDL, and snapshot tests pass. ✅

---

## 3. Turso-Exclusive Features

> **Product priority:** This phase is the core BoutiqueDB advantage — every Turso capability SQLite lacks must land as a typed Apple-app surface (DSL + LiveQuery + capability gates), not raw SQL.

- [x] 3.0 Add `TursoCapabilities` probe API and gate feature entry points with `BoutiqueError.featureUnavailable` / typed errors.
- [x] 3.1 Implement MVCC API: `beginConcurrent`, `commitConcurrent`, `writeConcurrent` with `SQLITE_BUSY` retry/backoff.
- [x] 3.2 Add runtime guard throwing `BoutiqueError.cdcMutuallyExclusiveWithMVCC`.
- [x] 3.3 Implement `Vector32` / `Vector32Sparse` value types and `QueryBindable` conformance.
- [x] 3.4 Add vector distance helpers (`vectorDistanceCos`, `vectorDistanceL2`, `vectorDistanceDot`, `vectorDistanceJaccard`) to the DSL.
- [x] 3.5 Add FTS DSL helpers (`.match`, `.score`, `.highlight`) and live search overload for `@LiveQuery`.
- [x] 3.6 Implement `createMaterializedView<T: MaterializedView>(_:)`, gated by experimental-views availability.
- [x] 3.7 Add `BoutiqueDB(url:encryption:)` for at-rest encryption (C setter or `sdk-kit` migration as needed).
- [x] 3.8 Add `BoutiqueDB(url:multiProcess:)` and `.tshm` sidecar management.
- [x] 3.9 Add typed scalar extension helpers (`UUID.v4`, `String.regexp`, `Date` time functions, `percentile`, `fuzzy`, `ipaddr`).
- [x] 3.10 Add custom type support: `QueryBindable` for `RawRepresentable` enums and structs.
- [x] 3.11 Add integration tests for each Turso feature; skip features not supported by the current C build.

**Phase deliverable:** All Turso-exclusive features accessible via typed APIs; MVCC writes safe; encryption/multi-process APIs available. ✅ (gated throws where lib lacks flags)

---

## 4. CloudKit Sync Hardening

- [x] 4.1 Define `SyncAdapter` protocol and make `TursoCKSyncEngine` conform via `CloudKitSyncAdapter`.
- [x] 4.2 Implement `SyncStatus` enum and `AsyncStream` on the adapter.
- [x] 4.3 Add batching controls: `maxBatchSize` ≤ 250 and `drainCDC` limit of 500.
- [x] 4.4 Implement `zoneNotFound` retry path (`retryZones` / `retryRecords`).
- [x] 4.5 Implement conflict policies: `.serverWins`, `.clientWins`, `.lastWriterWins(field:)`.
- [x] 4.6 Implement crash-safe account change detection and `wipeAndRebootstrap()`.
- [x] 4.7 Add multi-table sync support with `syncedTables` array and per-table `CKRecord` mapping.
- [x] 4.8 Add simulated two-device round-trip tests using `enablesCloudKit: false`.
- [x] 4.9 Add live two-simulator CloudKit test instructions and a manual QA checklist.
- [x] 4.10 Add performance benchmarks for `drainCDC`, sync round-trip, and `writeConcurrent`.

**Phase deliverable:** Production sync; conflict resolution stable; account change safe; benchmarks documented. ✅

---

## 5. Developer Experience & Release

- [x] 5.1 Implement `BoutiqueMigrationPlan` and `BoutiqueMigration` helpers (`create`, `ensureColumn`, migrator).
- [x] 5.2 Add `BoutiqueDBDependencyKey` for `swift-dependencies` integration.
- [x] 5.3 Rewrite `README.md` with quick-start, module overview, and migration guide.
- [x] 5.4 Add `docs/` (Migrations, CloudKit QA, Sync benchmarks).
- [~] 5.5 SampleApp — **cancelled** (package DX; SQLiteData-style, no required sample app).
- [x] 5.6 Add `.github/workflows/swift.yml` for `swift build` / `swift test`.
- [~] 5.7 SPI / xcframework — **deferred to refinement R2** (BD-008 residual).
- [~] 5.8 Sample macOS release pipeline — **cancelled** (no SampleApp); package GitHub release instead.
- [~] 5.9 Tag `v1.0.0` — **deferred until refinement P0 complete** (see BoutiqueDB-Refinement-Tasks.md).

**Phase deliverable:** Migrations + open DX; CI; docs. SampleApp cancelled. SPI/v1.0.0 still open (5.7, 5.9).

---

## 6. Cross-Cutting Verification

- [x] 6.1 macOS `swift build` / `swift test` green for implementation cycle.
- [x] 6.2 No `bindings/c` source changes required in this cycle.
- [x] 6.3 Full workspace clippy deferred to pre-bundle refinement / release.
- [x] 6.4 Living spec: `BoutiqueDB-Framework-Spec-v2.md`; change archived under `openspec/changes/archive/2026-07-22-boutiquedb-v2/`.
- [x] 6.5 OpenSpec change archived 2026-07-22 (v1.0.0 package tag still gated by refinement P0).

**Phase deliverable:** Cycle closed; refinement backlog owns packaging/release.

---

## Archive note

This change is **archived** (implementation cycle complete).  
Pre-bundle work: `BoutiqueDB-Refinement-Tasks.md`.  
BD issues: all closed/decided in `BoutiqueDB-Issues.md`.
