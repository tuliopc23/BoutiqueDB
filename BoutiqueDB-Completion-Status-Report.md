# BoutiqueDB — Completion Status Report

**Date:** 2026-07-22  
**Scope:** Framework (BoutiqueDB-Swift) + planning monorepo (BoutiqueDB)  
**Audience:** Product owner — reality check + next steps  

---

## 1. Executive summary

| Program | Status | Reality |
|---------|--------|---------|
| **OpenSpec v2** (`boutiquedb-v2`) | **Archived complete** | All phase tasks `[x]` |
| **BD issues 001–014** | **Closed / decided** | No open blockers; BD-013 superseded by sdk-kit |
| **Quality / prod readiness** | **Complete** | PR-001…037 done; tests green |
| **sdk-kit integration S0–S4** | **Landed** | Official features + cooperative async |
| **Public package / SPI / tag** | **Not done** | Intentional residual |
| **Live CloudKit in production** | **Not validated** | Offline tests only; **your next app** |

**Bottom line:**  
The **framework is complete enough to dogfood** in a real Apple app (local Turso + LiveQuery + migrations + offline CK path + official experimental open).  
It is **not** “public SPM package finished” and **not** “multi-device CloudKit proven in the field.”

---

## 2. What is actually done

### 2.1 Product / API (BoutiqueDB-Swift)

| Area | State |
|------|--------|
| Async `read` / `write` / `DatabaseActor` | Done |
| LiveQuery / LiveQueryOne + CDC observation | Done |
| Dual concurrent writes + CDC-safe contract | Done |
| Migrations + schemaSync columns | Done |
| Macros (table / FTS / vector / MV) | Done |
| CloudKit offline (drain, conflicts, wipe, status) | Done |
| `BoutiqueDBSyncEngine.attach` auto-drain | Done |
| **sdk-kit open** + `TursoOpenOptions` | Done |
| FTS / views via official CSV (vendor with `fts`) | Done (S0 proven) |
| Encryption / multiProcess official tokens | Done (experimental; opt-in) |
| Cooperative `asyncIO` (TURSO_IO + yield) | Done (opt-in `.tursoEnhancedAsync`) |
| `swift test` | Green (47+ BoutiqueDB + kit/sync/macros) |

### 2.2 Planning / governance (synced this session)

| Tracker | Status |
|---------|--------|
| `BoutiqueDB-Issues.md` | Open = none; BD-001–004 resolutions updated for sdk-kit |
| `BoutiqueDB-Refinement-Tasks.md` | R1.1/1.3/1.4/1.5 closed via sdk-kit; R2 packaging open |
| `BoutiqueDB-Prod-Readiness-Tasks.md` | Complete; deferred table updated |
| `BoutiqueDB-Quality-Program.md` | Complete + sdk-kit follow-on noted |
| OpenSpec archive README | Points to follow-on programs |
| OpenSpec `tasks.md` | All `[x]` (already) |

### 2.3 What “bids” means here

Interpreted as **backlog / trackers** (BD + refinement + OpenSpec + PR tasks). All **implementation** bids for v2 + quality + sdk-kit phases are closed or archived. Remaining bids are **packaging, CI breadth, live CK, DocC**.

---

## 3. Honest completion scorecard

| Dimension | % complete (opinion) | Notes |
|-----------|----------------------|--------|
| v2 OpenSpec scope | **~95%** | Offline CK, not live CI |
| Prod correctness (audit P0/P1) | **~95%** | Core contracts fixed + tests |
| Turso feature enablement (official) | **~90%** | Open path works; vector **index** still engine/manual-dependent |
| Async engine integration | **~85%** | Cooperative path exists; default remains sync IO drive |
| CloudKit production readiness | **~60%** | Strong offline suite; **no live multi-device proof** |
| Public distribution (SPI) | **~30%** | unsafeFlags + large static lib |
| Docs / DocC polish | **~70%** | Architecture/Open-Options good; no DocC |

**Overall framework maturity for internal next app: ~85%.**  
**Overall “ship to SPI / 1.0 tag”: ~55%.**

---

## 4. Residual risks (real, not theoretical)

1. **Live CloudKit** — never exercised against real `CKSyncEngine` + account in CI.  
2. **Packaging** — `unsafeFlags` + ~400MB static lib hurts consumers and SPI.  
3. **Experimental Turso flags** — still experimental upstream; opt-in by design.  
4. **Rare flaky signal 6** — occasional process abort noise in full `swift test` runs observed historically; latest filtered runs green; watch under stress.  
5. **Table-scoped invalidation** — may still be global invalidate (perf, not correctness).  
6. **Multi-process** — token open works; **Share/Safari concurrent writers** not E2E tested.

---

## 5. Next steps (ordered)

### A. Your immediate path (highest value)

| # | Step | Why |
|---|------|-----|
| 1 | **Build the next real app** on BoutiqueDB-Swift | Only real proof of LiveQuery + migrations + UX |
| 2 | **Live CloudKit** (personal/dev containers) | Close the 40% gap on sync; use `attach` + QA checklist |
| 3 | **Share / App Group** only if product needs multi-process | Validate `multiProcess: true` with two processes |
| 4 | Prefer `.tursoEnhanced` or `.tursoEnhancedAsync` for imports | Official features + optional cooperative IO |

### B. Framework maintainers (before public 1.0)

| # | Step | Tracker |
|---|------|---------|
| 1 | Multi-arch **xcframework** / binary target; drop unsafeFlags | R2.1 |
| 2 | CI builds or caches sdk-kit artifact | R2.2 |
| 3 | SPI clean + version tag | R2.3 / R2.5 |
| 4 | Optional iOS simulator CI | R5.4 |
| 5 | DocC catalog | R4.2 |

### C. Explicitly deprioritize

| Item | Why |
|------|-----|
| Zig / dual Rust binding | Rejected; sdk-kit is official |
| Sample app target | Cancelled; template docs exist |
| Live CK in CI | Expensive; app validation first |
| Full fuzzy/ipaddr DSL | R1.6 nice-to-have |

---

## 6. Recommended “definition of done” variants

### Dogfood-ready (you are here)

- [x] OpenSpec + BD closed  
- [x] Prod readiness PR tasks closed  
- [x] sdk-kit features + async path  
- [x] Offline tests green  
- [ ] Live CK in **your** app  

### Public 1.0

- [ ] Dogfood-ready + live CK lessons landed  
- [ ] R2.1 packaging without unsafeFlags  
- [ ] Tagged release + CHANGELOG  
- [ ] SPI or documented install path  

---

## 7. One-page status board

```text
OpenSpec v2 .............. CLOSED (archive)
BD-001..014 .............. CLOSED (sdk-kit updates applied)
Quality / PR tasks ....... CLOSED
sdk-kit S0–S4 ............ CLOSED
Refinement R1 engine ..... CLOSED (via sdk-kit)
Refinement R2 packaging .. OPEN
Live CloudKit ............ YOUR APP
SPI / tag ................ LATER
```

---

## 8. References

| Doc | Role |
|-----|------|
| `openspec/changes/archive/2026-07-22-boutiquedb-v2/` | v2 design archive |
| `BoutiqueDB-Issues.md` | BD log |
| `BoutiqueDB-Refinement-Tasks.md` | Pre-bundle residual |
| `BoutiqueDB-Prod-Readiness-Tasks.md` | Quality PRs |
| `BoutiqueDB-SdkKit-Integration-Spec.md` | Official binding phases |
| `BoutiqueDB-Swift/docs/Turso-Open-Options.md` | App open flags |
| `audits/2026-07-22/` | Audit trail |
