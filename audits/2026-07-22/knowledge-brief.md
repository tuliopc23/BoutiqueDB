# Knowledge brief (consolidated) — 2026-07-22

---

# K-Turso — Turso-only features vs BoutiqueDB Swift surface

**Agent:** K-Turso  
**Date:** 2026-07-22  
**Mode:** read-only knowledge extraction  
**Audience:** BoutiqueDB quality program / Apple-framework design

This note separates **facts** (what the engines, manuals, COMPAT matrix, and Swift sources state today) from **recommendations** (how a best-in-class Apple app should use each feature without raw SQL).

---

## Sources (absolute paths)

| Area | Path |
|------|------|
| Manuals | `/Users/tuliopinheirocunha/Developer/BoutiqueDB/cli/manuals/` (`cdc.md`, `vector.md`, `encryption.md`, `materialized-views.md`, `index.md`, `custom-types.md`) |
| Compatibility | `/Users/tuliopinheirocunha/Developer/BoutiqueDB/COMPAT.md` |
| CLI / journal / experimental flags | `/Users/tuliopinheirocunha/Developer/BoutiqueDB/docs/manual.md` |
| C binding experimental toggle | `/Users/tuliopinheirocunha/Developer/BoutiqueDB/bindings/c/src/lib.rs` |
| Rust Builder flags | `/Users/tuliopinheirocunha/Developer/BoutiqueDB/bindings/rust/src/lib.rs` |
| Swift feature map | `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/BoutiqueDB-TursoFeatures.md` |
| TursoKit | `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/Sources/TursoKit/` |
| StructuredQueriesTurso | `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/Sources/StructuredQueriesTurso/` |
| BoutiqueDB facade / MVCC dual conn | `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/Sources/BoutiqueDB/BoutiqueDB.swift` |
| Capability probes | `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/Sources/BoutiqueDB/Schema/TursoCapabilities.swift` |
| Schema DDL / macros | `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/Sources/BoutiqueDB/Schema/BoutiqueSchema.swift`, `Macros.swift` |
| Live observation | `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/Sources/TursoObservation/TursoInvalidation.swift`, `LiveQuery.swift` |

---

## 1. Turso-only features vs SQLite (facts)

| Feature | Turso | Stock SQLite | Notes (facts) |
|---------|-------|--------------|---------------|
| **CDC** | Yes — `PRAGMA capture_data_changes_conn` → table `turso_cdc` (modes: off/id/before/after/full) | No equivalent | Turso-specific PRAGMA (`COMPAT.md` § Turso-specific PRAGMAs). Manual: `cli/manuals/cdc.md`. |
| **MVCC / `BEGIN CONCURRENT`** | Yes — `PRAGMA journal_mode = mvcc` + concurrent txns | No | Snapshot isolation; write-write conflict → `SQLITE_BUSY`. Docs mark **not production ready**. `COMPAT.md` journaling table lists `wal` only (not `mvcc` row); MVCC described elsewhere (`docs/manual.md`, transaction sections). |
| **FTS (Tantivy)** | Yes — `CREATE INDEX … USING fts` + `fts_match` / `fts_score` / `fts_highlight` | Different: FTS3/4/5 virtual tables | Turso **does not** implement SQLite FTS3/4/5 (`COMPAT.md` § Full-Text Search). |
| **Vector functions** | Yes — `vector`/`vector32`/`vector64`, distances, extract/concat/slice | No built-in | Compatible with libSQL-style vector API (`COMPAT.md` § Vector). |
| **Vector / sparse indexes** | Intended via `CREATE INDEX … USING vector` (experimental index methods) | No | **Tension:** `cli/manuals/vector.md` states **vector indexes are not yet supported** (brute-force scan only). Swift stack still models `USING vector` DDL + capability probe. |
| **Materialized views (IVM)** | Yes — `CREATE MATERIALIZED VIEW` with incremental maintenance | No live IVM (manual refresh only if any) | Requires `--experimental-views`. Limits: not all SQL functions; no nested views (`cli/manuals/materialized-views.md`). |
| **Encryption at rest** | Yes — `PRAGMA cipher` / `hexkey`; ciphers AES-GCM + AEGIS family | No (SQLCipher is third-party) | Requires `--experimental-encryption`. Not production ready (`docs/manual.md`). |
| **Custom types / domains** | Yes — `CREATE TYPE` / STRICT type system | No | Needs experimental custom types path; STRICT tables always enabled in custom-types manual narrative. |
| **Generated columns** | Partial — VIRTUAL only | Full GENERATED ALWAYS AS | Flag: `--experimental-generated-columns` / `turso_enable_experimental()`. |
| **WITHOUT ROWID** | Partial — effectively insert-only path | Full | Flag: `--experimental-without-rowid`. UPDATE/DELETE/UPSERT/secondary indexes/FK/**CDC**/MV rejected (`COMPAT.md`). |
| **STRICT tables** | Yes (normal) | Yes (SQLite 3.37+) | Not Turso-exclusive. |
| **Multi-process WAL** | Experimental `.tshm` | Different multi-process WAL model | `--experimental-multiprocess-wal`. |
| **Turso PRAGMAs** | `require_where`, `list_types`, `mvcc_checkpoint_threshold`, `data_sync_retry`, … | Absent | `COMPAT.md` Turso-specific PRAGMAs table. |
| **UUID / regexp / time / percentile / CSV** | In-tree extensions | Optional loadable extensions | Not all exclusive, but **bundled** in Turso. |

### Experimental flag matrix (facts)

| Flag / API | Enables | CLI | Rust `Builder` | C `libturso_sqlite3` |
|------------|---------|-----|----------------|----------------------|
| `--experimental-encryption` / `experimental_encryption` | Cipher + hexkey / URI open | Yes | Yes | **Not** in `turso_enable_experimental()` |
| `--experimental-views` / `experimental_materialized_views` | Materialized views | Yes | Yes | **Not** in blanket C toggle |
| `--experimental-index-method` / `experimental_index_method` | Custom index methods (FTS/vector methods) | Yes (`docs/manual.md`) | Yes | **Not** in blanket C toggle |
| `--experimental-multiprocess-wal` | Multi-process WAL | Yes | Yes | **Not** |
| `--experimental-generated-columns` | Generated columns | Yes | Yes | **Yes** via `turso_enable_experimental()` |
| `--experimental-without-rowid` | WITHOUT ROWID | Yes | Yes | **Yes** via blanket toggle |
| vacuum experimental | VACUUM | — | Yes | **Yes** via blanket toggle |
| `experimental_custom_types` | CREATE TYPE / domains | — | Yes | **Not** in blanket toggle |
| `experimental_attach` | ATTACH | — | Yes | **Not** |
| MVCC | `PRAGMA journal_mode=mvcc` | Runtime pragma | N/A (runtime) | SQL path only |
| CDC | `PRAGMA capture_data_changes_conn` | Runtime pragma | N/A | SQL path only |

**Fact:** `turso_enable_experimental()` only sets `with_generated_columns(true)`, `with_vacuum(true)`, `with_without_rowid(true)` for subsequent opens (`bindings/c/src/lib.rs`).

**Fact:** BoutiqueDB-Swift opens via SQLite C API (`sqlite3_open_v2` in `TursoDatabase.swift`) — no Rust `Builder` feature flags at open time.

---

## 2. How each maps to current Swift API (or gap)

| Feature | Swift surface today | Gap / status |
|---------|---------------------|--------------|
| **CDC** | `TursoConnection.enableCaptureDataChanges(mode:)`; `cdcChanges` / `cdcDecodedJSON`; `TursoDatabase.connect(enableCDC:)`; `BoutiqueDB(enableCDC:)`; `TursoStore` CDC poll → generation; `@LiveQuery` / `@LiveQueryOne` | **Done for observation.** Future: true CDC `AsyncStream` instead of 50–100 ms poll (`BoutiqueDB-TursoFeatures.md`). |
| **MVCC** | Dual-connection design in `BoutiqueDB`: primary CDC connection + lazy second `PRAGMA journal_mode=mvcc` writer; `writeConcurrent`, `beginConcurrent`/`commitConcurrent`/`rollbackConcurrent`; `DatabaseActor` busy-retry; low-level `TursoConnection.writeConcurrent` | **API present.** Same-handle CDC+MVCC rejected (`BoutiqueError.cdcMutuallyExclusiveWithMVCC`). |
| **FTS query** | `QueryExpression.match` / `.score` / `.highlight` → `fts_*` (`TursoFunctions.swift`) | **Query DSL present.** |
| **FTS index DDL** | `FTSIndexDescriptor`, `@FTSIndex` macro, `BoutiqueDB.createFTSIndex` / `create(_:)`, capability gate | **Gated** by `capabilities.ftsIndex` (BD-002). |
| **Vector values** | `Vector32`, `Vector32Sparse` (`Vector32.swift`); `QueryBindable` as JSON text | **Types present.** Sparse parse incomplete (`init?(rawValue:)` empties). |
| **Vector distance** | `vectorDistanceCos/L2/Dot/Jaccard` (`TursoFunctions.swift`) | **DSL present.** COMPAT lists cos/l2 (+ vector_extract etc.); dot/jaccard in Swift may outpace documented COMPAT rows. |
| **Vector index DDL** | `VectorIndexDescriptor`, `@VectorIndex`, `createVectorIndex` | **Gated** by `capabilities.vectorIndex`. Manual says indexes unsupported — expect probe often **false**. |
| **Materialized views** | `MaterializedViewDescriptor`, `@MaterializedView(as:)`, `createMaterializedView` | **Gated** by `capabilities.materializedViews` (BD-003). Source is **raw SQL string** in macro — not typed SQ expression yet. |
| **Encryption** | `EncryptionConfig.aegis256/aes256gcm` on `BoutiqueDB` init | **Hard fail:** any non-nil encryption throws `encryptionUnavailable` (BD-001). No PRAGMA/URI wiring. |
| **Multi-process WAL** | `multiProcess:` init param | **Hard fail:** `multiProcessWALUnavailable` (BD-004). |
| **Generated / WITHOUT ROWID / STRICT** | `@BoutiqueTable(withoutRowid:strict:)`, `BoutiqueTableDescriptor` | DDL helpers exist; engine flags may still reject WITHOUT ROWID / generated unless C experimental toggle / rebuild. |
| **Custom types** | Guidance + `StringQueryBindable` only | **No** CREATE TYPE API. Map domains via Swift enums + `QueryBindable`. |
| **CRUD / SQL DSL** | `StructuredQueriesTurso` `fetchAll`/`execute` on `TursoConnection`; `BoutiqueDB.read`/`write` via `DatabaseActor` | Core path is type-safe SQL, not raw strings. |
| **CloudKit sync + CDC** | `BoutiqueDBSyncEngine.drainCDC` | Sync layer consumes CDC independently of LiveQuery cursor (`TursoStore` comment). |

