## ADDED Requirements

### Requirement: Vector search DSL is type-safe
`StructuredQueriesTurso` SHALL expose `Vector32`/`Vector32Sparse` value types and `vectorDistanceCos`, `vectorDistanceL2`, `vectorDistanceDot`, and `vectorDistanceJaccard` query helpers.

#### Scenario: Nearest-neighbor query
- **WHEN** `Document.where { vectorDistanceCos($0.embedding, query) < 0.2 }.order { vectorDistanceCos($0.embedding, query) }.limit(10).fetchAll(conn)` is executed
- **THEN** the query returns up to 10 rows ordered by cosine distance

### Requirement: Full-text search DSL is type-safe
`StructuredQueriesTurso` SHALL expose `.match(_:)`, `.score(_:) > 0`, and `.highlight(query:before:after:)` methods on FTS-indexed columns.

#### Scenario: Text search with ranking
- **WHEN** `Note.where { $0.title.match("swift") && $0.title.score("swift") > 0 }.fetchAll(conn)` is executed
- **THEN** the query returns matching notes ordered by relevance

### Requirement: Materialized views are queryable like tables
`BoutiqueDB` SHALL provide `createMaterializedView(_:)` and treat `@MaterializedView` models as `Table` sources in `SelectOf<T>` queries.

#### Scenario: Query materialized view
- **GIVEN** a created `CustomerTotals` materialized view
- **WHEN** `CustomerTotals.all.fetchAll(conn)` is called
- **THEN** the view's maintained results are returned

### Requirement: Encryption at rest is configurable at init
`BoutiqueDB` SHALL accept an `encryption` configuration (`.aegis256(key:)`) and issue the appropriate `PRAGMA cipher` / `PRAGMA hexkey` statements.

#### Scenario: Encrypted database open
- **WHEN** `BoutiqueDB(url: ..., encryption: .aegis256(key: key))` is created
- **THEN** the database file is encrypted and subsequent opens with the wrong key fail

### Requirement: Multi-process WAL is configurable at init
`BoutiqueDB` SHALL accept a `multiProcess: Bool` option and manage the `.tshm` sidecar file.

#### Scenario: Two processes share a WAL
- **GIVEN** `BoutiqueDB(url: ..., multiProcess: true)` opened in Process A
- **WHEN** Process B opens the same URL with `multiProcess: true`
- **THEN** both processes can read and write without corrupting the WAL

### Requirement: Turso scalar extensions are exposed as typed helpers
`BoutiqueDB` SHALL provide typed helpers for UUID (`uuid4()`), regexp (`regexp_like`), time (`dur_*`), percentile, fuzzy matching, and IP address functions.

#### Scenario: UUID and regexp in a query
- **WHEN** `Note.where { $0.title.regexp("^Hello") && $0.id.eq(UUID.v4()) }` is constructed
- **THEN** the generated SQL uses `regexp_like(title, "^Hello")` and `uuid4()` correctly

### Requirement: Custom types are QueryBindable
`BoutiqueDB` SHALL allow `RawRepresentable` enums and structs to conform to `QueryBindable` and be used as column values.

#### Scenario: Enum stored as TEXT
- **GIVEN** a `Status` enum conforming to `String, QueryBindable`
- **WHEN** it is used as a `@Column` property
- **THEN** inserts and queries map to/from `TEXT` values
