# ADR-010: Ring Buffer Sized to the Cache Hierarchy

**Status:** Accepted
**Date:** 2026-09-18
**Deciders:** Rajenthar
**Supersedes:** the 2¹⁶ sizing in the original design
**Related:** ADR-004 (Disruptor), ADR-011 (cache-line layout)

---

## Context

The original design specified a Disruptor ring buffer of 2¹⁶ (65,536) slots per symbol shard. That number was chosen the way such numbers usually are — it sounded generously large — and it is actively harmful.

Ring buffer slots are pre-allocated and remain resident for the lifetime of the process. With a 64-byte event per slot (ADR-011):

| Size | Memory per symbol | vs. Ampere Altra cache |
|---|---|---|
| 2¹⁶ (65,536) | **4 MB** | 4× larger than the 1 MB per-core L2 |
| 2¹² (4,096) | **256 KB** | Comfortably inside L2 |

Ten symbols at 2¹⁶ would be 40 MB — exceeding the entire 32 MB shared system-level cache, before the order books themselves are counted. The producer writing to a slot and the consumer reading it shortly after would frequently miss cache, turning a handoff designed to cost tens of nanoseconds into one costing hundreds.

The general failure here is worth naming: **a buffer sized for imagined peak burst, with no reference to the memory hierarchy it lives in.** Bigger is not free; it is paid for in cache residency.

## Decision

**2¹² (4,096) slots per symbol shard, and the size is configurable, not compiled in.**

- 4,096 slots × 64 bytes = 256 KB, sitting comfortably within the 1 MB per-core L2.
- At a realistic 10k orders/sec per symbol, 4,096 slots is roughly 400ms of buffering — ample, since the consumer runs at millions of ops/sec and only falls behind under genuine pathology.
- **Raise it only against evidence.** `ringBufferRemainingCapacity` is exported as a metric; sustained low remaining capacity is the only justification for increasing the size.
- A full buffer applies backpressure via blocking `publishEvent` (ADR-004). It never drops.

## Options Considered

### Option A: 2¹⁶ (65,536) slots — original

| Dimension | Assessment |
|---|---|
| Burst absorption | Very high |
| Cache behaviour | **Poor** — 4 MB, 4× over L2 |
| Multi-symbol scaling | **Poor** — 10 symbols exceed the whole SLC |

**Pros:** Absorbs enormous bursts without ever applying backpressure.
**Cons:** Evicts the order book itself from cache — the buffer competes with the data structure it feeds. Scales badly with symbol count. Optimises for a burst scenario that backpressure already handles correctly.

### Option B: 2¹² (4,096) slots (chosen)

| Dimension | Assessment |
|---|---|
| Burst absorption | ~400ms at realistic rates |
| Cache behaviour | **Good** — 256 KB, inside L2 |
| Multi-symbol scaling | Good — 10 symbols = 2.5 MB total |

**Pros:** Stays L2-resident, so the handoff hits cache. Leaves room for the order book in the same cache. Scales to tens of symbols without SLC pressure. Ample buffering for any non-pathological load.
**Cons:** A sustained producer-faster-than-consumer condition applies backpressure sooner — which is correct behaviour, but will be visible in metrics and needs to be understood rather than alarmed at.

### Option C: 2¹⁰ (1,024) slots

**Pros:** 64 KB — fits even in L1d.
**Cons:** Too little headroom for ordinary jitter; backpressure would engage during normal GC pauses or scheduling hiccups, coupling the producer to transient consumer stalls for no real benefit. The consumer is orders of magnitude faster than the producer, so L1 residency of the *buffer* is not where the win is.

### Option D: Size per symbol by observed volume

**Pros:** Optimal allocation — hot symbols get more, quiet ones less.
**Cons:** Requires volume data that does not exist before the system runs. Premature. Revisit once real traffic patterns are measured; the configurability in this decision makes that a config change rather than a code change.

## Trade-off Analysis

The trade is **burst absorption against cache residency**, and the original design got it wrong by treating only the first as a cost. A larger buffer looks free — it is "just memory" — but memory that competes for L2 with the order book is not free at all.

The decisive observation is that **backpressure is the correct response to sustained overload, not a failure to be buffered away.** If the producer outruns the consumer persistently, a bigger buffer only delays the same outcome while consuming cache the whole time. Blocking `publishEvent` (ADR-004) propagates the pressure to Kafka consumer lag, which is exactly where it is visible and actionable.

Option D is the right long-term answer; the configurability here is what makes it reachable without a redesign.

## Consequences

**Easier**
- The handoff stays cache-resident, preserving Disruptor's latency advantage.
- Scales to tens of symbols on a 2-OCPU VM without SLC exhaustion.
- Memory footprint is predictable: `symbols × 256 KB`.

**Harder**
- Backpressure engages sooner and must be understood as designed behaviour, not an incident.
- Requires the remaining-capacity metric to be in place before the size can be tuned with confidence.

**To revisit**
- Raise the size if `ringBufferRemainingCapacity` shows sustained pressure under realistic load.
- Per-symbol sizing (Option D) once traffic patterns are observed.
- Re-derive the number if the target host changes — this sizing is specific to Ampere Altra's 1 MB L2.

## Action Items

1. [ ] Ring buffer size 4,096, read from configuration — never a compiled-in constant
2. [ ] Export `ringBufferRemainingCapacity` per symbol to Grafana
3. [ ] Document the derivation (slot size × count vs. L2) beside the config value, so the number is not mistaken for arbitrary
4. [ ] Load test: confirm backpressure engages gracefully and surfaces as consumer lag
5. [ ] Record total ring buffer memory as `symbols × 256 KB` in the deployment topology table
