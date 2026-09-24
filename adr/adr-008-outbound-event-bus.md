# ADR-008: Outbound Event Bus Over Direct Downstream Calls

**Status:** Accepted — amended 2026-09-21 (Option E supersedes Option C)
**Date:** 2026-09-18
**Deciders:** Rajenthar
**Related:** ADR-002 (zero-I/O core), ADR-004 (Disruptor), ADR-009 (allocation-free), ADR-015 (deterministic replay)

---

## Context

A match produces trades and book updates. Four consumers need them:

- Kafka publisher → `trades.out`, `book-updates.out`
- Persistence writer → Postgres
- WebSocket broadcaster → subscribed clients
- Metrics collector → Grafana

The naive wiring is for the matching engine to call each in turn. That is fatal here for a reason worth stating precisely: **the matching thread would run at the speed of its slowest consumer.** A Postgres write taking 5ms means the symbol's entire order flow stalls for 5ms. One WebSocket client on a poor connection would throttle matching for every other client on that symbol. The engine's p99 would be determined by whichever downstream system was having the worst day.

It also violates ADR-002 outright — a direct call to a Kafka producer or JDBC connection is I/O inside `apex-core`.

## Decision

**The matching core emits into an outbound ring buffer and never performs I/O. A single publisher drains that buffer into Kafka; every other consumer reads from Kafka.**

- `apex-core` writes trades through a `TradeSink` callback (ADR-009). The sink writes into a Disruptor ring buffer slot; it does not perform I/O.
- **The Kafka publisher is the only gating consumer** on the outbound ring buffer. It batches and sends asynchronously to `trades.out` and `book-updates.out`.
- **Persistence and WebSocket consume from Kafka topics**, with independent consumer-group offsets. They are not attached to the ring buffer and therefore cannot gate it.
- **Metrics is a second, non-gating handler** on the ring buffer. It is explicitly allowed to drop and never gates the producer.
- Matching's only cost is a slot write: tens of nanoseconds, zero allocation.

Kafka retention, not the ring buffer's 4,096 slots, is what absorbs downstream slowness. A persistence worker may be offline for hours and resume from its committed offset without matching ever noticing.

This is safe *only* because ADR-015 makes the durable input log the source of truth: any output lost between matching and Kafka is reproducible by replay. The two decisions are inseparable — fire-and-forget without deterministic replay would be data loss.

### Correction: what a bounded ring buffer actually does

The original version of this ADR stated that a slow handler causes the ring buffer to wrap and lose events, and that a slow consumer "cannot propagate backpressure into matching." **Both statements are wrong**, and Option C was chosen on the strength of them.

Handlers registered via `handleEventsWith()` become *gating sequences*. The producer's claim computes:

```
wrapPoint = nextSequence - bufferSize
if (wrapPoint > minimumGatingSequence) -> park and spin until it advances
```

It refuses to claim a slot that any gating consumer has not yet read. **The Disruptor blocks; it does not overwrite.** With four gating handlers, a stalled Postgres writer parks the matching thread after 4,096 events — precisely the outcome this ADR was written to prevent, merely deferred by 4,096 events.

With one bounded buffer and N gating consumers there is no third option: either the slowest consumer gates the producer, or it is not gating and has no safe way to detect that it was lapped. Isolation cannot come from the ring buffer. It has to come from making the fan-out point durable and effectively unbounded, which is what a log is for.

## Options Considered

### Option A: Direct synchronous calls from the matching engine

| Dimension | Assessment |
|---|---|
| Complexity | Low |
| Latency | **Catastrophic** — slowest consumer sets the pace |
| Isolation | None |
| ADR-002 compliance | Violated |

**Pros:** Simplest possible; trivially ordered; errors surface immediately at the call site.
**Cons:** Matching latency becomes the sum of all downstream latencies. A single slow consumer stalls an entire symbol. Puts I/O directly in `apex-core`. Rejected.

