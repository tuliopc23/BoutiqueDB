# Research: Enable all Turso features + async for BoutiqueDB (Apple)

**Date:** 2026-07-22  
**Goal:** Enable Turso experimental capabilities + cooperative async in the Apple framework **without unofficial hacks** that diverge from Turso’s supported integration surfaces.  
**Constraint:** Apple-only product. Prefer **upstream-official** paths; change surface deliberately when the current door has no official knob.

---

## 0. Critical: two C ABIs — pick carefully

Turso does **not** give every feature “on any C call.” Official story differs by surface:

| | **SQLite3 C ABI** | **SDK-kit C ABI** |
|--|-------------------|-------------------|
| **Where** | `bindings/c` → `turso_sqlite3` / `sqlite3.h` | `sdk-kit` → `turso_sdk_kit` / `turso.h` |
| **Purpose (upstream)** | SQLite **compatibility** for apps that speak sqlite3 | **Language-binding** ABI (build SDKs on this) |
| **How docs enable experimental features** | **Not** listed on Experimental Features page for full CSV flags | Same model as Python/Go: `experimental_features` string (+ encryption fields) |
| **What code actually exposes** | `turso_enable_experimental()` → only generated_columns + vacuum + without_rowid | Full feature token list + cipher/hexkey + `async_io` |
| **Step / IO** | `sqlite3_step` → **blocks** (`run_one_step_blocking`) | Can return **`TURSO_IO`**; caller drives IO |
| **BoutiqueDB today** | **This one** | Not linked |

Official Experimental Features doc documents: CLI, Rust Builder, Python, JS, Go, Java (.NET encryption).  
It does **not** say “use `sqlite3_open` + secret flags.”

SDK-kit README (official): *“Low-level C ABI for building language bindings”* with async IO and clear status codes.

### Policy (so we don’t fuck everything)

1. **Use only public, documented (or sdk-kit-header-stable) APIs** for feature enablement.  
2. **If sqlite3 C has no official knob** for a feature → do **not** private-patch `default_db_opts()` in a one-off fork as “the product forever.” Either:
   - **switch to sdk-kit** (official binding path), or  
   - **upstream** a proper bindings/c API (URI / `turso_*` config) matching doc tokens.  
3. **Token names** must match docs: `views`, `index_method`, `encryption`, `multiprocess_wal`, … (see experimental-features.mdx).  
4. **Experimental = opt-in** per app/open; never force-all-on in default BoutiqueDB open without product choice.  
5. **Change one surface at a time** (open/config first; async step second). Don’t rewrite SQL, CK, and open in one megacommit.

---

## 1. Executive recommendation (careful)

| Rank | Approach | When | Upstream-risk |
|------|----------|------|----------------|
| **#1** | **TursoKit → sdk-kit C API** for open/connect/(later) step | Want enable-all + async the **official binding way** | **Low** if we stay on `turso.h` only |
| **#2** | **Stay on sqlite3 C**; ship CDC/CK/async-actors only; feature ceiling accepted | Next app / Live CK **now** | **Lowest** churn |
| **#3** | **Upstream PR to bindings/c** for full feature CSV (aligned with docs) if we must keep sqlite3 forever | Long-term sqlite3-shaped apps | **Correct** but needs review |
| **#4** | Private always-on all `DatabaseOpts` in forked sqlite3 open | — | **Reject** |
| **#5** | Zig layer / dual Rust+frozen sqlite3 | — | **Reject** for this goal |

**Bottom line:**  
The knobs exist in the engine. **Official C path for full flags + async is sdk-kit**, not the sqlite3 compatibility layer. BoutiqueDB currently chose the compatibility layer — correct for “feel like SQLite,” incomplete for “enable all Turso experimental.” **Migrate or upstream; don’t hack.**

---


## 2. What Turso actually provides (facts)

### 2.1 Core: `DatabaseOpts` (`core/lib.rs`)

```text
enable_views
enable_custom_types
enable_encryption
enable_index_method
enable_autovacuum
enable_vacuum
enable_attach
enable_generated_columns
enable_multiprocess_wal
enable_without_rowid
enable_experimental_mvcc_passive_checkpoint
unsafe_testing
```

These are **engine open options**, not “missing from Turso.”

### 2.2 Rust Builder (`bindings/rust`)

Maps 1:1 onto those opts via `experimental_*` methods + `with_encryption(opts)`.

### 2.3 `bindings/c` — SQLite ABI (what BoutiqueDB uses today)

