# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What kache is

A zero-copy, content-addressed build cache distributed as a **single `kache` binary** that is multi-modal:
- A `RUSTC_WRAPPER` for Rust compiles.
- A `cc` / `c++` compiler wrapper for C/C++ object compiles (recognizes `cc`/`c++`/`gcc`/`g++`/`clang`/`clang++` + versioned variants, and `clang-cl` / `--driver-mode=cl` on any OS).
- A CLI for cache management (`kache init`, `monitor`, `stats`, `gc`, `sync`, …) plus a background daemon.

Cache keys are blake3 hashes of normalized compiler inputs. Hits restore **zero-copy**: reflink (CoW clone) where the FS supports it (APFS, btrfs, XFS-w/-reflink, ReFS on Windows), else hardlink, else copy. Identical blobs are stored once. Local Rust + local C/C++ caching and direct S3 sync (Rust artifacts only) work today; C/C++ is local-only.

## Build & dev commands

The repo uses **`just`** as its single task runner — prefer it over raw `cargo`. `mise.toml` pins the toolchain (Rust **1.95+ MSRV**, edition 2024) and `just`.

```sh
just check          # fmt-check + lint + test — run before every PR
just ci             # mirrors CI: fmt-check + lint + image-service-print + helm-lint + coverage
just test           # cargo test --workspace
just lint           # cargo clippy --workspace --all-targets -- -D warnings (warnings are errors)
just fmt            # cargo fmt --all
just fix            # auto-fix fmt + clippy
just build          # cargo build --release
just coverage       # cargo-llvm-cov; CI enforces a JSON-threshold gate
```

Run a single test: `cargo test --workspace <test_name>` or target one file, e.g. `cargo test --test cache_key_underkeying_test`.

**CRITICAL — bootstrapping:** The `Justfile` exports `RUSTC_WRAPPER := ""` so kache never builds itself through kache. Any new recipe that compiles must keep this. At runtime, a re-entrancy guard (`KACHE_ACTIVE` env, set in `run_wrapper_mode`) breaks nested `kache → cc → kache` loops.

### E2E and benchmarks

`kache-scenario` (in `crates/kache-e2e`) is the harness. Scenarios live in `scenarios/` as `scenario.toml` files (`e2e-*` = correctness fixtures, `bench-*` = benchmark workloads). Each scenario builds a project **twice against one shared cache** — cold build in one checkout, warm build in a second checkout at a *different absolute path* — to exercise cross-clone key stability.

```sh
just e2e                 # run e2e fixture scenarios (suite:e2e tier:gate), assert contracts vs `kache report --json`
just e2e-docker [ARGS]   # CI-equivalent e2e inside a Linux container (needed because host clang ≠ CI clang)
just bench [PROFILE]     # list/run kache benchmark profiles (firefox, substrate, llvm); slow, tens of GB
just bench-sccache PROFILE  # same shape with sccache for comparison
just bench-trace PROFILE    # also diff per-crate key inputs across clones → key-diff.{json,md}
```

## Architecture

### Wrapper hot path (the core value)
`src/main.rs` `run_wrapper_mode` is entered when arg[1] looks like a compiler path. It guards re-entrancy, loads `Config`, intercepts the `cc`-crate family probe (`kache -E <file>`), then dispatches via `compiler::detect_compiler`:
- `RUSTC_ID` → `wrapper::run` (`src/wrapper.rs`)
- `CC_ID` → `wrapper::run_cc` (`src/wrapper.rs`)

The flow: parse args → compute blake3 key (`src/cache_key.rs`) → look up in `Store` → on hit, restore artifacts zero-copy (`src/link.rs`); on miss, run the real compiler (`src/compile.rs`), then record outputs into the store. Incremental compilation is disabled while wrapping (artifact caching replaces it; also avoids APFS corruption).