### Option B: Direct calls to an async executor (`CompletableFuture` / thread pool)

| Dimension | Assessment |
|---|---|
| Complexity | Medium |
| Latency | Better, but allocation-heavy |
| Isolation | Partial |

**Pros:** Non-blocking; familiar Java idiom; reasonable isolation between consumers.
**Cons:** Allocates per event — a `CompletableFuture` and a task object per trade, violating ADR-009's 0 B/op target. Unbounded queues risk memory exhaustion; bounded ones reintroduce blocking. Task submission still touches a concurrent queue from the matching thread.

### Option C: Outbound ring buffer with four gating handlers (superseded)

| Dimension | Assessment |
|---|---|
| Complexity | Medium |
| Latency | Excellent — tens of nanoseconds to publish |
| Isolation | **None under sustained lag** — every gating handler gates the producer |
| Allocation | Zero — pooled slots |

**Pros:** Publishing costs tens of nanoseconds. Zero allocation via pre-allocated slots. Reuses the mechanism already introduced by ADR-004, so no new concept. Per-handler lag is directly observable as a metric.

**Cons:** The isolation it appears to provide does not exist. Any handler that falls 4,096 events behind parks the matching thread, so Postgres latency and slow WebSocket clients both reach the hot path — the failure this ADR exists to prevent. The buffer offers roughly 0.4 seconds of headroom at 10k events/sec, which is far below the duration of an ordinary Postgres stall or consumer restart. Originally chosen on the mistaken belief that the buffer would wrap and drop instead of block.

### Option D: Kafka producer called directly from the matching thread

**Pros:** Single fan-out mechanism; durable by construction; consumers fully decoupled.
**Cons:** Puts a Kafka producer call on the matching thread — I/O in `apex-core`, violating ADR-002. Even async Kafka sends involve a serializer, a record accumulator and a lock. Rejected for the same reason as Option A.

Note that this is **not** Option E. The objection here is the producer call sitting on the matching thread, not the use of Kafka for fan-out.

### Option E: Outbound ring buffer, single Kafka publisher, Kafka fan-out (chosen)

| Dimension | Assessment |
|---|---|
| Complexity | Medium — one more topic pair, no new concept |
| Latency | Excellent on the hot path — tens of nanoseconds to publish |
| Isolation | **Full and real** — only the publisher gates; it is the fastest consumer |
| Allocation | Zero on the hot path — pooled slots |
| Buffer depth | Kafka retention (hours) rather than 4,096 slots (~0.4s) |

**Pros**
- The matching thread still only writes a slot. No Kafka API is touched in `apex-core`, so ADR-002 holds exactly as under Option C.
- The single gating consumer is the fastest available one: a batched asynchronous send into the OS page cache, orders of magnitude quicker than a Postgres commit or a socket write to a slow client.
- Consumers that genuinely stall are no longer gating anything. Postgres can be down for an hour and resume from its committed offset.
- Per-consumer overflow policy disappears. Independent offsets, retention and replay are consumer-group semantics that Kafka already provides.
- Consumers restart, scale and rewind without coordinating with matching.
- A WebSocket client can resync by offset instead of being dropped.

**Cons**
- Adds single-digit milliseconds to WebSocket market data, which now traverses Kafka rather than going straight out.
- The Kafka publisher becomes the single gating consumer: if Kafka itself is unavailable, the publisher stalls and matching eventually blocks. This is one failure mode in one place with one policy, rather than four, and ADR-015 replay makes drop-and-alert a legitimate response.
- `trades.out` and `book-updates.out` now carry internal fan-out traffic as well as external, so partitioning and retention must be sized for both.

## Trade-off Analysis

The decision is about **who absorbs downstream slowness**. Options A and B let it flow back into matching directly. Option C appeared to confine it to the individual consumer but does not: a bounded buffer with gating consumers converts consumer slowness into producer blocking. The isolation was assumed rather than verified against the Disruptor's actual semantics.

