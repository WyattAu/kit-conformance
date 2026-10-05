# kit-conformance

Shared conformance harness for the WyattAu Rust kit ecosystem.

Every kit in the estate is expected to be boring in the same ways:
`forbid(unsafe_code)`, documented public API, a declared MSRV, no
`unwrap`/`expect` in library code, and a documented semver policy. This repo
holds that expectation in one place so a new kit inherits it on day one
instead of rediscovering standards, and so "the kits are production grade"
is a checkable claim rather than an aspiration.

## What it checks

| Check | Tool | Why |
| --- | --- | --- |
| `forbid(unsafe_code)` | rustc lint | Kits are dependencies of everything; unsafe must be opt-in per crate, never ambient |
| `missing_docs` | rustc lint | A kit's public API is a contract; undocumented items are invisible contract |
| MSRV | `cargo msrv` (or the declared `rust-version`) | Consumers pin toolchains; a silent bump is a breaking change |
| Semver | `cargo semver-checks` | Catches accidental breaking changes before release, not after |
| `unwrap`/`expect` in lib | `clippy::unwrap_used` | A library panic is a consumer outage |
| Doc examples | `cargo test --doc` | Documentation that does not compile is a lie |
| Tests | `cargo test --all-features` | The kit must work in the configuration consumers actually pick |

## Usage

The harness is a composite GitHub Action so a kit repo gets all of it in
three lines:

```yaml
# .github/workflows/conformance.yml
name: conformance
on: [push, pull_request]

jobs:
  kit:
    uses: WyattAu/kit-conformance/.github/workflows/conformance.yml@main
    with:
      crate-path: .
      msrv: "1.85"
    secrets: inherit
```

The workflow expects the kit to satisfy these in its own source:

```rust
#![forbid(unsafe_code)]
#![deny(missing_docs)]
```

and to declare its toolchain floor:

```toml
[package]
rust-version = "1.85"
```

## Why a repo and not a template

A template is copied once and then drifts: six kits later, seven different
definitions of "conforming", none of which anyone remembers updating. A
workflow referenced by tag cannot drift — a fix here reaches every kit on
its next run, and a kit that stops passing is visibly broken rather than
quietly non-conforming.

## Local use

```bash
# what CI will run
cargo test --all-features
cargo clippy --all-targets --all-features -- -D warnings
cargo doc --no-deps --all-features
```

`cargo semver-checks` and `cargo msrv` are optional locally; CI installs
them. Their absence should not block iteration, but a release should not
happen without them.

## Scope boundary

This repo checks *mechanical* conformance. It does not judge whether an API
is a good abstraction — that is what dogfooding in a real consumer
(CivitForge, `flaglab`) is for. Conformance without dogfooding yields
consistently mediocre code; this harness exists so dogfooding is the thing
that varies, not the baseline.