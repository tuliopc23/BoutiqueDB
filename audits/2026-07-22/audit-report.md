# BoutiqueDB-Swift Audit Report

**Date:** 2026-07-22  
**Scope:** `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift` (Sources, Tests, docs, Package.swift) + open items in `BoutiqueDB-Refinement-Tasks.md`  
**Mode:** read-only, evidence-first  
**SQLiteData bar:** Point-Free SQLiteData / GRDB CloudKit maturity as quality reference  

---

## Executive summary

BoutiqueDB-Swift has a coherent layered design (`BoutiqueDB` → `DatabaseActor` / dual MVCC writer → `TursoKit` → C bind; observation via `TursoStore`; CloudKit via `TursoCKSync`) and a non-trivial offline test suite (~54 `@Test`s). It is **not** production-ready for a public SPI/bundle claim.

Highest risks: **per-connection CDC + dual MVCC writer breaks sync capture**, **encryption/multi-process always fail**, **experimental FTS/vector/views gated but likely off in vendor lib**, **SPM `unsafeFlags` packaging**, **large public-API test holes**, and **DX/docs drift vs SQLiteData**.

Open refinement P0 still open: **R1.1**, **R2.1**, **R2.5**. Other open: R0.6, R1.3–R1.6, R2.2–R2.3, R3.2–R3.4, R4.2, R4.4, R5.4.

---

## Findings

### [A-001] Dual-connection `writeConcurrent` writes are not captured by CDC

- **severity:** critical  
- **area:** concurrency | sync | turso  
- **evidence:**  
  - Turso docs: `PRAGMA capture_data_changes_conn` enables CDC **for the current connection** (`BoutiqueDB/docs/sql-reference/pragmas.mdx` via engine tree; COMPAT: “per-connection Change Data Capture”).  
  - Primary open path: `BoutiqueDB.swift:59` `connect(enableCDC: enableCDC)`.  
  - Concurrent writer: `BoutiqueDB.swift:86–90` `connect(enableCDC: false)` + `PRAGMA journal_mode = mvcc`.  
  - Sync drain: `TursoCKSyncEngine.drainCDC` reads `connection.cdcChanges` on the **CDC** handle only (`TursoCKSyncEngine.swift:156–162`).  
  - Stress test does **not** assert pending changes after concurrent write: `RefinementStressTests.swift:76` `#expect(!engine.pendingRecordZoneChanges.isEmpty || true)` (always true).  
- **recommendation:** Either (1) document and enforce mutual exclusion: `concurrentWrites` incompatible with CloudKit/CDC-backed LiveQuery cross-connection discovery; or (2) after `writeConcurrent`, synthesize outbound sync (table snapshot / primary-key delta) and still call `store.invalidate()`; or (3) research a global CDC table mode and enable capture on the writer if engine allows without MVCC conflict. Add a failing test: concurrent insert must appear in `drainCDC` or be rejected at API boundary.  
- **prod-block:** yes  

### [A-002] Encryption API is a hard fail; config type is dead surface

- **severity:** critical (if encryption is marketed); major (if documented as unavailable)  
- **area:** security | turso | api  
- **evidence:** `BoutiqueDB.swift:51–53` always `throw BoutiqueError.encryptionUnavailable` when `encryption != nil`. `EncryptionConfig` still public (`BoutiqueDB.swift:267–270`). README/CHANGELOG acknowledge unavailability. Refinement **R1.3** open.  
- **recommendation:** Until C setter exists: remove or `@available(*, unavailable)` the init parameter, or keep throwing but hide `EncryptionConfig` from public products. Do not list encryption as a shipping advantage without R1.3.  
- **prod-block:** yes (for any app requiring at-rest encryption)  

### [A-003] Multi-process WAL always unavailable

