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