---

## 3. Capability gates / experimental flag requirements

### Runtime probes (`TursoCapabilities.probe`)

**File:** `…/BoutiqueDB/Schema/TursoCapabilities.swift`

| Field | How probed (fact) | Assumed / hard-coded |
|-------|-------------------|----------------------|
| `cdc` | — | Always `true` |
| `generatedColumns` | — | Always `true` |
| `multiProcessWAL` | — | Always `false` (BD-004) |
| `vectorFunctions` | `SELECT vector32('[1.0,0.0]') IS NOT NULL` | — |
| `ftsIndex` | TEMP table + `CREATE INDEX … USING fts` in SAVEPOINT | — |
| `vectorIndex` | TEMP + `CREATE INDEX … USING vector` | — |
| `materializedViews` | TEMP + `CREATE MATERIALIZED VIEW …` | — |
| `mvcc` | SAVEPOINT + `BEGIN CONCURRENT` + ROLLBACK | **May false-negative** if connection is not already in `journal_mode=mvcc` |
| `encryption` | `PRAGMA hexkey` succeeds | Weak signal: PRAGMA existence ≠ encryption feature enabled for create/open |

### Init-time hard gates (`BoutiqueDB.init`)

| Request | Behavior |
|---------|----------|
| `encryption: .some` | Always throws `BoutiqueError.encryptionUnavailable` |
| `multiProcess: true` | Always throws `BoutiqueError.multiProcessWALUnavailable` |
| `enableCDC && enableMVCC` (legacy convenience) | Throws `cdcMutuallyExclusiveWithMVCC` |
| `concurrentWrites: true` + `enableCDC: true` | Allowed: second connection for MVCC writer (BD-005) |

### DDL gates (`createFTSIndex` / `createVectorIndex` / `createMaterializedView` / `requireCapability`)

Missing capability → `BoutiqueError.featureUnavailable` with BD-00x message citing rebuild flags (`--experimental-index-method`, `--experimental-views`).

### Additive schema sync

`SchemaSync.syncSchema` treats FTS/vector/MV unavailability as **non-fatal**: still applies plain CREATE TABLE statements (`Migration/SchemaSync.swift`).

---

## 4. Best-in-class Apple app usage (without raw SQL)

*Recommendations — preferred product shapes.*

### CDC + live UI

1. Open `BoutiqueDB(url:, enableCDC: true)` (default).
2. Model tables with `@Table` / `@BoutiqueTable` + StructuredQueries.
3. Bind UI with `@LiveQuery` / `@LiveQueryOne` (observe `store.subscribe()` generation).
4. Prefer `db.write { … }` for mutations so actor path + `store.invalidate()` keep UI snappy even before CDC poll.
5. Do **not** enable MVCC on the same connection as CDC; use dual-handle concurrent writer if needed.

### MVCC concurrent writes

1. Init with `concurrentWrites: true` (and leave CDC on primary).
2. Route high-contention writers through `await db.writeConcurrent { … }` (built-in SQLITE_BUSY backoff).
3. Keep exclusive schema changes on primary `write` / IMMEDIATE path.
4. Treat MVCC as **experimental**; ship retry UX and telemetry on busy rates.

### FTS

1. Declare `@FTSIndex("title", "body", tokenizer: .default)` (or `FTSIndexDescriptor`).
2. `try await db.create(Note.self)` / `createFTSIndex` only after `capabilities.ftsIndex`.
3. Query with typed DSL, e.g. `Note.where { $0.title.match(q) }.order { $0.title.score(q) }` — no hand-written `fts_match`.
4. Drive search UI via `@LiveQuery` + `setQuery` when the search string changes.

### Vector search

1. Store embeddings as `Vector32` columns (BLOB/text binding via `QueryBindable`).
2. Similarity: `Document.where { vectorDistanceCos($0.embedding, query) < threshold }.order { vectorDistanceCos(…) }.limit(k)`.
3. **Assume linear scan** until vector indexes are real (manual fact). Pre-filter with WHERE; prefer `vector32` over `vector64`.
4. Gate any `@VectorIndex` DDL on `capabilities.vectorIndex`; degrade gracefully if false.

### Materialized views

1. Prefer dashboard/report aggregates as `@MaterializedView` or `MaterializedViewDescriptor` once capability true.
2. Keep view SQL simple (no nested views; supported functions only).
3. Until typed `source:` DSL exists, generate `sourceSQL` from StructuredQueries SQL emission if possible rather than free-form strings in app code.
4. Do not put MV on ultra-hot write tables (write amplification).

### Encryption

1. **Today:** do not pass `encryption:`; use iOS Data Protection / file protection / Keychain for secrets at app layer.
2. **When C/Builder open is fixed:** init with `EncryptionConfig.aegis256(key:)` (preferred cipher per manuals), key from Keychain, never log hexkey.
3. Do not treat experimental encryption as compliance story until docs drop “not production ready.”

### Generated columns / STRICT / WITHOUT ROWID

1. Prefer `@BoutiqueTable(strict: true)` for typed discipline.
2. Avoid WITHOUT ROWID for app tables that need UPDATE/DELETE or CDC.
3. Use generated columns only if vendored lib has experimental generated columns enabled.

### Custom types

1. Model domains as Swift `enum` / value types with `QueryBindable` / `StringQueryBindable`.
2. Skip SQL `CREATE TYPE` until C binding exposes custom types.

### Capability-first bootstrap pattern

```text
open BoutiqueDB → read db.capabilities →
  create base tables always →
  create FTS/vector/MV only if flags true →
  surface reduced UX if false (keyword search only, etc.)
```

---

## 5. Risks if lib lacks flags / wrong open path

| Missing / wrong | Symptom | Product impact | Mitigation in current Swift stack |
|-----------------|---------|----------------|-----------------------------------|
| No `--experimental-index-method` (or FTS not built) | `CREATE INDEX … USING fts/vector` fails; probe false | No Tantivy FTS / no vector ANN indexes | `featureUnavailable`; additive sync skips those DDL |
| No `--experimental-views` | Materialized view DDL fails | Dashboards must re-aggregate live | Gated create; fall back to queries / app cache |
| No `--experimental-encryption` / no open-with-key API | Cannot set cipher; open of encrypted file fails | App that *assumes* encryption ships **plaintext** DB if it silently skips; current API **throws** instead | Hard throw on `encryption:` — good fail-closed for request; still no encryption path |
| App uses PRAGMA cipher without flag | Errors / non-encrypted file | False sense of security | Do not DIY PRAGMA until open API exists |
| No `turso_enable_experimental` for WITHOUT ROWID / generated | DDL rejected | Macros emit unusable DDL | Prefer STRICT-only tables; avoid WITHOUT ROWID for CRUD apps |
| CDC + MVCC same connection | Undefined / mutual exclusion | Broken live queries or concurrent writers | Dual connection + error on legacy same-handle |
| MVCC without busy retry | Frequent SQLITE_BUSY to UI | Perceived flakiness | Use `writeConcurrent` only |
| Vector index assumed present | DDL fails or planner ignores | Latency O(n) embeddings | Design for brute force; size limits / prefilter |
| Probe `encryption` true from hexkey PRAGMA only | False confidence | UI enables encryption settings that throw | Trust init throw / BD-001 until real open-with-key |
| Probe `mvcc` without journal_mode=mvcc | False negative | App hides concurrent writes wrongly | Enable concurrent path via `concurrentWrites` and second connection; re-probe on MVCC handle if needed |
| WITHOUT ROWID + CDC/MV | Engine rejects | Sync/live broken for those tables | Never combine |
| Multi-process without flag | Share DB across processes fails | Widget / extension / app concurrent file use unsafe | Keep single-process; App Group only after BD-004 |

### C binding structural risk (fact)

`BoutiqueDB-TursoFeatures.md` and `bindings/c/src/lib.rs` agree: the **SQLite C compatibility layer does not expose the Rust Builder’s per-feature open flags**. Features that need encryption, views, index methods, attach, multiprocess WAL, or custom types require either:

1. New C setters / open options, or  
2. A custom-built `libturso_sqlite3` with flags baked in, or  
3. Moving off plain `sqlite3_open` toward sdk-kit/Builder.

Until then, **capability probes + fail-closed init** are the safety net; **macros must not assume experimental DDL succeeds**.

---

## Cross-cutting invariants (facts)