- **severity:** major  
- **area:** turso | concurrency  
- **evidence:** `BoutiqueDB.swift:48–50` always throws `multiProcessWALUnavailable`. `TursoCapabilities.multiProcessWAL = false` hardcoded (`TursoCapabilities.swift:36`). R1.4 open.  
- **recommendation:** Same pattern as encryption: fail closed in docs; don’t expose `multiProcess: Bool` as if optional success path exists.  
- **prod-block:** yes (for multi-process / app-group shared DB)  

### [A-004] FTS / vector index / materialized views depend on experimental vendor build (R1.1)

- **severity:** major  
- **area:** turso | api  
- **evidence:** Gates throw `featureUnavailable` (`BoutiqueDB.swift:213–236`, `240–250`). Probes use `CREATE INDEX … USING fts/vector` and `CREATE MATERIALIZED VIEW` (`TursoCapabilities.swift:51–105`). Refinement **R1.1** open: rebuild vendor lib with experimental flags. Macros still generate DDL (`BoutiqueDBMacrosTests`). Tests soft-skip: `TursoFeaturesTests.ftsIndexCreateGatedByCapability` only creates if `capabilities.ftsIndex`.  
- **recommendation:** Ship a capability matrix in README (probed at runtime). Rebuild lib (R1.1) or demote macros/docs from “supported” to “experimental / requires custom build”. Add CI that fails if advertised features probe false on the released binary.  
- **prod-block:** yes (for FTS/vector-index/MV product claims)  

### [A-005] SPM packaging requires `unsafeFlags` linker path

- **severity:** major  
- **area:** dx  
- **evidence:** `Package.swift:8–8`, `42–46` always `-L…/Vendor/turso/lib -lturso_sqlite3`. Comment references SPI binary target; `useUnsafeTursoLink` is unused (always unsafe path). R2.1 partial; R2.3 blocked. CI `.github/workflows/swift.yml` does not build engine binary.  
- **recommendation:** Complete R2.1 binary/`systemLibrary` target; remove unsafe flags; SPI dump-package clean.  
- **prod-block:** yes (for public package consumers / SPI)  

### [A-006] Public `connection` bypasses `DatabaseActor` serialization

- **severity:** major  
- **area:** concurrency | api  
- **evidence:** `BoutiqueDB.swift:17` `public let connection: TursoConnection`. Architecture (`docs/Architecture.md:28`) requires all I/O via `DatabaseActor`. Tests use raw `db.connection.execute` (`BoutiqueDBTests.swift:151–154`). `BoutiqueDBSyncEngine` / `performLocalWrite` also touch the raw connection outside actor wrappers (`BoutiqueDBSyncEngine.swift:50–61`, `TursoCKSyncEngine.performLocalWrite`).  
- **recommendation:** Make connection internal or `package`; expose only actor-mediated APIs. Sync engine should accept a serial executor / actor-bound handle. Document if escape hatch is intentional with “unsafe” naming.  
- **prod-block:** yes (data races / SQLITE_MISUSE under concurrent app use)  

### [A-007] `BoutiqueDBConnection.fetchOne` swallows errors

- **severity:** major  
- **area:** api  
- **evidence:** `BoutiqueDBConnection.swift:45–50` `try? T.find(...)` — SQL/decode failures become `nil`, indistinguishable from missing row. Higher-level `BoutiqueDB.fetchOne` uses the same path (`BoutiqueDB.swift:191–196`).  
- **recommendation:** Propagate errors; use `fetchOne` that returns optional only on empty result. Align with StructuredQueries / SQLiteData behavior.  
- **prod-block:** yes (silent data-loss / wrong nil)  

### [A-008] Migration bodies are not transactional with bookkeeping (partial apply)

- **severity:** major  
- **area:** schema  
- **evidence:** `BoutiqueMigrator.swift:117–136` comment admits DDL may auto-commit; id inserted only on full success. Failed mid-migration leaves DDL applied but id **not** recorded — retry re-runs body (may fail on “already exists” unless IF NOT EXISTS).  
- **recommendation:** Require migration helpers to be idempotent; document contract loudly; optionally wrap DML in savepoints; provide repair tooling. Add tests for multi-statement failure mid-body.  
- **prod-block:** yes (production schema evolution)  

