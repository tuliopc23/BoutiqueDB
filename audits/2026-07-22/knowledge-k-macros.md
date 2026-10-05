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