| Item | Behavior |
|------|----------|
| Open | `sqlite3_open` / `sqlite3_open_v2` → `Database::open_file_with_flags(..., default_db_opts(), ...)` |
| `default_db_opts()` | Only if `turso_enable_experimental()`: **generated_columns, vacuum, without_rowid** |
| Step | `sqlite3_step` → `run_one_step_blocking` → **busy-waits IO inside the call** |
| Encryption open | **Not** wired (no cipher/hexkey on this path) |
| Feature list | **Not** exposed as config |

### 2.4 `sdk-kit` — Turso-native C API (already “Builder in C”)

From `sdk-kit/turso.h`:

```c
typedef struct {
    uint64_t async_io;                 // non-zero = caller-driven cooperative IO
    const char *path;
    const char *experimental_features; // comma-separated feature tokens
    const char *vfs;                   // memory | syscall | io_uring | ...
    const char *encryption_cipher;
    const char *encryption_hexkey;
} turso_database_config_t;
```

Feature tokens (from `sdk-kit` / rsapi mapping):

```text
views, index_method, custom_types, autovacuum, vacuum,
encryption, attach, generated_columns, multiprocess_wal,
without_rowid, mvcc_passive_checkpoint
```

Status codes include **`TURSO_IO = 3`**: operation needs the host to run the IO backend and call again.

```c
// open/step can return TURSO_IO when async_io != 0
turso_database_open(...);
turso_statement_step(...);
// then: execute one iteration of underlying IO backend
```

This is the **documented, up-to-date path** for:

1. Enabling **all** experimental features from C  
2. **Cooperative async** instead of blocking step  

**BoutiqueDB is simply not using this API yet.**

---

## 3. Possibility matrix: enable-all features

| Feature | Engine | bindings/c today | sdk-kit C today | Swift framework today |
|---------|--------|------------------|-----------------|------------------------|
| CDC (pragma) | Yes | Yes (SQL) | Yes (SQL) | Yes |
| MVCC / BEGIN CONCURRENT | Yes | Yes (SQL) | Yes (SQL) | Yes (with CDC dual-path) |
| FTS index method | Yes (flag) | Off | **On via `index_method`** | Gated / throws |
| Vector index method | Yes (flag) | Off | **On via `index_method`** | Gated |
| Materialized views | Yes (flag) | Off | **On via `views`** | Gated |
| Encryption | Yes (flag + key) | No | **Config fields** | Throws |
| Multi-process WAL | Yes (flag) | No | **`multiprocess_wal`** | Throws |
| Custom types | Yes (flag) | No | **`custom_types`** | Unused |
| ATTACH experimental | Yes (flag) | No | **`attach`** | Unused |
| Generated / vacuum / WITHOUT ROWID | Yes | Partial global toggle | Full list | Partial |
| Cooperative async step | Core `IOResult` | **Blocked in step** | **`async_io` + TURSO_IO** | Actor wraps blocking only |

### Conclusion on “can we enable all?”

| Question | Answer |
|----------|--------|
| Is it possible without rewriting Turso core? | **Yes** |
| Do we need Zig? | **No** |
| Do we need a new dual Rust binding from scratch? | **No** — sdk-kit is that surface |
| Smallest **correct** work | Point open/config (and later step) at **sdk-kit**, or **upstream** bindings/c — not private forks |

---

## 3b. Feature-by-feature: official path vs gap (pick carefully)

Official feature tokens (from `docs/sql-reference/experimental-features.mdx`):

`views` · `custom_types` · `encryption` · `index_method` · `autovacuum` · `vacuum` · `attach` · `generated_columns` · `without_rowid` · `multiprocess_wal` · `mvcc_passive_checkpoint`

| Need | Official enable path exists? | On **sqlite3 C** today | On **sdk-kit C** today | BoutiqueDB should |
|------|------------------------------|------------------------|------------------------|-------------------|
| CDC / SQL CRUD | N/A (not experimental) | Yes | Yes | Keep (either surface) |
| MVCC `BEGIN CONCURRENT` | SQL after open | Yes | Yes | Keep; don’t invent |
| Generated / vacuum / WITHOUT ROWID | Yes | Partial via `turso_enable_experimental()` only | Full CSV | Prefer CSV on sdk-kit; or call existing enable for those three only |
| Materialized views / views | Yes (`views`) | **No official sqlite3 knob** | `experimental_features=views` | **sdk-kit** or upstream sqlite3 |
| FTS / vector **index method** | Yes (`index_method`) | **No** | `index_method` | **sdk-kit** or upstream |
| Encryption | Yes (`encryption` + cipher/key on SDKs) | **No** (throws in Swift) | cipher + hexkey fields | **sdk-kit** only until bindings/c gains official API |
| Multi-process WAL | Yes (`multiprocess_wal`) | **No** | CSV token | **sdk-kit** + App Group product design |
| Custom types / attach | Yes | **No** | CSV tokens | Only if product needs; sdk-kit |
| Cooperative async step | sdk-kit design | **No** (blocking step) | `async_io` + `TURSO_IO` | Phase 2 on sdk-kit; don’t fake on sqlite3 |
| UI non-block import | App architecture | Actor wrap works | Same | **Keep now** on current stack |

