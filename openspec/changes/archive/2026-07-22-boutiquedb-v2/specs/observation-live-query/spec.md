## ADDED Requirements

### Requirement: TursoStore exposes an AsyncStream of change events
`TursoStore` SHALL expose a public `changes: AsyncStream<ChangeEvent>` property and a method `invalidate()` that yields a new event.

#### Scenario: Background CDC listener receives a change
- **WHEN** a write completes and advances the CDC cursor
- **THEN** `store.changes` emits a `ChangeEvent.generation(UInt64)` without blocking the main thread

#### Scenario: Manual invalidation
- **WHEN** `store.invalidate()` is called
- **THEN** `store.changes` emits a new event and `store.generation` increments

### Requirement: LiveQuery refreshes from AsyncStream events
`@LiveQuery` SHALL subscribe to `db.store.changes` and re-run its query when a relevant event arrives, using `withObservationTracking` (or `withPerceptionTracking` on iOS 15/macOS 12).

#### Scenario: Write triggers LiveQuery update
- **WHEN** a row is inserted into a table referenced by the query
- **THEN** the property wrapper's `wrappedValue` updates within 500 ms

#### Scenario: LiveQuery can be force-refreshed
- **WHEN** `forceRefresh()` is called
- **THEN** the query re-runs immediately, independent of stream events

### Requirement: Perception backport supports older OS versions
`BoutiqueDB` SHALL use `swift-perception` to provide `Perceptible`/`PerceptionRegistrar` behavior when `Observation` is unavailable.

#### Scenario: iOS 15 device uses LiveQuery
- **GIVEN** an app running on iOS 15
- **WHEN** `@LiveQuery` is attached to a SwiftUI view
- **THEN** the view updates when the underlying property changes
