# BoutiqueDB — Full-Blown Swift Persistence Framework Specification v2

> Modern, macro-first, full-featured Swift persistence layer for Turso Database with seamless SwiftUI Observation (`AsyncStream`-based), Apple CloudKit sync (`CKSyncEngine`), first-class Turso-only APIs (CDC, MVCC, FTS, Vector, Materialized Views, Encryption, Multi-process WAL, Async Writes), and intelligent macro/schema generation. Integrates the existing `BoutiqueDB-Swift` package (`TursoKit`, `StructuredQueriesTurso`, `TursoCKSync`, `TursoObservation`, `BoutiqueDB`).

---

## 1. Vision & Executive Summary

**Baseline:** `BoutiqueDB-Swift` (`~/Developer/BoutiqueDB-Swift`) is a working local-first framework (local Turso `.db` + `CKSyncEngine` sync, no Turso Cloud). It uses `bindings/c` (`libturso_sqlite3`), `swift-structured-queries` (`0.33.0`), `TursoKit`, `StructuredQueriesTurso`, `TursoCKSync`, `TursoObservation`, and `BoutiqueDB`.

**Target:** A production-grade framework that:
- Replaces `Timer` polling (`pollInterval: 0.5`) with `AsyncStream` over CDC (`SELECT change_id FROM turso_cdc`)
- Adds `async` reads/writes (`DatabaseActor`) and concurrent transactions (`BEGIN CONCURRENT`)
- Introduces `BoutiqueDBMacros` (`swift-syntax`) for `@BoutiqueTable`, `@FTSIndex`, `@VectorIndex`, `@MaterializedView`
- Exposes full Turso feature matrix via typed DSL + macros
- Keeps CloudKit (`TursoCKSyncEngine`) as default sync, with `SyncAdapter` protocol for Turso Cloud
- Integrates `swift-dependencies`, `swift-perception`, `swift-testing`, `swift-snapshot-testing`

**References:**
- Design docs: `BoutiqueDB-Swift/BoutiqueDB-Design.md`, `BoutiqueDB-TursoFeatures.md`
- OpenSpec change: `openspec/changes/archive/2026-07-22-boutiquedb-v2/` (proposal, design, specs, tasks)
- Open issues: `BoutiqueDB-Issues.md`
- Source: `Sources/BoutiqueDB/`, `Sources/TursoKit/`, `Sources/StructuredQueriesTurso/`, `Sources/TursoObservation/`, `Sources/TursoCKSync/`

---

## 2. Architecture (Current + Enhanced)

```
App Process (SwiftUI / @Observable)
├── BoutiqueDB (@MainActor, Sendable) — container, async read/write
│   ├── TursoKit (TursoDatabase, TursoConnection)
│   ├── StructuredQueriesTurso (Statement+Turso, Table+Turso)
│   └── BoutiqueDBConnection (transaction handle)
├── TursoObservation (AsyncStream CDC, TursoStore, TursoQueryBox)
│   └── Replaces Timer polling with cooperative await
├── TursoCKSync (TursoCKSyncEngine, CKSyncEngine delegate)
│   ├── SyncAdapter protocol (CloudKitSyncAdapter default)
│   └── StateSerialization durability, conflict policies
├── BoutiqueDBMacros (.macro target, swift-syntax)
│   ├── @BoutiqueTable (extends @Table)
│   ├── @FTSIndex, @VectorIndex, @MaterializedView
│   └── @LiveQuery / @LiveQueryOne (property wrapper macros)
└── SwiftUI Integration (@LiveQuery, @LiveQueryOne, Observation/Perception)
```

**Binding:** `bindings/c` (`libturso_sqlite3`) — stable SQLite3 C API. `sdk-kit` (`turso_sync.h`, `turso.h`) is the future migration path for native async/encryption/custom-index features (`Phase 3`).

---

## 3. Modern Async & Observation Design

### Async Writes (`DatabaseActor`)
- `read` / `write` are `async throws`. Closure runs on background cooperative pool (`IOResult`-compatible).
- `BoutiqueDB` stays `@MainActor`; inner `DatabaseActor` handles concurrent I/O.
- `LiveQuery` refresh uses `AsyncStream` event (`store.generation` change) instead of `Task.sleep(for: .milliseconds(100))`.

### CDC → AsyncStream Observation
```swift
public final class TursoStore: @unchecked Sendable {
  public private(set) var generation: UInt64 = 0
  public let changes: AsyncStream<ChangeEvent>
  private var continuation: AsyncStream<ChangeEvent>.Continuation!

  public init(connection: TursoConnection) {
    var cont: AsyncStream<ChangeEvent>.Continuation!
    (changes, cont) = AsyncStream.makeStream(of: ChangeEvent.self)
    self.continuation = cont
    // Background task listens to CDC or commit hook
  }

  public func invalidate() {
    generation += 1
    continuation.yield(.generation(generation))
  }
}
```

