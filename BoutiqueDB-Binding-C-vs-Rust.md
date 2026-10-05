# BoutiqueDB binding strategy: C ABI vs Rust direct (FFI)

**Date:** 2026-07-22 (enhanced)  
**Product:** Apple-only framework — SQLiteData-style DX, Turso engine, CloudKit, SwiftUI  
**Not about:** multi-language SDKs or “Turso for the world”

---

## 0. TL;DR

| Path | What it is | BoutiqueDB today |
|------|------------|------------------|
| **C ABI** | Swift → sqlite3-shaped C API → `turso_core` | **Current** (`TursoKit` + `libturso_sqlite3`) |
| **Rust direct** | Swift → FFI/UniFFI/`extern "C"` Turso API → Builder / `DatabaseOpts` → `turso_core` | **Not built** |

Both run the **same Rust engine**. They differ in **how you open and configure** the DB and how much Turso power you can turn on.

**For next app + Live CloudKit:** stay on **C**.  
**For full Turso knobs without endless C patches:** plan **C+ open opts** or **Rust open path**.

---

## 1. Questions this doc answers

1. C binds vs Rust direct — comparison  
2. Features hard/unavailable on C as wired  
3. Advantages given up on C  
4. Advantages of a Rust binding  
5. Are we really different from stock SQLite on C?  
6. **Possibility assessment: what we can do if C vs if Rust** (Apple app scenarios)  
7. In-app concurrency (import/markdown) vs multi-process (extensions)

Evidence:

- `bindings/c` — `sqlite3_open*`, `turso_enable_experimental`  
- `bindings/rust` — `Builder` experimental flags  
- `COMPAT.md`, `BoutiqueDB-Swift` stack, audits  

---

## 2. First principle: engine is always Rust

```text
                         turso_core (Rust)
                                ▲
                 ┌──────────────┴──────────────┐
                 │                             │
        Rust Builder / rsapi            C sqlite3 ABI
        (bindings/rust, sdk-kit)        (bindings/c → libturso_sqlite3)
                 │                             │
          full open knobs               SQLite-shaped open
                 │                             │
         Swift via Rust FFI               BoutiqueDB today
         (hypothetical)                   TursoKit / DatabaseActor
```

- **C is not Turso rewritten in C** — it is a compatibility shell that opens `turso_core`.  
- **Rust direct** is a different shell that can call `Builder` / full `DatabaseOpts`.  
- Multi-language reuse is **not** a BoutiqueDB product requirement. Optimize for **Apple apps**.

---

## 3. Side-by-side comparison

| Dimension | **C ABI (current)** | **Rust direct (FFI)** |
|-----------|---------------------|------------------------|
| Swift links | `libturso_sqlite3` / xcframework | Custom static lib (UniFFI / thin `extern "C"` / sdk-kit) |
| Open | `sqlite3_open_v2` | `Builder::new_local` + `experimental_*` / encryption opts |
| SQL | prepare / bind / step | Same SQL surface *or* richer API |
| Experimental features | Partial; weak global toggle | First-class Builder flags |
| Mental model | “SQLite, Turso engine” | “Turso for Apple” |
| StructuredQueries | Excellent | Fine if SQL or driver exists |
| SQLiteData-like DX | High | High if Swift API is good |
| Turso-only marketing | Partial (SQL/PRAGMA reach) | Strong (open-time parity) |
| Async core | C is **sync**; Swift actors wrap | Could map better (harder to build) |
| Maintenance | Shared Turso C investment | **You own** bridge + Apple slices |
| iOS | Proven | Proven pattern; more custom work |
| Risk | Feature lag | Bridge bugs, rewrite cost |
| Bootstrap | **Already paid** | Large TursoKit rewrite |

### Short verdict

| Product goal | Prefer |
|--------------|--------|
| SQLiteData UX + CDC + CK, “good enough Turso” | **C** (extend knobs only when needed) |
| Every Turso experimental feature seamless | **Rust FFI** long-term *or* full C `DatabaseOpts` parity |
| Ship next app + Live CK **now** | **Stay on C** |

---

## 4. What C exposes today