1. **CDC commits only** — rolled-back txns produce no CDC rows (`cli/manuals/cdc.md`).
2. **CDC table columns:** `change_id`, `change_time`, `change_type` (1/0/-1), `table_name`, `id`, `before`/`after`/`updates` blobs.
3. **One active transaction per connection**; concurrent writers need **multiple connections** (`docs/manual.md`).
4. **LiveQuery** is generation-driven, not a full CDC row stream; sync engine has its own CDC cursor.
5. **I/O isolation:** `BoutiqueDB` is `@MainActor`; engine work on `DatabaseActor` (BD-014).

---

## Summary table: feature → Swift readiness

| Feature | Engine maturity | Swift API | Can ship without raw SQL? |
|---------|-----------------|-----------|---------------------------|
| CDC + LiveQuery | Practical | Strong | **Yes** |
| MVCC concurrent writes | Experimental | Strong (dual conn) | **Yes** (with retries) |
| FTS Tantivy | Yes (index method flag) | Strong (DSL + macro) | **Yes** if capability true |
| Vector functions | Yes | Strong (`Vector32` + distance) | **Yes** (scan-only) |
| Vector indexes | Manual: no; code: experimental | Macro + gate | **No** until engine+lib agree |
| Materialized views | Experimental | Partial (SQL string macro) | **Partial** |
| Encryption | Experimental | Declared, always unavailable | **No** (fail-closed) |
| Multi-process WAL | Experimental | Declared, always unavailable | **No** |
| Custom types | Experimental | Guidance only | Use Swift types instead |
| Generated / WITHOUT ROWID | Partial + flags | Macro params | Prefer STRICT only |

---

## Recommendations for the quality program

1. **Treat C-binding experimental surface as the critical path for v1.0** (BD-001 encryption open, BD-002 index methods, BD-003 views, BD-004 multiprocess) — Swift already models ideal APIs.
2. **Align docs:** either implement vector indexes or stop advertising `@VectorIndex` as ready; resolve `vector.md` vs macros.
3. **Harden probes:** probe MVCC on an MVCC-mode connection; treat encryption probe as “pragma exists,” not “usable.”
4. **Apple-app default profile:** CDC on, concurrentWrites optional, no encryption param, FTS if probe, vectors as functions-only, no WITHOUT ROWID on synced tables.
5. **Never ship silent feature drop for security** (encryption); continue fail-closed. Soft-drop search indexes only (as SchemaSync already does).

---

*End of K-Turso knowledge note.*

---

# K-SwiftUI — BoutiqueDB Observation & LiveQuery Knowledge

**Date:** 2026-07-22  
**Scope:** SwiftUI observation stack in `BoutiqueDB-Swift`  
**Sources:**  
- `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/Sources/TursoObservation/TursoInvalidation.swift`  
- `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/Sources/BoutiqueDB/LiveQuery.swift`  
- `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/Sources/BoutiqueDB/LiveQueryOne.swift`  
- `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/Sources/BoutiqueDB/BoutiqueDB.swift`  
- `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/Sources/BoutiqueDB/DatabaseActor.swift`  
- `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/docs/Architecture.md`  
- `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/docs/App-Template.md`  
- Tests: `LiveQueryIntegrationTests.swift`, `BoutiqueDBTests.swift`, `RefinementStressTests.swift`

---

## 1. Observation model (AsyncStream multi-consumer, generation, invalidate)

### `TursoStore` (core invalidator)

Path: `Sources/TursoObservation/TursoInvalidation.swift`

| Piece | Behavior |
|---|---|
| `@MainActor @Observable final class TursoStore` | UI-safe generation counter + subscription hub |
| `generation: UInt64` | Monotonic (wrapping add `&+= 1`) invalidation epoch |
| `lastChangeID: Int64` | In-memory cursor over `turso_cdc.change_id` |
| `ChangeEvent.generation(UInt64)` | Sole event type on the stream |
| `subscribe() -> AsyncStream<ChangeEvent>` | **Multi-consumer**: each call registers a new continuation under a `UUID` key |
| Buffering | `.bufferingNewest(1)` — late consumers keep only the latest generation bump |
| `invalidate()` | Bumps `generation`, yields `ChangeEvent.generation` to **all** live continuations |
| `changes` | Convenience alias that creates a **new** subscription each access (`public var changes: AsyncStream { subscribe() }`) |
| CDC listener | Cooperative `Task` loop: `SELECT COALESCE(MAX(change_id),0) FROM turso_cdc`; if advanced → MainActor `invalidate()`; else sleep `idlePollInterval` (default **50 ms**) |
| `advanceFromCDC()` | One-shot cursor advance + invalidate (tests / same-connection manual path) |
| Cursor isolation | Observation `lastChangeID` is **independent** of TursoCKSync’s persistent `ck_cdc_cursor` — observation never advances sync state |

**Termination:** `continuation.onTermination` hops to `@MainActor` and removes the UUID from `continuations`. `deinit` cancels the listener and finishes all continuations.

**Design intent (doc + comments):** Prefer explicit `invalidate()` after local writes when the writer already knows data changed; the CDC poll covers **other connections** / external mutators. Local `BoutiqueDB.write` always invalidates immediately (see §2).

### `TursoQueryBox<Value>` (lower-level reactive box)

Same file. Synchronous `fetch: () throws -> Value` re-run on every stream event via `forceRefresh()`. Tracks `lastGeneration` for optional `refreshIfNeeded()`. Useful for non-`Table` / non-StructuredQueries consumers; **LiveQuery does not use this type** — it reimplements the subscribe + reload loop with async `db.read`.

### LiveQuery subscription pattern

Both `LiveQuery` and `LiveQueryOne`:

1. Create `observationTask` on init.  
2. `await load()` once (initial fetch).  
3. `for await _ in db.store.subscribe() { await load() }`.  
4. `deinit` cancels the task.

Every LiveQuery therefore holds **its own** multi-consumer slot. Dual LiveQuery tests assert both refresh on one write (`twoLiveQueriesBothRefresh`, `dualLiveQueryWithWritesAndDrain`).

**Invalidation fan-out is coarse:** every generation bump reloads **every** subscribed query (no table / row filter). Correctness-first; cost grows with N queries × full refetch.

---

## 2. MainActor / DatabaseActor integration

From `docs/Architecture.md` and implementation:

```
SwiftUI / @Observable models
        │
        ▼
BoutiqueDB (@MainActor)          ← open + migrate + LiveQuery host
   ├── DatabaseActor             ← serialized I/O (reads/writes)
   ├── concurrent DatabaseActor  ← optional MVCC writer (CDC ⊥ MVCC)
   └── TursoStore                ← AsyncStream / generation invalidation
```

### Rules (BD-014 / BD-005)

| Surface | Isolation | Role |
|---|---|---|
| `BoutiqueDB` | `@MainActor`, `Sendable` | SwiftUI-safe container; holds `store`, opens connections |
| `DatabaseActor` | `actor` | All SQLite I/O for one `TursoConnection` |
| `TursoStore` | `@MainActor @Observable` | Generation + streams; never blocks on long SQL in invalidate path |
| `LiveQuery` / `LiveQueryOne` | `@MainActor @Observable` | Own `wrappedValue` / `isLoading` / `loadError` |

### Hop pattern

```text
UI / LiveQuery (MainActor)
  → await db.read / db.write  (BoutiqueDB MainActor methods)
    → await databaseActor.read|write  (DatabaseActor)
      → TursoConnection blocking SQLite
  ← result Sendable-bound back
  → store.invalidate()          // only after successful write paths
```

Write paths that call `store.invalidate()` after I/O completes:

- `BoutiqueDB.write`  
- `BoutiqueDB.writeConcurrent`  
- `BoutiqueDB.commitConcurrent`  
- `BoutiqueDB.execute` (via `write`)

Reads never invalidate. Listener path: background poll → `MainActor.run { lastChangeID = …; invalidate() }`.

### Deadlock hazard (documented)

**Do not** call MainActor-isolated `BoutiqueDB` methods from inside a `DatabaseActor` body. LiveQuery correctly only uses `db.read { … }` from the observation task (MainActor), never re-enters BoutiqueDB from the actor closure.

### Dependencies

`Sources/BoutiqueDB/Dependencies/BoutiqueDBDependencyKey.swift` exposes `DependencyValues.boutiqueDB`. App template sets it in `.task` after `BoutiqueDB.open`. LiveQuery still requires an explicit `BoutiqueDB` instance at construction — no automatic `@Dependency(\.boutiqueDB)` injection inside the property wrapper.

---

## 3. LiveQuery lifecycle and `setQuery`

### `LiveQuery<Element>` — `Sources/BoutiqueDB/LiveQuery.swift`

```text
init(wrappedValue: [], db, query)
  → startObserving()
       → Task: load() once
       → for await store.subscribe() → load()

setQuery(newFactory)
  → replace query closure
  → forceRefresh() → Task { await load() }

load()
  → isLoading = true
  → db.read { query().fetchAll(connection) }
  → wrappedValue / loadError
  → isLoading = false
```

| API | Notes |
|---|---|
| `wrappedValue: [Element]` | Observed; drives SwiftUI when LiveQuery is held as `@Observable` state or re-exported |
| `loadError` / `isLoading` | Surface for error UI / skeletons |
| `setQuery(_:)` | Dynamic query swap (FTS search text, filters); **async reload** — tests poll until count matches |
| `forceRefresh()` / `refresh()` | Manual kick; same as stream-driven `load` |
| Query type | `@Sendable () -> SelectOf<Element>` — factory re-evaluated each load |

**Class + propertyWrapper:** `LiveQuery` is a `@propertyWrapper` **and** a reference type (`final class`). Practical app pattern (from `docs/App-Template.md` and tests) is **not** `@LiveQuery var notes` on a View, but:

