# BoutiqueDB — Pre-Bundle Refinement Tasks

> OpenSpec v2 **archived**. Quality program **complete**.  
> This file tracks **remaining** work before public package bundle / SPI / tag.  
> Updated 2026-07-22 after **sdk-kit** integration.

Legend: `[x]` done · `[ ]` open · `[~]` partial

---

## R0 — Correctness & API freeze

| ID | Pri | Status | Task |
|---|---|---|---|
| R0.1 | P0 | [x] | API surface aligned with README / Architecture.md |
| R0.2 | P0 | [x] | fatals only for unregistered `\.boutiqueDB` dependency |
| R0.3 | P0 | [x] | Concurrency rules documented in `docs/Architecture.md` |
| R0.4 | P1 | [x] | `LiveQuery.setQuery` for dynamic reloads |
| R0.5 | P1 | [x] | Failed migration never recorded (`migrationFailed`) |
| R0.6 | P1 | [x] | SchemaSync additive **column** ensure from schema |
| R0.7 | P2 | [x] | `beginConcurrent` / `commitConcurrent` / `rollbackConcurrent` |

---

## R1 — Engine / bindings

| ID | Pri | Status | Task |
|---|---|---|---|
| R1.1 | P0 | [x] | **Done via sdk-kit:** vendor `libturso_sdk_kit` with `--features fts,encryption`; open CSV `index_method,views,…` (not sqlite3 rebuild) |
| R1.2 | P1 | [x] | `Scripts/build-turso-sdk-kit.sh` (+ legacy `build-turso.sh` for sqlite3 if needed) |
| R1.3 | P1 | [x] | **Done via sdk-kit:** encryption cipher/hexkey on open (not separate C setter on sqlite3) |
| R1.4 | P2 | [x] | **Done via sdk-kit:** `multiprocess_wal` token on open; App Group multi-process **validation** still app-level |
| R1.5 | P2 | [x] | Capability probes + official open options (`docs/Turso-Open-Options.md`) |
| R1.6 | P2 | [ ] | fuzzy / ipaddr / time helpers expansion (nice-to-have DSL) |

---

## R2 — Packaging & SPI

| ID | Pri | Status | Task |
|---|---|---|---|
| R2.1 | P0 | [~] | **In progress:** path `binaryTarget` TursoSDK.xcframework, **no unsafeFlags**; macos-arm64 slice works; iOS multi-arch still TODO |
| R2.2 | P0 | [~] | CI builds xcframework when engine available; release workflow on `v*` tags |
| R2.3 | P1 | [~] | `swift package dump-package` OK; SPI listing after rename + iOS slices |
| R2.4 | P1 | [x] | CHANGELOG.md |
| R2.5 | P1 | [ ] | Tag `v0.2.0` after first release asset + checksum |
| R2.6 | P0 | [x] | LICENSE, NOTICE, `.spi.yml`, Publishing.md, icon assets |

---

## R3 — Sync polish

| ID | Pri | Status | Task |
|---|---|---|---|
| R3.1 | P1 | [x] | `ck_meta_version` / format_version |
| R3.2 | P1 | [x] | needsAuthentication from account status (injectable + apply) |
| R3.3 | P2 | [ ] | Fill benchmark tables on device |
| R3.4 | P2 | [ ] | Live CloudKit CI (optional) — **user validates in next app** |

---

## R4 — Docs & DX

| ID | Pri | Status | Task |
|---|---|---|---|
| R4.1 | P0 | [x] | `docs/Architecture.md` |
| R4.2 | P1 | [ ] | DocC catalog |
| R4.3 | P1 | [x] | `docs/App-Template.md` |
| R4.4 | P2 | [ ] | Pre-async API migration guide (optional; async open is opt-in) |
| R4.5 | P1 | [x] | `docs/Turso-Open-Options.md` (sdk-kit official path) |

---

## R5 — Test hardening

| ID | Pri | Status | Task |
|---|---|---|---|
| R5.1 | P0 | [x] | `swift test` green (sdk-kit path) |
| R5.2 | P1 | [x] | Dual LiveQuery + concurrent write + drainCDC stress |
| R5.3 | P1 | [x] | Failed migration not recorded |
| R5.4 | P2 | [ ] | iOS simulator CI matrix |
| R5.5 | P1 | [x] | asyncIO open + write test |

---

## Remaining before “final public bundle”

1. **R2.1** — multi-arch binary / remove unsafeFlags for SPI  
2. **R2.5** — version tag  
3. Optional: R4.2 DocC, R5.4 iOS CI, R3.4 live CK CI  

**Not blockers for dogfooding / next app:** packaging SPI, DocC, live CK CI.

---

## Framework icon

- Source: `BoutiqueDB/Frameowrk-icon/BoutiqueDB.png` (typo folder kept)  
- Canonical package assets: `BoutiqueDB-Swift/Assets/BoutiqueDB.png`, `BoutiqueDB-Swift/icon.png`  
