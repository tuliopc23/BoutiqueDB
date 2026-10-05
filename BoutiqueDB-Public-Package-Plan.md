# Plan: Public-ready BoutiqueDB package (SPI format) before app integration

## Status (2026-07-22)

| Phase | Status |
|-------|--------|
| P0 metadata | **Done** (LICENSE, NOTICE, .spi.yml, README, Publishing.md) |
| P1 binary / no unsafeFlags | **Partial** — macos-arm64 xcframework + path binaryTarget; iOS slices TODO |
| P2 CI | **Scaffolded** — workflows need engine repo access on GitHub |
| P3 tag + SPI | **Pending** — rename repo + tag v0.2.0 |
| P4–P5 docs | **Prod docs exist**; DocC still open |

---

## Goal

Make **BoutiqueDB-Swift** publication-ready as a **Swift Package Index–compatible** package (icon, metadata, GitHub mirror), including **R2.1 / R2.2 / R2.5** (multi-arch binary, drop `unsafeFlags`, CI for `libturso_sdk_kit`, tag + SPI), **then** production docs, **then** fuller docs.  

**Order locked by you:**

```text
1. Bundle / SPI-format package + GitHub mirror metadata
2. R2.1 multi-arch binary / drop unsafeFlags
3. CI cache/build of libturso_sdk_kit
4. Tag + SPI readiness
5. Prod-ready docs (install, open options, CloudKit QA)
6. Broader docs (DocC / guides)
7. Good to go (dogfood app)
```

**Not in this plan:** Live CloudKit field validation (your next app), SampleApp, Zig.

---

## Current reality (blockers for SPI)

| Item | State |
|------|--------|
| Package name / products | `BoutiqueDB` — OK |
| Platforms | iOS 17+, macOS 14+ — OK |
| Icon | `Assets/BoutiqueDB.png`, `icon.png` — present |
| README / CHANGELOG | Present, need SPI-oriented polish |
| LICENSE | **Missing** at package root (required for public) |
| `.spi.yml` | **Missing** |
| Git remote | `github.com/tuliopc23/TursoCloudKit.git` — rename/mirror to **BoutiqueDB** recommended |
| Engine link | `unsafeFlags` + local `Vendor/turso-sdk/lib/libturso_sdk_kit.a` (~**388 MB**) |
| Legacy | Still has `CTursoSQLite3` + `Vendor/turso` (~241 MB) unused by TursoKit |
| CI | macOS `swift test` only; **no** engine build/cache |
| SPI | Will **reject / fail analysis** while `unsafeFlags` remain |

SPI needs a package that **clones cleanly** and builds **without** local absolute-path linker flags. Official approach for native libs: **binary target** (xcframework zip on GitHub Releases) or system library (poor DX).

---

## Architecture for public package

```text
GitHub repo (BoutiqueDB)
  Package.swift
    → products: BoutiqueDB, TursoKit, …
    → binaryTarget: TursoSDK (xcframework zip URL or path)
    → CTursoSDK: thin wrapper / system module over binary
    → TursoKit → CTursoSDK → BoutiqueDB …

Release assets
  TursoSDK-vX.Y.Z.xcframework.zip  (ios-arm64, ios-arm64-simulator, macos-arm64, macos-x86_64 optional)
  checksum in Package.swift

CI
  - cache Rust target dir
  - build sdk-kit for each slice (or matrix)
  - package xcframework
  - swift test (macOS arm64 host slice)
  - on tag: upload release asset + update checksum
```

**Policy (unchanged):** only official sdk-kit surface; no private DatabaseOpts hacks.

---

## Phase P0 — Package identity & SPI metadata (no engine change yet)

**Outcome:** Repo looks like a real public Swift package even before binary lands.

1. **LICENSE**  
   - Add root `LICENSE` (match monorepo MIT or chosen license; align with Turso/engine redistribution terms for the static lib).  
   - Note in README: redistributed `libturso_sdk_kit` / Turso license notice.

2. **GitHub package identity**  
   - Prefer repo name **`BoutiqueDB`** (or `BoutiqueDB-Swift`); update remote / create mirror from `TursoCloudKit` if needed.  
   - GitHub About: description, website, topics (`swift`, `spm`, `sqlite`, `turso`, `swiftui`, `cloudkit`).  
   - Social preview: use `Assets/BoutiqueDB.png` / `icon.png`.  
   - Default branch `main`; protect if desired.

3. **SPI-oriented files**  
   - `.spi.yml` (SPIManifest): document platforms, Swift versions, optional DocC later.  
   - Root `Package.swift` already tools-version 6.1 — keep.  
   - Ensure `README.md` has: badges placeholder, install snippet (`https://github.com/…/BoutiqueDB`), modules table, platforms, license.  
   - `CHANGELOG.md` Keep a Changelog format; prepare Unreleased → 0.2.0 or 1.0.0-rc.1.