### SwiftUI Observation (`Observation` + `Perception`)
- Default: native `Observation` (`iOS 17+`, `macOS 14+`).
- Backport: `swift-perception` (`Perception` library) for `iOS 15` / `macOS 12`.
- `LiveQuery` uses `withObservationTracking` + `AsyncStream` subscription (not `Timer`).

---

## 4. Full Feature Matrix (Modern API Shape)

### CDC / Live Queries (Done + Modernized)
- `LiveQuery<Element>` / `LiveQueryOne<Element>` property wrappers (`Observation`-friendly).
- `AsyncStream` backing (`store.changes`), not `Timer`.
- `refresh()` / `forceRefresh()` / `refreshIfNeeded()` backed by stream events.

### MVCC (`BEGIN CONCURRENT`)
- `BoutiqueDB.beginConcurrent()` / `.commitConcurrent()` / `.writeConcurrent()`.
- Separate connection from CDC/sync (CDC ⊥ MVCC).
- Auto-retry with exponential backoff (`SQLITE_BUSY`).
- Config guard: `mvcc == true && cdc == true` throws `BoutiqueError.cdcMutuallyExclusiveWithMVCC`.

### Async Writes (Modern Async/Concurrency)
- `read<T>` / `write<T>` / `transaction<T>` are `async throws`.
- `BoutiqueDBConnection` supports async closures (`inout` for writes).
- Background `DatabaseActor` for cooperative I/O.

### Full-Text Search (`FTS` — Tantivy)
- `@FTSIndex` macro (`swift-syntax`): generates `CREATE INDEX ... USING fts` DDL; validates tokenizer / column types.
- DSL: `Note.where { $0.title.match("swift") }`, `.score("swift") > 0`, `.highlight(query: "swift")`.
- `LiveQuery` overload: `@LiveQuery(search: Note.titleSearch, "swift") var results: [Note]`.
- Note: `snippet()` missing (Tantivy limitation); `MATCH` syntax supported.

### Vector Search
- `@VectorIndex` macro: generates `CREATE INDEX ... USING vector`; validates metric (`.cosine`, `.l2`, `.dot`, `.jaccard`).
- Type: `Vector32` (`Codable`, `Sendable`, `RawRepresentable`).
- DSL: `Document.where { vectorDistanceCos($0.embedding, query) < 0.2 }`.

### Materialized Views (`IVM`)
- `@MaterializedView` macro: generates `CREATE MATERIALIZED VIEW ... AS <source SQL>`; validates source expression (no nested views, limited functions).
- Runtime: `createMaterializedView<T: MaterializedView>(_ view: T.Type) async throws`.
- Note: `IVM` is experimental (`--experimental-views` not enabled by `turso_enable_experimental()`); requires `sdk-kit` migration or new C setter for full production.

### Generated Columns / WITHOUT ROWID / STRICT
- Extend `@Table` / `@Column` macro parameters (upstream `swift-structured-queries` or `@BoutiqueTable` wrapper):
  - `@Table(withoutRowid: true, strict: true)`
  - `@Column(generated: .virtual, expression: "lower(title)")`
- Note: `turso_enable_experimental()` turns on generated columns and WITHOUT ROWID; `STRICT` is normal SQLite feature.

### Encryption at Rest
- `BoutiqueDB(url: ..., encryption: .aegis256(key: ...))`.
- Note: `bindings/c` does not expose `sqlite3_open_v2_with_key`. Must add C setter or migrate `TursoKit` to `sdk-kit` native API (`--experimental-encryption`).

### Multi-Process WAL (`multiProcessWAL`)
- `BoutiqueDB(url: ..., multiProcess: Bool)`.
- `.tshm` sidecar file management for cross-process WAL readers/writers.
- Low priority unless sharing DB with extension / widget.

### Extensions (UUID, regexp, time, CSV, percentile, fuzzy, ipaddr)
- Typed scalar helpers: `UUID.v4()` (`uuid4()`), `String.regexp(...)` (`regexp_like(...)`), `Date.timeInterval(...)` (`dur_` functions), `percentile(...)`, `fuzzy(...)`, `ipaddr(...)`.
- Added to `StructuredQueriesTurso` DSL layer or `BoutiqueDB` extension.

---

## 5. Macro & Schema Generation Strategy

### 5.1 Existing (`swift-structured-queries` 0.33.0)
- `@Table`, `@Column(primaryKey:)`, `#sql` macro, query DSL (`select`, `where`, `join`, `group`, `order`, `limit`, `insert`, `update`, `delete`, `upsert`).

### 5.2 New (`BoutiqueDBMacros` — `.macro` target)

