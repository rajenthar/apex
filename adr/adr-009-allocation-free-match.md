# ADR-009: Allocation-Free `match()` via Callback Sink

**Status:** Accepted
**Date:** 2026-09-18
**Deciders:** Rajenthar
**Related:** ADR-002 (zero-I/O core), ADR-005 (fixed-point), ADR-006 (fastutil), ADR-008 (event bus), ADR-012 (GC choice)

---

## Context

The original design specified `List<TradeEvent> match(Order incoming)` — the obvious, idiomatic Java signature. It is also, at this system's throughput, the single largest threat to p99 latency.

The arithmetic is unforgiving. At 4M matches/sec, allocating even 100 bytes per match — a `List`, its backing array, one or more `TradeEvent` objects, possibly a boxed `Long` from the old `TreeMap` — produces **400 MB/s of garbage**. Against a 400 MB heap with roughly a 150 MB young generation, that triggers a young GC approximately every 0.4 seconds. Each young GC is a stop-the-world pause of several milliseconds.

The consequence is worth stating bluntly: **the matching algorithm would run in 250 nanoseconds and the p99 would be measured in milliseconds**, entirely because of allocation. Every other optimisation in this project would be invisible next to it.

Allocation is also the thing most likely to creep back in silently. Unlike a blocking call, it produces no obvious symptom in a microbenchmark — JMH's default output reports throughput, not garbage.

## Decision

**`apex-core` allocates nothing on the matching path. Target and verify 0 B/op.**

```java
@FunctionalInterface
public interface TradeSink {
    void onTrade(long makerOrderId, long takerOrderId,
                 long price, long quantity, long timestamp);
}

/** @return remaining unfilled quantity; 0 means fully filled */
long match(Order incoming, TradeSink sink);
```

Four rules, together sufficient for 0 B/op:

1. **No collection return.** Trades are emitted through `sink` as they occur. The sink writes into a pre-allocated ring buffer slot (ADR-008).
2. **Primitive parameters only.** No `TradeEvent` object is constructed inside the core.
3. **No boxing.** Guaranteed by ADR-005 (`long` prices) and ADR-006 (fastutil primitive-keyed maps).
4. **No lambda capture on the hot path.** The `TradeSink` instance is created once at wiring time, never per call — a capturing lambda allocates.

**Verification is mandatory, not aspirational:** `mvn ... && java -jar target/benchmarks.jar -prof gc` must report `gc.alloc.rate.norm` of **0 B/op**. This runs in CI and fails the build on regression.

## Options Considered

### Option A: `List<TradeEvent> match(Order)` (original, rejected)

| Dimension | Assessment |
|---|---|
| Complexity | Low — idiomatic Java |
| Allocation | **~100+ B/op** |
| p99 impact | Severe |
| Testability | Excellent |

**Pros:** Natural signature, easy to test, composable, immediately readable.
**Cons:** 400 MB/s of garbage at target throughput. Young GC every ~0.4s, each pausing for milliseconds. Destroys the p99 the project exists to demonstrate.

### Option B: Reusable output buffer owned by the engine

```java
int match(Order incoming, TradeEvent[] out);  // returns count
```

| Dimension | Assessment |
|---|---|
| Complexity | Medium |
| Allocation | Zero, if the array is pre-sized |
| Safety | **Poor** — aliasing hazard |

**Pros:** Zero allocation; keeps a return-value shape that feels familiar.
**Cons:** Caller must not retain the array beyond the call — a silent, severe aliasing bug waiting to happen. Requires a maximum-trades-per-match bound, and overflow behaviour must be defined. Mutable shared state across an API boundary.

### Option C: Callback sink (chosen)

| Dimension | Assessment |
|---|---|
| Complexity | Medium — inversion of control |
| Allocation | **Zero** |
| Safety | Good — no shared mutable buffer |
| Testability | Good — test sink collects into a list |

**Pros:** Zero allocation. No aliasing hazard, because nothing is handed back. No maximum-trades bound needed — trades stream out as they happen. Composes naturally with ADR-008's ring buffer, where the sink writes directly into a pooled slot. Tests use a collecting sink and read normally.
**Cons:** Inverted control flow is less familiar. The sink must not allocate either — the discipline propagates outward to every implementation. Harder to compose functionally.

### Option D: Object pool of `TradeEvent` instances

**Pros:** Keeps the object-returning shape.
**Cons:** Pool management on the hot path (borrow, return, exhaustion handling) is itself overhead, and lifetime bugs in a pool are subtle and severe. Modern GCs handle short-lived objects well — the answer to allocation pressure is to allocate less, not to hand-roll memory management.

## Trade-off Analysis

The trade is **API ergonomics against p99 latency**, and for this system the ergonomics lose decisively. A 100-byte-per-match API is not a minor inefficiency; it is the difference between a ~1µs p99 and a multi-millisecond one.

Option B is tempting because it preserves a return-value feel, but the aliasing hazard is exactly the kind of bug that passes review, passes tests, and corrupts data under load. Option C removes the hazard entirely by never handing a mutable structure across the boundary.

The discipline propagating outward is a genuine cost worth naming: every `TradeSink` implementation must also be allocation-free, or the benefit is lost one layer up. This is why verification is a build-failing CI check rather than a code review convention — the property is easy to state and easy to lose.

## Consequences

**Easier**
- p99 becomes a property of the algorithm, not of GC timing.
- ADR-012's GC choice becomes far less critical when there is almost nothing to collect.
- `gc.alloc.rate.norm = 0 B/op` is a single, unambiguous pass/fail number.

**Harder**
- Inverted control flow is less readable to newcomers.
- The zero-allocation obligation propagates to every sink implementation.
- Allocation regressions are invisible without the `-prof gc` check, which makes the CI gate essential rather than optional.
- Some functional composition patterns become awkward.

**To revisit**
- Nothing structural. If a future feature genuinely requires returning a collection, it belongs outside `apex-core`, built from emitted events.

## Action Items

1. [ ] Define `TradeSink` in `apex-common`; primitive parameters only
2. [ ] `long match(Order, TradeSink)` returning remaining quantity
3. [ ] Single `TradeSink` instance created at wiring time; never per call
4. [ ] Ring-buffer-writing sink implementation in `apex-ingestion` (ADR-008)
5. [ ] Collecting test sink for unit tests
6. [ ] **CI gate: JMH `-prof gc` asserts `gc.alloc.rate.norm == 0 B/op`; build fails on regression**
7. [ ] async-profiler in `-e alloc` mode as a second check on the full pipeline
8. [ ] Publish the 0 B/op figure in the README benchmark table
