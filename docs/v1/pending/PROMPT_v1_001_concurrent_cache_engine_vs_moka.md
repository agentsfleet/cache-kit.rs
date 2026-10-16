# PROMPT — cache-kit v1 concurrent cache engine: replace moka in production

> **Audience:** Claude or Codex, acting as **CTO**.
> **Owner / approval gate:** Indy (Kishore). The final approach below is presented to Indy for approval **before** any spec is filed.
> **Origin:** read-only review of cache-kit 0.9.0 performed Oct 16, 2026, plus Indy's problem statement, captured here in full.
> **This file is a prompt, not a spec.** The spec is filed by `orly-spec-new` into this folder only after Indy approves the approach.

---

## 1. Problem statement (Indy's intent, captured verbatim)

Indy uses the `moka` crate in production in **two repositories**:

- `~/Projects/agentsfleet`
- `~/Projects/e2e-observability-platform`

He wants to **replace moka** with a **performance-refactored, re-engineered concurrent cache** built inside `cache-kit.rs`, with backend support for **Redis, InMemory, and Dragonfly**, and **benchmarks proving it is more performant than moka** — on in-memory, Redis, and Dragonfly — with a clear explanation of *why* it wins.

The end goal: cache-kit.rs embedded in both repositories in production, then a **second blog article after shipping agentsfleet** — *"we use cache-kit.rs in agentsfleet in production"* — carrying the benchmark numbers. The benchmarks must therefore be **reproducible and publishable**, not a one-off run.

## 2. Verified moka usage today (read from source — re-verify on current HEAD before relying on it)

### 2.1 agentsfleet (`rustd/`, moka 0.12.16, `features = ["future"]`)

| Call site | What it caches | The moka property it depends on |
|---|---|---|
| `rustd/crates/afd_credential/src/credential/broker.rs` | Minted tokens in front of the mints | **Single-flight entry API** — `or_try_insert_with`: one caller resolves, the rest park on the same future; a refusal reaches every waiter and is cached by nobody (two cold misses on a rotating refresh token trip reuse detection and revoke the whole family). LRU (Least Recently Used) capacity bound `MAX_CACHED_TOKENS = 8192` — "a bound on memory rather than a tuning knob." Expiry checked against an **injected clock** (`broker/cache.rs`, `moka::Expiry`). |
| `rustd/crates/afd_identity/src/capability.rs` | Provider capability/scope sets | **Two caches** (hot + stale) — a hand-rolled stale-while-revalidate; hand-written `Clone` so clones share the cache. Also `moka::ops::compute::Op`. |
| `rustd/crates/afd_fleet_lifecycle/src/live_set.rs` | "Which fleets exist in a workspace" for live-stream ticks | **Plain time-to-live (TTL) cache**, deliberately *not* coalesced today; its doc comment names `try_get_with` + the shared-`Arc`-error problem as the blocker for single-flight adoption. |

### 2.2 e2e-observability-platform (`rust/`, moka 0.12, `features = ["future"]`)

- `rust/src/services/auth_service.rs:23` — `pub type TokenCache = moka::future::Cache<String, ValidatedToken>` — auth token cache in front of per-request database hits.
- `rust/src/services/auth_service.rs:65` — `pub type TokenNegativeCache = moka::future::Cache<String, ()>` — negative caching of invalid tokens.
- `rust/src/grpc/auth.rs:40` — the gRPC auth path hits the `TokenCache` first, falls back to the database.
- `rust/benches/auth_benchmark.rs` — an existing benchmark harness to extend, not replace.

**Gap in the prior analysis:** only agentsfleet's usage was analyzed in depth. The e2e-observability-platform embedding (token shapes, negative-cache TTL policy, gRPC hot-path latency budget, multi-tenant fan-out) must be analyzed with the same rigor before the approach is final.

## 3. Prior analysis to review (read-only review of cache-kit 0.9.0)

### 3.1 What cache-kit is today

```
CacheService (Arc wrapper)
   └─ CacheExpander ──── strategy: Fresh / Refresh / Invalidate / Bypass
        │                  retry + backoff, TTL policy per prefix, metrics
        ├─ repository.fetch_by_id()  ← miss path, per caller (no single-flight)
        └─ CacheBackend (bytes: Vec<u8>)
             ├─ InMemory: DashMap<String, {data: Vec<u8>, expires_at: Option<Instant>}>
             ├─ Redis (feature-gated)
             └─ Memcached (feature-gated)
```

Maturity: 0.1.0 → 0.9.0 in six days (CHANGELOG, Dec 25–31, 2025), 193 tests, orly-governed. The framework layer is well-scaffolded; the **in-process engine is the missing organ**.