```swift
@MainActor
@Observable
final class NotesModel {
  let db: BoutiqueDB
  @ObservationIgnored private let live: LiveQuery<Note>
  var notes: [Note] { live.wrappedValue }

  init(db: BoutiqueDB) {
    self.db = db
    self.live = LiveQuery(db) { Note.all.asSelect() }
  }
}
```

`@ObservationIgnored` avoids double-observation of the wrapper object; the model’s computed `notes` re-reads `live.wrappedValue`. Because `LiveQuery` is `@Observable`, mutations to `wrappedValue` notify observation clients that track through the live instance — the template relies on the model re-exposing values (SwiftUI sees model updates when `live`’s observable fields change only if something observes `live`; **the re-export pattern depends on either observing `live` or having the parent model re-trigger**). Tests hang `LiveQuery` directly or read `liveNotes.wrappedValue` via a model computed property and poll until refresh — integration coverage validates end-to-end after `db.write`.

### `LiveQueryOne<Element>` — `Sources/BoutiqueDB/LiveQueryOne.swift`

Same observe/load loop, but:

- `wrappedValue: Element?` via `fetchOne`  
- **No `setQuery`** — filter changes require constructing a new wrapper  
- Same `forceRefresh` / `load` / `loadError` / `isLoading`

### Lifecycle edge cases

| Case | Behavior |
|---|---|
| Init before schema ready | First `load` may set `loadError`; later invalidations retry |
| Write on primary path | Immediate invalidate → reload without waiting for CDC poll |
| Foreign connection write | CDC listener (50 ms idle) eventually invalidate |
| `startListening: false` | Still invalidates on `db.write`; CDC path needs `advanceFromCDC` / manual invalidate |
| Deinit while loading | Task cancel; in-flight load may complete if already on actor (no explicit generation token / load cancellation token) |
| Multi LiveQuery | Independent streams; both receive same generation events |

### Tests map

| Test | Path | Asserts |
|---|---|---|
| `liveQueryAutoRefreshes` | `Tests/BoutiqueDBTests/BoutiqueDBTests.swift` | Write → array updates ≤ 500 ms |
| `liveQueryOneAutoRefreshes` | same | Single-row update refresh |
| `twoLiveQueriesBothRefresh` | same | Multi-consumer fan-out |
| `forceRefreshWorks` | same | `advanceFromCDC` + manual load |
| `liveQueryUpdatesWithinOneSecond` | `LiveQueryIntegrationTests.swift` | `@Observable` model + `LiveQuery` |
| `liveQuerySetQueryReloads` | `RefinementStressTests.swift` | `setQuery` filters to one row |
| `dualLiveQueryWithWritesAndDrain` | same | Dual LQ + concurrent write + CDC drain |

---

## 4. Gaps vs SQLiteData `@FetchAll` UX

BoutiqueDB’s LiveQuery is a solid **v1 reactive fetch**, but SQLiteData / SharingGRDB-style `@FetchAll` remains a higher-bar product UX. Gaps:

| Dimension | BoutiqueDB today | SQLiteData `@FetchAll` style (target UX) |
|---|---|---|
| **Declaration site** | Manual model: hold `LiveQuery`, re-export `wrappedValue` | `@FetchAll var notes: [Note]` (or similar) on View / `@Observable` model with minimal glue |
| **DB injection** | Explicit `LiveQuery(db) { … }` | Often `@Dependency` / environment / shared database default |
| **Query dynamism** | Imperative `setQuery` (array only); no `setQuery` on `LiveQueryOne` | Query identity tracks bound state (search string, selected id) with less ceremony |
| **Property-wrapper ergonomics** | `@propertyWrapper` exists but app template avoids pure `@LiveQuery` on Views | First-class wrapper + projected value for bindings / load state |
| **Invalidation granularity** | Global generation — every query refetches | Table- (or statement-) scoped invalidation reduces thrash |
| **CDC delivery** | Cooperative poll 50 ms when idle (docs still mention “true CDC AsyncStream” as future) | Ideally push-driven or tighter engine hook; less background wake |
| **Loading / error UX** | `isLoading` / `loadError` present but no standardized View helpers | Skeleton / error View patterns often shipped in framework samples |
| **Observation wiring** | Easy to misuse (`@ObservationIgnored` + computed property required knowledge) | Fewer footguns; values update Views automatically |
| **App bootstrap** | `.task { open; prepareDependencies }` then optional model | Clearer “database ready” environment / `View` modifier patterns |
| **Fetch one / keyed** | Separate `LiveQueryOne` type | Unified API family (`@FetchAll` / `@FetchOne` / `@Fetch`) |
| **Search / FTS live** | Documented future: `@LiveQuery(search:text:)` overload (`BoutiqueDB-TursoFeatures.md`) | Built-in search binding patterns |
| **Cancellation / generation races** | Last `load` wins only by accident (no load generation guard) | Stale response discard when query changes mid-flight |
| **Preview / test fakes** | Dependency key fatalErrors if unset | Easy in-memory preview databases |

**What already matches the direction well:**

- Local writes invalidate **immediately** (no poll wait for own commits).  
- Multi-consumer AsyncStream scales to multiple LiveQueries.  
- `@MainActor` + actor I/O split matches SwiftUI concurrency guidance.  
- StructuredQueries `SelectOf` factories compose FTS/vector filters once DSL is ready.  
- Tests assert dual consumers, `setQuery`, and model integration.

---

## 5. Recommendations for seamless SwiftUI apps

### Product / API (high leverage)

1. **`@FetchAll` / `@FetchOne` façade (or real View-friendly property wrappers)**  
   - Default to `@Dependency(\.boutiqueDB)`.  
   - Allow `LiveQuery` construction without threading `db` through every model.  
   - Support both `@Observable` models and SwiftUI `View` storage (`@State` / `@StateObject`-like ownership).

2. **Reactive query binding**  
   - e.g. `LiveQuery(db, query: { Note.where { $0.title.contains(search) } })` where `search` is read from an `@Observable` source each reload, **or** `bindQuery` that rebuilds on observed property change.  
   - Add `setQuery` to `LiveQueryOne` for parity (detail screens with changing id).

3. **Stale-load protection**  
   - Capture `generation` (or query token) at start of `load()`; apply results only if still current after `db.read`. Prevents out-of-order updates under rapid `setQuery` / bursty invalidation.

4. **Table-scoped invalidation (medium term)**  
   - When CDC rows are available, map `turso_cdc` table names → interested queries.  
   - Keep global `invalidate()` for “unknown / write path” correctness.  
   - Cuts N× full refetch cost in multi-screen apps.

5. **True change push (longer term)**  
   - Replace idle poll with engine notification or shared-memory wake when bindings allow (`BoutiqueDB-TursoFeatures.md` already flags this).  
   - Keep 50 ms poll as fallback.

6. **Projected value / View helpers**  
   - `$live.isLoading`, `$live.loadError`, `LiveQueryView` / `.task` modifiers for “database not ready” gating matching `docs/App-Template.md` ProgressView pattern.

7. **Document the canonical ownership pattern** in README next to Architecture:  
   - Always `@ObservationIgnored private let live` + computed surface, **or** make LiveQuery updates automatically poke a parent `Observable` via callback / `withMutation` if wrapper-on-model remains the recommended style.

### App architecture (today, without new APIs)

```text
App.task
  → BoutiqueDB.open(migrations:)
  → prepareDependencies { $0.boutiqueDB = db }
  → store already startListening (default true)

@Observable model
  → LiveQuery / LiveQueryOne owned privately
  → mutations only via db.write / writeConcurrent  (auto-invalidate)
  → search: call live.setQuery { … } on text change

Views
  → @State model or @Environment / dependency-injected model
  → List(model.notes) — no direct SQLite in body
```

### Testing guidance

- Prefer `db.write` paths (invalidate included) over raw `connection.execute` unless testing CDC.  
- Use `waitFor` polling helpers as existing suites do (`LiveQueryIntegrationTests`).  
- For listener-off tests: `advanceFromCDC()` or `store.invalidate()` + `await load()`.  
- Stress dual LiveQuery + concurrentWrites + CDC drain already in `RefinementStressTests`.

### Priority order for “SQLiteData-class” feel

1. Stale-load tokens + `LiveQueryOne.setQuery` (correctness, small).  
2. Dependency-default + View-friendly wrappers (DX).  
3. Query binding to search/filter state (DX).  
4. Table-scoped invalidation (scale).  
5. Engine push CDC (efficiency).

---

## Quick reference

| Symbol | Path |
|---|---|
| `TursoStore` / `ChangeEvent` / `TursoQueryBox` | `BoutiqueDB-Swift/Sources/TursoObservation/TursoInvalidation.swift` |
| `LiveQuery` | `BoutiqueDB-Swift/Sources/BoutiqueDB/LiveQuery.swift` |
| `LiveQueryOne` | `BoutiqueDB-Swift/Sources/BoutiqueDB/LiveQueryOne.swift` |
| `BoutiqueDB.write` → `invalidate` | `BoutiqueDB-Swift/Sources/BoutiqueDB/BoutiqueDB.swift` |
| `DatabaseActor` | `BoutiqueDB-Swift/Sources/BoutiqueDB/DatabaseActor.swift` |
| Architecture concurrency rules | `BoutiqueDB-Swift/docs/Architecture.md` |
| Copy-paste app model | `BoutiqueDB-Swift/docs/App-Template.md` |
| LiveQuery tests | `BoutiqueDB-Swift/Tests/BoutiqueDBTests/{BoutiqueDBTests,LiveQueryIntegrationTests,RefinementStressTests}.swift` |

**Bottom line:** BoutiqueDB’s SwiftUI story is **generation-based multi-consumer observation** with MainActor store + actor I/O and immediate local-write invalidation. It is production-usable via `@Observable` models, but still **one layer below** SQLiteData `@FetchAll` for declaration ergonomics, query binding, scoped invalidation, and stale-fetch safety.

