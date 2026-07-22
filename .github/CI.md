# BoutiqueDB fork CI policy (Apple / Swift only)

This repository is a **Turso fork used only as the engine for BoutiqueDB-Swift**.

## Product surface

| Repo | Role |
|------|------|
| [`BoutiqueDB-Swift`](https://github.com/tuliopc23/BoutiqueDB-Swift) | **Only** public package (SPM / SPI, iOS + macOS) |
| This repo (`BoutiqueDB`) | Engine source (`sdk-kit`) for multi-arch `TursoSDK.xcframework` |

## Active workflows

- `sdk-kit-apple.yml` — path-filtered build of `libturso_sdk_kit` for Apple

## Never re-add

- Language binding tests/publish: .NET/NuGet, Java/Maven, Python, Go, NAPI, RN
- crates.io / npm CLI publish, cargo-dist release, serverless publish
- Upstream Turso bench/fuzz/antithesis/codspeed/elle/conformance matrices on this fork

Agents: keep CI minimal and Apple-focused. Do not restore multi-language registry jobs.
