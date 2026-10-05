## ADDED Requirements

### Requirement: BoutiqueDBMacros is a dedicated .macro target
`Package.swift` SHALL declare a `.macro` target named `BoutiqueDBMacros` that depends on `SwiftSyntaxMacros` and `SwiftCompilerPlugin`.

#### Scenario: Package builds
- **WHEN** `swift build` runs
- **THEN** `BoutiqueDBMacros` compiles without linker or sandbox errors

### Requirement: @BoutiqueTable extends @Table with advanced options
`@BoutiqueTable` SHALL support `withoutRowid:`, `strict:`, and `generated` column options while generating `TableDescriptor`, `Codable` conformance helpers, and static DSL helpers.

#### Scenario: Model with generated column compiles
- **GIVEN** a model marked with `@BoutiqueTable` and a generated column
- **WHEN** the macro expands
- **THEN** the generated DDL includes `GENERATED ALWAYS AS ... VIRTUAL` and the model compiles

### Requirement: @FTSIndex generates a typed full-text index
`@FTSIndex` SHALL accept a tokenizer and one or more `String`/`Text` columns and generate `CREATE INDEX ... USING fts` DDL plus a typed `FTSConfig` descriptor.

#### Scenario: Full-text index is created
- **GIVEN** `@FTSIndex extension Note { static let titleSearch = FTS(title, body, tokenizer: .default) }`
- **WHEN** the migration runs
- **THEN** `CREATE INDEX note_title_search ON notes USING fts (title, body)` is executed

### Requirement: @VectorIndex generates a typed vector index
`@VectorIndex` SHALL validate the metric (`.cosine`, `.l2`, `.dot`, `.jaccard`) and column type (`Vector32` or `Vector32Sparse`) and generate `CREATE INDEX ... USING vector` DDL.

#### Scenario: Vector index is created
- **GIVEN** `@VectorIndex extension Document { static let embeddingIndex = VectorIndex(embedding, metric: .cosine) }`
- **WHEN** the migration runs
- **THEN** `CREATE INDEX document_embedding_idx ON documents USING vector (embedding)` is executed

### Requirement: @MaterializedView generates a queryable view
`@MaterializedView` SHALL define a `Table`-like struct whose `source` expression is translated into `CREATE MATERIALIZED VIEW ... AS <source>` DDL.

#### Scenario: Materialized view is created
- **GIVEN** a `@MaterializedView` model with a valid `QueryExpression` source
- **WHEN** `db.createMaterializedView(CustomerTotals.self)` is called
- **THEN** the materialized view is created and `CustomerTotals.all.fetchAll(conn)` returns rows

### Requirement: Macro expansions are snapshot-tested
`BoutiqueDBTests` SHALL include snapshot tests for each macro's generated SQL using `swift-snapshot-testing`.

#### Scenario: SQL snapshot matches
- **WHEN** a macro's generated SQL is snapshotted
- **THEN** the output matches the checked-in reference and any drift fails CI