### 3.2 Gap table (moka vs cache-kit 0.9.0, agentsfleet-verified)

| Capability | moka 0.12 | cache-kit 0.9.0 | Evidence |
|---|---|---|---|
| Single-flight init | `or_try_insert_with` | **absent** — every concurrent miss runs its own fetch | `src/expander.rs:395-412` |
| Bounded eviction | W-TinyLFU, capacity + weights | **absent** — unbounded DashMap | `src/backend/inmemory.rs` |
| TTL that doesn't leak | per-entry expiry + housekeeping | checked on access only; expired-but-unread entries live forever | `src/backend/inmemory.rs:113-121` |
| Injectable clock | `Expiry` against caller clock | `Instant::now()` hardcoded | `src/backend/inmemory.rs:21,26` |
| Cheap reads | hash + `Arc<V>` refcount | full `Vec<u8>` memcpy per get + postcard deserialize per hit | `src/backend/inmemory.rs:116,161`; `src/expander.rs:397` |
| Stale-while-revalidate | rolled by hand by users | absent | — |
| Atomic compute | `ops::compute` | absent | — |
| Typed in-process values | generic `V` | everything is `Vec<u8>` + 8-byte envelope | `src/serialization/mod.rs:12-14` |

### 3.3 Proposed plan (from the review — critique this)

- **P0 — typed fast path.** Split the abstraction: a typed `LocalCache<K, V>` engine (moka-shaped) inside cache-kit; keep the byte-oriented `CacheBackend` for Redis/Memcached only. In-process hit becomes hash + `Arc<V>` refcount instead of serialize + clone + deserialize.
- **P1 — entry API with single-flight.** `get_or_try_insert_with`: one caller runs init, waiters park on a shared future, `Err` propagates to all waiters and is cached by nobody. Publish **both** failure semantics (propagate-to-waiters and per-waiter-retry), because `afd_fleet_lifecycle` rejected shared-`Arc` errors for its error model.
- **P2 — bounded eviction + non-leaking TTL.** Segmented LRU per shard, entry-count and byte-weight bounds (blob sizes already known at `src/backend/inmemory.rs:84`); amortized expiry sweeping on the write path plus an optional sweeper task. W-TinyLFU (frequency-sketch admission) is the stretch goal, not the promise.
- **P3 — injected clock + native stale-while-revalidate.** `Arc<dyn Clock>` through expiry checks; `get_with_stale(key, init, soft_ttl, hard_ttl)` — serve stale, background-refresh past soft TTL, single-flight the refresh. Deletes `afd_identity`'s two-cache pattern.
- **Micro-levers.** `Arc` values instead of per-get clone; `Arc<str>` keys instead of per-set `to_string()` (`inmemory.rs:128`); single shard visit on expired-get (`remove_if`); make `exists()` remove expired like `get()` does; incremental stats counters instead of the O(n) walk; dashmap 5.5 → 6.x.
- **Migration order for agentsfleet:** `afd_fleet_lifecycle` (simplest) → `afd_identity` (needs compute + stale) → `afd_credential` (full entry semantics + injected clock; highest stakes).

### 3.4 Honesty note from the review

moka's moat is W-TinyLFU hit-rates plus a heavily benchmarked buffered-write/housekeeping design. Do **not** promise beating moka universally. The winnable claim: on the workloads these two repositories actually run — small working sets, skewed access, TTL-driven, single-flight-sensitive — a typed engine with `Arc` reads, segmented LRU, and real single-flight should beat moka, and the benchmarks must scope the claim to exactly the lanes they measure.

### 3.5 Pending specs carried forward — `docs/v1/pending/`

Three specs, previously PENDING under v0.9.2 (never started; `VERSION` is still 0.9.0), were relocated into this folder on Oct 16, 2026 via `git mv` — content unchanged, identities retained until the §5 reconciliation decides their disposition. Reviewed at relocation — **all three are relevant to v1; none is dead weight:**