**Macro Package Design:**
```swift
.macro(
  name: "BoutiqueDBMacros",
  dependencies: [
    .product(name: "SwiftSyntaxMacros", package: "swift-syntax"),
    .product(name: "SwiftCompilerPlugin", package: "swift-syntax"),
  ]
)
```

**Macro Definitions:**
- `@BoutiqueTable` (wraps/extends `@Table`): adds `withoutRowid`, `strict`, `generated` column options; generates `TableDescriptor`, `Codable` mapping, DSL helpers (`Task.all`, `Task.filter(...)`).
- `@FTSIndex`: `CREATE INDEX ... USING fts` DDL; `FTSConfig` descriptor.
- `@VectorIndex`: `CREATE INDEX ... USING vector` DDL; `VectorIndex` descriptor (`embedding`, `.cosine`).
- `@MaterializedView`: `CREATE MATERIALIZED VIEW ... AS <source>`; validates source expression.
- `@LiveQuery` / `@LiveQueryOne`: `PropertyWrapper` macro that generates `ObservationRegistrar` / `PerceptionRegistrar` and `AsyncStream` subscription (replaces `Timer` polling).

**Macro Validation:**
- `@FTSIndex`: validates `tokenizer` (`.default`, `.raw`, `.simple`, `.whitespace`, `.ngram`) and column types (`String`, `Text`) against `Table` schema.
- `@VectorIndex`: validates metric (`.cosine`, `.l2`, `.dot`, `.jaccard`) and column type (`Vector32` / `Vector32Sparse`).
- `@MaterializedView`: validates source expression has no nested `MaterializedView` references and uses supported SQL functions (no `TEMPORARY`, no `WITHOUT ROWID` in source).

**Reference:** `BoutiqueDB-TursoFeatures.md` §4-§7, `BoutiqueDB-Design.md` §4.

---

## 6. Seamless Query Builder — StructuredQueries + BoutiqueDB Integration

This section defines the **developer experience** for building, composing, observing, and synchronizing queries with zero friction between `StructuredQueriesTurso`, `BoutiqueDB`, `TursoObservation`, and SwiftUI.

### 6.1 Design Principles

1. **One DSL, three layers:** Schema (`@Table`), Query (`SelectOf<T>`), Execution (`BoutiqueDB.read` / `.write`).
2. **Type-safe interpolation:** `#sql` macro (`swift-structured-queries`) for raw SQL fragments; `StructuredQueriesTurso` drivers execute safely against `TursoConnection`.
3. **Observation-first:** Every query result is observable via `LiveQuery` / `LiveQueryOne` backed by `AsyncStream` (not polling).
4. **Turso-aware DSL:** `Statement+Turso.swift` and `Table+Turso.swift` extend StructuredQueries with `execute`, `fetchAll`, `fetchOne`, `fetchCount` that work directly on `TursoConnection`.
5. **Macro-backed boilerplate reduction:** `@BoutiqueTable` generates descriptor, `Codable`, DSL helpers; `@FTSIndex` / `@VectorIndex` generate DDL; `@LiveQuery` generates observer registration.

### 6.2 Basic Query Flow (Seamless)

```swift
import BoutiqueDB
import StructuredQueries
import StructuredQueriesTurso

@Table
struct Note: Identifiable, Sendable {
  @Column(primaryKey: true) let id: UUID
  var title: String
  var body: String
  var updatedAt: Date = Date()
}

@MainActor
final class NotesModel: Observable {
  let db = try! BoutiqueDB(url: BoutiqueDB.applicationSupportURL())

  // Observable array backed by AsyncStream CDC events
  @LiveQuery(db) { Note.all.order { $0.updatedAt.desc() }.asSelect() }
  var notes: [Note] = []

  // Observable single row
  @LiveQueryOne(db) { Note.where { $0.id.eq(noteID) }.asSelect() }
  var note: Note?

  func addNote(title: String) async throws {
    try await db.write { conn in
      try Note.insert { Note(id: UUID(), title: title) }.execute(conn.connection)
    }
    // AsyncStream event triggers LiveQuery refresh automatically
  }

  func searchNotes(query: String) async throws -> [Note] {
    try await db.read { conn in
      // Type-safe FTS DSL (generated by @FTSIndex macro)
      try Note.where { $0.title.match(query) }
        .order { $0.title.asc() }
        .fetchAll(conn.connection)
    }
  }

  func nearestDocuments(queryEmbedding: Vector32) async throws -> [Document] {
    try await db.read { conn in
      // Type-safe vector DSL (generated by @VectorIndex macro)
      try Document.where { vectorDistanceCos($0.embedding, queryEmbedding) < 0.2 }
        .order { vectorDistanceCos($0.embedding, queryEmbedding) }
        .limit(10)
        .fetchAll(conn.connection)
    }
  }
}
```