---

# K-Sync knowledge: BoutiqueDB CloudKit / SyncAdapter

**Date:** 2026-07-22  
**Scope (read-only):**
- `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/Sources/TursoCKSync/`
- `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/docs/CloudKit-QA-Checklist.md`
- `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/Tests/TursoCKSyncTests/`
- `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/Sources/BoutiqueDB/BoutiqueDBSyncEngine.swift`

**Related:** `docs/Architecture.md`, `docs/App-Template.md`, `docs/Sync-Benchmarks.md`, `BoutiqueDB-Design.md`

---

## 1. SyncAdapter architecture

### Layering

```
SwiftUI / app
    │
    ▼
BoutiqueDBSyncEngine          (@MainActor façade; BoutiqueDB product)
    │  adapter: CloudKitSyncAdapter
    ▼
SyncAdapter protocol          (pluggable multi-device surface)
    └── CloudKitSyncAdapter   (default; status fan-out)
            │  engine: TursoCKSyncEngine
            ▼
TursoCKSyncEngine             (CDC ↔ CKSyncEngine / local pending queue)
    ├── SyncMetadataStore     (ck_* tables in the Turso file)
    ├── RecordMapper          (row ↔ CKRecord + system fields)
    ├── RowSQL                (upsert/delete against user tables)
    └── SyncedTable / TursoCKSyncConfiguration
```

| Type | Path | Role |
|------|------|------|
| `SyncAdapter` | `Sources/TursoCKSync/SyncAdapter.swift` | Protocol: `start`, `stop`, `syncStatus()`, `drainLocalChanges()`, `applyRemoteChanges` |
| `CloudKitSyncAdapter` | same | Default impl; owns status stream + wraps engine |
| `SyncStatus` | same | `idle` / `syncing` / `failed(String)` / `needsAuthentication` / `accountChanged` |
| `RemoteChange` | same | Transport-agnostic: `.upsert(CKRecord)` / `.delete(CKRecord.ID)` |
| `TursoCKSyncEngine` | `Sources/TursoCKSync/TursoCKSyncEngine.swift` | CDC drain, inbound apply, `CKSyncEngineDelegate`, conflicts, account wipe |
| `BoutiqueDBSyncEngine` | `Sources/BoutiqueDB/BoutiqueDBSyncEngine.swift` | Thin `@MainActor` wrapper: builds `TursoCKSyncConfiguration` + adapter; exposes `start`, `syncStatus`, `drainCDC`, `drainLocalChanges`, `performLocalWrite` |
| `SyncedTable` | `Sources/TursoCKSync/SyncedTable.swift` | Table name, PK column (default `id`), synced columns, record type |
| `ConflictPolicy` | same | `.serverWins` (default), `.clientWins`, `.lastWriterWins(field:)` |
| `TursoCKSyncConfiguration` | same | container ID, zone (`app.default`), tables, policy, `maxBatchSize` (≤250), `drainCDCLimit` (default 500), `enablesCloudKit` |
| `SyncMetadataStore` | `Sources/TursoCKSync/SyncMetadataStore.swift` | On-disk CK state, CDC cursor, record meta, account hash, format version |
| `RecordMapper` / `RowSQL` | `RecordMapper.swift`, `RowSQL.swift` | Encode/decode system fields; SQL upsert/delete |

### Design intent

- **Pluggable transport:** `SyncAdapter` is documented for a future Turso Cloud adapter without changing app APIs (`SyncAdapter.swift` header; design doc §7).
- **Offline / unit-test mode:** `enablesCloudKit: false` skips `CKContainer` / `CKSyncEngine` and keeps `localPendingRecordZoneChanges` in-process (`TursoCKSyncEngine.start`).
- **Record identity:** `table:rowPK` record names (`RecordIdentity` in `SyncedTable.swift`); zone `configuration.zoneName` / owner `CKCurrentUserDefaultName`.
- **CDC isolation:** engine sets `connection.isSynchronizing` during inbound apply so `drainCDC` no-ops mid-apply; cursor advanced past echo CDC rows.

### Status stream

`CloudKitSyncAdapter` multiplexes `SyncStatus` to subscribers via `AsyncStream` (newest buffer 8). Engine pushes via `statusSink` (wired in adapter init). Adapter also publishes around `start` / `drainLocalChanges` / `applyRemoteChanges`.

**PROD-BLOCK gap:** `SyncStatus.needsAuthentication` is defined but **never published** anywhere in the package (only mention is the enum case). Live apps cannot observe “not signed into iCloud” through the official stream without extra work.

---

## 2. CDC drain → pending → apply remote

### Outbound (local → CloudKit)

1. **Prerequisite:** connection opened with CDC (`BoutiqueDB` defaults `enableCDC: true`; engine also calls `enableCaptureDataChanges(mode: .full)` on init).
2. **App writes** to user tables (via `BoutiqueDB.write` or `TursoCKSyncEngine.performLocalWrite`).
3. **`drainCDC(limit:)`** (`TursoCKSyncEngine`):
   - Returns 0 if `isSynchronizing` or `stopped`.
   - Loads `ck_cdc_cursor.last_change_id`.
   - Reads `connection.cdcChanges(after:cursor, limit:)` (default limit 500).
   - Filters to `configuration.syncedTableNames`.
   - Resolves row PK (TEXT PK, or rowid → PK column, or CDC before/after payload via `bin_record_json_object`).
   - Builds `CKRecord.ID` as `table:pk` in the app zone.
   - Deletes → `.deleteRecord`; inserts/updates → `.saveRecord`.
   - **`enqueuePending`:** either `CKSyncEngine.state.add(pendingRecordZoneChanges:)` or local queue.
   - Advances CDC cursor to last seen change ID **even for filtered tables** (non-synced table CDC is skipped but cursor still moves with `lastID`).
4. **Send path (CloudKit only):** `CKSyncEngineDelegate.nextRecordZoneChangeBatch` caps to `maxBatchSize` (≤250), `makeRecord` loads row + hydrates system fields; missing local row drops the save.
5. **After successful save:** `handleSentRecordZoneChanges` upserts `ck_record_meta` with encoded system fields.

Convenience APIs:
- `performLocalWrite { ... }` → write then `drainCDC()`.
- `CloudKitSyncAdapter.drainLocalChanges()` / `BoutiqueDBSyncEngine.drainLocalChanges()` → `drainCDC` with config limit + status transitions.

### Inbound (CloudKit → local)

1. **Live:** `fetchedRecordZoneChanges` → `applyModification` / `applyDeletion`.
2. **Tests / custom transport:** `applyRemoteRecord` / `applyRemoteDeletion` or adapter `applyRemoteChanges([RemoteChange])`.
3. **`applyModification`:**
   - Resolve table by record type or record-name prefix.
   - Map CK fields → row via `RecordMapper.rowDictionary`.
   - `isSynchronizing = true`, upsert row + `ck_record_meta`, advance CDC cursor past echo.
4. **`applyDeletion`:** resolve via record name or `ck_record_meta`, delete row + meta, advance cursor.

### Metadata tables (`SyncMetadataStore.schemaSQL`)

| Table | Purpose |
|-------|---------|
| `ck_sync_state` | `CKSyncEngine.State.Serialization` blob |
| `ck_record_meta` | per-row record name, zone, system fields (unique on record_name) |
| `ck_cdc_cursor` | last drained CDC change_id |
| `ck_account` | account_hash for BD-007 identity change |
| `ck_meta_version` | format_version (currently 1) |

### Echo suppression

Inbound writes would re-appear in `turso_cdc`. Engine advances cursor with `advanceCDCCursorPastEcho(from:)` (up to 10k CDC rows after apply) so a subsequent drain does not re-pend the same row (asserted in `inboundApplyAndEchoSuppression` test).

### Coupling gaps (integration)

| Issue | Severity |
|-------|----------|
| **No automatic drain after `BoutiqueDB.write`** — apps must call `drainCDC` / `drainLocalChanges` / `performLocalWrite` themselves | **PROD-BLOCK** for “sync just works” |
| **`BoutiqueDB.open` does not create or start `BoutiqueDBSyncEngine`** | App must wire sync separately |
| **App template (`docs/App-Template.md`) has zero sync wiring** | New apps ship local-only by default |
| CDC cursor advanced for all CDC rows including non-synced tables (filter only skips enqueue) | Usually fine; document behavior |

---

## 3. Conflicts, account, batching, multi-table

### Conflicts (`ConflictPolicy`)

Handled on `.serverRecordChanged` in `handleSentRecordZoneChanges` / `handleServerRecordChanged`, and via test hook `resolveConflictForTesting`:

| Policy | Behavior |
|--------|----------|
| `.serverWins` | `applyModification(serverRecord)` |
| `.clientWins` | Apply server (absorb system fields), then re-pend `.saveRecord(failedRecord.recordID)` |
| `.lastWriterWins(field:)` | Compare comparable stamps (NSNumber, ISO8601 string, NSDate) on field; newer local → update meta system fields from server + re-pend; else apply server |

Other save failures:
- `zoneNotFound` → re-save zone + record
- `unknownItem` → clear system fields meta, re-pend save
- network / busy / notAuthenticated / cancelled → no immediate retry in handler (rely on CKSyncEngine)

### Account

