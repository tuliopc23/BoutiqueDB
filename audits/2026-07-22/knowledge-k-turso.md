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