### 4.1 Works via SQL (no Builder) — real Turso differentiation

| Capability | How | BoutiqueDB |
|------------|-----|------------|
| **CDC** | `PRAGMA capture_data_changes_conn` → `turso_cdc` | LiveQuery + CK drain |
| **MVCC / concurrent tx** | `journal_mode=mvcc` + `BEGIN CONCURRENT` | `writeConcurrent` (CDC-safe fallback) |
| **Vector functions** | `vector32`, distances (if in build) | `Vector32`, DSL |
| **Bundled helpers** | uuid, regexp, percentile… | Partial DSL |
| **File + SQL compat** | Turso COMPAT goals | Migrations, StructuredQueries |

### 4.2 Partial / missing on C open

| Capability | Rust Builder | C today |
|------------|--------------|---------|
| Generated cols / vacuum / WITHOUT ROWID | yes | `turso_enable_experimental()` **only these three** |
| FTS / vector **index methods** | `experimental_index_method` | Not in toggle → probe often **false** |
| Materialized views | `experimental_materialized_views` | Often **false** |
| Encryption | `experimental_encryption` + opts | **No** → `encryptionUnavailable` |
| Multi-process WAL | `experimental_multiprocess_wal` | **No** → `multiProcessWALUnavailable` |
| Custom types / attach | Builder flags | Not wired |
| MVCC passive checkpoint | Builder flag | Not wired |

### 4.3 What `turso_enable_experimental()` actually does

```text
generated_columns + vacuum + without_rowid
// NOT: index_method, views, encryption, multiprocess, custom_types, attach
```

---

## 5. Feature usability for BoutiqueDB (Apple apps)

Not “does the engine have it” — **do we want it for this product?**

| Feature | Useful for us? | Why |
|---------|----------------|-----|
| **CDC + LiveQuery + CK** | **Core** | Already on C |
| **In-app concurrent writers (MVCC)** | **High** | Import + user edits; markdown parse off-main + DB write |
| **FTS index** | **High** | Search product quality |
| **Multi-process WAL** | **High if** Share / Safari / extensions share one DB file | App Group + concurrent processes |
| **Vector functions** | Medium–High | AI features; often OK on C |
| **Vector index** | Medium later | Scale optimization |
| **Materialized views** | Medium | Dashboards / aggregates |
| **Encryption (engine)** | Low–Medium | Prefer **Data Protection** for most iOS apps |
| **Custom types** | Low | Prefer Swift enums + bindables |
| **ATTACH** | Low | One app DB is the norm |
| **MVCC passive checkpoint** | Very low | Tuning only |

### Concurrency: don’t mix these up

| Goal | Solution | Binding impact |
|------|----------|----------------|
| Parse markdown / import **without freezing UI** | Swift `Task` + `DatabaseActor` + `async write` | **C already enough** |
| User edits **while** bulk import writes | `writeConcurrent` / MVCC (in-process) | **C already (SQL)** |
| Share sheet + main app **same file, both write** | Multi-process WAL + App Group | **Needs flag (C+ or Rust)** |

---

## 6. Possibilities if we stay on **C**

### 6.1 Can ship now (no binding change)

| Possibility | Notes |
|-------------|--------|
| Local CRUD + StructuredQueries | Ready |
| LiveQuery / observation via CDC | Ready |
| CloudKit offline path + attach auto-drain | Ready; live CK = next app |
| Migrations / schema sync / columns | Ready |
| Async import without UI freeze | Ready (actors + async) |
| Contending in-app writers | Ready (`writeConcurrent` + CDC-safe contract) |
| Honest fail-closed encryption / multiProcess | Ready (throws) |
| Capability-gated FTS/MV macros | Ready (soft fail if probe false) |

### 6.2 Can unlock later **without** abandoning C (“C+”)

Extend `bindings/c` open path / `DatabaseOpts` (Apple-only still):

| Unlock | Work |
|--------|------|
| FTS + vector **indexes** | Wire `index_method` into C open or per-db opts |
| Materialized views | Wire `views` flag |
| Engine encryption | Open-with-cipher/key C API + Swift `EncryptionConfig` |
| Multi-process WAL | Wire multiprocess flag + App Group docs + multi-target tests |
| Richer `turso_enable_*` or `turso_db_config` | Replace single weak global toggle |