| Path | Behavior |
|------|----------|
| `CKSyncEngine.Event.accountChange` | `.signIn` → re-upload zone + `enqueueAllLocalRows`; `.switchAccounts` / `.signOut` → `wipeAndRebootstrap(preserveLocalUserData: true)` |
| `noteAccountIdentity(_:)` | If stored hash differs → status `.accountChanged`, wipe meta + re-enqueue local rows, save new hash |
| `detectAccountIdentityChangeIfNeeded()` | **Stub:** only loads stored hash; comment says live apps must call `noteAccountIdentity` from CK account status callbacks |
| Zone deleted (`fetchedDatabaseChanges` deletions for zone) | `wipeAndRebootstrap(preserveLocalUserData: false)` — **wipes user synced tables** |

`wipeAndRebootstrap`:
- Optionally `RowSQL.deleteAllSyncedData` for all `syncedTables`
- `metadata.wipeAll()` (meta, state, cursor, account hash)
- Clears engine / local pending; `start()` again
- If preserving data → `enqueueAllLocalRows()`

**PROD-BLOCK:** Account detection is not automatic. Apps that never call `noteAccountIdentity` miss crash-safe rebootstrap across process death (BD-007 path only partially wired). `detectAccountIdentityChangeIfNeeded` in adapter `start()` does not perform a live iCloud identity probe.

### Batching

| Knob | Default | Cap / notes |
|------|---------|-------------|
| `drainCDCLimit` | 500 | Per `drainCDC` call; large writes need multiple drains |
| `maxBatchSize` | 250 | Hard-capped to CloudKit 250 in configuration init |
| `pendingBatches()` | — | Splits current pending into ≤ maxBatchSize chunks (test/diagnostics) |
| `nextRecordZoneChangeBatch` | — | Live send path prefixes pending to maxBatchSize |

QA checklist expects ≥600 local inserts → outbound ≤250/batch, CDC steps ≤500.

### Multi-table

- Config holds `[SyncedTable]`; drain filters by name set.
- Each table → own record type (default table name) and `table:pk` names.
- `enqueueAllLocalRows` iterates all synced tables.
- Zone wipe deletes data from **all** synced tables when not preserving user data.

---

## 4. Test coverage present vs missing

**Harness:** `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/Tests/TursoCKSyncTests/TursoCKSyncTests.swift`  
**Mode:** almost exclusively `enablesCloudKit: false` (documented in QA checklist).  
**Extra:** `Tests/BoutiqueDBTests/RefinementStressTests.swift` lightly drains CDC with LiveQuery stress.

### Present (automated)

| Area | Test(s) |
|------|---------|
| Metadata migrate / cursor / record meta | `metadataAndStatePersistence` |
| Outbound CDC → pending + `makeRecord` | `outboundCDCDrainBuildsPendingChanges` |
| Inbound upsert + echo suppression + remote delete | `inboundApplyAndEchoSuppression` |
| System fields round-trip on rebuild | `systemFieldsRoundTrip` |
| Wipe without preserve (user data deleted) | `accountWipeRebootstrap` |
| Simulated A→B insert + delete | `simulatedTwoDeviceRoundTrip` |
| Multi-table pending names | `multiTableSyncRoundTrip` |
| Batching 600 rows / max 250 | `pendingBatchesRespectMaxBatchSize` |
| LWW conflict (local newer) | `lastWriterWinsConflict` |
| Server wins conflict | `serverWinsConflict` |
| Account hash change preserves rows | `accountHashChangePreservesLocalData` |
| Preserve wipe re-enqueues | `wipePreservingDataReenqueues` |
| Adapter status stream + start/drain | `cloudKitSyncAdapterStatusStream` |
| Observation CDC generation (TursoStore) | `observationInvalidatesOnCDC` |

### Missing or weak (relative to QA checklist / prod)

| Gap | Severity |
|-----|----------|
| **Live CloudKit** (entitlements, two devices, real `CKSyncEngine`) — checklist only | **PROD-BLOCK** (manual) |
| **`.clientWins` conflict** automated test | Missing |
| LWW when **server newer** (should apply server) | Missing (only local-newer case) |
| Concurrent edit race under real send path | Missing (test hook only) |
| Network mid-sync / retry / no duplicate PKs | Manual only |
| Zone deletion event path | Logic exists; not driven by synthetic CK events offline |
| `needsAuthentication` / not signed in | Unimplemented + untested |
| `BoutiqueDBSyncEngine` façade | No dedicated tests |
| Auto-drain after `BoutiqueDB.write` | N/A (feature missing) |
| Multi-table **inbound** apply for two record types on peer | Outbound pending only |
| Drain pagination loop for >500 CDC rows until empty | Partially implied by batch test; no explicit multi-drain API test |
| Integer PK / non-TEXT PK tables | Coverage biased to TEXT UUID PK |
| Benchmarks tables in `docs/Sync-Benchmarks.md` | Empty placeholders |
| `applyRemoteChanges` multi-batch via adapter | Not covered beyond status test |

---

## 5. Prod Apple app integration checklist gaps

Map of **required production work** vs what the package provides. Items marked **PROD-BLOCK** block a real App Store multi-device launch if left undone.

### Entitlements & packaging

| Checklist item | Package support | Gap |
|----------------|-----------------|-----|
| iCloud + CloudKit capability; container ID = `TursoCKSyncConfiguration.containerIdentifier` | Config accepts identifier; default dev id `iCloud.com.turso.cloudkit.dev` if nil | **PROD-BLOCK:** wrong/missing container traps or silent wrong DB |
| Embed `libturso_sqlite3` with CDC | `BoutiqueDB` / build scripts | Ensure release app does not strip CDC; vendor path in Package.swift |
| `enablesCloudKit: true` | Default true on `BoutiqueDBSyncEngine` | Tests always false — easy to ship a test-only mental model |
| Min OS iOS 17 / macOS 14 for `CKSyncEngine` | Design BD-006 | Document in app target deployment target |

### Runtime wiring (not in App-Template)

| Step | Status |
|------|--------|
| Open DB with `enableCDC: true` (default) | OK |
| **Never** enable CDC + MVCC on same connection; use `concurrentWrites` for second handle | Documented Architecture / README |
| Construct `BoutiqueDBSyncEngine` / `CloudKitSyncAdapter` with `syncedTables` matching schema | **App responsibility** |
| Call `start(automaticallySync: true)` (or adapter `start()`) | **App responsibility** |
| After every local commit path: `drainLocalChanges()` / `drainCDC()` or use `performLocalWrite` | **PROD-BLOCK** if forgotten — changes sit only in CDC |
| UI bind to `syncStatus()` | Status partial; no `needsAuthentication` |
| Call `noteAccountIdentity` from `CKContainer.accountStatus` / userRecordID | **PROD-BLOCK** — `detectAccountIdentityChangeIfNeeded` is a no-op probe |
| Handle `.accountChanged` / rebootstrap UX | Status exists; app must preserve UX for local data |
| Optional: pull-to-sync / timer drain for large backlogs | Not built-in |
| Conflict policy choice per product (default serverWins) | Config only; document for users |

### Manual QA (from `docs/CloudKit-QA-Checklist.md`)

Still required before release:

1. Two devices same iCloud — insert / edit / delete ≤30s  
2. Multi-table both record types  
3. Sign-out / switch account → local preserved, re-upload  
4. Network off mid-sync → failed/retry, no corruption, no duplicate PKs  
5. ≥600 rows batching  
6. Conflict policy spot-checks for all three policies  
7. Sign-off table (empty in checklist)

### Façade vs engine API mismatches

| `BoutiqueDBSyncEngine` | Underlying gap |
|------------------------|----------------|
| `start(automaticallySync:)` is **sync throws**, not `async` like `SyncAdapter.start` | Apps using protocol vs façade differ |
| No `stop`, `applyRemoteChanges`, `noteAccountIdentity`, `wipeAndRebootstrap` on façade | Apps must reach `adapter.engine` for account/prod recovery |
| Hard-codes `drainCDCLimit: 500` in init | Cannot tune without constructing adapter/config manually |

### Architecture doc vs code

- Architecture diagram lists `SyncAdapter` under `BoutiqueDB` — correct product-wise, but **not auto-attached** to open.
- Design checklist marks `BoutiqueDBSyncEngine` and `SyncAdapter` done; **integration completeness** (auto-drain, account probe, app template, live CI) remains open for production.

---

## Prod-block summary

1. **No automatic CDC drain** after normal `BoutiqueDB` writes — silent data never leaves device.  
2. **Account identity** depends on app-called `noteAccountIdentity`; start-time detect is a stub.  
3. **`needsAuthentication` never emitted** — incomplete status model for unsigned users.  
4. **No live CloudKit automated tests** — only offline queue; two-device/network/zone QA is manual.  
5. **App template omits sync entirely** — high risk of incomplete integration.  
6. **Façade incomplete for account wipe / stop / identity** — easy to miss BD-007 paths.  
7. **`.clientWins` and full LWW matrix untested** — policy bugs ship if used in prod without manual QA.

---

## Source index

| Path | Contents |
|------|----------|
| `BoutiqueDB-Swift/Sources/TursoCKSync/SyncAdapter.swift` | Protocol, status, adapter |
| `BoutiqueDB-Swift/Sources/TursoCKSync/TursoCKSyncEngine.swift` | CDC, apply, delegate, conflicts, account |
| `BoutiqueDB-Swift/Sources/TursoCKSync/SyncMetadataStore.swift` | Schema + persistence |
| `BoutiqueDB-Swift/Sources/TursoCKSync/SyncedTable.swift` | Tables, identity, config, policies |
| `BoutiqueDB-Swift/Sources/TursoCKSync/RecordMapper.swift` | CKRecord mapping |
| `BoutiqueDB-Swift/Sources/TursoCKSync/RowSQL.swift` | Local SQL mutations |
| `BoutiqueDB-Swift/Sources/BoutiqueDB/BoutiqueDBSyncEngine.swift` | App-facing wrapper |
| `BoutiqueDB-Swift/docs/CloudKit-QA-Checklist.md` | Manual RC checklist |
| `BoutiqueDB-Swift/Tests/TursoCKSyncTests/TursoCKSyncTests.swift` | Offline unit coverage |