- **`M092_003_SIMPLE_CACHE.md` (P0, May 16, 2026)** — moka-parity entry point: `SimpleCache<K, V, B = InMemoryBackend>` with `get`, `insert`, `insert_with_ttl`, `remove`, `clear`, `get_or_insert_with`, `try_get_or_insert_with`; zero user-implemented cache-kit traits; ≤15 lines to first read; README Quick Start budgeted at 50 lines enforced by `scripts/check-readme-quickstart-budget.sh`. Its adoption diagnosis is exactly v1's user-facing goal (moka: 3 lines, 0 traits; cache-kit today: ~50 lines, 3 traits). **But its implementation surface wraps `CacheExpander<B>` — the byte backend — so it inherits every engine gap in §3.2, and its own gates table doc-states the thundering-herd gap on `get_or_insert_with` instead of fixing it.** v1 must land this API surface *on the new typed engine* (§3.3 P0/P1), not as a wrapper over the old path. Two of its invariants carry over unchanged: the simplicity-first rule (every public method works with zero user-implemented traits, proven by a doc-test with only `Serialize + Deserialize + Clone` derives) and the README budget.
- **`M092_001_ERROR_CODES.md` (P1, May 12, 2026)** — typed error codes distinguishing failure modes programmatically, with backend-source errors separated from validation/serialization errors at the call site (the source proposal's intent is inlined as the spec's Appendix A; `plans/` was retired in this same cleanup). v1's engine error surface — single-flight refusal propagation, eviction, backend-source failures — needs this taxonomy from day one, and both consumer repos carry their own error standards (agentsfleet: `docs/RUST_ERROR_STANDARD.md`, `ErrorKind` + `error_shell!`, `[CODE]` Display). Fold the taxonomy into the engine's error design rather than bolting it on after.
- **`M092_002_OBSERVABILITY.md` (P1, May 16, 2026)** — production observability: `tracing` spans on backend operations (Round-Trip Time (RTT) visibility in distributed traces), optional `metrics`-crate adapter, low-cardinality keys (today's `CacheMetrics` routes raw cache keys, which breaks Prometheus), typed `record_*` methods, and a built-in stats accessor (moka has one; cache-kit doesn't). v1 needs the stats accessor anyway — the §6 benchmark lanes read from it — so design the engine's incremental counters (hits/misses/evictions/bytes, §3.3 micro-levers) as the substrate this spec consumes.

**Disposition to decide in the approach, not execute here:** whether v1 supersedes M092_003 outright (same goal, stronger engine) and whether M092_001 / M092_002 land as parallel v0.9.x workstreams or are absorbed into the v1 spec. The reviewing agent proposes; Indy decides.

### 3.6 Serialization & codec decision — postcard today, and what the field does

**How moka does it: it doesn't.** Direct evidence from Indy's own code: `pub type TokenCache = moka::future::Cache<String, ValidatedToken>` (`e2e-observability-platform/rust/src/services/auth_service.rs:23`) — the cache stores the **typed value** directly, `Arc`-shared. Zero codec cost because moka is in-process only; there are no bytes to write. Other in-process Rust caches (quick_cache, `cached`, stretto) do the same. Hybrid engines (foyer) serialize only when crossing to the disk tier. Redis-family clients (redis-rs, fred) speak bytes on the wire and leave the codec to the caller.

**cache-kit today pays postcard on every path, including in-process.** The envelope is `CKIT` magic (4 bytes) + `u32` schema version + postcard payload (`src/serialization/mod.rs:52-56,70-90`), applied even for `InMemoryBackend` hits (`src/expander.rs:397`, `src/backend/inmemory.rs:116`). The envelope gives magic rejection and version eviction — but **no payload integrity**: a corrupted payload under intact magic can deserialize into wrong-but-valid data (grep for `crc|checksum|xxhash|integrity` across `src/` finds nothing; postcard carries no integrity check itself).

**The architecture rule for v1 (land this in the spec):** *serialization never runs on the local tier.* The typed engine (§3.3 P0) stores `Arc<V>` — the moka model — and the codec/envelope applies only when bytes cross a process boundary (Redis, Memcached, Dragonfly). This removes the entire codec cost from every in-process hit.

**Candidate matrix for the remote tier — decide with the §6 bench, not by taste:**

| Codec | Shape | Strengths | Fault-tolerance notes |
|---|---|---|---|
| postcard (current) | compact binary, serde-native | small, fast, `alloc`-only, already the envelope's payload | no integrity check — payload corruption under intact magic is silent |
| bincode 2 | compact binary, native `Encode`/`Decode` | comparable size/speed to postcard; no serde dependency on that path | no integrity check; native traits mean non-serde derives for cached types |
| rkyv | zero-copy archive | fastest *deserialization* — reads without parsing, wins on large payloads | heavier toolchain; validation is the unsafe/`bytecheck` surface; envelope and wire-compatibility story needs care |
| serde_json | text | human-readable, debuggable | largest, slowest — debug lanes only |
| flexbuffers | schemaless binary | zero-copy-ish reads, schema-flexible | mid-table size/speed; weaker typing guarantees |

