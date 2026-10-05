## ADDED Requirements

### Requirement: Public read/write APIs are async
`BoutiqueDB` SHALL provide `read<T>`, `write<T>`, and `transaction<T>` methods that are `async throws` and isolate SQLite work to a `DatabaseActor`.

#### Scenario: Read returns a value on the background actor
- **WHEN** `try await db.read { try $0.fetchAll(Note.self) }` is called from `@MainActor`
- **THEN** the closure runs off the main thread and the result is returned asynchronously

#### Scenario: Write mutates state and emits a change event
- **WHEN** `try await db.write { conn in try Note.insert(...).execute(conn.connection) }` succeeds
- **THEN** the row is persisted and `db.store.changes` emits a `ChangeEvent`

### Requirement: DatabaseActor serializes writes
`DatabaseActor` SHALL serialize write operations so that no two writes run concurrently on the same connection.

#### Scenario: Concurrent writes are queued
- **WHEN** two `db.write` calls are issued concurrently
- **THEN** they complete in some order and the database remains consistent

### Requirement: Concurrent transactions use BEGIN CONCURRENT
`BoutiqueDB` SHALL provide `writeConcurrent`, `beginConcurrent`, and `commitConcurrent` APIs that use `PRAGMA journal_mode = mvcc; BEGIN CONCURRENT;`.

#### Scenario: Non-conflicting concurrent writes commit
- **WHEN** two `writeConcurrent` blocks modify different rows
- **THEN** both commit successfully

#### Scenario: Conflicting concurrent write retries
- **WHEN** two `writeConcurrent` blocks modify the same row and the second receives `SQLITE_BUSY`
- **THEN** the framework retries the second write with exponential backoff up to a configurable maximum

### Requirement: CDC and MVCC are mutually exclusive on one connection
`BoutiqueDB` SHALL throw `BoutiqueError.cdcMutuallyExclusiveWithMVCC` if a configuration tries to enable both CDC and MVCC on the same connection handle.

#### Scenario: Invalid configuration is rejected
- **WHEN** `BoutiqueDB` is initialized with both `enableCDC: true` and `enableMVCC: true` on the same connection
- **THEN** initialization throws `BoutiqueError.cdcMutuallyExclusiveWithMVCC`