---

# K-Macros — BoutiqueDB-Swift schema macros & migration surface

**Date:** 2026-07-22  
**Scope:** BoutiqueDB-Swift macros, `BoutiqueSchema`, migrator, `ensureColumn`, schema sync  
**Sources (absolute paths under BoutiqueDB-Swift unless noted):**

| Area | Path |
|------|------|
| Macro implementations | `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/Sources/BoutiqueDBMacros/` |
| Public macro decls + schema | `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/Sources/BoutiqueDB/Schema/` |
| Migration / sync | `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/Sources/BoutiqueDB/Migration/` |
| Open + create | `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/Sources/BoutiqueDB/BoutiqueDB+Open.swift`, `BoutiqueDB.swift` |
| Macro expansion tests | `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/Tests/BoutiqueDBMacrosTests/` |
| Runtime migration tests | `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/Tests/BoutiqueDBTests/MigrationTests.swift` |
| Docs | `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/docs/Migrations.md` |

---

## 1. Macro surface and generated DDL

### Plugin registration

`BoutiqueDBMacrosPlugin` (`Sources/BoutiqueDBMacros/Plugin.swift`) registers four macros:

1. `BoutiqueTableMacro` → `@BoutiqueTable`
2. `FTSIndexMacro` → `@FTSIndex`
3. `VectorIndexMacro` → `@VectorIndex`
4. `MaterializedViewMacro` → `@MaterializedView`

Public declarations live in `Sources/BoutiqueDB/Schema/Macros.swift` and attach **member** + **extension** (`BoutiqueSchema` conformance).

### `@BoutiqueTable(name:withoutRowid:strict:)`

**Impl:** `Sources/BoutiqueDBMacros/BoutiqueTableMacro.swift`  
**Generates:**

- `static var boutiqueTableName: String`
- `static var boutiqueCreateStatements: [String]`
- `extension Type: BoutiqueSchema`

**DDL shape:**

```sql
CREATE TABLE IF NOT EXISTS "notes" (
  "id" TEXT PRIMARY KEY NOT NULL,
  "title" TEXT NOT NULL,
  ...
) [WITHOUT ROWID] [STRICT]
```

**Column derivation** (stored properties only; skips computed/`accessorBlock`):

| Swift | SQL type (`SQLHelpers.swift`) |
|-------|--------------------------------|
| `Int`, `Int64`, `Int32`, `UInt*`, `Bool` | `INTEGER` |
| `Double`, `Float`, `CGFloat` | `REAL` |
| `Data`, `Vector32`, `Vector32Sparse` | `BLOB` |
| everything else (`String`, `Date`, `UUID`, …) | `TEXT` |

**Constraints / flags:**

- Primary key: property named `id`, **or** `@Column(primaryKey: …)` (attribute name `"Column"`; accepts `true` or enum-style `.something` via `boolishPrimaryKey`).
- Non-optional → `NOT NULL` (including PK).
- Optional (`T?` / `Optional<T>`) → nullable, no `NOT NULL`.
- `@GeneratedColumn(expression: "…")` → `GENERATED ALWAYS AS (…) VIRTUAL` (recognized by macro **only**; no public `@GeneratedColumn` macro declaration in `Macros.swift`).
- Stacked `@FTSIndex` / `@VectorIndex` on the same struct: their DDL is **appended** into `boutiqueCreateStatements`.

**Table name:** `name:` argument, else StructuredQueries-light pluralization (`Note` → `notes`, `*y` → `*ies`, already-`s` left as-is) in `BoutiqueSQL.defaultTableName`.

**Test evidence:** `BoutiqueDBMacrosTests.boutiqueTableGeneratesStrictDDL` expands `@BoutiqueTable(strict: true)` on `Note` to STRICT table DDL + `BoutiqueSchema` extension.

### `@FTSIndex(_ columns:…, tokenizer:, name:)`

**Impl:** `FTSIndexMacro.swift`  
**DDL:**

```sql
CREATE INDEX IF NOT EXISTS "articles_title_body_fts"
  ON "articles" USING fts("title", "body")
  WITH (tokenizer = 'default')
```

- Tokenizers allowed: `default`, `raw`, `simple`, `whitespace`, `ngram` (compile-time reject otherwise).
- Alone → owns `boutiqueCreateStatements` + `BoutiqueSchema`.
- With `@BoutiqueTable` → also emits `boutiqueFTSCreateStatements` (table macro already embeds FTS into create statements).

### `@VectorIndex(_ column:, metric:, name:)`

**DDL:**

```sql
CREATE INDEX IF NOT EXISTS "documents_embedding_vector"
  ON "documents" USING vector("embedding")
  WITH (metric = 'cosine')
```

- Metrics: `cosine`, `l2`, `dot`, `jaccard`.
- Same alone / stacked pattern as FTS (`boutiqueVectorCreateStatements` when stacked).

### `@MaterializedView(name:as:)`

**DDL:**

```sql
CREATE MATERIALIZED VIEW IF NOT EXISTS "customerTotals" AS <sourceSQL>
```

- Requires non-empty `as:` source SQL.
- Rejects nested `MATERIALIZED VIEW`, `TEMPORARY`, `WITHOUT ROWID` substrings in source (string heuristics).

### Manual (non-macro) descriptors

`Sources/BoutiqueDB/Schema/BoutiqueSchema.swift` also provides:

- `FTSIndexDescriptor` / `VectorIndexDescriptor` / `MaterializedViewDescriptor` / `BoutiqueTableDescriptor`
- Same DDL strings as macros; used by `createFTSIndex` / `createVectorIndex` / `createMaterializedView`.

### What macros **do not** generate

- Ordinary B-tree indexes (`CREATE INDEX` without FTS/vector)
- `UNIQUE`, `CHECK`, `FOREIGN KEY`, composite PKs, `DEFAULT` on columns
- `GENERATED … STORED` (only VIRTUAL path)
- Drop / alter / rename SQL
- Query-layer `@Table` integration is separate (StructuredQueries); `@BoutiqueTable` is Turso DDL–oriented, not a full SQ replacement for `@Table`

---

## 2. BoutiqueSchema / create / migrator flow

### Protocol

```text
BoutiqueSchema
  boutiqueTableName: String          // default pluralized type name
  boutiqueCreateStatements: [String] // default []
```

Macros fill both; manual enums/structs can conform by hand (see `MigrationTests`).

### `db.create(Schema.self)`

`BoutiqueDB.create` (`BoutiqueDB.swift`):

1. Require non-empty `boutiqueCreateStatements`.
2. For each statement: `requireCapability(for:)` then `execute`.
3. Capability gates (from `TursoCapabilities.probe`):
   - `USING fts` → `capabilities.ftsIndex`
   - `USING vector` → `capabilities.vectorIndex`
   - `MATERIALIZED VIEW` → `capabilities.materializedViews`
4. Failures → `BoutiqueError.featureUnavailable` (BD-002 / BD-003 messaging).

Related helpers: `createFTSIndex`, `createVectorIndex`, `createMaterializedView` (descriptor-based, same gates).

### Open + migrate pipeline

`BoutiqueDB.open` (`BoutiqueDB+Open.swift`):

```text
BoutiqueDB.init(...)
  → (optional) BoutiqueMigrator.migrate(plan)
       on schemaErasedForDebug: wipe handled, reopen, re-migrate, optional sync
  → (optional) syncSchema(schemaModels, policy: schemaSync)
  → return db
```

Parameters of interest:

- `migrations: BoutiqueMigrationPlan?`
- `schemaModels: [any BoutiqueSchema.Type]`
- `schemaSync: SchemaSyncPolicy` (default `.off`)

### Migrator model (GRDB / SQLiteData style)

| Piece | Role |
|-------|------|
| `BoutiqueMigration(id) { db in … }` | Single named step |
| `BoutiqueMigrationPlan` + result builder | Ordered list; `eraseDatabaseOnSchemaChange` |
| `BoutiqueMigrator` | Applies pending IDs |
| Tracking table | `boutique_schema_migrations (id TEXT PK, applied_at REAL)` |

**Apply semantics** (`BoutiqueMigrator.migrate`):

1. `CREATE TABLE IF NOT EXISTS boutique_schema_migrations …`
2. If `eraseDatabaseOnSchemaChange`: erase DB files when **applied IDs ∉ plan** (unknown IDs only); throw `schemaErasedForDebug` so open can reopen.
3. Skip IDs already recorded.
4. Run migration body; **only on success** `INSERT` id + timestamp.
5. Body failure → `BoutiqueError.migrationFailed` — id **not** recorded (`failedMigrationIsNotRecorded` test).

**Documented caveats:**

- Append-only named migrations; never edit shipped bodies (`docs/Migrations.md`).
- DDL may auto-commit; body + bookkeeping insert are **not** one atomic SQLite transaction (comment in migrator).
- `eraseDatabaseOnSchemaChange` is DEBUG-oriented; production should leave it `false`.

### Intended app pattern

```swift
// docs/Migrations.md
BoutiqueMigration("v1_create_notes") { db in
  try await db.create(NoteSchema.self)  // @BoutiqueTable / BoutiqueSchema
}
BoutiqueMigration("v2_add_updated_at") { db in
  try await db.ensureColumn(table: "notes", name: "updatedAt", sqlType: "TEXT", default: "…")
}
```

Open with plan; optional `schemaModels` + `.additiveOnly` for DEBUG table create.

---

## 3. `ensureColumn` and schemaSync limits

### `ensureColumn` (`SchemaHelpers.swift`)

