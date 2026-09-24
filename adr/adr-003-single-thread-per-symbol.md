# ADR-003: Single Thread Per Symbol

**Status:** Accepted
**Date:** 2026-09-18
**Deciders:** Rajenthar
**Related:** ADR-004 (Disruptor), ADR-015 (deterministic replay), ADR-016 (client idempotency)

---

## Context

An order book is shared mutable state under concurrent access — the classic setup for locks. The instinct is to reach for `ReentrantReadWriteLock` or a concurrent data structure and let multiple threads match against one book.

That instinct is wrong here, for three reasons that compound:

1. **Matching is inherently serial.** Price-time priority means order N's outcome depends on the exact state left by order N−1. Concurrency does not speed up a fundamentally sequential algorithm; it only adds coordination cost.
2. **Lock contention destroys tail latency.** Under load, contended locks produce unpredictable p99s — exactly the metric the project is built to demonstrate.
3. **Concurrency destroys determinism.** Thread interleaving is non-reproducible, so ADR-015's replay-based recovery cannot work, and ADR-016's dedupe check would need its own synchronization.

The parallelism that actually exists is *across symbols*: AAPL and BTC-USD share no state and can match simultaneously without coordination.

## Decision

**One pinned platform thread per symbol. Each `OrderBook` is touched by exactly one thread, forever.**

- Symbols shard across threads; `SymbolRouter` maps symbol → ring buffer → thread.
- Orders reach that thread only via its Disruptor ring buffer (ADR-004).
- No locks, no `synchronized`, no concurrent collections inside `apex-core`.
- **Platform threads, not virtual threads.** A virtual thread unmounts on blocking and adds scheduling indirection; the matching thread never blocks and wants to stay hot on one core.

Scaling is by symbol count, not by cores-per-symbol. This is stated as an explicit limitation: a single very hot symbol is capped at one core's matching rate.

## Options Considered

### Option A: Lock-based concurrent order book

| Dimension | Assessment |
|---|---|
| Complexity | High — lock ordering, deadlock risk |
| Throughput | Lower than single-threaded under contention |
| Tail latency | Poor and unpredictable |
| Determinism | None |

**Pros:** Familiar; scales one symbol across cores in principle.
**Cons:** Matching is serial, so the locks serialize anyway — you pay coordination cost for no parallelism. Contended `synchronized` can cost hundreds of nanoseconds, comparable to the entire match. Non-deterministic, breaking ADR-015. Deadlock risk once cancel/amend touch multiple structures.

### Option B: Lock-free concurrent data structures (CAS-based)

| Dimension | Assessment |
|---|---|
| Complexity | **Very high** — ABA, memory ordering, correctness proofs |
| Throughput | Good in theory |
| Tail latency | Better than locks, worse than single-threaded |
| Determinism | None |

**Pros:** No blocking; a genuine throughput gain if correct.
**Cons:** A lock-free order book with price-time priority is a research-grade problem — CAS retry storms under contention, and correctness is extremely hard to establish. Still non-deterministic. High risk of a subtly wrong implementation that benchmarks well and loses money.

### Option C: Single thread per symbol (chosen)

| Dimension | Assessment |
|---|---|
| Complexity | **Low** — no concurrency in the core at all |
| Throughput | Highest per symbol; scales across symbols |
| Tail latency | Predictable |
| Determinism | **Total** |

**Pros:** No locks means no contention, no deadlock, no memory-ordering bugs. Excellent cache locality — the book stays in one core's L1/L2 (see ADR-011). Deterministic, enabling ADR-015 replay and ADR-016's race-free dedupe check. Simple enough to reason about completely. This is how LMAX and real exchange engines work.
**Cons:** One hot symbol cannot exceed one core. Uneven symbol activity means uneven thread utilization. Scaling a single symbol would require a redesign (shard rebalancing, leader election).

## Trade-off Analysis

The apparent trade is throughput-per-symbol against simplicity, but that framing is misleading: options A and B do not actually deliver more throughput for a serial algorithm. They deliver *coordination overhead* dressed as parallelism. The real trade is **horizontal scalability of one symbol** against **everything else** — latency, determinism, correctness, and comprehensibility.

For this system, losing single-symbol horizontal scale costs little. Real exchanges shard by symbol too; a single instrument's matching is serial by definition, because price-time priority is a total order.

The determinism benefit is the one that turned out to matter most. It was not the original motivation for this decision, but ADR-015's entire recovery model rests on it.

## Consequences

**Easier**
- `apex-core` contains no concurrency primitives whatsoever.
- Cache locality is excellent: one book, one core, no coherence traffic.
- ADR-015 replay and ADR-016 dedupe both become trivially correct.
- Tests are deterministic and reproducible.

**Harder**
- A single hot symbol is capped at one core's rate — must be stated honestly in the README.
- Thread-to-symbol assignment needs thought as symbol count grows beyond core count.
- Uneven symbol activity produces uneven CPU utilization.

**To revisit**
- Symbol-to-thread assignment strategy once symbol count exceeds available cores (fixed sharding vs work-stealing — note that work-stealing would reintroduce non-determinism and is likely incompatible with ADR-015).
- Hot-standby replicas for HA, which would need their own ADR.

## Action Items

1. [ ] `OrderBook` instance per symbol; never shared between threads
2. [ ] Pinned platform threads for matching; explicitly **not** virtual threads
3. [ ] `SymbolRouter` maps symbol → ring buffer at the ingestion edge only
4. [ ] ArchUnit test: no `synchronized`, `java.util.concurrent.locks`, or concurrent collections in `apex-core`
5. [ ] Document the single-hot-symbol ceiling in README constraints
6. [ ] Define thread assignment policy for symbol count > core count
