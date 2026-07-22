# BoutiqueDB fork CI policy (Apple / Swift only)

This repository is a **fork of Turso used as the engine for BoutiqueDB-Swift**.

## Scope

| In scope | Out of scope |
|----------|----------------|
| Rust engine / `sdk-kit` correctness | Publishing to NuGet, Maven, npm, crates.io |
| Building staticlibs for Apple | .NET / Java / Python / Go / NAPI / RN bindings CI |
| Conformance useful for engine quality | Multi-language package registries |

The **public Swift package** lives in  
[`tuliopc23/BoutiqueDB-Swift`](https://github.com/tuliopc23/BoutiqueDB-Swift)  
and is the only surface we publish for app integration (SPM / SPI).

## Removed workflows (do not re-add)

- `dotnet-publish.yml`, `dotnet-test.yml`
- `java-publish.yml`, `java.yml`
- `python.yml`, `go.yml`, `napi.yml`, `react-native.yml`
- `publish-cli-npm.yml`, `publish-crates.yml`
- cargo-dist `release.yml`, `turso-serverless.yml`

Agents: **never** restore language-binding publish workflows or registry publish jobs on this fork.