**Effort:** medium on engine C binding; **TursoKit/BoutiqueDB mostly call new setters**.  
**Keeps:** StructuredQueries, prepare/step, existing tests.

### 6.3 Hard limits of pure C shape (even with C+)

| Limit | Why |
|-------|-----|
| Always “SQLite theater” | API looks like sqlite3 forever |
| C surface lags Builder | New Turso flags appear on Rust first |
| Sync C API | Long ops block a thread (mitigated by actors, not eliminated) |
| Dual-maintain if Turso prioritizes non-C SDKs | Risk over years |

### 6.4 C path — recommended possibility set for v1 product

```text
MUST:     CDC, LiveQuery, CK, migrations, async I/O, writeConcurrent
SHOULD:   C+ for index_method (FTS) when search ships
COULD:    C+ multiprocess when Share/Safari share one DB
DEFER:    engine encryption (Data Protection), custom types, ATTACH, checkpoint
```

---

## 7. Possibilities if we go **Rust direct**

### 7.1 What becomes natural

| Possibility | How |
|-------------|-----|
| Open with full feature matrix | `Builder.experimental_index_method(true)` etc. |
| Encryption at open | `experimental_encryption` + `with_encryption(opts)` |
| Multi-process from day of wire-up | `experimental_multiprocess_wal(true)` |
| MV / FTS index without “probe false” surprises | Flags match capabilities |
| Turso-first narrative | “BoutiqueDB opens Turso,” not “sqlite3_open with gaps” |
| Faster follow of new engine flags | Bind Builder once; map new methods to Swift |

### 7.2 What we must still build (Rust does not give for free)

| Still ours | Notes |
|------------|--------|
| LiveQuery, Observation, MainActor model | Swift |
| CloudKit / SyncAdapter | Swift |
| Migrations, macros, Dependencies | Swift |
| StructuredQueries driver | Re-point at new connection type or keep SQL execute API |
| Static lib for ios/ios-sim/macos | Packaging either way |
| Async story across FFI | Non-trivial if exposing async core |

### 7.3 Cost and risk of Rust direct

| Cost | Detail |
|------|--------|
| Rewrite TursoKit open/session | Largest piece |
| Choose bridge tech | UniFFI vs hand `extern "C"` vs sdk-kit |
| Dual stack during migration | Or big-bang cutover |
| Testing matrix | Device + sim + mac for new lib |
| Lose free ride on C sqlite3 maturity | More code we own |

### 7.4 Rust path — possibility set

```text
BEST FOR: full Turso open control, multi-process+FTS+encrypt as first-class open options
NOT REQUIRED FOR: Live CK validation, import UX, LiveQuery
DO ONLY IF: C+ debt keeps exploding OR brand = full Builder parity on Apple
```

---

## 8. Possibility matrix (feature × binding)

| Feature / goal | **If C (today)** | **If C+ (extend open)** | **If Rust FFI** |
|----------------|------------------|-------------------------|-----------------|
| CRUD + SQL | ✅ | ✅ | ✅ |
| CDC / LiveQuery | ✅ | ✅ | ✅ (need SQL/CDC still) |
| CloudKit sync layer | ✅ | ✅ | ✅ |
| UI non-blocking import | ✅ actors | ✅ | ✅ |
| In-app MVCC writers | ✅ SQL | ✅ | ✅ |
| FTS index | ❌/gated | ✅ | ✅ |
| Vector index | ❌/gated | ✅ | ✅ |
| Materialized views | ❌/gated | ✅ | ✅ |
| Engine encryption | ❌ | ✅ | ✅ |
| Multi-process WAL | ❌ | ✅ | ✅ |
| Custom types | ❌ | ✅ possible | ✅ |
| ATTACH experimental | ❌ | ✅ possible | ✅ |
| Builder-complete open UX | ❌ | ~parity if complete | ✅ native |
| Effort to first green app | Low (done) | Medium | High |

Legend: ✅ usable · ❌ not / fail-closed · gated = probe often false

---