### Safe product tiers (don’t mix)

| Tier | Binding | Enable | Async | Use for |
|------|---------|--------|-------|---------|
| **T0 Stable path** | sqlite3 C (current) | none / only existing `turso_enable_experimental` if you call it | Actor + blocking step | Live CK app, CDC, CRUD, import UX |
| **T1 Feature-rich official** | sdk-kit | official CSV tokens + encryption fields | still can start `async_io=0` | FTS index, views, multiprocess, encrypt |
| **T2 Async official** | sdk-kit | same | `async_io=1` + IO loop | Fair long imports / cooperative IO |
| **T3 Upstream sqlite3 parity** | bindings/c after PR | same tokens as docs | maybe later | If we refuse to leave sqlite3.h |

**Rule:** Ship **T0** until T1 is deliberate. Don’t half-enable experimental features on T0 with private patches.

---


## 4. Async core: what “async” means here

### 4.1 Turso’s real model (not Rust async/await)

From engine skill / core:

- Ops return `IOResult::Done | IOResult::IO(completions)`  
- Caller must **re-enter** after completions  
- Explicit state machines; re-entrancy bugs if you mutate before yield  

**C sqlite3 path:** `run_one_step_blocking` loops:

```text
step → if IO → io.step() → step again  (all inside one C call)
```

Swift only sees “blocking step.” `DatabaseActor` + `async` **does not** make the engine cooperative; it only **moves the block off MainActor**.

### 4.2 What Swift can do today (without engine change)

| Pattern | Freezes UI? | True engine yield? |
|---------|-------------|---------------------|
| MainActor SQL | Yes | No |
| `await db.write` on actor | No (UI) | No (thread still blocked in step) |
| Chunked writes + await between chunks | Better fairness | Partial |
| `writeConcurrent` | Multi-writer | Still blocking steps |

**Enough for:** markdown parse off-main + import without freezing UI.  
**Not enough for:** fine-grained scheduling, “one SQLite step interleaved with other work on same thread,” or matching Node’s non-blocking story.

### 4.3 What Swift can do with sdk-kit `async_io`

```text
loop {
  status = turso_statement_step(stmt)
  if status == TURSO_ROW / DONE / ERROR → handle
  if status == TURSO_IO →
      await runIOBackendOnce()   // or schedule on runloop / task
      continue
}
```

Map to Swift:

```swift
// Conceptual
func stepAsync() async throws -> Step {
  while true {
    switch nativeStep() {
    case .row: return .row
    case .done: return .done
    case .io: await ioTick()   // yield to Swift concurrency
    case .busy: throw / retry
    }
  }
}
```

Benefits for BoutiqueDB:

- Import can **yield between IO waits** without inventing fake chunking  
- Long migrations less “stuck thread”  
- Aligns with Turso’s design instead of fighting it  

Cost:

- Rewrite `TursoConnection` / statement driver  
- StructuredQueries execute path becomes async-aware  
- Testing TURSO_IO interleaving  

### 4.4 “new std:io” / Zig std.io — does the answer lie there?

| Candidate | Relevance |
|-----------|-----------|
| **Turso `IO` trait + PlatformIO / io_uring / …** | **Yes** — this *is* Turso’s IO layer; sdk-kit `vfs` + `async_io` expose it |
| **Rust async std / tokio** | Turso core is **not** primarily tokio-async; don’t force it |
| **Zig `std.io`** | Zig can implement C-compatible IO backends, but Turso already has Rust IO backends; Zig doesn’t unlock experimental SQL features |
| **Swift concurrency (async/await)** | **Yes** — host scheduler for cooperative `TURSO_IO` |

**Verdict:** The async answer is **Turso cooperative IO + Swift tasks**, already surfaced as `async_io` / `TURSO_IO` in **sdk-kit**, not a new language’s std.io.

---

## 5. Approach evaluation (detail)

### A. Expand `bindings/c` only (C+)

**Only acceptable if done as an upstream Turso change** (or temporary branch destined for PR), using the **same feature tokens** as experimental-features.mdx — not a BoutiqueDB-only secret fork.