### 6.3 Query Composition (Chained DSL)

All query builder methods (`select`, `where`, `order`, `group`, `join`, `limit`) take trailing closures describing joined tables (`{ remindersLists, reminders in ... }`). The DSL is fully composable:

```swift
// Multi-table join with aggregate + filter + order + limit
try await db.read { conn in
  try Note
    .join(RemindersList.all) { $0.remindersListID.eq($1.id) }
    .where { notes, lists in notes.title.contains("swift") }
    .group(by: \Note.remindersListID)
    .select {
      Aggregate(
        remindersListID: $0.remindersListID,
        count: count,
        totalLength: sum(\.$0.body.length())
      )
    }
    .order { $0.remindersListID.asc() }
    .limit(50)
    .fetchAll(conn.connection)
}
```

### 6.4 Raw SQL Escape Hatch (`#sql` + `execute`)

When the DSL cannot express a feature (e.g. experimental PRAGMA, custom `WITHOUT ROWID`, `STRICT` table definition, complex CTE), use `#sql` safe interpolation:

```swift
// Migration / schema drift
try await db.execute(#sql("""
  CREATE TABLE IF NOT EXISTS notes (
    id TEXT PRIMARY KEY NOT NULL,
    title TEXT NOT NULL,
    body TEXT NOT NULL,
    updatedAt TEXT DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now'))
  ) WITHOUT ROWID STRICT
"""))

// Raw query with safe interpolation
try await db.read { conn in
  try Note.where(#sql("lower(title) LIKE ?", [.text("%swift%")])).fetchAll(conn.connection)
}
```

### 6.5 Query Builder Integration with Turso-Only Features

**FTS DSL Extension (`Statement+Turso.swift` + `@FTSIndex` macro):**
```swift
// Macro generates: Note.titleSearch (FTSConfig descriptor)
// DSL helpers added to StructuredQueriesTurso
try Note.where { $0.title.match("swift") }.fetchAll(conn.connection)
try Note.select { FTSHighlight(title: $0.title, before: "<b>", after: "</b>", query: "swift") }.fetchAll(conn.connection)
```

**Vector DSL Extension (`Statement+Turso.swift` + `@VectorIndex` macro):**
```swift
try Document.where { vectorDistanceCos($0.embedding, queryEmbedding) < 0.2 }
  .order { vectorDistanceCos($0.embedding, queryEmbedding) }
  .limit(10)
  .fetchAll(conn.connection)
```

**Materialized View DSL (`Statement+Turso.swift` + `@MaterializedView` macro):**
```swift
// Macro defines: CustomerTotals.source (QueryExpression)
// Runtime query treats materialized view as Table
try await db.read { conn in
  try CustomerTotals.all.fetchAll(conn.connection)
}
```

**Concurrent Transaction DSL (`ConcurrentConnection`):**
```swift
try await db.writeConcurrent { conn in
  try Note.insert { Note(id: UUID(), title: "Concurrent") }.execute(conn.connection)
  try conn.commitConcurrent()
}
```

### 6.6 Migration & Schema Drift Queries

```swift
try await db.migrate(using: BoutiqueMigrationPlan {
  BoutiqueMigration(1) { db in
    try await db.create(Note.self)
  }
  BoutiqueMigration(2) { db in
    try await db.alter(Note.self) { table in
      table.add(\.updatedAt, type: .date, default: Date())
    }
  }
  BoutiqueMigration(3) { db in
    // FTS index created by macro at build time, but DDL applied at migration time
    try await db.execute("CREATE INDEX IF NOT EXISTS notes_title_fts ON notes USING fts (title)")
  }
})
```

### 6.7 Query Performance & Type Safety

- **Query decoding:** `TursoQueryDecoder.swift` maps `TursoValue` rows to `Codable` models (`Table.QueryOutput`).
- **Type-safe bindings:** `TursoValue` enum (`.text`, `.integer`, `.double`, `.blob`, `.null`) ensures SQL → Swift mapping is safe.
- **Sendable conformance:** `Table` models (`Note`, `Document`) must conform to `Sendable` for concurrent reads (`@Sendable () -> SelectOf<Element>` closures in `LiveQuery`).
- **Performance:** `read` uses read-only connection; `write` uses write connection; concurrent writes use separate `ConcurrentConnection`. `AsyncStream` event delivery is faster than `Timer` polling.

---

*This section (§6) defines the seamless developer experience: one DSL (`StructuredQueriesTurso`), one container (`BoutiqueDB`), one observation mechanism (`AsyncStream` + `LiveQuery`), and full Turso feature access through typed APIs and macros. It replaces the previous polling-based design with modern Swift concurrency patterns while maintaining full compatibility with `swift-structured-queries` and the existing `BoutiqueDB-Swift` package.*


