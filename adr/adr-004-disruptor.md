# ADR-004: LMAX Disruptor Over BlockingQueue

**Status:** Accepted
**Date:** 2026-09-18
**Deciders:** Rajenthar
**Related:** ADR-003 (single thread per symbol), ADR-009 (allocation-free), ADR-010 (ring buffer sizing), ADR-015 (no silent drops)

---

## Context

ADR-003 establishes one matching thread per symbol. Orders arrive from `apex-ingestion` on a different thread. Something must hand off between them, and that handoff sits directly on the critical path — it happens once per order, before every match.

The default choice is `ArrayBlockingQueue` or `LinkedBlockingQueue`. Both work correctly. Both are also poor fits for a sub-microsecond path:

- **Lock-based.** Every `put`/`take` acquires a `ReentrantLock`. Under contention this parks threads, and a park/unpark cycle costs microseconds — an order of magnitude more than the match it precedes.
- **Allocating.** `LinkedBlockingQueue` allocates a node per element, directly violating ADR-009's zero-allocation rule.
- **False sharing.** Head and tail cursors typically live on the same cache line, so producer and consumer invalidate each other's cache lines on every operation.

## Decision

**LMAX Disruptor ring buffer, one per symbol shard, single-producer configuration.**

- Pre-allocated slots, populated at startup, reused forever — no allocation on publish.
- Cursors are cache-line padded, eliminating false sharing between producer and consumer.
- `ProducerType.SINGLE`, because one ingestion thread feeds each symbol's buffer. This removes the CAS required by multi-producer mode.
- **Blocking `publishEvent`, never `tryPublishEvent`** — a full buffer must apply backpressure, not drop orders (ADR-015).
- Wait strategy: start with `BlockingWaitStrategy` for CPU-frugal behaviour on a 2-OCPU VM; consider `YieldingWaitStrategy` only if profiling shows wake-up latency matters and spare cores exist.

Sizing is covered separately in ADR-010.

## Options Considered

### Option A: `ArrayBlockingQueue`

| Dimension | Assessment |
|---|---|
| Complexity | Low |
| Latency | Poor — lock + park/unpark |
| Allocation | None per element (array-backed) |
| Familiarity | High |

**Pros:** Standard library, zero learning curve, bounded, correct.
**Cons:** Lock acquisition on every operation; park/unpark costs microseconds; false sharing between head and tail cursors. The handoff would dominate the match it precedes.

### Option B: `LinkedBlockingQueue`

**Pros:** Standard, unbounded by default (which is itself a liability).
**Cons:** Everything wrong with Option A, plus a node allocation per element — violating ADR-009. Unbounded by default means memory exhaustion instead of backpressure. Rejected outright.

### Option C: LMAX Disruptor (chosen)

| Dimension | Assessment |
|---|---|
| Complexity | Medium — unfamiliar API, sizing and wait strategies to understand |
| Latency | Excellent — tens of nanoseconds |
| Allocation | **Zero** — slots pre-allocated |
| Familiarity | Low, but well documented |

**Pros:** No locks, no allocation, cache-line padded cursors. Single-producer mode avoids even a CAS. Mechanical sympathy is the library's explicit design goal. Battle-tested in production trading systems, so it is a credible choice rather than an exotic one.
**Cons:** Unfamiliar API and mental model. Sizing and wait strategy are real decisions with real consequences (see ADR-010). `tryPublishEvent` is a footgun that silently drops.

### Option D: JCTools `SpscArrayQueue`

| Dimension | Assessment |
|---|---|
| Complexity | Low — it is just a queue |
| Latency | Excellent, comparable to Disruptor |
| Allocation | Zero |

**Pros:** Genuinely competitive on raw latency; far simpler API; purpose-built single-producer-single-consumer.
**Cons:** A queue, not an event-processing framework — no multi-consumer fan-out, no batching, no event handler pipeline. ADR-008's outbound event bus benefits from Disruptor's multi-handler support.

## Trade-off Analysis

Options C and D are close on the metric that matters, and JCTools would be a defensible choice. Disruptor wins on two secondary grounds: its multi-consumer event handler model suits ADR-008's fan-out (Kafka publisher, persistence, WebSocket, metrics all consuming the same output stream in parallel), and it is the canonical implementation of the pattern this project is demonstrating.

The honest framing: for the inbound order path alone, JCTools would be equivalent and simpler. Disruptor earns its complexity on the *outbound* side, and using one mechanism for both is preferable to two.

The cost is a real learning curve plus two genuine footguns — wait strategy choice and `tryPublishEvent`. Both are addressed by explicit decisions here rather than left to defaults.

## Consequences

**Easier**
- Handoff cost drops to tens of nanoseconds and stops dominating the match.
- Zero allocation on the handoff path, supporting ADR-009's 0 B/op target.
- Multi-consumer fan-out for the outbound bus comes free.
- Ring buffer occupancy is a natural, meaningful backpressure metric.

**Harder**
- Wait strategy is a real tuning decision on a 2-OCPU VM, where a spinning strategy would starve other work.
- Sizing matters for cache behaviour (ADR-010).
- `tryPublishEvent` must be banned in review and by static check.

**To revisit**
- Wait strategy, once CPU utilization and wake-up latency are measured under Gatling load.

## Action Items

1. [ ] Disruptor 4.x, `ProducerType.SINGLE`, one ring buffer per symbol shard
2. [ ] `BlockingWaitStrategy` initially; revisit after profiling
3. [ ] Use blocking `publishEvent` exclusively; ArchUnit or grep check banning `tryPublishEvent`
4. [ ] Export ring buffer remaining capacity as a metric (feeds ADR-010's sizing decision)
5. [ ] Benchmark Disruptor vs `ArrayBlockingQueue` handoff in `apex-benchmarks` — this before/after number is worth publishing
