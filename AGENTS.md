# AGENTS.md

Agent instructions for this repository. This file records the locked decisions of the
approved `varda-libstack` initiative (pin authority: `.prime/libstack-research/consolidated.md`);
it records decisions, it does not make them.

## What this repo is

Varda is a Rust-based, multi-agent LLM harness, and this repository is docs-first: the
four crates in the workspace — `varda-core`, `varda-daemon`, `varda-node`, `varda-tui` —
are cargo scaffolds, and the design lives in `README.md` and `docs/philosophy/`, which
are the spec the crates get built to.

## Vendored dependencies (`libs/`)

All 29 third-party dependencies are vendored as git submodules under `libs/` (23
distinct repos), each pinned to an exact commit. The table below is the manifest —
crate → `libs/` path → version @ tag — built verbatim from the locked pin table (plan
Appendices A and C); where a repo covers several crates (`tracing`, `bollard`, `turso`,
`rig`, `msgpack-rust`), each crate gets its own row, and the tag is the tag the pin is
referenced from, never a tracked branch.

| crate | `libs/` path | version @ tag |
|---|---|---|
| `anyhow` | `libs/anyhow` | 1.0.104 @ `1.0.104` |
| `bollard` | `libs/bollard` | 0.21.1 @ `v0.21.1 (tag mislabeled - pin = release commit)` |
| `bollard-stubs` | `libs/bollard/codegen/swagger` | 1.53.1-rc.29.3.1 @ `v0.21.1 (tag mislabeled - pin = release commit)` |
| `chrono` | `libs/chrono` | 0.4.45 @ `v0.4.45` |
| `clap` | `libs/clap` | 4.6.6 @ `v4.6.6` |
| `comfy-table` | `libs/comfy-table` | 8.0.0 @ `v8.0.0` |
| `crossterm` | `libs/crossterm` | 0.29.0 @ `0.29` |
| `indoc` | `libs/indoc` | 2.0.7 @ `2.0.7` |
| `owo-colors` | `libs/owo-colors` | 4.3.0 @ `v4.3.0` |
| `rand` | `libs/rand` | 0.10.2 @ `0.10.2` |
| `ratatui` | `libs/ratatui/ratatui` | 0.30.2 @ `ratatui-v0.30.2` |
| `rig` | `libs/rig` | 0.41.0 @ `v0.41.0` |
| `rig-agent` | `libs/rig/crates/rig-agent` | 0.41.0 @ `v0.41.0` |
| `rig-core` | `libs/rig/crates/rig-core` | 0.41.0 @ `v0.41.0` |
| `rig-derive` | `libs/rig/crates/rig-derive` | 0.41.0 @ `v0.41.0` |
| `rmp` | `libs/msgpack-rust/rmp` | 0.8.15 @ `— (commit pin)` |
| `rmp-serde` | `libs/msgpack-rust/rmp-serde` | 1.3.1 @ `— (commit pin)` |
| `serde` | `libs/serde/serde` | 1.0.229 @ `v1.0.229` |
| `thiserror` | `libs/thiserror` | 2.0.20 @ `2.0.20` |
| `tokio` | `libs/tokio/tokio` | 1.53.1 @ `tokio-1.53.1` |
| `tracing` | `libs/tracing/tracing` | 0.1.44 @ `tracing-subscriber-0.3.23 (covering commit)` |
| `tracing-subscriber` | `libs/tracing/tracing-subscriber` | 0.3.23 @ `tracing-subscriber-0.3.23 (covering commit)` |
| `tui-input` | `libs/tui-input` | 0.15.4 @ `v0.15.4` |
| `tui-markdown` | `libs/tui-markdown/tui-markdown` | 0.3.9 @ `tui-markdown-v0.3.9` |
| `tui-tree-widget` | `libs/tui-rs-tree-widget` | 0.24.1 @ `v0.24.1` |
| `turso` | `libs/turso/bindings/rust` | 0.7.2 @ `v0.7.2` |
| `url` | `libs/rust-url/url` | 2.5.8 @ `v2.5.8` |
| `uuid` | `libs/uuid` | 1.24.1 @ `v1.24.1` |
| `zenoh` | `libs/zenoh/zenoh` | 1.10.0 @ `1.10.0 (annotated)` |

Rules:

- The source of any imported third-party crate lives in `libs/<repo>` — read it there.
- In-scope deps are path-based, not registry (transitive deps resolve from crates.io as usual).
- `libs/` is pinned and read-only — never edit vendored source, never "fix" a pin; surface pin problems instead.
- A pin may only change by updating BOTH this table and the submodule commit together (user authority).
- Submodules are pinned by commit, not branch (no `branch` key in `.gitmodules`).
- New dependencies require an entry in this table first.

## Workspace rules

All deps are declared exactly once in the `[workspace.dependencies]` of the workspace root manifest (`app/Cargo.toml`) as
`{ path, version, features?, default-features? }`. Member manifests reference each dep
with `{ workspace = true }` only — no path, version, or features in members.
`Cargo.lock` is committed and changes in the same commit as the manifest it locks.

## Queue design (locked)

The queue is **Eclipse Zenoh 1.10.0**: its in-process router is embedded in
`varda-daemon` (`varda-node` and `varda-tui` connect as sessions). Transport is
`transport_tcp` only — no external broker, no container. The queue is transient (no
queue durability): the `.varda/` database (`turso`) is the system of record, and the
DB sequence carries order. Runtime knobs (max_sessions 10000, `CongestionControl::Block`)
live in daemon config, not this doc.

## Persistence (locked)

Persistence is **`turso` 0.7.2** (the Turso Rust API; `bindings/rust`, defaults + `sync`).
The local `.varda/` file DB is primary, with remote Turso cloud sync as a local-first
push/pull target. At-pin caveats (accepted at the pin): FTS is Turso-native — the
tantivy `USING fts` index, experimental and feature-gated — not SQLite FTS5; vector
search is exact over BLOB, with dense ANN on the roadmap.

## Git discipline

- Work in the feature worktree, never on `main` — `main` is the frozen base and is never written, reset, or moved.
- Commits are unsigned: `git commit --no-gpg-sign`, with lowercase-scope subjects (`<lowercase-scope>: <imperative summary>`).
- Push only when the maintainer says.
- Planning artifacts live under `.prime/` at the bare-repo root (outside every worktree) and are never committed.