### 6.8 Modern Observation Design (`AsyncStream` + `LiveQuery`)

**Current (`TursoObservation`):**
- `TursoStore` (`@Observable`): `pollInterval: 0.5`, `Timer` polling (`poll()` every 100 ms).
- `LiveQuery` (`@Observable`): `Timer` polling (`pollInterval`), `withObservationTracking` on `store.generation`.
- `LiveQueryOne`: same mechanism.
- `refresh()` / `forceRefresh()` / `poll()` — synchronous (blocks `MainActor`).

**Modern Target Design:**
- Replace `Timer` with `AsyncStream` over CDC (`SELECT change_id FROM turso_cdc` cooperative await).
- `LiveQuery` uses `withObservationTracking` + `AsyncStream` subscription (`Task` loop, not `Task.sleep`).
- `LiveQueryOne` uses same mechanism.
- `TursoStore` exposes `changes: AsyncStream<ChangeEvent>`; `invalidate()` yields event.
- `Perception` backport (`swift-perception`): `PerceptionRegistrar` for `iOS 15` / `macOS 12`.
- `Observation` default (`iOS 17+` / `macOS 14+`).

**Reference:** `pfw-observable-models` skill (`@ObservationIgnored`, `Equatable` identity, async methods), `axiom-swiftui` skill (`Observation` debugging, `LiveQuery` patterns), `BoutiqueDB-TursoFeatures.md` §1, `LiveQuery.swift`, `TursoInvalidation.swift`.

### 6.9 AsyncStream-Based Store & LiveQuery (Modern `PropertyWrapper`)

```swift
@propertyWrapper
public final class LiveQuery<Element: Table & Sendable>: Observable
where Element == Element.QueryOutput {
  public private(set) var wrappedValue: [Element]
  private let db: BoutiqueDB
  private let query: @Sendable () -> SelectOf<Element>
  private var observationTask: Task<Void, Never>?

  public init(
    wrappedValue: [Element] = [],
    _ db: BoutiqueDB,
    query: @escaping @Sendable () -> SelectOf<Element> = { Element.all.asSelect() }
  ) {
    self.wrappedValue = wrappedValue
    self.db = db
    self.query = query
    refresh()
    observeAsyncStream()
  }

  private func observeAsyncStream() {
    observationTask = Task { [weak self] in
      guard let self else { return }
      for await event in await self.db.store.changes {
        await MainActor.run {
          self.refresh()
        }
      }
    }
  }

  public func refresh() {
    do {
      wrappedValue = try db.read { try query().fetchAll($0.connection) }
    } catch {
      // IssueReporting / loadError surface
    }
  }

  public func forceRefresh() {
    // Immediate refresh, skip stream event
  }
}
```

**Perception Backport (`swift-perception`):**
- `Perception` library (`PerceptionRegistrar`) for `iOS 15` / `macOS 12`.
- `TursoStore` exposes `withPerceptionTracking` interface; `LiveQuery` uses it when `Observation` unavailable.
- `Perception` requires `Perceptible` protocol conformance (similar to `Observable`).

**AsyncStream-based Store:**
```swift
public final class TursoStore: @unchecked Sendable {
  public private(set) var generation: UInt64 = 0
  public let changes: AsyncStream<ChangeEvent>
  private var continuation: AsyncStream<ChangeEvent>.Continuation!

  public init(connection: TursoConnection) {
    var cont: AsyncStream<ChangeEvent>.Continuation!
    (changes, cont) = AsyncStream.makeStream(of: ChangeEvent.self)
    self.continuation = cont
    // Background cooperative await over CDC or commit hook
  }

  public func invalidate() {
    generation += 1
    continuation.yield(.generation(generation))
  }
}
```

**LiveQuery (Modern `PropertyWrapper`):**
```swift
@propertyWrapper
public final class LiveQuery<Element: Table & Sendable>: Observable
where Element == Element.QueryOutput {
  public private(set) var wrappedValue: [Element]
  private let db: BoutiqueDB
  private let query: @Sendable () -> SelectOf<Element>
  private var observationTask: Task<Void, Never>?

  public init(
    wrappedValue: [Element] = [],
    _ db: BoutiqueDB,
    query: @escaping @Sendable () -> SelectOf<Element> = { Element.all.asSelect() }
  ) {
    self.wrappedValue = wrappedValue
    self.db = db
    self.query = query
    refresh()
    observeAsyncStream()
  }

  private func observeAsyncStream() {
    observationTask = Task { [weak self] in
      guard let self else { return }
      for await event in await self.db.store.changes {
        await MainActor.run {
          self.refresh()
        }
      }
    }
  }

  public func refresh() {
    do {
      wrappedValue = try db.read { try query().fetchAll($0.connection) }
    } catch {
      // IssueReporting / loadError surface
    }
  }

  public func forceRefresh() {
    // Immediate refresh, skip stream event
  }
}
```