**Work (upstream-shaped):**

1. Add public API, e.g. URI `?experimental=views,index_method` (Go already documents DSN style for its driver) or `turso_config_*` next to sqlite3.  
2. Map CSV → `DatabaseOpts` (copy sdk-kit mapping; don’t invent names).  
3. Encryption: dedicated open helper mirroring .NET/Java `OpenWithEncryption` if that’s the bindings/c story.  
4. Async on sqlite3 step is **not** part of classic sqlite3 contract — don’t pretend; leave cooperative async to sdk-kit.

**Pros:** Keep `sqlite3_*` for Swift.  
**Cons:** Reimplements sdk-kit; async still second-class; **must not** land as unreviewed private forever.

**When:** Team insists on sqlite3.h forever and will land the PR in Turso.

---

### B. Migrate TursoKit → **sdk-kit C API** (recommended)

**Work:**

1. Build/link `sdk-kit` static lib for Apple slices (same problem as today with turso_sqlite3).  
2. Replace open:

```swift
var config = turso_database_config_t(
  async_io: 1, // or 0 initially
  path: path,
  experimental_features: "views,index_method,encryption,multiprocess_wal,generated_columns,vacuum,without_rowid,attach,custom_types,mvcc_passive_checkpoint",
  vfs: nil,
  encryption_cipher: ...,
  encryption_hexkey: ...
)
turso_database_new(&config, &db, &err)
turso_database_open(db, &err)  // may TURSO_IO
turso_database_connect(db, &conn, &err)
```

3. Replace prepare/step/bind with `turso_statement_*`.  
4. Map `TURSO_IO` → Swift `async` loop.  
5. Keep StructuredQueries by implementing the same driver interface on new connection.  
6. BoutiqueDB capabilities: enable features at open → probes should go true (engine willing).

**Pros:**

- **All features** already designed on this C API  
- **Async path designed** (`async_io`, `TURSO_IO`)  
- One Turso-maintained surface (sdk-kit), not boutique reinvention  
- Still C ABI → excellent Swift interop (`@_cdecl` / module map / xcframework)

**Cons:**

- Real TursoKit rewrite (medium–large)  
- Not binary-compatible with “assume sqlite3.h only”  
- Need solid packaging of sdk-kit for iOS/macOS  

**When:** Goal is enable-all + serious async. **This is the best approach.**

---

### C. Rust UniFFI / direct Builder from Swift

**Pros:** Idiomatic feature flags in generated Swift.  
**Cons:** New binding stack; still need SQL execution surface; duplicates sdk-kit C which already wraps Builder/rsapi.  

**When:** Only if sdk-kit is inadequate for Apple (it is not, for features/async).

---

### D. Zig middle layer

**What people hope:** Zig as “better C” for FFI, or new IO.

**Reality for BoutiqueDB:**

| Claim | Assessment |
|-------|------------|
| Zig unlocks Turso experimental flags | **False** — flags live in Rust `DatabaseOpts` |
| Zig improves Swift interop vs C | **Weak** — Swift still talks C ABI; Zig becomes another compile step |
| Zig std.io replaces Turso IO | **Wrong layer** — would mean reimplementing VFS against turso_core (huge) |
| Zig useful as thin glue | Only if writing custom C API wrappers; Rust already did that in sdk-kit |

**Verdict:** **Do not introduce Zig** for this goal. Extra language, no feature unlock, no async unlock beyond what sdk-kit already exposes.

---

### E. Hybrid: sqlite3 for SQL + Rust only for open

Previously judged **impractical** for long-term dual stacks: open opts belong to the **same** database open as the connection. sdk-kit already unifies open+connect+step.

---

## 6. Swift interoperability notes

| Concern | C sqlite3 | sdk-kit C | Rust UniFFI |
|---------|-----------|-----------|-------------|
| SPM / xcframework | Known | Same packaging shape | Same + generate Swift |
| StructuredQueries | Easy | Need driver adapter | Need driver adapter |
| Sendable / isolation | Manual (you did locks + actor) | Same | Better types possible |
| Async | Fake (actor) | Real with TURSO_IO | Possible if exposed |
| Apple App Store | Static link fine | Static link fine | Static link fine |

Swift does not need “Rust objects in Swift.” It needs a **stable C (or generated) ABI** and a clean async loop. **sdk-kit is that ABI.**

---

## 7. Phased plan (recommended)

### Phase 0 — Research lock (this doc)

- Binding decision: **target sdk-kit for enable-all + async**  
- Zig: **out**  
- Dual Rust+sqlite3: **out**

