# ADR-012: Garbage Collector Selection

**Status:** Proposed
**Date:** 2026-09-18
**Deciders:** Rajenthar
**Related:** ADR-009 (allocation-free match), ADR-016 (dedupe map memory)

---

## Context

The JVM's garbage collector determines the tail latency of any Java system that allocates. Apex's profile is unusual and shapes the choice:

- **The matching hot path allocates nothing** (ADR-009), verified at 0 B/op.
- **Some allocation remains off the hot path** — deserialization in ingestion, Kafka producer internals, HTTP handling in the gateway, `PriceLevel` creation on level churn.
- **The live set is small but not trivial** — order books, plus the ADR-016 dedupe map, which is the single largest consumer and may run to hundreds of megabytes.
- **Heap is ~400 MB** in the matching container.
- **The host is a 2-OCPU Ampere Altra VM** — GC threads compete directly with the matching thread for a very small number of cores.

That last point is the one most easily missed. A concurrent collector trades throughput and CPU for shorter pauses. With two cores, "spend CPU to avoid pauses" may simply move the stall from a GC pause to matching-thread starvation.

The honest position: **ADR-009 makes this decision much less critical than it would otherwise be.** A collector's pause behaviour matters in proportion to how much there is to collect. This ADR is therefore proposed rather than accepted — it should be settled by measurement, not by reputation.

## Decision

**Start with G1 (the default), measure, and switch only on evidence.**

- Baseline: G1 with `-XX:MaxGCPauseMillis=10`, heap fixed with `-Xms == -Xmx` to avoid resizing pauses.
- `-XX:+UseCompressedOops` (default at this heap size, and assumed by ADR-011).
- Instrument GC pause time, frequency and allocation rate into Grafana from day one.
- **Decision gate:** if measured GC pauses contribute materially to end-to-end p99 under Gatling load, evaluate Generational ZGC. If they do not, G1 stays and the simplicity is kept.

Pre-committing to a collector before measuring would be choosing on reputation rather than evidence, which is precisely the error ADR-010 corrected for ring buffer sizing.

## Options Considered

### Option A: G1 (chosen as baseline)

| Dimension | Assessment |
|---|---|
| Pause target | ~10ms achievable, not guaranteed |
| CPU cost | Moderate |
| Fit for 2 OCPU | Good |
| Maturity | Highest — the default since Java 9 |

**Pros:** Default, so no configuration risk and the best-understood failure modes. Region-based collection handles a small live set efficiently. Modest CPU appetite, which matters on two cores. Excellent tooling and diagnostics.
**Cons:** Pauses in the millisecond range, which would be visible in core-only p99 if the hot path allocated — it does not. Full GC, though rare, is a multi-hundred-millisecond stall.

### Option B: Generational ZGC

| Dimension | Assessment |
|---|---|
| Pause target | **Sub-millisecond**, largely heap-size independent |
| CPU cost | **High** — concurrent work on every core |
| Fit for 2 OCPU | **Questionable** |
| Maturity | Good in Java 21+ |

**Pros:** Sub-millisecond pauses are exactly what a latency-focused system wants, and generational ZGC (Java 21) fixed the throughput regression of the non-generational version. Scales to large heaps, which matters if ADR-016's dedupe map grows.
**Cons:** Concurrent collection consumes CPU continuously, and on two cores that CPU is taken directly from the matching thread. Load barriers add a small cost to every reference read — including inside the matching loop. May well produce *worse* end-to-end throughput here despite better pause numbers. This is the counterintuitive result worth measuring rather than assuming.

### Option C: Shenandoah

**Pros:** Low pause times, similar goals to ZGC, sometimes lighter.
**Cons:** Not in all JDK builds; less common on ARM; same fundamental CPU-contention concern as ZGC. No clear advantage over Generational ZGC to justify the extra variable.

### Option D: Epsilon (no-op collector)

**Pros:** Zero GC overhead, zero pauses. Would be a striking demonstration if the system truly allocated nothing.
**Cons:** The heap fills and the JVM dies. Only viable if allocation were zero *everywhere*, which it is not — ingestion, Kafka and the gateway all allocate.
**However:** Epsilon is genuinely useful as a **test tool**. Running the `apex-core` JMH benchmark under Epsilon proves the 0 B/op claim absolutely — if the core allocated anything, the benchmark would eventually die. Adopted for that purpose, not for production.

## Trade-off Analysis

The instinct for a low-latency system is to reach for ZGC, and on larger hardware that would likely be right. On 2 OCPUs the reasoning inverts: ZGC's concurrent threads and load barriers consume the same scarce cores the matching thread needs, and its advantage — shorter pauses — applies to a hot path that, thanks to ADR-009, barely triggers collection at all.

The general principle: **ADR-009 is the real GC strategy. The collector choice is secondary.** Eliminating allocation on the hot path does more for p99 than any collector can, and it makes the collector decision low-stakes rather than critical. Reversing that priority — tuning GC while allocating 400 MB/s — would be optimising the wrong layer.

The one factor that could change the answer is ADR-016's dedupe map. If it grows to several hundred megabytes of long-lived state, old-generation collection becomes significant and ZGC's heap-size-independent pauses become genuinely attractive. This is a concrete trigger to re-evaluate, not a vague one.

## Consequences

**Easier**
- No premature configuration complexity; the default is the starting point.
- GC metrics are instrumented from day one, so the decision gate has data when it is reached.
- Epsilon gives an absolute, unambiguous proof of the 0 B/op claim.

**Harder**
- The decision stays open until load testing, which means it cannot be written up as settled yet.
- GC tuning on ARM has a smaller body of published experience than x86.

**To revisit — explicit triggers**
- GC pauses contribute materially to end-to-end p99 under Gatling load → evaluate Generational ZGC.
- ADR-016's dedupe map exceeds ~200 MB of live set → re-evaluate, since old-gen pressure changes the calculus.
- Deployment moves to a host with more cores → ZGC's CPU cost becomes far less relevant.

## Action Items

1. [ ] Baseline: G1, `-Xms == -Xmx` at 400 MB, `-XX:MaxGCPauseMillis=10`
2. [ ] Export GC pause count, pause duration and allocation rate to Grafana
3. [ ] Run the `apex-core` JMH benchmark under `-XX:+UseEpsilonGC` as an absolute 0 B/op proof
4. [ ] Under Gatling load, attribute end-to-end p99 between GC pauses and everything else
5. [ ] If GC is material: A/B G1 vs Generational ZGC on the same load profile and **publish both results** — the comparison is more interesting than the winner
6. [ ] Record the measured outcome here and move this ADR to Accepted