**Perception Backport (`swift-perception`):**
- `Perception` library (`PerceptionRegistrar`) for `iOS 15` / `macOS 12`.
- `TursoStore` exposes `withPerceptionTracking` interface; `LiveQuery` uses it when `Observation` unavailable.
- `Perception` requires `Perceptible` protocol conformance (similar to `Observable`).

**Reference:** `pfw-observable-models` skill (`@ObservationIgnored`, `Equatable` identity, async methods), `axiom-swiftui` skill (`Observation` debugging, `LiveQuery` patterns), `BoutiqueDB-TursoFeatures.md` §1 (CDC / observation), `LiveQuery.swift`.

---

## 7. Sync Deep Dive (`TursoCKSync` / `CKSyncEngine`)

### 7.1 Current Architecture (`TursoCKSyncEngine`)

- `CKSyncEngine` initialized with `database: container.privateCloudDatabase`, `stateSerialization: state`, `delegate: self`.
- Custom zone (`zoneID`) saved via `pendingDatabaseChanges` (`.saveZone`).
- Outbound (`drainCDC()`): reads `turso_cdc`, maps to `CKRecord.ID` (`table:rowPK`), builds `PendingRecordZoneChange` (`.saveRecord` / `.deleteRecord`), enqueues.
- Inbound (`fetchedRecordZoneChanges`): applies modifications/deletions with `connection.isSynchronizing = true` (skips CDC echo suppression via `advanceCDCCursorPastEcho`).
- Conflict policies: `.serverWins`, `.clientWins`, `.lastWriterWins(let field:)`.
- Account change (`signIn` / `signOut` / `switchAccounts`): `wipeAndRebootstrap()` or re-upload.

**Reference:** `openspec/changes/archive/2026-07-22-boutiquedb-v2/specs/cloudkit-sync/spec.md` (requirements and scenarios), `openspec/changes/archive/2026-07-22-boutiquedb-v2/design.md` (decisions and risks), `TursoCKSyncEngine.swift`.

### 7.2 Modern Enhancements

**Sync Adapter Protocol (`SyncAdapter`):**
```swift
public protocol SyncAdapter: Sendable {
  func start() async throws
  func stop() async
  func syncStatus() -> AsyncStream<SyncStatus>
  func drainLocalChanges() async throws -> Int
  func applyRemoteChanges(_ changes: [RemoteChange]) async throws
}

public struct CloudKitSyncAdapter: SyncAdapter {
  // Existing TursoCKSyncEngine
}

public struct TursoCloudSyncAdapter: SyncAdapter {
  // Future Turso Cloud SDK (`sdk-kit` / `turso_sync.h`)
}
```

**State Serialization Durability:**
- Persist `stateSerialization` with applied mutations; crash-safe writes.
- `AsyncStream` for `SyncStatus` updates (not polling).

**Batching & Performance:**
- `maxBatchSize` (≤ 250 records/request) enforced at `nextRecordZoneChangeBatch`.
- Handle `zoneNotFound` retry (`retryZones` / `retryRecords`).
- Optimize `drainCDC()` for large batches (`limit: 500`, cooperative await).

**Conflict Resolution:**
- `.serverWins`: apply server record directly.
- `.clientWins`: apply server record (keep system fields) + re-pend local save.
- `.lastWriterWins(let field:)`: compare `comparableStamp` (timestamp / version); if local > server, re-pend local; else apply server.

---

## 8. Package & Dependency Updates

### `Package.swift` Updates (`BoutiqueDB-Swift`)

```swift
dependencies: [
  .package(url: "https://github.com/pointfreeco/swift-structured-queries", from: "0.33.0"),
  .package(url: "https://github.com/pointfreeco/swift-dependencies", from: "1.0.0"),        // ADD
  .package(url: "https://github.com/pointfreeco/swift-perception", from: "1.0.0"),         // ADD
  .package(url: "https://github.com/apple/swift-syntax", from: "600.0.0"),                // ADD (macros)
  .package(url: "https://github.com/pointfreeco/swift-testing", from: "1.0.0"),           // ADD (tests)
  .package(url: "https://github.com/pointfreeco/swift-snapshot-testing", from: "2.0.0"), // ADD
],
```

**New `.macro` Target (`BoutiqueDBMacros`):**
```swift
.macro(
  name: "BoutiqueDBMacros",
  dependencies: [
    .product(name: "SwiftSyntaxMacros", package: "swift-syntax"),
    .product(name: "SwiftCompilerPlugin", package: "swift-syntax"),
  ]
)
```

