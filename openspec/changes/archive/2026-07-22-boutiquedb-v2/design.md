# BoutiqueDB v2 — Design

## Context

`BoutiqueDB-Swift` is a local-first Swift persistence package built on the Rust Turso engine via `bindings/c` (`libturso_sqlite3`). It already has:

- `TursoKit` for low-level SQLite3-compatible open/connection/statement handling.
- `StructuredQueriesTurso` as the Point-Free `swift-structured-queries` driver.
- `TursoObservation` with a `Timer`-polling `TursoStore` and `LiveQuery`/`LiveQueryOne` property wrappers.
- `TursoCKSync` with a `CKSyncEngine`-based CloudKit sync pipeline.
- `BoutiqueDB` high-level container with synchronous `read`/`write`.

The v2 framework spec modernizes every layer. This design explains the big choices and sequencing.

## Goals / Non-Goals

**Goals:**
- Make the public API fully `async`/`await` and Swift 6 concurrency-friendly (`Sendable`, `@MainActor` container, background `DatabaseActor`).
- Replace polling observation with `AsyncStream` over Turso CDC.
- Deliver macro-generated schema/index DDL while preserving `swift-structured-queries` compatibility.
- Surface Turso-only features through typed Swift DSLs.
- Harden CloudKit sync for production: batching, conflict resolution, account changes, crash-safe state.
- Ship `v1.0.0` with CI, docs, sample app, and SPI-ready packaging.

**Non-Goals:**
- Rewriting the Rust Turso engine or `bindings/c` (except small C setters if absolutely required).
- Turso Cloud sync as the default in v2 (it remains a future `SyncAdapter` implementation).
- Full compatibility with iOS < 15 (we target iOS 15+ via `swift-perception`; iOS 17+ uses native `Observation`).
- Reimplementing the SQL parser or query planner (we rely on `swift-structured-queries`).

## Decisions

### Decision 1: Keep `bindings/c` as the engine binding for v2

**Rationale:** `libturso_sqlite3` gives us a normal SQLite3 API surface that `swift-structured-queries` already consumes. CDC, MVCC, FTS, vector functions, and materialized views are all reachable through SQL. Features requiring per-open Rust `Builder` flags (encryption, custom index methods, multi-process WAL, `--experimental-views`) need either a new C setter or a future migration to `sdk-kit`.

**Alternatives considered:**
- Migrate to `sdk-kit/turso.h` now. Rejected because it would require rewriting the Swift driver and dropping the SQLite3-compatible stack mid-cycle.

### Decision 2: `DatabaseActor` owns all I/O; `BoutiqueDB` stays `@MainActor`

**Rationale:** Keeps SwiftUI/model integration simple (`BoutiqueDB` is safe to use from `@MainActor`) while running blocking SQLite work on a cooperative background actor. This matches SwiftData/SQLiteData patterns and avoids re-entrancy bugs.

**Alternatives considered:**
- Use a `DispatchQueue`. Rejected because `DatabaseActor` gives us `Sendable` checking and structured concurrency.

### Decision 3: `AsyncStream<ChangeEvent>` for observation, not `Timer`

**Rationale:** CDC already emits `change_id` rows to `turso_cdc`. A background task can `SELECT change_id FROM turso_cdc` cooperatively and yield events. This eliminates the 100 ms polling latency and reduces battery/CPU use.

**Alternatives considered:**
- `sqlite3_update_hook` / `commit_hook`. Rejected because `bindings/c` does not expose them; CDC is the available hook.

### Decision 4: Separate `SyncAdapter` protocol with `CloudKitSyncAdapter` as the default

**Rationale:** Hardcoding CloudKit blocked future Turso Cloud sync. A protocol lets us swap adapters without changing `BoutiqueDB` public API.

### Decision 5: `@BoutiqueTable` wraps/extends `@Table`; advanced options are peer macros

**Rationale:** `swift-structured-queries` owns `@Table`/`@Column`. Instead of forking it, we layer `@BoutiqueTable` (for `withoutRowid`/`strict`/generated columns) and peer macros for indexes/views. This keeps upstream compatibility.

### Decision 6: MVCC and CDC are mutually exclusive at the connection level

**Rationale:** Turso MVCC and CDC cannot run on the same connection. The sync engine needs CDC; concurrent writes need a dedicated MVCC connection. `BoutiqueDB` throws `BoutiqueError.cdcMutuallyExclusiveWithMVCC` if both are enabled on the same handle.

## Risks / Trade-offs

- [Risk] `bindings/c` lacks setters for encryption / multi-process WAL / custom index methods → [Mitigation] Implement per-feature C setters in `bindings/c` if the Rust side exposes them, or schedule a Phase 4 `sdk-kit` migration only after the macro and DSL layers are stable.
- [Risk] Materialized views require `--experimental-views`, which is not enabled by `turso_enable_experimental()` → [Mitigation] Gate `@MaterializedView` behind a runtime check and document the experimental flag requirement.
- [Risk] Macro snapshot tests become brittle → [Mitigation] Use `swift-snapshot-testing` for generated SQL only, not full macro expansion AST, and review diffs manually.
- [Risk] CloudKit `CKSyncEngine` account changes can corrupt local state → [Mitigation] Persist a hash of the account identifier and crash-safe `stateSerialization`; `wipeAndRebootstrap()` only after confirmation.
- [Risk] `AsyncStream` CDC listener may miss changes across WAL checkpoints → [Mitigation] Use a dedicated read-only CDC connection and poll only when no new `change_id` appears within a cooperative timeout (fallback, not hot loop).

## Migration Plan

1. Introduce `BoutiqueDB` v2 APIs side-by-side with v1 synchronous APIs where possible, then remove v1 in `v1.0.0`.
2. `BoutiqueMigrationPlan` migrates existing local databases: create new tables, add columns, build indexes, copy data.
3. Sync state (`ck_*` metadata, `stateSerialization`) is preserved across upgrades.
4. After release, archive this OpenSpec change and keep `BoutiqueDB-Framework-Spec-v2.md` as the living architecture doc.

## Open Questions

See `BoutiqueDB-Issues.md` for the consolidated list of blockers, risks, and decisions that must be resolved as implementation proceeds. The most critical ones are:

- Does the vendored `libturso_sqlite3` expose `PRAGMA cipher`/encryption, or do we need a new C setter?
- Can the CLI build enable `--experimental-index-method` for FTS/vector indexes by default, or do we gate index creation at runtime?
- What is the minimum CKSyncEngine `stateSerialization` migration path for existing users of `TursoCKSyncEngine`?
