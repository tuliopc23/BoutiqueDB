# K-Sync knowledge: BoutiqueDB CloudKit / SyncAdapter

**Date:** 2026-07-22  
**Scope (read-only):**
- `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/Sources/TursoCKSync/`
- `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/docs/CloudKit-QA-Checklist.md`
- `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/Tests/TursoCKSyncTests/`
- `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/Sources/BoutiqueDB/BoutiqueDBSyncEngine.swift`

**Related:** `docs/Architecture.md`, `docs/App-Template.md`, `docs/Sync-Benchmarks.md`, `BoutiqueDB-Design.md`

---

## 1. SyncAdapter architecture

### Layering

```
SwiftUI / app
    │
    ▼
BoutiqueDBSyncEngine          (@MainActor façade; BoutiqueDB product)
    │  adapter: CloudKitSyncAdapter
    ▼
SyncAdapter protocol          (pluggable multi-device surface)
    └── CloudKitSyncAdapter   (default; status fan-out)
            │  engine: TursoCKSyncEngine
            ▼
TursoCKSyncEngine             (CDC ↔ CKSyncEngine / local pending queue)
    ├── SyncMetadataStore     (ck_* tables in the Turso file)
    ├── RecordMapper          (row ↔ CKRecord + system fields)
    ├── RowSQL                (upsert/delete against user tables)
    └── SyncedTable / TursoCKSyncConfiguration
```

| Type | Path | Role |
|------|------|------|
| `SyncAdapter` | `Sources/TursoCKSync/SyncAdapter.swift` | Protocol: `start`, `stop`, `syncStatus()`, `drainLocalChanges()`, `applyRemoteChanges` |
| `CloudKitSyncAdapter` | same | Default impl; owns status stream + wraps engine |
| `SyncStatus` | same | `idle` / `syncing` / `failed(String)` / `needsAuthentication` / `accountChanged` |
| `RemoteChange` | same | Transport-agnostic: `.upsert(CKRecord)` / `.delete(CKRecord.ID)` |
| `TursoCKSyncEngine` | `Sources/TursoCKSync/TursoCKSyncEngine.swift` | CDC drain, inbound apply, `CKSyncEngineDelegate`, conflicts, account wipe |
| `BoutiqueDBSyncEngine` | `Sources/BoutiqueDB/BoutiqueDBSyncEngine.swift` | Thin `@MainActor` wrapper: builds `TursoCKSyncConfiguration` + adapter; exposes `start`, `syncStatus`, `drainCDC`, `drainLocalChanges`, `performLocalWrite` |
| `SyncedTable` | `Sources/TursoCKSync/SyncedTable.swift` | Table name, PK column (default `id`), synced columns, record type |
| `ConflictPolicy` | same | `.serverWins` (default), `.clientWins`, `.lastWriterWins(field:)` |
| `TursoCKSyncConfiguration` | same | container ID, zone (`app.default`), tables, policy, `maxBatchSize` (≤250), `drainCDCLimit` (default 500), `enablesCloudKit` |
| `SyncMetadataStore` | `Sources/TursoCKSync/SyncMetadataStore.swift` | On-disk CK state, CDC cursor, record meta, account hash, format version |
| `RecordMapper` / `RowSQL` | `RecordMapper.swift`, `RowSQL.swift` | Encode/decode system fields; SQL upsert/delete |

### Design intent

- **Pluggable transport:** `SyncAdapter` is documented for a future Turso Cloud adapter without changing app APIs (`SyncAdapter.swift` header; design doc §7).
- **Offline / unit-test mode:** `enablesCloudKit: false` skips `CKContainer` / `CKSyncEngine` and keeps `localPendingRecordZoneChanges` in-process (`TursoCKSyncEngine.start`).
- **Record identity:** `table:rowPK` record names (`RecordIdentity` in `SyncedTable.swift`); zone `configuration.zoneName` / owner `CKCurrentUserDefaultName`.
- **CDC isolation:** engine sets `connection.isSynchronizing` during inbound apply so `drainCDC` no-ops mid-apply; cursor advanced past echo CDC rows.