**New `Dependencies` Integration (`BoutiqueDB`):**
- Create `BoutiqueDBDependencyKey` (`defaultDatabase`).
- Update `BoutiqueDB.init` to accept `@Dependency(\.defaultDatabase)` or `prepareDependencies` setup.

---

## 9. Production Roadmap (`Phase 1` → `Phase 5`)

### Phase 1 — Foundation & Modern Observation (Week 1-2)
- Dependency setup (`swift-dependencies`, `swift-perception`).
- Async writes (`DatabaseActor`).
- `LiveQuery` modernization (`AsyncStream`, not `Timer`).
- Migration helpers (`BoutiqueMigrationPlan`, `BoutiqueMigration`).
- SwiftUI integration test (`LiveQuery` < 1 s refresh).

**Deliverable:** `swift build` + `swift test` passing; `LiveQuery` refresh < 500 ms; `Migration` helpers working.

### Phase 2 — Macro Layer & Schema Generation (Week 3-4)
- `BoutiqueDBMacros` package (`.macro` target, `swift-syntax`).
- `@BoutiqueTable`, `@FTSIndex`, `@VectorIndex`, `@MaterializedView` macros.
- Schema drift / snapshot tests (`swift-snapshot-testing`).

**Deliverable:** Macros generate valid SQL; snapshot tests pass; DDL validated by macro.

### Phase 3 — Full Turso Features (Week 5-8)
- MVCC (`BEGIN CONCURRENT`, concurrent transaction API, retry/backoff).
- Encryption (`init(url:encryption:)` — C setter or `sdk-kit` migration).
- Multi-process WAL (`multiProcess: Bool`).
- Extensions (typed scalar DSL).
- Custom types (`RawRepresentable` + `QueryBindable`).
- Generated columns / WITHOUT ROWID / STRICT (macro parameter extension).

**Deliverable:** All Turso-exclusive features accessible via typed API; MVCC concurrent writes safe; encryption / multi-process APIs available.

### Phase 4 — CloudKit Sync Hardening & Production (Week 9-12)
- `SyncAdapter` protocol; `CloudKitSyncAdapter` (default); `TursoCloudSyncAdapter` (future).
- Batching optimization (`limit: 500`, `maxBatchSize` ≤ 250); `zoneNotFound` retry.
- Conflict resolution (`.lastWriterWins` timestamp comparison).
- Account change (`wipeAndRebootstrap` crash-safe).
- Multi-table sync (`syncedTables` array).
- Live two-simulator test (`enablesCloudKit: true` + `false`).
- Performance benchmarks (`LiveQuery`, `drainCDC`, `writeConcurrent`, `Migration`).

**Deliverable:** Production sync (two-device round-trip); conflict resolution stable; account change safe; benchmarks documented.

### Phase 5 — Polish & Release (Week 13-16)
- Documentation (`README.md`, `docs/`, quick-start, migration guide).
- Sample app (`SampleApp/` — SwiftUI + `LiveQuery` + sync + FTS + vector demo).
- CI (`.github/workflows/` — `swift build`, `swift test`, simulator tests).
- SPI (`.xcframework` binary target or `systemLibrary` for `libturso_sqlite3` — replace `.unsafeFlags`).
- App Store / Mac Distribution (`macos-release` skill pipeline: archive, notarize, DMG, Sparkle).
- Public release (`v1.0.0` tag, GitHub release notes).

**Deliverable:** `v1.0.0` release; SPI ready; CI green; sample app working.

---

## 10. External Packages & Tools (Integration List)

### Swift / iOS Packages (SPM Updates)

| Package | Version | Products Added | Target Dependencies | Status |
|---------|---------|----------------|---------------------|--------|
| `swift-structured-queries` | `0.33.0` | `StructuredQueries`, `StructuredQueriesCore` | `StructuredQueriesTurso`, `BoutiqueDB` | ✅ Done |
| `swift-dependencies` | `1.0.0` | `Dependencies` | `BoutiqueDB` | ⚠️ Add |
| `swift-perception` | `1.0.0` | `Perception` | `BoutiqueDB` | ⚠️ Add |
| `swift-syntax` | `600.0.0` | `SwiftSyntaxMacros`, `SwiftCompilerPlugin` | `BoutiqueDBMacros` (new `.macro`) | ⚠️ Add |
| `swift-testing` | `1.0.0` | `Testing` | `BoutiqueDBTests` | ⚠️ Add |
| `swift-snapshot-testing` | `2.0.0` | `SnapshotTesting` | `BoutiqueDBTests` | ⚠️ Add |

### Rust / Build Tools (`AGENTS.md`)

