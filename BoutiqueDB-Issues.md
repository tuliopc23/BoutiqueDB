# BoutiqueDB — Design & Implementation Issues

> **Status as of 2026-07-22 (post sdk-kit integration):** All BD-001–014 remain **CLOSED** or **DECIDED**.
> Resolutions below updated to match **sdk-kit** binding (supersedes early “bindings/c only” notes where relevant).
> Pre-bundle packaging work: `BoutiqueDB-Refinement-Tasks.md`.
> Official feature/async path: `BoutiqueDB-SdkKit-Integration-Spec.md`.

## How to use this file

- Closed issues stay for audit trail.
- New production blockers: add `BD-015+` under **Open Issues**.

---

## Open Issues

### Production beta hardening (implemented for v0.3.0-beta.1)

The active tracker is
`openspec/changes/boutiquedb-production-beta-hardening/tasks.md`. Issues below were confirmed
against `~/Developer/BoutiqueDB-Swift` on 2026-07-22 using FFF, ast-grep, the available
CodeGraph index, baseline builds/tests, and current SQLiteData/CKSyncEngine references.

BD-015–038 were implemented and verified in the Swift package. The only external
release gate that cannot be completed in this checkout is signed physical-device
CloudKit validation against the host application's production container; the package
documents that boundary and does not claim it as locally verified.

| ID | Pri | Area | Confirmed gap / acceptance criterion |
|---|---|---|---|
| **BD-015** | P0 | Concurrency | Public `TursoConnection`/`TursoStatement` native handles are `@unchecked Sendable`; async methods bypass locking and prepared statements escape connection isolation. Enforce one verifiable ownership model and add racing consumer tests. |
| **BD-016** | P0 | Core API | `TursoStatement.clearBindings()` is an empty successful no-op. Implement against the ABI, safely emulate it, or remove/deprecate the API with tests. |
| **BD-017** | P0 | Lifecycle | Add idempotent shutdown for listeners, connections, concurrent writers, and sync. A failed commit/rollback must not mark a potentially open transaction idle. |
| **BD-018** | P0 | Errors | Commit hooks, CDC observation, asset reads, account probes, and several cleanup paths swallow errors. Preserve underlying context in stable typed errors and observable status. |
| **BD-019** | P0 | Bootstrap | `BoutiqueDB.open` omits `openOptions`, encryption, and multi-process settings. One sendable configuration must reach every open/reopen path. |
| **BD-020** | P0 | Migrations | Migration body and tracking insert are non-atomic; duplicate/order drift is not validated; debug erase deletes live DB/WAL/SHM files. Make transactional default, validate plan, close before erase/reopen, and inject a deterministic clock. |
| **BD-021** | P1 | Schema | StructuredQueries models, macro DDL, additive columns, and `SyncedTable` repeat schema as unrelated/stringly metadata. Define one canonical typed descriptor and derive consumers from it. |
| **BD-022** | P1 | Macros | `@BoutiqueTable` does not provide the advertised StructuredQueries model integration or additive-column metadata; unknown types silently become TEXT. Complete generation and diagnostics. |
| **BD-023** | P1 | DDL | FTS/vector/materialized-view validation is incomplete and schema creation can partially succeed or silently omit unavailable objects. Validate and apply atomically, returning an explicit report when optional features are skipped. |
| **BD-024** | P0 | Observation | CDC still polls synchronously every 50 ms from a `@MainActor` task, bypasses database actor ownership, and ignores failures. Move work off-main, serialize access, expose failures, and use adaptive/event-driven behavior where available. |
| **BD-025** | P1 | LiveQuery | Concurrent subscription/manual refresh tasks can publish stale results after `setQuery`. Add ordered cancellation/generation semantics, structured load state, teardown and error tests. |
| **BD-026** | P1 | DX/DI | Documentation places `await` inside synchronous `prepareDependencies`; no safe preview/test factory exists and the concrete dependency is hard to substitute. Provide compiling bootstrap patterns and controlled factories. |
| **BD-027** | P0 | Sync durability | `drainCDC` advances the persistent cursor across configured-table rows whose primary key cannot be resolved, permanently losing changes. Fail/retry or durably dead-letter; validate schemas before start. |
| **BD-028** | P0 | Conflicts | `.clientWins` applies server data before re-enqueueing, so the retry uploads server values. Preserve the client payload while adopting server system fields; test actual outbound round-trip behavior. |
| **BD-029** | P0 | Commit/outbox | One mutable `onLocalCommit` callback can be clobbered and its errors are discarded. Support multiple subscribers and durable/observable enqueue failure semantics without rolling back a committed local transaction. |
| **BD-030** | P0 | CloudKit config | CloudKit-enabled sync falls back to a hard-coded development container. Require an explicit/injected host container and validate zone/table configuration. |
| **BD-031** | P0 | Sync schema | Record-name constraints crash with `precondition`; reserved CloudKit fields, compound/auto-increment keys, unique constraints, unsupported deletes/value types, and migration compatibility are not validated. Fail before syncing. |
| **BD-032** | P0 | Sync concurrency | CK engine mutable state is only partially locked; account probing is callback-based with `try?`; synchronization flag is a non-nested boolean; CKAsset read failures become NULL. Actor-isolate and make failures explicit. |
| **BD-033** | P1 | Sync API | `SyncAdapter` claims transport independence but exposes `CKRecord`. Introduce transport-neutral envelopes, add manual fetch/send/sync/status APIs, and keep CloudKit sharing mapping in its adapter. |
| **BD-034** | P1 | Public API | Five public products expose unsafe low-level details; Perception is unused at iOS 17/macOS 14; docs/errors still mention legacy sqlite3. Decide supported products/SPI and create API baselines. |
| **BD-035** | P1 | Verification | Add deterministic dependency, migration-from-release, failure injection, concurrent read/write/sync, Thread Sanitizer, observation fan-out, restart, partial failure, WAL/checkpoint, and performance coverage. |
| **BD-036** | P1 | Docs | Add DocC and a compiling consumer fixture; document entitlements, remote notifications, sharing acceptance/permissions, account-change policy, privacy logging, backup/WAL rules, limitations, and troubleshooting. |
| **BD-037** | P0 | Binary | Released `v0.2.0` archive links objects built for macOS 26/27 while the package declares macOS 14. Rebuild every required slice with explicit minimum targets and make warning-free verification mandatory. |
| **BD-038** | P0 | Release | Validate clean URL-binary builds on macOS 14+ and iOS 17 device/simulator architectures, checksum/tag workflow, SPI matrix, semver/API compatibility, and prerelease installation before publishing beta. |