## 9. Advantages given up on C (as-is)

| Lost / reduced | Why |
|----------------|-----|
| Full Turso SDK open parity | Builder unreachable |
| Easy experimental power | Weak C toggle |
| Strongest “not SQLite” story | Feels like SQLite + PRAGMAs |
| Engine encryption / multi-process | Fail-closed |
| Future flags land late | C ports lag |

**Not given up:** Turso engine binary, CDC, concurrent SQL, Apple framework layer.

---

## 10. Advantages of Rust direct

| Gain | Detail |
|------|--------|
| Full open-time control | All `experimental_*` |
| Fewer permanent gaps | New flags map cleanly |
| Turso-first product story | Cleaner branding |
| Honest capabilities | Open flags ⇒ probes true |
| Apple-only design freedom | No multi-lang C constraints |

**Does not replace:** LiveQuery, CK, migrations, packaging work.

---

## 11. Are we really different from SQLite on C?

| Layer | vs stock SQLite |
|-------|-----------------|
| API shape | Same idea (sqlite3) |
| Engine | **Different** — `turso_core` |
| CDC / concurrent / Turso SQL | **Different** |
| Apple framework (LiveQuery, CK, …) | **Different product** |

```text
BoutiqueDB on C ≈ Turso engine + SQLite-shaped door + Apple app framework
SQLiteData       ≈ SQLite engine + SQLite door + Apple app framework
```

Different **if we use** CDC, concurrent, Turso SQL, CK bridge.  
Only a swapped `.a` if we only run portable SQL.

---

## 12. Recommended roadmap (possibilities ordered)

| Phase | Binding | Unlock |
|-------|---------|--------|
| **Now** | C | Next app + Live CloudKit; import via actors + `write` / `writeConcurrent` |
| **Next product need: search** | C+ | `index_method` → FTS |
| **Next product need: Share/Safari same DB** | C+ | `multiprocess_wal` + App Group guide |
| **If open-flag debt explodes** | Rust FFI open | Builder parity; keep SQL execute for StructuredQueries |
| **Encryption** | Prefer OS | Data Protection; engine cipher only if required |

### Hybrid (often best for BoutiqueDB)

1. Keep **C** for SQL / StructuredQueries / CDC / CK.  
2. **C+** for blocked open flags that product needs.  
3. **Rust open** only if C+ becomes a permanent tax.

---

## 13. Decision log

| Decision | Choice | Date |
|----------|--------|------|
| Binding for app + Live CK validation | **C (current)** | 2026-07-22 |
| Long-term open path | ☐ C+full opts · ☐ Rust FFI · ☐ undecided | |
| Encryption for apps | ☐ Data Protection only · ☐ Turso cipher later | |
| FTS | ☐ LIKE/SQL · ☐ Turso FTS when C+ · ☐ both | |
| Multi-process | ☐ defer · ☐ C+ when extensions share DB | |
| In-app import concurrency | **Actors + writeConcurrent (C)** | 2026-07-22 |

---

## 14. Summary

| Question | Answer |
|----------|--------|
| C vs Rust? | Same engine; different **open control** and ownership cost |
| C possibilities? | Full Apple SQLiteData+CK path **now**; C+ for FTS/MV/encrypt/multiprocess |
| Rust possibilities? | Full Builder; higher rewrite cost; not needed for CK/import UX |
| Let go on C today? | Engine encrypt, multi-process, reliable FTS/vector indexes, MV |
| Different from SQLite? | Yes at engine + CDC + concurrent + framework; shape is SQLite-like |
| Care about other langs? | **No** for this product |

---

## Related files

- **`BoutiqueDB-Enable-All-Features-Research.md`** — enable-all + async + sdk-kit vs Zig (2026-07-22)  
- `bindings/c/src/lib.rs`  
- `bindings/rust/src/lib.rs`  
- `sdk-kit/turso.h` — full experimental_features + async_io (preferred C surface)  
- `COMPAT.md`  
- `BoutiqueDB-Swift/BoutiqueDB-TursoFeatures.md`  
- `BoutiqueDB-Swift/docs/Architecture.md`  
- `audits/2026-07-22/assessment.md`  

