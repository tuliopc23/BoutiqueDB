# BoutiqueDB — Production Readiness Tasks

**Status:** COMPLETE — all waves implemented  
**Program:** Quality Program Phase A → implement  
**Constraint was:** not final packaging / SPI / tag / sample app.  
**Binding note:** After this program, engine open moved to **sdk-kit** (see SdkKit-Integration-Spec); deferred R1.1/R1.3/R1.4 below are **superseded** by that work.

Legend: `[ ]` open · `[~]` partial · `[x]` done

---

## Waves 0–3

All **PR-001…PR-037** tasks: **`[x]`** (see git history / audit mapping).

Definition of done for this program: **all checked**.

---

## Deferred at time of quality program → current reality

| Item | Then | Now (2026-07-22) |
|------|------|------------------|
| R1.1 experimental lib | Deferred | **Done** via sdk-kit + fts cargo feature |
| R1.3 / R1.4 encryption / multi-process C | Deferred | **Done** via sdk-kit official config (experimental still) |
| R2 packaging SPI | Deferred | **Still open** |
| Live CloudKit CI | Deferred | **User next app** |
| iOS CI / DocC | Deferred | **Still open** (optional) |

---

## Mapping quick ref

| PR | Audit | Status |
|----|-------|--------|
| PR-001–003 | A-001, A-023, A-025 | [x] |
| PR-010–013 | A-007, A-006, A-017, A-011 | [x] |
| PR-020–024 | A-008, A-009, A-010, A-012, A-019 | [x] |
| PR-030–037 | honesty + DX | [x] |