### [A-009] SchemaSync does not ensure additive columns (R0.6)

- **severity:** major  
- **area:** schema  
- **evidence:** `SchemaSync.swift:16–17` explicitly “Does **not** add individual columns”. Refinement **R0.6** open. Migrations.md claims helpers include `ensureColumn` but schema sync only IF NOT EXISTS create.  
- **recommendation:** Implement column ensure from `BoutiqueSchema` metadata or document that schemaSync never evolves columns; force migrations only.  
- **prod-block:** no (if documented); yes if apps rely on additiveOnly for column adds  

### [A-010] Live CloudKit auth / account status not wired (`needsAuthentication`)

- **severity:** major  
- **area:** sync  
- **evidence:** `SyncStatus.needsAuthentication` exists (`SyncAdapter.swift:32`) but engine never publishes it. `detectAccountIdentityChangeIfNeeded` is a no-op vs live status (`TursoCKSyncEngine.swift:132–138`). R3.2 open. All automated tests use `enablesCloudKit: false`.  
- **recommendation:** Subscribe to `CKContainer.accountStatus` / account change notifications; map `.noAccount`/`.couldNotDetermine` → `.needsAuthentication`. Integration test with mock container if possible.  
- **prod-block:** yes (real multi-device apps)  

### [A-011] `isSynchronizing` flag is unlocked public mutable state

- **severity:** major  
- **area:** concurrency | sync  
- **evidence:** `TursoConnection.swift:23` `public var isSynchronizing: Bool = false` without lock. `drainCDC` reads it (`TursoCKSyncEngine.swift:157`); inbound apply sets it (`263–264`). Concurrent drain during apply can race.  
- **recommendation:** Store under connection lock or sync-engine-private atomic; do not expose as public API.  
- **prod-block:** yes (under concurrent drain + inbound)  

### [A-012] Low-level `beginConcurrent` / `commitConcurrent` lack session safety

- **severity:** major  
- **area:** concurrency | api  
- **evidence:** `DatabaseActor.swift:63–73` bare BEGIN/COMMIT/ROLLBACK. No nesting guard vs `write`/`writeConcurrent` which open their own transactions (`TursoConnection.swift:101–142`). Same actor serializes calls but interleaved `write` after `beginConcurrent` nests statements.  
- **recommendation:** Track transaction state on actor; forbid nested begin; or remove low-level API and keep only `writeConcurrent`.  
- **prod-block:** no (if undocumented power-user API); yes if promoted as primary concurrent API  

### [A-013] `@MainActor` + `Sendable` on types holding mutable observer maps

- **severity:** minor → major if misused  
- **area:** api | concurrency  
- **evidence:** `BoutiqueDB: @MainActor … Sendable` (`BoutiqueDB.swift:14–15`); `BoutiqueDBSyncEngine` same (`BoutiqueDBSyncEngine.swift:10–11`); `TursoStore` `@MainActor` only (`TursoInvalidation.swift:18–20`). Mutable `concurrentActor` is MainActor-isolated (OK). `TursoStore.continuations` mutated MainActor; `deinit` finishes continuations nonisolated (`43–48`) — potential teardown race. LiveQuery uses `nonisolated(unsafe)` for task handles (`LiveQuery.swift:21`).  
- **recommendation:** Audit deinit/teardown under Swift 6 complete checking; prefer `AsyncStream` continuation cleanup on MainActor via `Task { @MainActor in }`. Document that LiveQuery/BoutiqueDB must be MainActor-owned (SwiftUI models).  
- **prod-block:** no (current pattern mostly sound)  

### [A-014] `LiveQueryOne` missing `setQuery` parity

