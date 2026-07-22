# BoutiqueDB Production Beta Hardening

## Why

BoutiqueDB has a coherent local-first architecture and a green macOS test suite, but the
current implementation still contains correctness, concurrency, sync, schema, and packaging
gaps that prevent a trustworthy public beta. Several advertised APIs are incomplete or can
silently lose errors or changes, and the released Turso binary does not honor the package's
declared minimum macOS deployment target.

## Goal

Finish and align the framework as a production-beta Swift package: safe native ownership,
transactional persistence, race-free observation, durable CKSyncEngine integration, truthful
schema/macros and public APIs, deterministic tests, complete consumer documentation, and
reproducible iOS/macOS binary distribution.

## Scope

- Fix all confirmed BD-015 through BD-038 issues in `BoutiqueDB-Issues.md`.
- Preserve local SQLite/Turso as the source of truth and keep CloudKit optional.
- Align with current SQLiteData 1.7, StructuredQueries 0.34, Swift 6 concurrency, and Apple's
  CKSyncEngine lifecycle guidance where the Turso engine permits it.
- Ship a semver-compatible `0.x` beta only after clean consumer builds for every declared
  platform and architecture.

## Non-goals

- Claiming live CloudKit correctness without real-device validation.
- Supporting public CloudKit databases; CKSyncEngine supports private/shared databases.
- Hiding experimental Turso features behind claims of stable engine support.
- Publishing or tagging a release before all required checks are green.