### Compiler abstraction — `src/compiler/`
A `Compiler` trait (`mod.rs`) with two impls:
- `rustc.rs` — `RustcCompiler`
- `cc.rs` — `CcCompiler` (the largest module; `CcArgs` parsing, gcc/clang vs clang-cl handling, depinfo)
- `flags.rs` — flag classifier (uses linear-time `regex`, no backtracking — runs per cc argument, so bounded cost is a requirement). **Conservative by design: any flag kache hasn't classified causes passthrough/refusal rather than risk a wrong artifact.**
- `platform.rs` — target/arch identity.

### Storage & keys
- `src/store.rs` — SQLite-indexed content-addressed blob store at `{cache_dir}/store/blobs/<prefix>/<hash>`; `EntryMeta`, dup detection (`StorePutResult`).
- `src/cache_key.rs` — deterministic blake3 key over rustc/compiler version, crate name, source, deps, normalized flags. Portable across machines.
- `src/path_normalizer.rs` — strips machine/worktree-specific paths so keys are cross-checkout stable (gcc/clang via `-ffile-prefix-map`; clang-cl keys are machine-local). `KACHE_BASE_DIR` extends the stripped root; `KACHE_CC_PATH_NORMALIZE=0` is the literal-path escape hatch.
- `src/link.rs` — reflink/hardlink/copy strategies per platform.
- `src/extra_inputs.rs` — extra file inputs folded into the key.

### Daemon & remote — `src/daemon.rs`, `src/remote*.rs`
The optional background daemon (launchd/systemd via `src/service.rs`) does async S3 uploads, remote checks, and prefetch over an `interprocess` IPC socket. Local caching works without it (degrades gracefully). `src/remote.rs` + `src/remote_layout.rs` do S3 (AWS/Ceph/MinIO/R2). Prefetch planning: `src/probe/`, `src/build_intent.rs`, `src/planner_client.rs`, `src/remote_plan.rs`, `src/fallback_planner.rs`.

### CLI, config, TUI
`src/cli.rs` (subcommand impls), `src/config.rs` (env > config file > `.kache.toml` project-local), `src/config_tui.rs` (config editor), `src/tui.rs` (`kache monitor` live dashboard, ratatui), `src/report.rs` (text/json/markdown/github/perfetto/chrome-trace output), `src/events.rs` (`BuildEvent`/`EventResult`).

### Workspace crates
- `crates/kache-core` — shared planner types (`BuildIntent`, `PrefetchPlan`, `PlannerDataSource`), behind a `planning` feature.
- `crates/kache-service` — **PREVIEW** remote planner HTTP service (embedded SurrealDB, `charts/kache-service` Helm chart, `kunobi-auth`/`kunobi-ha` integration).
- `crates/kache-e2e` — the `kache-scenario` benchmark/e2e harness (`bench_runner.rs`, `runner.rs`, `assertions.rs`).

**Workspace boundary gotcha:** `scenarios/e2e-*/source` and `test-projects/*` each declare their own empty `[workspace]` so cargo treats them as standalone consumers. Don't add them to the root `members` list.

## Conventions & gotchas

- **TLS = `ring`, never `aws-lc-rs`** (`aws-lc-sys` hangs cross-compiling to `aarch64-pc-windows-msvc`). Every rustls consumer must opt out of aws-lc and avoid re-enabling it via additive features. See the long comment block in `Cargo.toml` before touching any AWS/reqwest/rustls dep or feature. The process-default ring `CryptoProvider` is installed at startup in `src/main.rs`.
- **`warnings = "deny"`** at the crate level — fix all warnings.
- Error handling: `anyhow::Result` with `.context()`; avoid bare `.unwrap()` on I/O/network. Document `unsafe` with `// SAFETY:` comments.
- Unit tests: `#[cfg(test)]` at the bottom of source files. Integration tests in `tests/` run real binaries against temp dirs.
- Trunk-based: all work lands on `main` via PRs; conventional-commit messages (`feat:`, `fix:`, `test:`, `docs:`). Releases tagged on `main`.
- Versioning: bump every workspace manifest at once with `just bump X.Y.Z` (use dotted prereleases like `0.5.0-rc.4`), then `just release` to cut the tag — never hand-draft the GitHub release (CI gates publish).