### Status stream

`CloudKitSyncAdapter` multiplexes `SyncStatus` to subscribers via `AsyncStream` (newest buffer 8). Engine pushes via `statusSink` (wired in adapter init). Adapter also publishes around `start` / `drainLocalChanges` / `applyRemoteChanges`.

**PROD-BLOCK gap:** `SyncStatus.needsAuthentication` is defined but **never published** anywhere in the package (only mention is the enum case). Live apps cannot observe “not signed into iCloud” through the official stream without extra work.

---

## 2. CDC drain → pending → apply remote

### Outbound (local → CloudKit)

1. **Prerequisite:** connection opened with CDC (`BoutiqueDB` defaults `enableCDC: true`; engine also calls `enableCaptureDataChanges(mode: .full)` on init).
2. **App writes** to user tables (via `BoutiqueDB.write` or `TursoCKSyncEngine.performLocalWrite`).
3. **`drainCDC(limit:)`** (`TursoCKSyncEngine`):
   - Returns 0 if `isSynchronizing` or `stopped`.
   - Loads `ck_cdc_cursor.last_change_id`.
   - Reads `connection.cdcChanges(after:cursor, limit:)` (default limit 500).
   - Filters to `configuration.syncedTableNames`.
   - Resolves row PK (TEXT PK, or rowid → PK column, or CDC before/after payload via `bin_record_json_object`).
   - Builds `CKRecord.ID` as `table:pk` in the app zone.
   - Deletes → `.deleteRecord`; inserts/updates → `.saveRecord`.
   - **`enqueuePending`:** either `CKSyncEngine.state.add(pendingRecordZoneChanges:)` or local queue.
   - Advances CDC cursor to last seen change ID **even for filtered tables** (non-synced table CDC is skipped but cursor still moves with `lastID`).
4. **Send path (CloudKit only):** `CKSyncEngineDelegate.nextRecordZoneChangeBatch` caps to `maxBatchSize` (≤250), `makeRecord` loads row + hydrates system fields; missing local row drops the save.
5. **After successful save:** `handleSentRecordZoneChanges` upserts `ck_record_meta` with encoded system fields.

Convenience APIs:
- `performLocalWrite { ... }` → write then `drainCDC()`.
- `CloudKitSyncAdapter.drainLocalChanges()` / `BoutiqueDBSyncEngine.drainLocalChanges()` → `drainCDC` with config limit + status transitions.

### Inbound (CloudKit → local)

1. **Live:** `fetchedRecordZoneChanges` → `applyModification` / `applyDeletion`.
2. **Tests / custom transport:** `applyRemoteRecord` / `applyRemoteDeletion` or adapter `applyRemoteChanges([RemoteChange])`.
3. **`applyModification`:**
   - Resolve table by record type or record-name prefix.
   - Map CK fields → row via `RecordMapper.rowDictionary`.
   - `isSynchronizing = true`, upsert row + `ck_record_meta`, advance CDC cursor past echo.
4. **`applyDeletion`:** resolve via record name or `ck_record_meta`, delete row + meta, advance cursor.

### Metadata tables (`SyncMetadataStore.schemaSQL`)

| Table | Purpose |
|-------|---------|
| `ck_sync_state` | `CKSyncEngine.State.Serialization` blob |
| `ck_record_meta` | per-row record name, zone, system fields (unique on record_name) |
| `ck_cdc_cursor` | last drained CDC change_id |
| `ck_account` | account_hash for BD-007 identity change |
| `ck_meta_version` | format_version (currently 1) |

### Echo suppression

Inbound writes would re-appear in `turso_cdc`. Engine advances cursor with `advanceCDCCursorPastEcho(from:)` (up to 10k CDC rows after apply) so a subsequent drain does not re-pend the same row (asserted in `inboundApplyAndEchoSuppression` test).

### Coupling gaps (integration)