```swift
ensureColumn(table:name:sqlType:default:)
```

**Behavior:**

1. `columnExists` via `PRAGMA table_info("table")`.
2. If present → no-op (idempotent; tested).
3. Else `ALTER TABLE … ADD COLUMN … [DEFAULT …]`.

**Also exposed:** `tableExists`, `columnExists`, `dropTableIfExists` (explicit only).

**Limits:**

| Supported | Not supported |
|-----------|----------------|
| Add missing column | Drop / rename column |
| Optional `DEFAULT` SQL fragment (caller-supplied string) | Typed defaults from Swift models |
| String `sqlType` (`TEXT`, `INTEGER`, …) | Infer type from `BoutiqueSchema` / macro model |
| | `NOT NULL` without default (SQLite rules; not guided) |
| | UNIQUE / FK / CHECK on add |
| | Backfill then tighten constraints |
| | Reordering, type changes, generated columns via alter |

### `SchemaSyncPolicy` / `syncSchema` (`SchemaSync.swift`)

| Policy | Effect |
|--------|--------|
| `.off` | No-op (production default) |
| `.additiveOnly` | For each model: `create(schema)` i.e. `CREATE … IF NOT EXISTS` statements |

**Explicitly does not:**

- Add individual columns (docs + code: use `ensureColumn` in a migration)
- Drop / rename tables or indexes
- Diff live `PRAGMA table_info` against model properties

**Capability soft-fail:** if `create` throws `.featureUnavailable`, sync still executes plain table statements that are not FTS/vector/MV (substring filters on SQL). FTS/vector indexes may be skipped while base tables still appear.

### Docs alignment (`docs/Migrations.md`)

| Auto (opt-in) | Never automatic |
|---------------|-----------------|
| `CREATE TABLE/INDEX IF NOT EXISTS` via schema sync | `DROP COLUMN` / `DROP TABLE` |
| `ensureColumn` when **called in a migration** | Silent rename |
| | Type changes |

Philosophy: Room-style complex AutoMigration / Prisma-style generate is **not** the default — corruption risk. Prefer explicit migrations for renames and data backfills.

---

## 4. Gaps for seamless schema evolution

Ordered by impact on “change the model, ship the app” ergonomics:

1. **No model → column diff**  
   Editing an `@BoutiqueTable` struct does not emit or apply `ensureColumn`. Developers must hand-write migration steps with string table/column/type. Drift between Swift model and SQL is easy.

2. **schemaSync is table-granularity only**  
   Missing tables/indexes (IF NOT EXISTS) yes; new properties on existing tables no. DEBUG “just work” story is incomplete for iterative model growth.

3. **No rename / drop / type-change story**  
   By design (safety). No helpers for table rebuild patterns, column rename with data copy, or multi-step backfill. Everything is raw SQL in a migration.

4. **`@GeneratedColumn` is half-wired**  
   Macro body recognizes the attribute, but there is no public `@GeneratedColumn` macro in `Schema/Macros.swift`. Apps cannot cleanly declare generated columns without undocumented attribute names or hand DDL.

5. **`@Column` is external**  
   Primary-key detection depends on StructuredQueries’ `@Column` attribute name. Boutique macros do not own or validate full column metadata (defaults, unique, collations).

6. **DDL expressiveness gaps**  
   No UNIQUE, FK, composite keys, ordinary indexes, STORED generated columns, or DEFAULT in macro expansion. Production schemas still need hand SQL for relational integrity.

7. **Type mapping coarseness**  
   `Date`/`UUID`/custom types → `TEXT` with no encoding policy; no link to StructuredQueries column representations.

8. **Migrator erase policy incomplete vs comment**  
   Comment mentions “order mismatch of common prefix”; implementation only erases on **unknown applied IDs**, not reordered/edited plan prefixes. Edited migration bodies of already-applied IDs are invisible.

9. **Non-atomic migration record**  
   Failed mid-migration after partial DDL can leave schema half-applied without a recorded id (or with applied DDL and failed later statements) — standard SQLite DDL issue, but no multi-statement dry-run or validation.

10. **Capabilities vs ship**  
    FTS/vector/MV require experimental libturso flags. Macros always emit Turso-specific DDL; create fails hard unless capability present. schemaSync softens this only for additive open path.

11. **No schema version / checksum of models**  
    Only migration ID list is tracked. No hash of `boutiqueCreateStatements` to detect model drift vs applied schema.

12. **Stacked FTS/Vector member APIs underused**  
    `boutiqueFTSCreateStatements` / `boutiqueVectorCreateStatements` exist when stacked with `@BoutiqueTable`, but `create` only runs `boutiqueCreateStatements` (which already includes them). Minor API surface noise.

13. **Materialized view evolution**  
    `CREATE MATERIALIZED VIEW IF NOT EXISTS` only — no replace/refresh/drop-and-recreate helper when source SQL changes.

14. **eraseDatabaseOnSchemaChange + CloudKit/sync**  
    File wipe is local-file only; no documented coordination with sync engine / remote schema (out of macros scope but relevant for “seamless” app upgrades).

---

## 5. Recommendations

### Near-term (keep safety, improve DX)

1. **Typed `ensureColumn` from schema metadata**  
   Generate (or reflect) a static column list on `BoutiqueSchema` (`name`, `sqlType`, `optional`, `defaultSQL`) from `@BoutiqueTable`, then:

   ```swift
   try await db.ensureColumns(of: Note.self)  // additive only, per missing column
   ```

   Still migration-invoked or DEBUG-sync-invoked — never silent destructive.

2. **Optional `.additiveOnly` column pass**  
   Extend `syncSchema` (still opt-in) to: create missing tables, then for each model property missing in `PRAGMA table_info`, `ADD COLUMN` with nullable or defaulted types only. Refuse non-null no-default.

3. **Ship `@GeneratedColumn` publicly**  
   Declare the macro (or document attribute) in `Schema/Macros.swift` and test STORED vs VIRTUAL if the engine supports it.

4. **Macro: `DEFAULT`, simple `UNIQUE`, ordinary indexes**  
   Extend column attributes and/or `@Index` for non-FTS indexes so fewer schemas need hand DDL.

5. **Migration helpers for rebuild**  
   Documented recipes + helpers: `renameColumn` via table copy, `rebuildTable`, with explicit opt-in — not auto.

6. **Stronger erase / drift detection (DEBUG)**  
   Implement the promised order/prefix check; optionally compare checksum of registered plan IDs + bodies hash in DEBUG builds.

7. **Expand macro tests**  
   Cover stacked `@BoutiqueTable` + `@FTSIndex` + `@VectorIndex`, PK/`@Column`, optional columns, `withoutRowid`, `@GeneratedColumn`, and diagnostic cases beyond bad tokenizer.

### Medium-term (evolution without Room AutoMigration)

8. **Explicit migration codegen (optional CLI / build plugin)**  
   Diff last committed schema snapshot vs current macro expansion → suggest new `BoutiqueMigration` stubs (`ensureColumn` / create index). Human commits the migration; never auto-apply destructive ops.

9. **Wire StructuredQueries `@Table` and `@BoutiqueTable`**  
   Single model surface for queries + Turso DDL, or documented dual-annotation pattern to avoid two sources of truth.

10. **Capability-aware macro diagnostics**  
    Docs/tooling that warn when models use FTS/vector/MV while vendored lib lacks flags (build-time or `TursoCapabilities` assert in DEBUG open).

### Non-goals (preserve deliberately)

- Silent DROP / rename / type coercion  
- Full Prisma/Room AutoMigration as default  
- Editing applied migration bodies  

These match current product rules in `docs/Migrations.md` and should stay **explicit**.

---

## Quick reference — call graph

```text
@BoutiqueTable / @FTSIndex / @VectorIndex / @MaterializedView
        │
        ▼
BoutiqueSchema.boutiqueCreateStatements  (+ table name)
        │
        ├─► BoutiqueDB.create(_:) ──► requireCapability ──► execute
        │         ▲
        │         │
        │   BoutiqueMigration body
        │         ▲
        │         │
BoutiqueDB.open ──► BoutiqueMigrator.migrate ──► boutique_schema_migrations
        │
        └─► syncSchema(.additiveOnly) ──► create (IF NOT EXISTS only)
                                              └─ featureUnavailable → plain TABLE only

ensureColumn ──► columnExists (PRAGMA) ──► ALTER TABLE ADD COLUMN
```

## Test inventory (macros + migrations)

| Test | File | Asserts |
|------|------|---------|
| STRICT table expansion | `Tests/BoutiqueDBMacrosTests/BoutiqueDBMacrosTests.swift` | DDL + conformance |
| FTS / vector / MV expansion | same | Turso-specific DDL |
| Bad tokenizer diagnostic | same | Compile-time error |
| Open applies migrations idempotently | `Tests/BoutiqueDBTests/MigrationTests.swift` | IDs + columns |
| `ensureColumn` idempotent | same | |
| Only new migrations run | same | |
| Failed migration not recorded | same | |
| schemaSync creates missing table | same | |
| Manual `BoutiqueSchema` + `create` | same | |

---

## Bottom line

BoutiqueDB-Swift macros give **Turso-advantage DDL** (STRICT / WITHOUT ROWID / generated VIRTUAL / FTS / vector / IVM) behind a clean `BoutiqueSchema` + `create` API, with a **GRDB-style append-only migrator** and **intentionally limited** additive sync. Seamless evolution stops at **new tables/indexes (IF NOT EXISTS)** and **hand-invoked `ensureColumn`**. Closing the gap without compromising safety means **schema-aware additive helpers**, **public generated-column API**, **richer column/index macro attributes**, and **optional migration stubs from model diffs** — not automatic destructive migrate.