The real question is how deep the buffer between matching and its slowest consumer needs to be. At 4,096 slots the answer is about 0.4 seconds, which is shorter than a GC pause on the persistence worker, a connection-pool timeout or a rolling restart — all routine events. Any of them would reach the matching thread.

Kafka's retention answers the same question in hours. That is the entire reason to put it at the fan-out point: not durability for its own sake, but a buffer deep enough that ordinary downstream disruption never becomes a matching problem.

The cost is latency for exactly one consumer — WebSocket market data, which gains a few milliseconds. That is a good trade for removing Postgres from the matching thread's critical path, and better still because it makes slow-client recovery an offset seek rather than a disconnect.

Metrics stays on the ring buffer as a non-gating handler. Routing counters through a broker would add operational weight for a consumer that is fast, allowed to drop, and never the problem.

| Consumer | Reads from | Gates matching? | On falling behind |
|---|---|---|---|
| Kafka publisher | Outbound ring buffer | **Yes** — the only one | Alert; drop-and-alert if Kafka is down, replay covers the gap |
| Persistence writer | `trades.out` | No | Consumer lag grows; resumes from committed offset. `ON CONFLICT DO NOTHING` makes replay safe |
| WebSocket broadcaster | `book-updates.out` | No | Client resyncs by offset; conflation if needed |
| Metrics collector | Outbound ring buffer (non-gating) | No | May drop; sampling loss is acceptable |

## Consequences

**Easier**
- Matching latency is genuinely independent of Postgres and of WebSocket clients, rather than independent only for the first 4,096 events.
- `apex-core` stays I/O-free, satisfying ADR-002.
- Consumers can be added, removed, restarted or rewound without touching the core.
- No per-handler overflow policy to design or maintain; Kafka consumer-group semantics cover it.
- Consumer lag is a standard Kafka metric rather than a bespoke one.

**Harder**
- One more hop for WebSocket market data, costing single-digit milliseconds.
- Kafka availability now matters to matching, indirectly and with a long fuse. Producer `buffer.memory` and `max.block.ms` need deliberate settings.
- Two classes of consumer to reason about: ring-buffer handlers and Kafka consumers.
- Debugging is harder: an error in a consumer surfaces far from the match that caused it.
- Correctness still depends on ADR-015 remaining true; weakening replay determinism silently converts publisher loss into data loss.

**To revisit**
- Outbound ring buffer size once publisher lag is measured under load. With one fast gating consumer, 4,096 may now be generous.
- Whether the WebSocket path needs a conflation stage for slow clients, or whether offset-based resync is sufficient.
- Whether metrics should also move to Kafka if the collector ever becomes a bottleneck.

## Action Items

1. [ ] Second Disruptor ring buffer for outbound events, pooled mutable slots
2. [ ] Kafka publisher registered as the **only** gating `EventHandler`; batched async sends
3. [ ] Metrics collector registered as a **non-gating** handler, explicitly permitted to drop
4. [ ] `TradeSink` implementation writes into a slot; performs no I/O
5. [ ] `apex-persistence` consumes `trades.out`; **not** wired to the ring buffer
6. [ ] WebSocket broadcaster consumes `book-updates.out`; resync by offset for slow clients
7. [ ] Publisher policy when Kafka is unavailable: block, then drop-and-alert past a bounded threshold; document the threshold
8. [ ] Export publisher gating lag and Kafka consumer lag as separate metrics
9. [ ] ArchUnit: no `EventHandler` implementation may live in `apex-core`
10. [ ] Chaos test: stall Postgres, assert matching throughput is unaffected indefinitely (not merely for 4,096 events)
11. [ ] Chaos test: stop Kafka, assert the documented publisher policy takes effect and matching degrades as specified
12. [ ] Reconcile `docs/apex-architecture.mermaid`, which already depicted this topology