| Tool | Command | Purpose | Status |
|------|---------|---------|--------|
| `cargo` | `cargo build`, `cargo test`, `cargo fmt`, `cargo clippy` | Turso engine (`bindings/c`, `core/`, `cli/`, `extensions/`) | ✅ Used |
| `swift` / `swiftc` | `swift build`, `swift test` | Swift package (`BoutiqueDB-Swift`) | ✅ Used |
| `xcodebuild` | `xcodebuild archive` / `xcodebuild -scheme BoutiqueDB` | Archive for iOS/macOS | ⚠️ Add to CI |
| `simctl` / `xcrun` | `simctl boot` / `simctl list devices` | iOS simulator testing | ⚠️ Add to CI |

---

## 11. References (Source & Design Files)

**Design Documents:**
- `<ref_file file="/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/BoutiqueDB-Design.md" />` (Architecture, macro/model layer, observation, sync, open questions)
- `<ref_file file="/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/BoutiqueDB-TursoFeatures.md" />` (Feature matrix, macro priorities, C-binding limitations)
- `<ref_file file="/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/README.md" />` (Quick start, modules, build instructions)

**Source Index (`BoutiqueDB-Swift`):**
- `Sources/BoutiqueDB/BoutiqueDB.swift` — Container (`read`, `write`, `execute`, `fetchAll`, `fetchOne`)
- `Sources/BoutiqueDB/BoutiqueDBConnection.swift` — Transaction handle (`execute`, `query`, `fetchAll`, `fetchOne`, `fetchCount`)
- `Sources/BoutiqueDB/LiveQuery.swift` — `LiveQuery` property wrapper (`Timer` polling)
- `Sources/BoutiqueDB/LiveQueryOne.swift` — `LiveQueryOne` property wrapper
- `Sources/TursoKit/TursoDatabase.swift` — Database open (`sqlite3_open_v2`), CDC enable
- `Sources/TursoKit/TursoConnection.swift` — Read/write closures, prepare/bind/step
- `Sources/StructuredQueriesTurso/Statement+Turso.swift` — `execute` / `fetchAll` / `fetchOne`
- `Sources/StructuredQueriesTurso/Table+Turso.swift` — `fetchAll` / `fetchOne` / `fetchCount`
- `Sources/TursoObservation/TursoInvalidation.swift` — `TursoStore` (`pollInterval`), `TursoQueryBox`
- `Sources/TursoCKSync/TursoCKSyncEngine.swift` — `CKSyncEngine` delegate (full pipeline)
- `Sources/TursoCKSync/RecordMapper.swift` — `CKRecord` ↔ Turso row mapping
- `Sources/TursoCKSync/RowSQL.swift` — SQL helpers for sync
- `Sources/TursoCKSync/SyncMetadataStore.swift` — Metadata persistence

**Repo Guidelines & Build:**
- `<ref_file file="/Users/tuliopinheirocunha/Developer/BoutiqueDB/AGENTS.md" />` (Rust build instructions, benchmark naming, CI note)
- `<ref_file file="/Users/tuliopinheirocunha/Developer/BoutiqueDB/BoutiqueDB-Framework-Spec.md" />` (Original v1 spec — superseded by this v2 spec)
- `openspec/changes/archive/2026-07-22-boutiquedb-v2/` (OpenSpec change: proposal, design, specs, tasks)
- `BoutiqueDB-Issues.md` (Open design & implementation issues / blockers)
- `.github/pull_request_template.md` (Contribution process)

**External References:**
- Apple `CKSyncEngine` docs (`developer.apple.com/documentation/cloudkit/cksyncengine`)
- Apple `CloudKit` docs (`developer.apple.com/documentation/cloudkit`)
- Point-Free `swift-structured-queries` (`github.com/pointfreeco/swift-structured-queries`)
- Point-Free `swift-dependencies` (`github.com/pointfreeco/swift-dependencies`)
- Point-Free `swift-observation` / `swift-perception` (`github.com/pointfreeco/swift-observation`, `swift-perception`)
- Turso `bindings/c` docs (`bindings/c/README.md`)
- Turso `docs/manual.md` (Full database manual)
- Turso `docs/agent-guides/` (Testing, debugging, async-io, MVCC guides)
- Turso `cli/manuals/` (CDC, MVCC, vector, FTS, materialized-views, encryption)

---

*This specification (v2) supersedes the previous framework spec (`BoutiqueDB-Framework-Spec.md`). It integrates the full `BoutiqueDB-Swift` architecture, modern Swift concurrency (`async`/`await`, `Sendable`, `Observation`/`Perception`, `AsyncStream`), a complete macro layer (`BoutiqueDBMacros`), full Turso feature exposure (CDC, MVCC, FTS, Vector, Materialized Views, Encryption, Multi-process WAL, Async Writes), a clear production roadmap (`Phase 1` → `Phase 5`), and all packages, tools, and external references needed for implementation. It is intended to guide the framework from its current working state to a full production release (`v1.0.0`).*
