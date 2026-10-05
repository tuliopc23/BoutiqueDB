# Spec: Official-path Turso features + async for BoutiqueDB

## Goal

Enable Turso experimental features and cooperative async for the Apple framework **only via official Turso integration surfaces**, without private forks of open/opts that will break or diverge.

**Product:** BoutiqueDB-Swift (SQLiteData-style + CDC + CloudKit).  
**Engine monorepo:** BoutiqueDB (turso).

---

## Policy (non-negotiable)

1. **Two C ABIs exist — do not conflate them**

   | Surface | Crate | Role | Full experimental flags? | Cooperative async? |
   |---------|-------|------|--------------------------|--------------------|
   | SQLite3 C | `turso_sqlite3` (`bindings/c`) | Compatibility (`sqlite3.h`) | **No** (only partial `turso_enable_experimental`) | **No** (blocking step) |
   | SDK-kit C | `turso_sdk_kit` (`sdk-kit/turso.h`) | **Official language-binding ABI** | **Yes** (`experimental_features` CSV) | **Yes** (`async_io` + `TURSO_IO`) |

2. **Official feature tokens only** (from `docs/sql-reference/experimental-features.mdx`):  
   `views`, `custom_types`, `encryption`, `index_method`, `autovacuum`, `vacuum`, `attach`, `generated_columns`, `without_rowid`, `multiprocess_wal`, `mvcc_passive_checkpoint`

3. **No private hacks:** do not silently hardcode all `DatabaseOpts` in a forked `default_db_opts()` as the permanent product.  
   - Prefer **sdk-kit** (official).  
   - Or **upstream PR** to `bindings/c` if we must keep pure sqlite3 forever.

4. **Experimental = opt-in** at open (app chooses flags). Do not force every experimental feature on by default in production opens.

5. **Change one layer at a time** so we don’t break CDC/CK/tests in one megacommit.

6. **Zig / dual full-Rust+frozen-sqlite3:** out of scope for this work.

References: `BoutiqueDB-Enable-All-Features-Research.md`, `BoutiqueDB-Binding-C-vs-Rust.md`.

---

## Target architecture

```text
                    turso_core
                         ▲
          ┌──────────────┴──────────────┐
          │                             │
   turso_sdk_kit (turso.h)        turso_sqlite3 (sqlite3.h)
   experimental_features          legacy / optional compat
   encryption_*, async_io
          │
   CTursoSDK (SPM C module)     ← NEW primary
          │
   TursoKit (Swift)             ← rewrite open/step/bind
          │
   BoutiqueDB / Observation / CK  ← mostly API-stable
```

**Phase outcome:** TursoKit uses **sdk-kit** as primary. Keep sqlite3 vendor path only if needed as transitional dual-build (prefer single primary after spike).

---

## Phases (implement in order)

### Phase S0 — Spike (prove official path, no product rewrite)

**Duration:** short. **Gate for rest of work.**

1. From BoutiqueDB monorepo:
   ```bash
   cargo build -p turso_sdk_kit --release
   ```
2. Confirm artifacts: `libturso_sdk_kit.a` (name verify), headers via `turso.h`.
3. Minimal C or Swift test:  
   - `turso_database_new` with  
     `experimental_features = "views,index_method,generated_columns,vacuum,without_rowid"`  
   - open + connect + `CREATE TABLE` + try `CREATE INDEX … USING fts` or MV if possible  
   - record: which features actually work on this machine/build (cargo features: `encryption`, `fts` on sdk-kit Cargo.toml)
4. Document cargo feature flags required for encryption/FTS in build script.

**Exit criteria:**  
- Static lib builds for host (macos-arm64).  
- At least one previously-gated capability becomes true with official CSV.  
- No BoutiqueDB-Swift breakage yet (spike can be scripts/ or Examples/).

**If spike fails** (e.g. packaging nightmare): fall back to **upstream-shaped** bindings/c PR plan (Phase S1b) rather than private hacks.

---

### Phase S1 — Vendor + SPM: ship sdk-kit to BoutiqueDB-Swift

