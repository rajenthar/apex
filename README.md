# Apex ⚡

A deterministic, low-latency order matching engine in Java 21.

Apex executes matching on a single-threaded-per-symbol hot path that performs no
I/O and allocates nothing. Persistence, messaging and client distribution are
fully asynchronous and off the critical path. Correctness under failure comes
from deterministic replay over a durable input log rather than from distributed
transactions.

> **Status: design complete, implementation not started.**
> Sixteen architecture decision records are written and reviewed. Code begins
> with the Maven scaffold. Every performance figure below is a *target* derived
> from design analysis, not a measured result. Measured numbers will replace
> them, labelled as such.

---

## Architecture

```
                    Client
                      │
              ┌───────▼────────┐
              │  apex-gateway  │  REST + WebSocket, virtual threads
              └───────┬────────┘  shape/auth validation only
                      │
              Kafka  orders.in    partitioned by symbol
                      │           the durable source of truth
              ┌───────▼────────┐
              │ apex-ingestion │  heavy validation, symbol routing
              └───────┬────────┘  offset carried as sequence number
                      │
          Disruptor ring buffer   4,096 slots, single producer
                      │
        ┌─────────────▼──────────────┐
        │        apex-core           │  one pinned thread per symbol
        │  OrderBook per symbol      │  zero I/O, zero allocation
        └─────────────┬──────────────┘
                      │  TradeSink callback
         Outbound Disruptor ring buffer
                      │
         ┌────────────┴─────────────┐
         │                          │
  Kafka publisher            Metrics collector
  (only gating consumer)     (non-gating, may drop)
         │
   trades.out / book-updates.out
         │
    ┌────┴──────────────────┐
    │                       │
apex-persistence      WebSocket broadcaster
batched → Postgres    → subscribed clients
```

Persistence and WebSocket consume from Kafka topics with independent offsets, so
neither can gate the matching thread. Kafka retention, not the ring buffer, is
what absorbs downstream slowness.

## Modules

| Module | Responsibility | Depends on |
|---|---|---|
| `apex-common` | DTOs, events, enums | nothing |
| `apex-core` | Matching engine and order book | `apex-common`, fastutil |
| `apex-ingestion` | Kafka consumer, validation, Disruptor wiring | core, common |
| `apex-persistence` | Async batched Postgres writer | common |
| `apex-gateway` | REST + WebSocket API (Spring Boot 3) | common |
| `apex-benchmarks` | JMH benchmarks | `apex-core` only |

`apex-core` depends on `apex-common` and fastutil, and nothing else — no Spring,
no Kafka client, no logging framework, no JDBC. This is enforced by the module
classpath, an ArchUnit test, and a Maven enforcer rule.

## Design decisions

All significant decisions are recorded as ADRs in [`adr/`](adr/), including the
options rejected and why. Start with [`adr/README.md`](adr/README.md).

The four that carry the design:

| ADR | Decision |
|---|---|
| [002](adr/adr-002-zero-io-core.md) | Zero-I/O, zero-framework matching core |
| [003](adr/adr-003-single-thread-per-symbol.md) | One pinned thread per symbol |
| [009](adr/adr-009-allocation-free-match.md) | Allocation-free `match()` via callback sink |
| [015](adr/adr-015-exactly-once-execution.md) | Exactly-once execution via deterministic replay |

Runtime tuning — GC avoidance, CPU pinning and cache residency, with the
commands to verify each — is documented in
[`adr/html/runtime-tuning.html`](adr/html/runtime-tuning.html).

## Targets

Derived from design analysis. To be replaced with measurements.

| Metric | Target |
|---|---|
| Allocation per match (`gc.alloc.rate.norm`) | **0 B/op** — a CI gate, not an aspiration |
| Core throughput, L1-resident microbenchmark | ~4M ops/sec |
| Core throughput, realistic price dispersion | 800k–1.5M ops/sec |
| `Order` object size | exactly 64 bytes, asserted by a JOL test |
| Inbound ring buffer | 4,096 slots (256 KB, L2-resident) |

Core-only and end-to-end latency are reported as separate metrics and are never
blended. They differ by orders of magnitude, for reasons the benchmark
documentation explains.

## Constraints and limitations

Stated deliberately rather than discovered later:

- **A single hot symbol is capped at one core's matching rate.** Scaling is by
  symbol count, not by cores per symbol. Sharding a single symbol would require
  a different design.
- **Kafka before matching adds latency** — roughly 200–500µs versus an
  in-memory-first design. Chosen for durability and replay.
- **A client ack means "durably accepted", not "matched".** The distinction is
  part of the API contract.
- **No HA in the MVP.** Recovery is snapshot plus log replay; failover and
  leader election are out of scope and would need their own ADR.
- **Kafka availability reaches matching indirectly.** The publisher is the only
  gating consumer of the outbound ring buffer; if Kafka is unreachable long
  enough, matching blocks. The policy for this is documented in ADR-008.
- **Idempotency is bounded to a 24-hour window**, per client.

## Deployment

Sized for a 2 OCPU / 12 GB ARM VM:

| Container | RAM | Role |
|---|---|---|
| apex-gateway | ~300 MB | REST/WS, request validation |
| apex-matching | ~400 MB | Core engine, fixed heap |
| apex-persistence-worker | ~150 MB | Kafka → batched Postgres |
| apex-metrics-sidecar | ~150 MB | Micrometer/OTel export |

Note that isolating a core for matching on a 2-core host leaves one core for
everything else. That is sufficient to demonstrate the technique at one-symbol
scale and does not stretch further.

## Stack

Java 21 · Spring Boot 3 (gateway only) · LMAX Disruptor · fastutil ·
Redpanda/Kafka · PostgreSQL · JMH · Gatling · async-profiler ·
Micrometer + OpenTelemetry · Docker

## Branches

| Branch | Purpose |
|---|---|
| `main` | Released state |
| `develop` | Integration branch; work lands here first |

## Licence

Not yet specified.
