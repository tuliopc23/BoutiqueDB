# BoutiqueDB Production Beta Hardening Tasks

Detailed acceptance criteria and evidence are tracked as BD-015…BD-038 in
`BoutiqueDB-Issues.md`.

## 1. Core ownership and lifecycle

- [x] 1.1 Make native connection/statement ownership race-safe and remove unjustified public
  `@unchecked Sendable` escape hatches (BD-015).
- [x] 1.2 Implement or remove every incomplete low-level API, beginning with
  `clearBindings()` (BD-016).
- [x] 1.3 Add explicit database/store/sync shutdown and poisoned-transaction handling (BD-017).
- [x] 1.4 Normalize typed errors without swallowing underlying failures (BD-018).

## 2. Open, migrations, and schema

- [x] 2.1 Introduce one open configuration used by every initializer/bootstrap path (BD-019).
- [x] 2.2 Make migrations atomic by default; validate IDs/order and safely handle debug erasure
  only after handles are closed (BD-020).
- [x] 2.3 Establish canonical typed schema metadata shared by StructuredQueries, macros,
  migrations, and sync (BD-021).
- [x] 2.4 Complete macro diagnostics and schema generation; never silently map unsupported
  Swift types or partially apply advanced DDL (BD-022, BD-023).

## 3. Observation and query DX

- [x] 3.1 Move CDC observation off `MainActor`, serialize it with database ownership, expose
  failures, and reduce/adapt polling (BD-024).
- [x] 3.2 Make LiveQuery refresh cancellation-safe, generation-ordered, table-aware where
  possible, and test teardown/error behavior (BD-025).
- [x] 3.3 Provide SQLiteData-style bootstrap, dependency, preview, and test factories with
  truthful examples (BD-026).

## 4. CloudKit and synchronization

- [x] 4.1 Never advance the CDC cursor past an unmappable configured-table change (BD-027).
- [x] 4.2 Implement genuine client/server/LWW conflict behavior with round-trip tests (BD-028).
- [x] 4.3 Replace the single swallowed commit callback with durable multi-subscriber enqueue
  semantics and observable failures (BD-029).
- [x] 4.4 Validate synchronized schemas, record identities, reserved names, value types, and
  explicit CloudKit container configuration before start (BD-030, BD-031).
- [x] 4.5 Serialize and lock sync state, make account probing awaitable, and make asset/mapping
  failures explicit (BD-032).
- [x] 4.6 Separate transport-neutral sync changes from CloudKit records and add manual
  fetch/send/sync lifecycle APIs; document sharing limitations (BD-033).

## 5. API, tests, performance, and documentation

- [x] 5.1 Freeze a documented supported public-product/API surface; remove unused dependencies
  and stale sqlite3-era messaging (BD-034).
- [x] 5.2 Add failure injection, concurrency stress/TSan, migration upgrade, observation fan-out,
  sync restart/conflict, and performance coverage (BD-035).
- [x] 5.3 Add DocC, a compiling consumer fixture, entitlements/sharing guidance, security and
  privacy logging guidance, and API compatibility checks (BD-036).

## 6. Distribution and beta release

- [x] 6.1 Rebuild and verify TursoSDK for macOS 14 and iOS 17 across all declared device and
  simulator architectures; fail CI on deployment-target/link warnings (BD-037).
- [x] 6.2 Validate clean URL-binary consumer builds, release/package workflows, checksums,
  SPI configuration, and declared platform matrix (BD-038).
- [ ] 6.3 Run a second full gap review, fix all findings, then publish a prerelease beta tag and
  document the remaining real-device CloudKit validation gate.

## Validation evidence

- Warning-free Swift build; strict `swift format`; shell syntax and ShellCheck.
- 94 tests across TursoKit, BoutiqueDB, TursoCKSync, and macros; focused concurrency/sync
  suites also pass under Thread Sanitizer.
- Crash reproducer passed 20 consecutive runs after removing the unsafe live CDC/MVCC
  journal transition; full suite passed repeatedly.
- DocC builds with warnings as errors; the standalone consumer fixture builds with warnings
  as errors; iOS 17 arm64 Simulator package link succeeds.
- TursoSDK v0.2.1 archive checksum and every Mach-O archive member were verified for macOS
  14/iOS 17 deployment targets across macOS arm64/x86_64, iOS arm64, and Simulator
  arm64/x86_64.
- Physical-device production CloudKit push/account/two-device convergence remains an explicit
  host-app QA gate; it is not represented as locally verified.