| Issue | Severity |
|-------|----------|
| **No automatic drain after `BoutiqueDB.write`** — apps must call `drainCDC` / `drainLocalChanges` / `performLocalWrite` themselves | **PROD-BLOCK** for “sync just works” |
| **`BoutiqueDB.open` does not create or start `BoutiqueDBSyncEngine`** | App must wire sync separately |
| **App template (`docs/App-Template.md`) has zero sync wiring** | New apps ship local-only by default |
| CDC cursor advanced for all CDC rows including non-synced tables (filter only skips enqueue) | Usually fine; document behavior |

---

## 3. Conflicts, account, batching, multi-table

### Conflicts (`ConflictPolicy`)

Handled on `.serverRecordChanged` in `handleSentRecordZoneChanges` / `handleServerRecordChanged`, and via test hook `resolveConflictForTesting`:

| Policy | Behavior |
|--------|----------|
| `.serverWins` | `applyModification(serverRecord)` |
| `.clientWins` | Apply server (absorb system fields), then re-pend `.saveRecord(failedRecord.recordID)` |
| `.lastWriterWins(field:)` | Compare comparable stamps (NSNumber, ISO8601 string, NSDate) on field; newer local → update meta system fields from server + re-pend; else apply server |

Other save failures:
- `zoneNotFound` → re-save zone + record
- `unknownItem` → clear system fields meta, re-pend save
- network / busy / notAuthenticated / cancelled → no immediate retry in handler (rely on CKSyncEngine)

### Account

| Path | Behavior |
|------|----------|
| `CKSyncEngine.Event.accountChange` | `.signIn` → re-upload zone + `enqueueAllLocalRows`; `.switchAccounts` / `.signOut` → `wipeAndRebootstrap(preserveLocalUserData: true)` |
| `noteAccountIdentity(_:)` | If stored hash differs → status `.accountChanged`, wipe meta + re-enqueue local rows, save new hash |
| `detectAccountIdentityChangeIfNeeded()` | **Stub:** only loads stored hash; comment says live apps must call `noteAccountIdentity` from CK account status callbacks |
| Zone deleted (`fetchedDatabaseChanges` deletions for zone) | `wipeAndRebootstrap(preserveLocalUserData: false)` — **wipes user synced tables** |

`wipeAndRebootstrap`:
- Optionally `RowSQL.deleteAllSyncedData` for all `syncedTables`
- `metadata.wipeAll()` (meta, state, cursor, account hash)
- Clears engine / local pending; `start()` again
- If preserving data → `enqueueAllLocalRows()`

**PROD-BLOCK:** Account detection is not automatic. Apps that never call `noteAccountIdentity` miss crash-safe rebootstrap across process death (BD-007 path only partially wired). `detectAccountIdentityChangeIfNeeded` in adapter `start()` does not perform a live iCloud identity probe.

### Batching

| Knob | Default | Cap / notes |
|------|---------|-------------|
| `drainCDCLimit` | 500 | Per `drainCDC` call; large writes need multiple drains |
| `maxBatchSize` | 250 | Hard-capped to CloudKit 250 in configuration init |
| `pendingBatches()` | — | Splits current pending into ≤ maxBatchSize chunks (test/diagnostics) |
| `nextRecordZoneChangeBatch` | — | Live send path prefixes pending to maxBatchSize |

QA checklist expects ≥600 local inserts → outbound ≤250/batch, CDC steps ≤500.

### Multi-table

- Config holds `[SyncedTable]`; drain filters by name set.
- Each table → own record type (default table name) and `table:pk` names.
- `enqueueAllLocalRows` iterates all synced tables.
- Zone wipe deletes data from **all** synced tables when not preserving user data.

---

## 4. Test coverage present vs missing

**Harness:** `/Users/tuliopinheirocunha/Developer/BoutiqueDB-Swift/Tests/TursoCKSyncTests/TursoCKSyncTests.swift`  
**Mode:** almost exclusively `enablesCloudKit: false` (documented in QA checklist).  
**Extra:** `Tests/BoutiqueDBTests/RefinementStressTests.swift` lightly drains CDC with LiveQuery stress.