- **severity:** minor  
- **area:** api | dx  
- **evidence:** `LiveQuery.setQuery` (`LiveQuery.swift:39–42`); `LiveQueryOne` stores `let query` (`LiveQueryOne.swift:25`) — cannot change.  
- **recommendation:** Mirror `setQuery` / mutable query factory on `LiveQueryOne`.  
- **prod-block:** no  

### [A-015] Observation is poll-based CDC (50ms), not push

- **severity:** minor  
- **area:** turso | dx  
- **evidence:** `TursoStore.startListening` sleeps `idlePollInterval` default 50ms (`TursoInvalidation.swift:36, 79–92`). TursoFeatures doc still notes poll vs true CDC stream. Local writes invalidate immediately (`BoutiqueDB.swift:130`).  
- **recommendation:** Accept for v1; optimize with longer idle backoff; investigate engine notifications if available.  
- **prod-block:** no  

### [A-016] `Vector32Sparse` cannot round-trip parse

- **severity:** major  
- **area:** api | turso  
- **evidence:** `Vector32.swift:68–70` `init?(rawValue:)` always `entries = [:]` (no parse). Dense `Vector32` parses correctly (`31–44`).  
- **recommendation:** Implement sparse JSON parse matching `jsonLiteral` format or mark unavailable.  
- **prod-block:** no (unless sparse vectors shipped)  

### [A-017] `StringQueryBindable` crashes on invalid DB data

- **severity:** major  
- **area:** api  
- **evidence:** `QueryBindable+Custom.swift:26–28` `preconditionFailure` in `init(queryOutput:)`.  
- **recommendation:** Throw decoding error (or failable path) so corrupt/legacy rows don’t abort the process.  
- **prod-block:** yes (production resilience)  

### [A-018] `capabilities.cdc` always true without probe

- **severity:** minor  
- **area:** turso  
- **evidence:** `TursoCapabilities.swift:34` `caps.cdc = true` unconditional.  
- **recommendation:** Probe `PRAGMA capture_data_changes_conn` / `turso_cdc` existence.  
- **prod-block:** no  

### [A-019] Conflict handler can write empty table/rowPK meta

- **severity:** minor  
- **area:** sync  
- **evidence:** `TursoCKSyncEngine.swift:329–335` uses `?? ""` for table/rowPK when parse fails in non-test `handleServerRecordChanged`. Test path (`resolveConflictForTesting`) is safer.  
- **recommendation:** Guard and skip / log instead of upserting empty keys.  
- **prod-block:** no  

### [A-020] Dual application-support directory conventions

- **severity:** minor  
- **area:** dx  
- **evidence:** `BoutiqueDB.applicationSupportURL` → `…/BoutiqueDB/` (`BoutiqueDB.swift:260`). `TursoDatabase.applicationSupportURL` → `…/TursoCloudKit/` (`TursoDatabase.swift:56`).  
- **recommendation:** Single canonical helper; deprecate the other.  
- **prod-block:** no  

### [A-021] README / App-Template DX defects

- **severity:** minor  
- **area:** dx  
- **evidence:**  
  - README incomplete snippet: `let rows = try await db.fetchAll /* or raw query */` (`README.md:60`).  
  - App template configures `prepareDependencies` inside `.task` after UI appears (`docs/App-Template.md:46–50`) with `try!` — dependency unavailable for early view init; crashes on open failure.  
  - No DocC catalog (R4.2 open).  
  - CHANGELOG 0.1.0 same day as Unreleased bulk (versioning unclear).  
- **recommendation:** Fix snippets; open DB before `WindowGroup` content that needs `@Dependency`; replace `try!` with error UI; add DocC.  
- **prod-block:** no  

### [A-022] CI matrix incomplete (R2.2, R5.4)

- **severity:** major  
- **area:** testing | dx  
- **evidence:** Only `macos-15` `swift test` (`.github/workflows/swift.yml`). No iOS simulator (R5.4), no vendor lib rebuild, no CloudKit live tests (R3.4).  
- **recommendation:** Add iOS sim job; cache prebuilt xcframework; optional nightly CK.  
- **prod-block:** yes (for multiplatform claim iOS 17+)  