### Phase 1 — Enable-all (sync first)

1. Package sdk-kit for Apple (or expand bindings/c opts as interim).  
2. BoutiqueDB open:

```text
experimental_features =
  "views,index_method,encryption,multiprocess_wal,generated_columns,vacuum,without_rowid"
  // add attach, custom_types, mvcc_passive_checkpoint as needed
```

3. Wire encryption config to cipher/hexkey when product needs engine cipher.  
4. `multiProcess: true` → `multiprocess_wal` (App Group docs + tests).  
5. Drop fail-closed throws once open succeeds; keep probes as safety.  
6. **async_io = 0** initially so behavior matches current blocking model while features light up.

**Success:** FTS/MV create not gated false; multi-process/encryption paths exist.

### Phase 2 — Cooperative async

1. Open with `async_io = 1`.  
2. TursoKit: `step` returns `.needsIO`; `await io.driveOnce()`.  
3. BoutiqueDB `write`/`read` stay async; internally use non-blocking step loop.  
4. Stress tests with forced IO yields (engine already has yield injection culture).  
5. Import/markdown story: parse off MainActor + **IO-yielding** writes.

**Success:** Thread not blocked inside one giant step loop for multi-IO statements; UI + imports more fair.

### Phase 3 — Product polish

- Capability matrix matches enabled flags  
- Doc: App Group multi-process  
- Optional: Data Protection still recommended even with engine encryption  

---

## 8. Risk register

| Risk | Mitigation |
|------|------------|
| Experimental features “not production ready” in Turso docs | Gate by product; enable per app; document experimental |
| sdk-kit Apple packaging incomplete | Fall back to C+ on bindings/c for flags only first |
| StructuredQueries rewrite cost | Keep SQL string execute bridge first; typed driver second |
| TURSO_IO + MainActor footguns | Never step on MainActor; only DatabaseActor / dedicated executor |
| Encryption key handling | Keychain; never log hexkey |
| Multiprocess + CDC | Test two processes; define writer rules |

---

## 9. Direct answers

| Question | Answer |
|----------|--------|
| Can we enable **all** missing features? | **Yes** — via **sdk-kit config** or C+ `DatabaseOpts` |
| Best approach? | **Migrate TursoKit to sdk-kit C API** (features + async designed in) |
| C opts alone enough? | **For features: yes.** For true async: need non-blocking step (`TURSO_IO`), not only actor wrapping |
| Zig / new std.io the answer? | **No** for features; IO answer is **Turso cooperative IO + Swift await**, already in sdk-kit |
| Keep sqlite3 C as-is forever? | Only if you accept feature/async ceiling; for enable-all, **change which C surface you use** |

---

## 10. Decision log

| Decision | Choice | Date |
|----------|--------|------|
| Enable-all strategy | **sdk-kit C (preferred) / bindings/c C+ interim** | 2026-07-22 |
| Async strategy | **sdk-kit `async_io` + TURSO_IO → Swift async loop** | 2026-07-22 |
| Zig middle layer | **Reject** | 2026-07-22 |
| Dual Rust + frozen sqlite3 | **Reject** | 2026-07-22 |
| Next implementation step | Spike: open DB via sdk-kit with `index_method,views` and probe capabilities from Swift | |

---

## 11. Related paths

| Path | Role |
|------|------|
| `core/lib.rs` | `DatabaseOpts` |
| `bindings/c/src/lib.rs` | sqlite3 open + blocking step (current) |
| `sdk-kit/turso.h` | Full config + async IO C API |
| `sdk-kit/src/rsapi.rs` | Feature string → opts + encryption |
| `core/statement.rs` | `run_one_step_blocking` vs cooperative step |
| `docs/agent-guides` / async-io skill | IOResult state machines |
| `BoutiqueDB-Binding-C-vs-Rust.md` | Prior binding strategy |
| BoutiqueDB-Swift `TursoKit` | Consumer to rewrite against chosen C surface |

---

## 12. One diagram

```text
                    turso_core (DatabaseOpts + IOResult)
                              ▲
              ┌───────────────┴────────────────┐
              │                                │
     sdk-kit C API                      bindings/c sqlite3
     experimental_features              default_db_opts (3 flags)
     encryption_*                       run_one_step_blocking
     async_io + TURSO_IO                sync-looking step
              │                                │
              │                         BoutiqueDB today
              │
         BEST TARGET for
         enable-all + async
              │
         TursoKit vNext → BoutiqueDB
         (Swift actors + await on TURSO_IO)
```