### Present (automated)

| Area | Test(s) |
|------|---------|
| Metadata migrate / cursor / record meta | `metadataAndStatePersistence` |
| Outbound CDC → pending + `makeRecord` | `outboundCDCDrainBuildsPendingChanges` |
| Inbound upsert + echo suppression + remote delete | `inboundApplyAndEchoSuppression` |
| System fields round-trip on rebuild | `systemFieldsRoundTrip` |
| Wipe without preserve (user data deleted) | `accountWipeRebootstrap` |
| Simulated A→B insert + delete | `simulatedTwoDeviceRoundTrip` |
| Multi-table pending names | `multiTableSyncRoundTrip` |
| Batching 600 rows / max 250 | `pendingBatchesRespectMaxBatchSize` |
| LWW conflict (local newer) | `lastWriterWinsConflict` |
| Server wins conflict | `serverWinsConflict` |
| Account hash change preserves rows | `accountHashChangePreservesLocalData` |
| Preserve wipe re-enqueues | `wipePreservingDataReenqueues` |
| Adapter status stream + start/drain | `cloudKitSyncAdapterStatusStream` |
| Observation CDC generation (TursoStore) | `observationInvalidatesOnCDC` |

### Missing or weak (relative to QA checklist / prod)

| Gap | Severity |
|-----|----------|
| **Live CloudKit** (entitlements, two devices, real `CKSyncEngine`) — checklist only | **PROD-BLOCK** (manual) |
| **`.clientWins` conflict** automated test | Missing |
| LWW when **server newer** (should apply server) | Missing (only local-newer case) |
| Concurrent edit race under real send path | Missing (test hook only) |
| Network mid-sync / retry / no duplicate PKs | Manual only |
| Zone deletion event path | Logic exists; not driven by synthetic CK events offline |
| `needsAuthentication` / not signed in | Unimplemented + untested |
| `BoutiqueDBSyncEngine` façade | No dedicated tests |
| Auto-drain after `BoutiqueDB.write` | N/A (feature missing) |
| Multi-table **inbound** apply for two record types on peer | Outbound pending only |
| Drain pagination loop for >500 CDC rows until empty | Partially implied by batch test; no explicit multi-drain API test |
| Integer PK / non-TEXT PK tables | Coverage biased to TEXT UUID PK |
| Benchmarks tables in `docs/Sync-Benchmarks.md` | Empty placeholders |
| `applyRemoteChanges` multi-batch via adapter | Not covered beyond status test |

---

## 5. Prod Apple app integration checklist gaps

Map of **required production work** vs what the package provides. Items marked **PROD-BLOCK** block a real App Store multi-device launch if left undone.

### Entitlements & packaging

| Checklist item | Package support | Gap |
|----------------|-----------------|-----|
| iCloud + CloudKit capability; container ID = `TursoCKSyncConfiguration.containerIdentifier` | Config accepts identifier; default dev id `iCloud.com.turso.cloudkit.dev` if nil | **PROD-BLOCK:** wrong/missing container traps or silent wrong DB |
| Embed `libturso_sqlite3` with CDC | `BoutiqueDB` / build scripts | Ensure release app does not strip CDC; vendor path in Package.swift |
| `enablesCloudKit: true` | Default true on `BoutiqueDBSyncEngine` | Tests always false — easy to ship a test-only mental model |
| Min OS iOS 17 / macOS 14 for `CKSyncEngine` | Design BD-006 | Document in app target deployment target |

### Runtime wiring (not in App-Template)