4. **Repo hygiene for large binaries**  
   - `.gitignore`: do **not** commit multi-hundred-MB `.a` long-term;  
   - Document: source build via script **or** release binary.  
   - During transition: either LFS or “release-only artifacts”.

5. **Remove or isolate legacy**  
   - Drop `CTursoSQLite3` product path from default Package if unused (or mark optional).  
   - Shrink clone size (critical for SPI clone timeouts).

**Exit:** `README` + `LICENSE` + `.spi.yml` + clean story; GitHub page complete. `swift package dump-package` works (still may list unsafeFlags until P1).

---

## Phase P1 — R2.1 Multi-arch binary + drop `unsafeFlags`

**Outcome:** Package builds for consumers with **zero** `unsafeFlags`.

### P1.1 Build script: xcframework

Extend `Scripts/build-turso-sdk-kit.sh` (or `Scripts/build-turso-sdk-xcframework.sh`):

| Slice | Triple / SDK |
|-------|----------------|
| macOS arm64 | `aarch64-apple-macosx` / macosx |
| macOS x86_64 | optional `x86_64-apple-macosx` |
| iOS device | `aarch64-apple-ios` |
| iOS simulator arm64 | `aarch64-apple-ios-sim` |
| iOS simulator x86_64 | optional |

Steps:

1. `cargo build -p turso_sdk_kit --release --features fts,encryption` per target (use `cargo zigbuild` / `cargo-xcode` / `RUSTFLAGS` + apple SDKs — document exact tool on macos-15 runners).  
2. Produce thin static libs per slice.  
3. `xcodebuild -create-xcframework` → `TursoSDK.xcframework`.  
4. Zip + `swift package compute-checksum`.

**Hard part:** Rust → multi-platform Apple static libs. Spike first on macos-arm64 only for binaryTarget path, then add iOS slices.

### P1.2 Package.swift

```swift
// Conceptual — exact URL/version after first release asset
.binaryTarget(
  name: "TursoSDK",
  url: "https://github.com/<org>/BoutiqueDB/releases/download/0.2.0/TursoSDK.xcframework.zip",
  checksum: "<sha256>"
)
// CTursoSDK: either
//  - target depending on TursoSDK binary + headers in Sources/CTursoSDK
//  - or systemModule pattern with binary’s headers
```

- Remove `-L… -lturso_sdk_kit` **unsafeFlags**.  
- Headers: keep `Sources/CTursoSDK/include/turso.h` in git (small).  
- Local dev override: env/`Package.local` or `#if` is awkward in SPM — prefer **always binary** + CI publishes, with script for maintainers to overwrite checksum after build.

**Local maintainer loop:**

```bash
./Scripts/build-turso-sdk-xcframework.sh
# updates Vendor/ or Release/ + prints checksum
# Package.swift uses path-based binaryTarget for local:
# .binaryTarget(name: "TursoSDK", path: "Vendor/TursoSDK.xcframework")
```

SPI and consumers use **URL** binaryTarget; maintainers may use **path** on a `Package.dev.swift` or documented swap — avoid two permanent Package.swift files if possible: **path binary for monorepo**, release script rewrites URL for tag (or use release-only).

**Recommended pattern:**

- `Package.swift` uses `.binaryTarget(..., path: "Vendor/TursoSDK.xcframework")` for reliability offline.  
- SPI accepts path binary targets if the xcframework is **in the repo or released**.  
- **Problem:**  xcframework still large for git.  

**Better for SPI + GitHub:**

- `Package.swift` uses **URL + checksum** only.  
- CI on tag builds xcframework, uploads release, opens PR or commits checksum update.  
- Clone of sources is small; SPM downloads binary separately.

### P1.3 Exit criteria

- [ ] No `unsafeFlags` in `Package.swift`  
- [ ] `swift build` / `swift test` on macOS with binary xcframework  
- [ ] Document iOS device/sim link  
- [ ] `swift package dump-package` clean for SPI  

---

## Phase P2 — CI: cache + build `libturso_sdk_kit` / xcframework

**File:** `.github/workflows/swift.yml` (+ `release.yml`)

### P2.1 PR / main workflow

1. Checkout (BoutiqueDB-Swift; optionally checkout engine monorepo or use submodule/`TURSO_SRC`).  
2. Cache:  
   - `~/.cargo/registry`, `~/.cargo/git`  
   - engine `target/` keyed by `Cargo.lock` + script hash  
3. Job A — **test** (macos-15):  
   - If binary path: download latest successful artifact or use committed path  
   - `swift test`  
4. Job B — **engine** (macos-15, optional on path change):  
   - Install Rust toolchain  
   - `./Scripts/build-turso-sdk-xcframework.sh` (at least macos-arm64 slice for test)  
   - Upload artifact  

