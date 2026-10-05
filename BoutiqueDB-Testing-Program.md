# BoutiqueDB — Testing Program (Phase B)

**Status:** baseline complete with prod-readiness suites; expand continuously  
**Principle:** fail-first, no weak asserts, high realistic coverage of public behavior

## Stack

| Layer | Approach |
|-------|----------|
| Swift | Swift Testing (`@Test`, `@Suite`, async) |
| Engine culture | Turso: correctness over coverage vanity; every fix needs a regression test |
| E2E offline | CDC × write × writeConcurrent × LiveQuery × drain × migrations |

## Current suites (BoutiqueDB-Swift)

| Suite | Focus |
|-------|--------|
| BoutiqueDB high-level API | CRUD, LiveQuery, concurrent serialize |
| CDC contracts | concurrent/primary drain; auto-drain attach |
| Prod readiness | fetchOne, migrations complete, schema columns, txn session, sparse, LQOne setQuery, auth status, isSynchronizing |
| Refinement stress | dual LQ + write + concurrent + drain (strict pending) |
| Migrations | open, ensureColumn, failed-not-recorded, schemaSync tables |
| Schema DDL | descriptors, create, capabilities |
| Turso features | vector, FTS gate, encryption/MP throws, concurrent dual |
| LiveQuery integration | @Observable model refresh |
| TursoKit | open/CDC, StructuredQueries driver |
| TursoCKSync | drain, inbound, conflicts, wipe, multi-table, batching, account hash |
| Macros | table/FTS/vector/MV expansions |

## Coverage targets

| Layer | Target | Status |
|-------|--------|--------|
| Critical contracts (A-001/007/017/023) | 100% exercised | **done** |
| Public BoutiqueDB methods | success + error path | **mostly done** — expand CI iOS later |
| Conflict policies | server/LWW/clientWins | server+LWW done; clientWins optional |
| Live CloudKit | optional nightly | deferred |

## Rules

1. No `#expect(… \|\| true)` or tautologies.
2. New public API → test in same change.
3. Prefer offline deterministic tests over live CloudKit.
4. Prefer extending existing suites over new files unless isolation requires it.

## Run

```bash
cd BoutiqueDB-Swift && swift test
```