| Step | Status |
|------|--------|
| Open DB with `enableCDC: true` (default) | OK |
| **Never** enable CDC + MVCC on same connection; use `concurrentWrites` for second handle | Documented Architecture / README |
| Construct `BoutiqueDBSyncEngine` / `CloudKitSyncAdapter` with `syncedTables` matching schema | **App responsibility** |
| Call `start(automaticallySync: true)` (or adapter `start()`) | **App responsibility** |
| After every local commit path: `drainLocalChanges()` / `drainCDC()` or use `performLocalWrite` | **PROD-BLOCK** if forgotten — changes sit only in CDC |
| UI bind to `syncStatus()` | Status partial; no `needsAuthentication` |
| Call `noteAccountIdentity` from `CKContainer.accountStatus` / userRecordID | **PROD-BLOCK** — `detectAccountIdentityChangeIfNeeded` is a no-op probe |
| Handle `.accountChanged` / rebootstrap UX | Status exists; app must preserve UX for local data |
| Optional: pull-to-sync / timer drain for large backlogs | Not built-in |
| Conflict policy choice per product (default serverWins) | Config only; document for users |

### Manual QA (from `docs/CloudKit-QA-Checklist.md`)

Still required before release:

1. Two devices same iCloud — insert / edit / delete ≤30s  
2. Multi-table both record types  
3. Sign-out / switch account → local preserved, re-upload  
4. Network off mid-sync → failed/retry, no corruption, no duplicate PKs  
5. ≥600 rows batching  
6. Conflict policy spot-checks for all three policies  
7. Sign-off table (empty in checklist)

### Façade vs engine API mismatches

| `BoutiqueDBSyncEngine` | Underlying gap |
|------------------------|----------------|
| `start(automaticallySync:)` is **sync throws**, not `async` like `SyncAdapter.start` | Apps using protocol vs façade differ |
| No `stop`, `applyRemoteChanges`, `noteAccountIdentity`, `wipeAndRebootstrap` on façade | Apps must reach `adapter.engine` for account/prod recovery |
| Hard-codes `drainCDCLimit: 500` in init | Cannot tune without constructing adapter/config manually |

### Architecture doc vs code

- Architecture diagram lists `SyncAdapter` under `BoutiqueDB` — correct product-wise, but **not auto-attached** to open.
- Design checklist marks `BoutiqueDBSyncEngine` and `SyncAdapter` done; **integration completeness** (auto-drain, account probe, app template, live CI) remains open for production.

---

## Prod-block summary

1. **No automatic CDC drain** after normal `BoutiqueDB` writes — silent data never leaves device.  
2. **Account identity** depends on app-called `noteAccountIdentity`; start-time detect is a stub.  
3. **`needsAuthentication` never emitted** — incomplete status model for unsigned users.  
4. **No live CloudKit automated tests** — only offline queue; two-device/network/zone QA is manual.  
5. **App template omits sync entirely** — high risk of incomplete integration.  
6. **Façade incomplete for account wipe / stop / identity** — easy to miss BD-007 paths.  
7. **`.clientWins` and full LWW matrix untested** — policy bugs ship if used in prod without manual QA.

---

## Source index

| Path | Contents |
|------|----------|
| `BoutiqueDB-Swift/Sources/TursoCKSync/SyncAdapter.swift` | Protocol, status, adapter |
| `BoutiqueDB-Swift/Sources/TursoCKSync/TursoCKSyncEngine.swift` | CDC, apply, delegate, conflicts, account |
| `BoutiqueDB-Swift/Sources/TursoCKSync/SyncMetadataStore.swift` | Schema + persistence |
| `BoutiqueDB-Swift/Sources/TursoCKSync/SyncedTable.swift` | Tables, identity, config, policies |
| `BoutiqueDB-Swift/Sources/TursoCKSync/RecordMapper.swift` | CKRecord mapping |
| `BoutiqueDB-Swift/Sources/TursoCKSync/RowSQL.swift` | Local SQL mutations |
| `BoutiqueDB-Swift/Sources/BoutiqueDB/BoutiqueDBSyncEngine.swift` | App-facing wrapper |
| `BoutiqueDB-Swift/docs/CloudKit-QA-Checklist.md` | Manual RC checklist |
| `BoutiqueDB-Swift/Tests/TursoCKSyncTests/TursoCKSyncTests.swift` | Offline unit coverage |
