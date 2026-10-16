# Repository instructions

`cache-kit` — a type-safe, fully generic, async caching framework for Rust
(crate `cache-kit`, published to crates.io; MIT; edition 2021; Minimum Supported
Rust Version (MSRV) 1.75). Async-only: every cache path needs a `tokio` runtime.

## Commands

The command table in `.orly/orly.json` owns `conform` (`make dev` — `cargo fmt` +
`clippy`), `verify.unit` (`make test`), and `verify.build` (`make build`).
Repo-specific additions:

- `make dev` — format + lint; the daily pre-commit loop.
- `make up` / `make down` — start/stop local Redis + Memcached. **Required
  before** any `redis`/`memcached`-feature test; without it those lanes fail to
  connect rather than skip.
- `make test FEATURES="--features redis"` (or `--all-features`) — scope tests to
  a backend feature set. Integration/database lanes run single-threaded
  (`cargo test -- --test-threads=1`).
- `make perf` / `perf-save` / `perf-diff` — criterion benches + Hypertext Markup
  Language (HTML) report and baseline comparison.
- `make version-bump VERSION=x.y.z` then `make release` — bump the version
  everywhere, then build/test/audit/publish to crates.io. Never hand-edit the
  version in one place.

## Terminology

- **Entity** — any cached type `T: CacheEntity`. **Repository** — a backend data
  source implementing `DataRepository`. **Backend** — the store: `InMemory`
  (default), `Redis`, `Memcached`.
- Backend selection is compile-time via Cargo features — `inmemory` (default),
  `redis`, `memcached`, `all`. Switching backends is a feature change, never a
  code change.
- Modules: `backend/{inmemory,redis,memcached}`, `entity`, `repository`,
  `service`, `strategy`, `key`, `expander`, `feed`, `serialization`,
  `observability`, `error`.

## Architecture triggers

- Editing `src/backend/**` → the change must hold for every backend behind its
  feature gate; build the cross-backend example
  (`cargo build --example multiple_backends --features redis,memcached`).
- Editing `src/serialization/**` or an entity's wire shape → `postcard` is the
  default codec; `rust_decimal::Decimal` needs custom serialization — round-trip
  test it.
- Adding or altering a Cargo feature → test the combination, not only
  `--all-features`; Continuous Integration (CI) builds per-feature examples.

## Local safety rules

- **Async-only** — no blocking cache calls; everything is `tokio`.
- **`InMemory` is single-instance** — never a production or multi-node store;
  use Redis/Memcached for anything shared.
- Public types are `Send + Sync`; preserve that guarantee when adding fields.
- `redis`/`memcached` tests need `make up` first; do not treat connection
  failures as skips.

<!-- orly:begin -->
**Engineering harness:** read [`.orly/AGENTS.md`](.orly/AGENTS.md) as well — it carries the safety rules,
the dispatch router that names which rule page to read before which edit, and the lifecycle
this repository gates on. Where the two disagree, this file wins.

The line below is an import, not decoration: a runtime that resolves it loads those rules
with this file, and one that does not still has the link above.

@.orly/AGENTS.md
<!-- orly:end -->