### [A-023] `BoutiqueDBSyncEngine` / sync path not integrated with `BoutiqueDB.write`

- **severity:** major  
- **area:** sync | api  
- **evidence:** High-level write invalidates store but never auto-`drainCDC`. Sync façade is separate (`BoutiqueDBSyncEngine`). Apps must remember drain after every write path (including concurrent). Architecture says “drain after local commits” but no API couples them.  
- **recommendation:** Optional `syncEngine` attached to `BoutiqueDB` that drains after write; or `write` option `notifySync: true`. Test end-to-end.  
- **prod-block:** yes (easy to ship silent non-syncing apps)  

### [A-024] No table-scoped invalidation (refresh amplification)

- **severity:** minor  
- **area:** api | dx  
- **evidence:** Any write → `store.invalidate()` → all LiveQueries reload full selects (`BoutiqueDB.swift:130`, `LiveQuery.swift:69–71`).  
- **recommendation:** Pass table names from CDC / write API; filter subscribers (SQLiteData-style observation granularity is still coarser than SQL triggers, but better than global).  
- **prod-block:** no  

### [A-025] Weak / no-op assertions in stress coverage

- **severity:** major  
- **area:** testing  
- **evidence:** `RefinementStressTests.swift:76` `|| true`. Concurrent write + drain not proven.  
- **recommendation:** Delete tautology; assert real pending record names for CDC-path writes; separate expected-fail test for concurrent-path until A-001 fixed.  
- **prod-block:** no (test quality) but hides A-001  

---

## Testing inventory

### What is tested (by suite)

| Suite | Coverage |
|---|---|
| `BoutiqueDBTests` | async R/W, LiveQuery/LiveQueryOne refresh, dual LQ, forceRefresh, serialized writes, CDC⊥MVCC convenience init, error case enum smoke |
| `LiveQueryIntegrationTests` | `@Observable` model + listening open, 1s refresh |
| `MigrationTests` | open+migrate idempotent, ensureColumn, only-new migrations, failed not recorded, additive schemaSync table create, `create(Schema)` |
| `SchemaDDLTests` | descriptor SQL strings, table create via DDL, capabilities probe non-throw, manual schema create |
| `TursoFeaturesTests` | Vector32 literal, vector SQL (if caps), FTS fragment, encryption/MP throws, dual-connection writeConcurrent, StringQueryBindable, FTS create gated |
| `RefinementStressTests` | dual LQ + write + writeConcurrent + drain (weak assert), `setQuery` |
| `TursoKitTests` | open CRUD+CDC, StructuredQueries driver CRUD |
| `TursoCKSyncTests` | metadata, outbound drain, inbound+echo, system fields, wipe, observation, simulated 2-device, multi-table, batching, LWW/serverWins, account hash, adapter status stream, preserve+reenqueue |
| `BoutiqueDBMacrosTests` | BoutiqueTable/FTS/Vector/MV expansions + bad tokenizer diagnostic |

### Public APIs lacking adequate tests (or untested)