1. Extend `Scripts/build-turso.sh` (or add `Scripts/build-turso-sdk-kit.sh`):
   - `cargo build -p turso_sdk_kit --release` (+ features: `encryption`, `fts` as appropriate)
   - Copy `libturso_sdk_kit.a` → `Vendor/turso-sdk/lib/`
   - Copy `sdk-kit/turso.h` → `Sources/CTursoSDK/include/` + modulemap
2. Package.swift:
   - New target `CTursoSDK` (like `CTursoSQLite3`) linking the new `.a` + same system frameworks (CoreFoundation, Security, c++, etc.)
   - TursoKit depends on `CTursoSDK` (primary)
3. **Decision after S0:**  
   - **A (preferred):** TursoKit **only** on sdk-kit; drop sqlite3 link for app path.  
   - **B (transitional):** dual link during migration (heavier); only if S1A blocked.

**Exit criteria:** `swift build` links `CTursoSDK`; empty smoke import compiles.

---

### Phase S2 — TursoKit rewrite on sdk-kit (sync mode first)

**Keep public Swift API as stable as possible** so BoutiqueDB/CK tests keep working.

#### S2.1 Open / connect

Replace `sqlite3_open_v2` with:

```text
turso_database_config_t {
  async_io = 0,                    // Phase S2: blocking-style host loop still OK
  path = file path or ":memory:",
  experimental_features = CSV or null,
  vfs = null | "memory" | "syscall",
  encryption_cipher / encryption_hexkey = optional
}
turso_database_new → open → connect
```

New Swift types (names adjustable):

```swift
public struct TursoOpenOptions: Sendable {
  public var experimentalFeatures: Set<TursoExperimentalFeature> // or [String]
  public var encryption: EncryptionConfig?  // maps to cipher+hexkey
  public var asyncIO: Bool = false          // S2 false; S4 true
  public var multiProcess: Bool = false     // adds multiprocess_wal token
}
```

Map enums → official tokens only.

#### S2.2 Statement lifecycle

Map:

| Old sqlite3 | New sdk-kit |
|-------------|-------------|
| `sqlite3_prepare_v2` | `turso_connection_prepare_single` |
| bind_* | sdk-kit bind APIs |
| `sqlite3_step` | `turso_statement_step` (+ if TURSO_IO and async_io=0, host may still need run_io in a tight loop — match sdk-kit blocking pattern) |
| column_* | sdk-kit row value APIs |
| finalize | sdk-kit finalize |

**Important:** With `async_io=0`, behavior should remain “call step until row/done” without Swift yielding — preserve current actor model.

#### S2.3 Connection helpers

- CDC: still **SQL pragma** after connect (`PRAGMA capture_data_changes_conn`) — official, surface-independent.  
- `changes` / `last_insert_rowid`: sdk-kit equivalents.  
- Close/deinit: follow sdk-kit ownership rules carefully (no UAF).

#### S2.4 BoutiqueDB wiring

- `BoutiqueDB.init` / `connect`:
  - Pass `TursoOpenOptions` from:
    - default: **empty or minimal** experimental set (product decision: default off vs curated default for “Turso advantage” apps)
    - `encryption:` → cipher+hexkey + `encryption` token  
    - `multiProcess: true` → `multiprocess_wal` token (stop hard-throw once open works)
  - Recommended **opt-in preset** e.g. `.tursoEnhanced` = `index_method,views,generated_columns,vacuum,without_rowid` (not encryption/multiprocess unless requested)
- `TursoCapabilities.probe`: after official enable, probes should flip true when engine supports them; keep probes as safety.

**Exit criteria:**

- `swift test` green (all existing suites).  
- New tests: open with `index_method` and/or `views` — capability or DDL succeeds when engine allows.  
- Encryption/multiProcess: either work via official config or still throw with clear “build/feature missing” (not fake success).

---

### Phase S3 — Product enablement (official flags only)

Wire BoutiqueDB public API carefully:

| API | Official token / field | Default |
|-----|------------------------|---------|
| `createFTSIndex` / macros | `index_method` | off unless preset |
| `createMaterializedView` | `views` | off unless preset |
| `encryption:` | `encryption` + cipher/hexkey | off; Data Protection still recommended in docs |
| `multiProcess:` | `multiprocess_wal` | off; App Group docs |
| custom types / attach | tokens | off unless explicit |

