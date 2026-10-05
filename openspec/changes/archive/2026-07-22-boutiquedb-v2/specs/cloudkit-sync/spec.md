## ADDED Requirements

### Requirement: SyncAdapter protocol defines a pluggable sync surface
`BoutiqueDB` SHALL declare a `SyncAdapter` protocol with `start()`, `stop()`, `syncStatus()`, `drainLocalChanges()`, and `applyRemoteChanges(_:)` methods.

#### Scenario: CloudKit adapter is the default
- **GIVEN** a `BoutiqueDB` with default sync configuration
- **WHEN** sync is started
- **THEN** `CloudKitSyncAdapter` is used and conforms to `SyncAdapter`

#### Scenario: Future Turso Cloud adapter can be swapped
- **GIVEN** a custom `SyncAdapter` implementation
- **WHEN** it is injected via configuration
- **THEN** `BoutiqueDB` uses it without changing public sync API

### Requirement: Batching respects CloudKit limits
`CloudKitSyncAdapter` SHALL batch record changes so that each `CKRecordZoneChangeBatch` contains at most 250 records and the local CDC drain reads at most 500 changes per cooperative await.

#### Scenario: Large local change set is split
- **GIVEN** 1,000 local CDC rows pending
- **WHEN** `drainLocalChanges()` runs
- **THEN** it produces 4 batches of 250 records each

### Requirement: Conflict resolution is configurable
`CloudKitSyncAdapter` SHALL support `.serverWins`, `.clientWins`, and `.lastWriterWins(field:)` conflict policies.

#### Scenario: Last-writer-wins chooses the newer row
- **GIVEN** a local row with `updatedAt` 2026-07-22T10:00:00Z and a server row with `updatedAt` 2026-07-22T10:05:00Z
- **WHEN** `.lastWriterWins(field: \.updatedAt)` resolves the conflict
- **THEN** the server row is applied and the local change is not re-pended

#### Scenario: Local newer wins
- **GIVEN** a local row with a later timestamp than the server row
- **WHEN** `.lastWriterWins` resolves the conflict
- **THEN** the server system fields are kept and the local content is re-pended

### Requirement: Account changes trigger safe rebootstrap
`CloudKitSyncAdapter` SHALL detect Apple ID changes and perform a crash-safe `wipeAndRebootstrap()` that clears local sync metadata and re-uploads the local database to the new account.

#### Scenario: Sign-out is detected
- **WHEN** the signed-in iCloud account changes or is removed
- **THEN** the adapter clears `ck_*` metadata, preserves local data, and restarts sync from scratch

### Requirement: Sync status is exposed as AsyncStream
`SyncAdapter` SHALL expose `syncStatus()` as `AsyncStream<SyncStatus>` with states such as `idle`, `syncing`, `failed`, and `needsAuthentication`.

#### Scenario: UI observes sync status
- **WHEN** a SwiftUI view subscribes to `adapter.syncStatus()`
- **THEN** it receives updates without polling