| API / surface | Gap |
|---|---|
| `BoutiqueDB.beginConcurrent` / `commitConcurrent` / `rollbackConcurrent` | **no tests** |
| `BoutiqueDB.transaction` | alias of write only; no distinct test |
| `BoutiqueDB.createVectorIndex` / `createMaterializedView` | only SQL string builders; no end-to-end when caps true |
| `BoutiqueDB.syncSchema` failure paths / FTS skip branch | partial (table create only) |
| `BoutiqueDB.dropTableIfExists` | **no tests** |
| `BoutiqueDB.migrate` / `hasCompletedMigrations` | migrate covered indirectly; `hasCompletedMigrations` **no tests** |
| `eraseDatabaseOnSchemaChange` / `schemaErasedForDebug` reopen path | code in `BoutiqueDB+Open` **no tests** |
| `BoutiqueDBSyncEngine` (all methods) | **no dedicated tests** (engine tested via TursoCKSync only) |
| `DependencyValues.boutiqueDB` live/test fatalError | **no tests** |
| `LiveQuery.loadError` / `isLoading` edge cases | minimal |
| `LiveQueryOne.setQuery` | N/A missing API |
| `TursoQueryBox` | **no tests** |
| `TursoStore.stopListening` / multi-subscriber buffering | thin |
| `TursoConnection.writeConcurrent` busy retry paths | only success path via BoutiqueDB |
| `TursoConnection.withPreparedStatement` / `prepare` | **no direct tests** |
| `cdcDecodedJSON` edge cases | smoke only in TursoKit |
| `CloudKitSyncAdapter.applyRemoteChanges` / `stop` | start/drain only |
| `ConflictPolicy.clientWins` | **no dedicated test** |
| `RecordMapper` CKAsset / date branches | **no tests** |
| `Vector32Sparse` | broken + untested |
| `TursoSQL.uuid4` / `percentile` / `.regexp` | **no tests** |
| vectorDistanceL2/Dot/Jaccard DSL | **no tests** |
| `EncryptionConfig` success path | impossible until R1.3 |
| Macro stacking `@BoutiqueTable`+`@FTSIndex` on same type | macro unit has separate attrs; integration create **no test** |
| iOS simulator / device | **none** |
| Live CloudKit | **none** (manual checklist only) |

---

## Area deep-dives

### 1. API / Sendable / MainActor

- Container is correctly `@MainActor` with I/O on `DatabaseActor` (BD-014).  
- Escapes: public `connection`, sync engine raw use, `@unchecked Sendable` on TursoKit types (justified by locks, but easy to misuse).  
- `BoutiqueDBConnection` is not `Sendable` yet used inside `@Sendable` closures — safe only because constructed inside actor.  
- Property wrappers require explicit `db` injection (unlike SQLiteData `@FetchAll` + dependencies). No `projectedValue`.  
- FatalError dependency keys match refinement R0.2 intent.

### 2. Concurrency dual connection

- Design is intentional BD-005: CDC ⊥ MVCC per handle.  
- Lazy second connection after schema exists is good.  
- **Correctness hole A-001** makes dual connection unsafe with sync.  
- `write` + `writeConcurrent` both invalidate store (UI OK).  
- No test that two concurrent MVCC writers conflict-retry under load.

### 3. Turso feature completeness vs gates

| Feature | API present | Runtime | Notes |
|---|---|---|---|
| CDC | yes | yes (primary) | per-connection |
| MVCC concurrent | yes | dual-conn | A-001 |
| Vector functions | yes | probed | soft-skip tests |
| Vector index | yes + macro | gated R1.1 | |
| FTS index + DSL | yes + macro | gated R1.1 | |
| Materialized views | yes + macro | gated R1.1 | |
| Encryption | config only | always throw | R1.3 |
| Multi-process WAL | flag only | always throw | R1.4 |
| uuid/percentile/regexp DSL | yes | untested | R1.6 |

### 4. Sync gaps

- Strong offline unit coverage for drain, echo, conflicts (partial), batching, account hash wipe.  
- Missing: live CK, auth status, auto-drain integration with BoutiqueDB, clientWins test, concurrent-write capture, zone delete full path via real events, large CKAsset.  
- `performLocalWrite` on engine bypasses BoutiqueDB observation unless store also advanced.

### 5. Schema / migration gaps

- Solid append-only plan + failed-not-recorded.  
- Partial apply / non-atomic DDL (A-008).  
- No column ensure in schemaSync (R0.6).  
- No destructive-change detection beyond unknown applied IDs.  
- Order mismatch of common prefix mentioned in comment but only unknown-id erase implemented (`BoutiqueMigrator.swift:145–155`).

### 6. DX / docs drift