Docs:

- README capability matrix: which open options enable which APIs.  
- Migrations.md: experimental opt-in.  
- Architecture.md: sdk-kit primary; sqlite3 legacy if any.

**Exit criteria:** README honest; tests for each enabled path; fail closed if flag not requested.

---

### Phase S4 — Cooperative async (official `async_io`)

Only after S2/S3 stable.

1. Open with `async_io = 1` (or per-connection option).  
2. Step loop:

```swift
while true {
  switch step() {
  case .row: ...
  case .done: return
  case .io: await runIO() // turso_statement_run_io
  case .busy: retry/backoff
  case .error: throw
  }
}
```

3. Keep all engine I/O off MainActor (`DatabaseActor`).  
4. Import/markdown story: parse off-main + yielding writes.  
5. Tests: long write under async_io; no main-thread assert.

**Exit criteria:** async_io path tested; blocking path still available or async is sole path with equivalent semantics.

---

### Phase S5 — Cleanup

- Remove dead sqlite3 vendor if fully replaced.  
- Single build script.  
- Update `BoutiqueDB-Prod-Readiness` / refinement R1.x: mark C-path feature gaps closed via sdk-kit.  
- CHANGELOG breaking notes if any.

---

## Explicit non-goals (this program)

- Zig middle layer  
- Private always-on all experimental flags  
- Dual permanent stacks (Rust open + sqlite3 step)  
- Live CloudKit CI / packaging SPI (separate)  
- Upstream Turso PR unless S0 proves sdk-kit unusable on Apple (then S1b)

---

## Risk register

| Risk | Mitigation |
|------|------------|
| sdk-kit API names differ slightly from turso.h snippets (create vs new) | S0 uses actual header only |
| Cargo features (`encryption`, `fts`) required | Build script enables documented features |
| Test suite assumes sqlite3 error codes | Map sdk-kit status → TursoError |
| CDC dual connection + multiprocess | Test separately; don’t enable both blindly |
| Large rewrite breaks CK | S2 keeps public BoutiqueDB API; full `swift test` gate |
| Experimental instability | Opt-in flags; document |

---

## Implementation order (after plan approval)

```text
S0 spike (build + one feature proof)
  → S1 vendor/SPM CTursoSDK
  → S2 TursoKit sync rewrite + green tests
  → S3 BoutiqueDB open options + docs
  → S4 async_io
  → S5 cleanup
```

**First commit after approve:** S0 spike script + notes in `BoutiqueDB-Swift/docs/` or `Scripts/`.

---

## Success metrics

| Metric | Target |
|--------|--------|
| Official path only | No private DatabaseOpts fork |
| Features | Opt-in CSV/tokens work for index_method + views at minimum |
| Regressions | Existing `swift test` green after S2 |
| Async | S4: TURSO_IO loop exists and is tested |
| Docs | Capability matrix matches open options |

---

## Artifacts to produce during implement

| File | Purpose |
|------|---------|
| `BoutiqueDB-Swift/Scripts/build-turso-sdk-kit.sh` | Official sdk-kit build |
| `BoutiqueDB-Swift/Sources/CTursoSDK/` | Header + modulemap |
| `BoutiqueDB-Swift/Sources/TursoKit/*` | sdk-kit-backed implementation |
| `BoutiqueDB-Swift/docs/Turso-Open-Options.md` | App-facing enable matrix |
| `BoutiqueDB/BoutiqueDB-Enable-All-Features-Research.md` | Keep in sync with decisions |
| Tests | OpenOptions, FTS gate, async_io smoke |

---

## Decision (locked by this plan)

| Decision | Choice |
|----------|--------|
| Primary C surface for enable-all + async | **sdk-kit (`turso.h`)** |
| sqlite3 C | Legacy / not used for experimental flags |
| Private opts hacks | **Forbidden** |
| Default experimental set | **Opt-in** (preset optional, not all-on) |
| Async | Phase S4 after sync sdk-kit path green |

