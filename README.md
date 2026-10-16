# cache-kit

**Typed, read-through caching for Rust — one code path over in-memory, Redis, and Memcached.**

[![Crates.io](https://img.shields.io/crates/v/cache-kit?style=flat-square)](https://crates.io/crates/cache-kit)
[![Docs.rs](https://img.shields.io/docsrs/cache-kit?style=flat-square)](https://docs.rs/cache-kit)
[![Documentation](https://img.shields.io/badge/guide-cachekit.org-blue?style=flat-square)](http://cachekit.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=flat-square)](https://opensource.org/licenses/MIT)
[![Rust Version](https://img.shields.io/badge/rust-1.75%2B-blue?style=flat-square&logo=rust)](https://www.rust-lang.org)
[![CI](https://github.com/agentsfleet/cache-kit.rs/actions/workflows/ci.yml/badge.svg)](https://github.com/agentsfleet/cache-kit.rs/actions/workflows/ci.yml)
[![codecov](https://codecov.io/gh/agentsfleet/cache-kit.rs/branch/main/graph/badge.svg)](https://codecov.io/gh/agentsfleet/cache-kit.rs)

Cache any `serde` type behind a small trait set and a strategy enum, so the
read path reads the same whether the store is process memory or a shared
backend. Database-agnostic: anything implementing `DataRepository` plugs in.

---

## Features

- **Read-through by strategy** — `Fresh`, `Refresh` (default), `Invalidate`, `Bypass`: cache behavior is an explicit enum, not boolean flags.
- **Backend-agnostic** — InMemory (default), Redis, Memcached. Switch by Cargo feature, never by code change.
- **Typed, versioned payloads** — postcard codec under a magic + schema-version envelope; a version bump evicts stale entries instead of misreading them.
- **Compile-time keys** — entity key and prefix come from the `CacheEntity` trait, verified by the compiler.
- **Thread-safe** — `Send + Sync` throughout; async-only on `tokio`.
- **Observable** — metrics hook plus per-prefix time-to-live (TTL) policies.

## Requirements

- Rust 1.75+ (edition 2021) and a `tokio` runtime — every cache path is async.
- Redis or Memcached for anything shared: `InMemory` is single-instance only.
- `rust_decimal::Decimal` fields need custom serialization (see the [docs](http://cachekit.org/)).

## Quick Start

```toml
[dependencies]
cache-kit = { version = "0.9", features = ["inmemory"] }
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
```

The full read-through pattern — entity, feeder, repository, strategy — in one file:

```rust
use cache_kit::{
    CacheEntity, CacheFeed, DataRepository, CacheExpander,
    backend::InMemoryBackend,
    strategy::CacheStrategy,
};
use serde::{Deserialize, Serialize};

// The cached type. Key and prefix are compile-time.
#[derive(Clone, Serialize, Deserialize)]
struct User {
    id: String,
    name: String,
}

impl CacheEntity for User {
    type Key = String;
    fn cache_key(&self) -> Self::Key { self.id.clone() }
    fn cache_prefix() -> &'static str { "user" }
}

// The receiver the cache operation feeds its outcome into.
struct UserFeeder {
    id: String,
    user: Option<User>,
}

impl CacheFeed<User> for UserFeeder {
    fn entity_id(&mut self) -> String { self.id.clone() }
    fn feed(&mut self, entity: Option<User>) { self.user = entity; }
}

// The source of truth the cache sits in front of.
struct UserRepository;

impl DataRepository<User> for UserRepository {
    async fn fetch_by_id(&self, id: &String) -> cache_kit::Result<Option<User>> {
        // Fetch from your database
        Ok(Some(User { id: id.clone(), name: "Alice".to_string() }))
    }
}

#[tokio::main]
async fn main() -> cache_kit::Result<()> {
    let expander = CacheExpander::new(InMemoryBackend::new());
    let mut feeder = UserFeeder { id: "user_001".to_string(), user: None };

    // Cache hit: returns cached. Miss: fetches, caches, returns.
    expander.with(&mut feeder, &UserRepository, CacheStrategy::Refresh).await?;

    if let Some(user) = feeder.user {
        println!("User: {}", user.name);
    }
    Ok(())
}
```

## Cache Strategies

| Strategy              | Behavior                   | Use case                     |
| --------------------- | -------------------------- | ---------------------------- |
| **Refresh** (default) | Try cache, fallback to DB  | Most reads                   |
| **Fresh**             | Cache only, no DB fallback | Data must be cached          |
| **Invalidate**        | Clear cache, fetch from DB | After mutations              |
| **Bypass**            | Skip cache, always use DB  | Testing, temporary disable   |

## Backends

Selected by Cargo feature: `inmemory` (default), `redis`, `memcached`, or `all`.

```rust
use cache_kit::backend::InMemoryBackend;
let expander = CacheExpander::new(InMemoryBackend::new());
```

```toml
# Redis
cache-kit = { version = "0.9", features = ["redis"] }
# Memcached
cache-kit = { version = "0.9", features = ["memcached"] }
```

```rust
use cache_kit::backend::{RedisBackend, RedisConfig};
let backend = RedisBackend::new(RedisConfig::default())?;
let expander = CacheExpander::new(backend);
```

## Observability

Logging via the `log` crate, a `CacheMetrics` hook, and TTL policies:

```rust
use cache_kit::observability::TtlPolicy;
use std::time::Duration;

let expander = CacheExpander::new(backend)
    .with_ttl_policy(TtlPolicy::Fixed(Duration::from_secs(300)));
```

## Examples

```bash
cargo run --example basic_usage
cargo run --example multiple_backends --features redis,memcached
cargo run --example advanced_builder
```

Full integration apps live under `examples/` (Actix + SQLx, Axum + gRPC).

## Testing

```bash
make up                                # local Redis + Memcached
make test FEATURES="--all-features"
```

## Publishing

```bash
make version-bump VERSION=x.y.z && make release
```

Builds, tests, audits, and publishes to crates.io. For the daily loop: `make dev`.

## Contributing

Contributions welcome — see [CONTRIBUTING.md](CONTRIBUTING.md). Repository rules
live in `AGENTS.md` + `.orly/`.

## License

MIT — Copyright (c) 2025 Kishore kumar Neelamegam. See [LICENSE](LICENSE).

## Links

[Repository](https://github.com/agentsfleet/cache-kit.rs) ·
[Issues](https://github.com/agentsfleet/cache-kit.rs/issues) ·
[crates.io](https://crates.io/crates/cache-kit) ·
[docs.rs](https://docs.rs/cache-kit) ·
[Guide (cachekit.org)](https://cachekit.org)