---

## Decided Issues (accepted design)

| ID | Decision | Reality (now) |
|---|---|---|
| **BD-005** | CDC ⊥ MVCC on one handle; dual connection for concurrent writes | **Shipped** — CDC-safe concurrent fallback when needed |
| **BD-010** | CDC cursor + immediate invalidate; multi-consumer `subscribe()` | **Shipped** |
| **BD-011** | Native Observation on iOS 17+; Perception dep only | **Shipped** |
| **BD-013** | Originally: keep `bindings/c` for v2; sdk-kit post-v2 | **SUPERSEDED (2026-07-22)** — TursoKit now uses **official sdk-kit** C ABI for open/step/features/async; sqlite3-compat retained as legacy target only |
| **BD-014** | `BoutiqueDB` `@MainActor`; I/O on `DatabaseActor` | **Shipped** — exclusive depth + cooperative async when `asyncIO` |

---

## Closed Issues

### BD-001: Encryption C API not exposed by `bindings/c`
- **Status:** CLOSED (updated 2026-07-22)
- **Resolution:** Encryption is enabled via **sdk-kit** `experimental_features=encryption` + cipher/hexkey on open (`BoutiqueDB(url:encryption:)`). Vendor lib built with `--features encryption`. Still **experimental** in Turso; apps may prefer Data Protection.
- **Was:** Always throw on sqlite3 path. **Now:** Official config path works when key/cipher valid.

### BD-002: FTS/Vector index methods need experimental flags
- **Status:** CLOSED (updated 2026-07-22)
- **Resolution:** Official token `index_method` via `TursoOpenOptions` / `.tursoEnhanced`. Vendor `turso_sdk_kit` built with `--features fts`. Runtime `TursoCapabilities` still probes. S0 spike: FTS create OK with CSV + fts cargo feature.

### BD-003: Materialized views require experimental views
- **Status:** CLOSED (updated 2026-07-22)
- **Resolution:** Official token `views` via open options. S0 spike: MV create OK.

### BD-004: Multi-process WAL needs Builder flag
- **Status:** CLOSED (updated 2026-07-22)
- **Resolution:** Official token `multiprocess_wal` via `multiProcess: true`. Single-process open works; multi-process App Group validation is **app-level next step** (user Live CloudKit / extension app).

### BD-006: Minimum OS / CKSyncEngine
- **Status:** CLOSED — iOS 17 / macOS 14.

### BD-007: stateSerialization migration
- **Status:** CLOSED — account hash + wipe/rebootstrap; live CK validation deferred to user app.

### BD-008: SPI / unsafeFlags
- **Status:** CLOSED as residual for v2 API — **still open for public package** (R2.x). Now links `libturso_sdk_kit` via unsafeFlags.

### BD-009: Macro snapshot brittleness
- **Status:** CLOSED — macro tests + Package.resolved pin.

### BD-012: Two-simulator CloudKit in CI
- **Status:** CLOSED — offline simulated tests + QA checklist; live CK = user next app.

### BD-005, BD-010, BD-011, BD-014
- Implemented in Phases 1–4 + quality program.

---

## References

| Artifact | Path |
|----------|------|
| OpenSpec v2 (archived) | `openspec/changes/archive/2026-07-22-boutiquedb-v2/` |
| Refinement / packaging | `BoutiqueDB-Refinement-Tasks.md` |
| Prod readiness | `BoutiqueDB-Prod-Readiness-Tasks.md` |
| Quality program | `BoutiqueDB-Quality-Program.md` |
| sdk-kit integration | `BoutiqueDB-SdkKit-Integration-Spec.md` |
| Completion report | `BoutiqueDB-Completion-Status-Report.md` |