- Architecture.md is accurate and useful.  
- README incomplete fetch example; TursoFeatures.md aspirational vs shipped.  
- Migrations.md quality is good (SQLiteData-aligned rules).  
- No DocC; no pre-async migration guide (R4.4).  
- Packaging story still “local vendor .a”.

### 7. SQLiteData-quality bar comparison

| Dimension | SQLiteData / GRDB bar | BoutiqueDB-Swift today |
|---|---|---|
| Query DX | Mature `@FetchAll`/`@FetchOne`, dependencies | LiveQuery needs explicit db; no View helpers |
| CloudKit | Production-hardened SyncEngine integration | Offline-simulated only; auth/status incomplete |
| Migrations | Proven GRDB migrator culture | Good shape; atomicity/partial apply weaker |
| Concurrency | Documented queue/pool model | Actor model good; dual-conn CDC hole |
| Packaging | Clean SPM | unsafeFlags + static .a |
| Docs | DocC + extensive guides | Architecture + checklists; no DocC |
| Testing | Broad integration + community battle-testing | ~54 tests; gaps listed above; weak stress assert |
| Encryption | App responsibility / SQLCipher ecosystem | API pretends Turso path, always throws |

**Verdict:** Directionally competitive as a Turso-native SQLiteData alternative, but **below** SQLiteData production bar on packaging, sync hardening, observation ergonomics, and test/CI breadth.

---

## Refinement task crosswalk (open)

| ID | Status | Audit mapping |
|---|---|---|
| R0.6 | open | A-009 |
| R1.1 | open P0 | A-004 |
| R1.3 | open | A-002 |
| R1.4 | open | A-003 |
| R1.5 | open | A-018 |
| R1.6 | open | untested DSL |
| R2.1 | partial P0 | A-005 |
| R2.2 | partial | A-022 |
| R2.3 | open | A-005 |
| R2.5 | open | packaging gate |
| R3.2 | open | A-010 |
| R3.3–R3.4 | open | benchmarks / live CK |
| R4.2 | open | A-021 |
| R4.4 | open | docs |
| R5.4 | open | A-022 |

---

## Recommended fix order (prod readiness)

1. **A-001 / A-023** — CDC×concurrentWrites×sync contract + tests (delete `|| true`).  
2. **A-006 / A-007** — seal connection surface; fix fetchOne error handling.  
3. **A-002 / A-003 / A-004 / A-005** — honest feature matrix + R1.1/R2.1 packaging.  
4. **A-008 / A-009** — migration atomicity docs/tests; schemaSync columns decision.  
5. **A-010 / A-011** — CloudKit auth + `isSynchronizing` safety.  
6. **A-017 / A-016** — decoding crash / sparse vector.  
7. **A-022** — iOS CI + broaden public API tests list above.  
8. DX polish (A-014, A-021, DocC) after correctness.

---

## Untested public API checklist (copy for Phase B)

```
BoutiqueDB.beginConcurrent / commitConcurrent / rollbackConcurrent
BoutiqueDB.transaction (distinct semantics)
BoutiqueDB.createVectorIndex / createMaterializedView (live)
BoutiqueDB.dropTableIfExists
BoutiqueMigrator.hasCompletedMigrations
eraseDatabaseOnSchemaChange → schemaErasedForDebug reopen
BoutiqueDBSyncEngine.*
DependencyValues.boutiqueDB configuration failures
TursoQueryBox.*
TursoStore.stopListening multi-subscriber
ConflictPolicy.clientWins
CloudKitSyncAdapter.stop / applyRemoteChanges error paths
Vector32Sparse
TursoSQL.uuid4 / percentile / QueryExpression.regexp
vectorDistanceL2 / Dot / Jaccard
RecordMapper CKAsset/date
iOS simulator suite
Live CloudKit (optional / manual gate)
writeConcurrent → drainCDC pending records (must fail or pass post A-001 fix)
```

---

*End of audit-report.md*
