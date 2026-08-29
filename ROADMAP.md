# Roadmap

Building a market-data and trading system for Kalshi and Polymarket, in Rust,
from scratch. **The system is the vehicle; learning Rust end-to-end is the
goal.** Every stage is chosen because of what it teaches, not because it is the
fastest path to a working service.

## Standing principles

- **Learning over shipping.** Doing something the hard way is often the right
  call here. Building a thing myself to understand it beats pulling a crate,
  *provided* I know which one I'm choosing and why.
- **Measure before optimizing.** Not because optimization is wasteful — chasing
  latency is a legitimate learning objective — but because optimizing without a
  benchmark teaches superstition instead of skill.
- **Paper trading is the default terminal state.** Live execution is a separate,
  deliberate decision (Stage 11), not a natural consequence of finishing. The
  project loses no educational value if it never places a real order.
- **Record real data early and continuously.** Recorded feeds become the fixture
  corpus for testing, replay, and benchmarking. Data not captured today cannot
  be captured retroactively.
- **Don't move on until the current stage runs.** Half-finished layers compound.

## Latency: the honest budget

| Segment | Order of magnitude |
|---|---|
| Network RTT to venue over public internet | 1–100 ms |
| Venue matching engine + dissemination | milliseconds |
| **My parse → decide → emit** | **microseconds** |
| Order signing (Polymarket, on-chain auth) | hundreds of µs – ms |

My code is roughly 1% of end-to-end latency. Region selection, warm connections,
and not blocking the event loop beat clever code by orders of magnitude. Both
venues also rate-limit, which binds long before processing speed does.

This does not make Rust the wrong choice. The real case for Rust here is **no
garbage collector, therefore predictable tail latency** — p99 stays near the
median, instead of stalling 50 ms at the worst possible moment — plus compile-time
elimination of whole bug classes in a system handling money.

Verify venue specifics against live API docs; exchange APIs change.

---

## Stages

### Stage 0 — Fundamentals

No trading code. The Book chapters 1–11, `rustlings` alongside.

- **Learn:** ownership, borrowing, lifetimes (intro), `Result`/`Option`, the `?`
  operator, structs, enums, pattern matching, traits, collections, `#[test]`.
- **Why first:** entering async while still fighting the borrow checker means
  learning neither. Chapter 11 is testing, so tests start here.
- **Done when:** I can read a borrow-checker error and predict the fix before
  reading the compiler's suggestion.

### Stage 1 — REST client

Pull the market list from Kalshi over HTTPS, deserialize, print. Unit-tested
from the first commit.

- **Learn:** external crates, `serde`, custom error types, `?` in anger.
- **Done when:** credentials work, JSON round-trips into my own types, tests pass.

### Stage 2 — One WebSocket feed, recorded to disk

Subscribe to a single Kalshi feed. Write every message to disk with a receive
timestamp. Run it continuously once it works.

- **Learn:** `tokio`, async/await, streams, `Send`/`Sync` bounds, `tokio::spawn`,
  channels, reconnect with backoff.
- **Note:** `Send`/`Sync` and channels appear here, not later — spawning a task
  demands them immediately, and reconnect logic needs message passing. The
  *advanced* concurrency material is deferred to Stage 8.
- **Done when:** it survives a network drop and keeps recording.

### Stage 3 — Normalization

One internal event type; both venues convert into it. The architectural
keystone — get this boundary right and a third venue is a day's work.

- **Learn:** enum modeling, `From`/`Into`, newtypes, generics vs. `dyn`,
  integration tests in `tests/`.
- **Done when:** no venue-specific detail leaks downstream of this layer.

### Stage 4 — Cargo workspace

Split the single binary into multiple crates with real boundaries.

- **Learn:** module system, crate boundaries, visibility, feature flags.
- **Why:** one `main.rs` teaches Rust the language; a workspace teaches Rust the
  engineering discipline.

### Stage 5 — Strategy interface

A trait defining what a strategy is: consumes normalized events, emits intents.
Backtest and live must be the same code path with a different event source.

- **Learn:** trait objects, dependency inversion, mocking through traits.
- **Done when:** swapping recorded data for a live feed is a config change.

### Stage 6 — Metrics and observability

Prometheus exporter, Grafana on top. Instrument the latency budget above.

- **Learn:** `tracing`, structured logging, shared state across tasks.

### Stage 7 — Deploy to cloud

EC2 or DigitalOcean. Region chosen by measured RTT to each venue, not by guess.

- **Learn:** cross-compilation, containers, systemd, config management.
- **Why before benchmarks:** I develop on arm64 macOS and deploy to x86-64
  Linux. Benchmark numbers do not transfer across that gap — different ISA,
  allocator behavior, syscall costs, plus cloud noisy-neighbor and CPU steal.
  Benchmarking first means optimizing against a machine I'll never run on.

### Stage 8 — Concurrency architecture

- **Learn:** atomics and memory ordering, locks vs. channels, tokio tasks vs. OS
  threads, lock-free structures.
- **Why here:** meaningless without a real workload to apply it to.

### Stage 9 — Benchmarks and profiling, on the deployed target

- **Learn:** `criterion`, flamegraphs, allocation control, zero-copy parsing.

### Stage 10 — Heavy testing

- **Learn:** deterministic replay of recorded feeds, `proptest`, `cargo-fuzz`,
  simulation testing.
- **Why here:** needs recorded fixtures (Stage 2) and a stable architecture
  (Stages 3–5) to test against.

### Stage 11 — Optional: live execution

Real orders, real money. Position limits, kill switch, reconciliation. A
deliberate decision, taken slowly, or never.

---

## Testing thread

Testing is not a stage. It escalates alongside everything:

| From | Technique |
|---|---|
| Stage 0 | `#[test]`, unit tests in-file |
| Stage 1 | Unit tests from the first commit |
| Stage 3 | Integration tests in `tests/` |
| Stage 5 | Trait-based mocking |
| Stage 10 | Replay, property tests, fuzzing |

## Open questions

- API credentials for Kalshi and Polymarket — obtained yet?
- Subscribing to one feed vs. all feeds: a Stage 2–3 concern that mostly falls
  out of the normalization design. Don't let it shape early decisions.