**Fault-tolerance requirements to add on the remote tier, whichever codec wins:**
- **Payload integrity in the envelope** — CRC32 or xxHash3 over the payload, cheap enough per wire-bound set/get; the magic header alone is not corruption detection.
- **Schema evolution** — keep the versioned envelope; the documented behavior (version mismatch → evict and recompute from source of truth, `src/serialization/mod.rs:63-66`) is right and stays.
- **Compression** — optional `lz4`/`zstd` behind a feature for large remote payloads; bench-gated, not assumed.

## 4. Dragonfly backend — design input

Dragonfly speaks the Redis Serialization Protocol (RESP). agentsfleet already runs Dragonfly cluster-only via redis-rs `cluster_async` over RESP3 (`rustd/crates/afd_dragonfly`, boot knob `DRAGONFLY_URL`; a seed that answers as a single server is refused at boot). The Dragonfly backend should ride the Redis backend with cluster mode + RESP3 rather than being a separate code path — evaluate, decide, and state the reasoning in the approach.

## 5. Your job, as CTO

1. **Critique the analysis in §3.** Where is it wrong, incomplete, or over-scoped? What is missing for a production claim (crash behavior, memory ordering, `Send`/`Sync` bounds, runtime requirements, dependency footprint)?
2. **Analyze the e2e-observability-platform embedding** with the same depth as agentsfleet: token sizes, negative-cache policy, gRPC per-request latency budget, benchmark baseline from `rust/benches/auth_benchmark.rs`. The engine's API must serve both repos or say plainly why it can't.
3. **Decide the architecture.** Typed local engine + byte backends, or one unified trait; where single-flight lives; eviction policy per tier; the codec/envelope decision per §3.6 (including the rule that the local tier never serializes); Dragonfly-over-RESP decision (§4); what is v1 and what is explicitly deferred.
4. **Reconcile the pending v0.9.2 specs (§3.5)** into the v1 plan — supersede M092_003 on the new engine, absorb or sequence M092_001 / M092_002 — and state the disposition for each.
5. **Define the benchmark plan with bars.** See §6 — no approach is final without the numbers that prove it and the thresholds that count as "better than moka."
6. **Present the final approach to Indy for approval.** Short plain-language decision asks; he decides by eyeball. Do not file the spec before approval.
7. **After approval:** file the spec in this folder (`docs/v1/pending/`) via `orly-spec-new` — this repository's naming, terminology, and template own the spec's shape, not this prompt.

## 6. Benchmark requirements (the proof — publishable, reproducible)

The blog article ships these numbers, so the methodology ships with them:

- **In-memory, head-to-head vs moka:** throughput and latency (p50/p99) for get/set under concurrent readers+writers; **stampede bench** (100 waiters, 1 init — the single-flight proof); **Zipf-skew hit-rate vs capacity** (the eviction proof); **Resident Set Size (RSS) over time** with expiring entries (the leak proof); clone-cost bench (`Arc` refcount vs byte copy).
- **Redis and Dragonfly, end-to-end:** cache-kit backend vs the realistic moka alternative (moka as an in-process front cache in front of redis-rs) and vs raw redis-rs — throughput, latency, and reconnect/failover behavior. Same harness, same workloads, three lanes.
- **Codec lane (feeds the §3.6 decision):** postcard vs bincode 2 vs rkyv vs serde_json on the actual production payload shapes — agentsfleet's credential `Cached` and identity `Cached`, e2e-observability's `ValidatedToken`, a fleet-set row — measuring encode/decode throughput + Central Processing Unit (CPU) cost and wire size, then end-to-end Redis/Dragonfly latency with the winner, with and without the integrity checksum. A codec is chosen by these numbers, not by reputation.
- **Reproducibility:** pinned crate versions (moka, redis-rs, dashmap), recorded hardware and load, fixed seeds, committed bench scripts and raw output. A number that cannot be re-run by a reader does not go in the blog.
- **Claims discipline:** every public claim names the bench that produced it. "Better than moka" is claimed only on measured lanes; deferred lanes are named as deferred with the reason.

## 7. Rules of engagement

- Read this repository's rules before writing: `.orly/AGENTS.md` (dispatch router), `RUST_GUIDELINES.txt`, and the spec template referenced by `.orly/docs/TEMPLATE.md`.
- The spec is filed by `orly-spec-new`, into `docs/v1/pending/`, only after Indy approves the approach. This prompt file is the input, not the spec.
- Re-verify every `file:line` citation in §2 and §3 against current HEAD before relying on it; the repos move.
- No edits outside the approved scope once the spec lands; no opportunistic refactors bundled in.
- Estimated effort is halved before being presented (house rule — estimates pad ~2x).

*Prompt captured Oct 16, 2026, from Indy's problem statement and the read-only review of cache-kit 0.9.0.*
