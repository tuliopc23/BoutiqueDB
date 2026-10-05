## ADDED Requirements

### Requirement: BoutiqueMigrationPlan provides schema versioning
`BoutiqueDB` SHALL provide a `BoutiqueMigrationPlan` DSL and `BoutiqueMigration` blocks that create and alter tables from `@Table`/`@BoutiqueTable` models.

#### Scenario: New table is created from a model
- **WHEN** `try await db.migrate(using: BoutiqueMigrationPlan { BoutiqueMigration(1) { try $0.create(Note.self) } })` runs
- **THEN** the `notes` table is created if it does not exist

#### Scenario: Alter table adds a column
- **GIVEN** a migration block that calls `db.alter(Note.self) { $0.add(\.updatedAt, type: .date, default: Date()) }`
- **WHEN** the migration runs
- **THEN** the column is added with the specified type and default

### Requirement: Package exposes a public SampleApp
The repo SHALL contain a `SampleApp/` target demonstrating SwiftUI + `LiveQuery` + sync + FTS + vector search.

#### Scenario: Sample app builds and runs
- **WHEN** the sample app target is built in Xcode
- **THEN** it compiles and displays a list of notes that updates on writes and supports search

### Requirement: CI validates every commit
`.github/workflows/` SHALL run `swift build`, `swift test`, and simulator tests on every PR.

#### Scenario: PR is opened
- **WHEN** a pull request is created
- **THEN** GitHub Actions builds and tests the package

### Requirement: Swift Package Index compatibility
`Package.swift` SHALL replace `.unsafeFlags` with a binary `.xcframework` target or `systemLibrary` so the package is SPI-eligible.

#### Scenario: Package resolves without unsafe flags
- **WHEN** `swift package resolve` runs in a fresh environment
- **THEN** no `unsafeFlags` warning appears and `swift package dump-package` succeeds

### Requirement: Release pipeline produces v1.0.0 artifacts
The project SHALL use the `macos-release` pipeline to archive, notarize, produce a DMG, and update a Sparkle `appcast.xml` for the sample macOS app.

#### Scenario: Tag is pushed
- **WHEN** `v1.0.0` is tagged
- **THEN** a GitHub release with signed artifacts is created

### Requirement: Documentation covers quick-start and migration
`README.md` and `docs/` SHALL include a quick-start guide, a migration guide from v1 synchronous APIs, and API reference links.

#### Scenario: New developer reads README
- **WHEN** a developer opens `README.md`
- **THEN** they can build the package, run the sample app, and understand the v2 API shape