### P2.2 Release workflow (on tag `v*`)

1. Build full multi-arch xcframework.  
2. Zip + checksum.  
3. Create GitHub Release with asset.  
4. Commit/PR: update `Package.swift` URL + checksum for that tag (or generate from template).  

### P2.3 Exit criteria

- [ ] CI green without developer machine Vendor  
- [ ] Cache hits documented  
- [ ] Tag pipeline produces xcframework asset  

---

## Phase P3 — Tag + SPI publication readiness

1. Choose version: **`0.2.0`** (framework mature, not “final 1.0” until live CK dogfood) or **`1.0.0-rc.1`**.  
2. CHANGELOG: cut Unreleased → version.  
3. Tag annotated: `git tag -a 0.2.0 -m "…"`.  
4. Push tag → release workflow.  
5. SPI:  
   - Ensure package builds on SPI macOS (and iOS if declared).  
   - Add package via SPI “Add a Package” with GitHub URL.  
   - Optional: DocC generation once docs phase starts.  
6. Verify install:

```swift
.package(url: "https://github.com/<org>/BoutiqueDB.git", from: "0.2.0")
```

### P3 exit

- [ ] Tagged release with binary asset  
- [ ] Package resolves on clean machine  
- [ ] SPI listing submitted / building  

---

## Phase P4 — Prod-ready docs (focus first)

**Minimal public docs set** (do these before long DocC):

| Doc | Content |
|-----|---------|
| README | Install, products, platforms, capability matrix, license, link to open options |
| `docs/Turso-Open-Options.md` | Already exists — polish for consumers |
| `docs/Architecture.md` | Keep; short “for app authors” section |
| `docs/Migrations.md` | Idempotent migrations |
| `docs/CloudKit-QA-Checklist.md` | Offline + live manual steps |
| `docs/App-Template.md` | Safe open pattern |
| `NOTICE` / third-party | Turso, Point-Free deps |

Also: **security** note (experimental encryption, Data Protection).

**Exit:** New consumer can install from GitHub tag and open a DB without reading monorepo audits.

---

## Phase P5 — Broader docs (after prod docs)

1. DocC catalog (`R4.2`): targets BoutiqueDB + TursoKit public API.  
2. SPI DocC hosting (optional `.spi.yml` documentation targets).  
3. Migration guide from sqlite3-era BoutiqueDB if any API break.  
4. Benchmarks page (optional R3.3).  

---

## Phase order (execution)

```text
P0  metadata + LICENSE + GitHub mirror + SPI files + drop legacy bloat from git
P1  multi-arch TursoSDK xcframework + Package.swift binaryTarget (no unsafeFlags)
P2  CI cache/build + release-on-tag
P3  tag + SPI submit
P4  prod-ready consumer docs
P5  DocC / expanded docs
```

---

## Risks & decisions

| Risk | Mitigation |
|------|------------|
| xcframework build for iOS Rust is hard | Spike macos-only binary first; add iOS before SPI iOS claim |
| 400MB in git | **Never** commit fat `.a`; release assets only |
| SPI timeout on huge downloads | Acceptable for binaryTarget; document size |
| License of Turso static link | NOTICE + MIT/Apache as required by Turso |
| Repo rename TursoCloudKit → BoutiqueDB | Redirects on GitHub; update Package URLs |
| Macro + binary packaging | Macros stay source; only engine is binary |

### Decision defaults (unless you override)

| Decision | Default |
|----------|---------|
| Version first public tag | `0.2.0` |
| Binary distribution | GitHub Release xcframework zip + URL binaryTarget |
| GitHub repo name | `BoutiqueDB` (rename or new + archive old) |
| Default product | `BoutiqueDB` library |
| Platforms claimed on SPI | macOS + iOS only after both slices build |

---

## Implementation notes (first sessions after approve)

1. **P0 quick wins** (same day): LICENSE, `.spi.yml`, README install block, `.gitignore` Vendor large libs, GitHub description, update refinement R2 status doc.  
2. **P1 spike**: produce `TursoSDK.xcframework` for **macos-arm64 only**, switch Package to path binaryTarget, green `swift test`.  
3. **P1 full**: iOS slices + zip + checksum.  
4. **P2/P3**: CI + tag.  
5. **P4**: docs pass.  

---

## Success metrics

| Metric | Target |
|--------|--------|
| `unsafeFlags` | **0** in Package.swift |
| Clean machine install | `swift package resolve` + build from tag |
| SPI | Package accepted / building |
| Icon + metadata | GitHub + README + Assets |
| CI | Test without local Vendor; release builds binary |
| Docs | Install + open + CK checklist complete before “good to go” |

---

## Explicit non-goals here

- Live multi-device CloudKit proof (your app)  
- SampleApp target  
- Changing Turso engine semantics  
- Dual sqlite3 + sdk-kit products long-term  

