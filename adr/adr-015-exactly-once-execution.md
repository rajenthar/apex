# ADR-015: Exactly-Once Execution via Deterministic Replay

**Status:** Proposed
**Date:** 2026-09-18
**Deciders:** Rajenthar
**Supersedes:** part of ADR-013 (snapshot + replay recovery)
**Related:** ADR-002 (zero-I/O core), ADR-007 (Kafka before matching), ADR-008 (outbound event bus), ADR-016 (client idempotency)

---

## Context

Apex matches orders. A duplicated match creates a trade that never should have existed; a dropped match silently loses a client's order. Both are unacceptable in a system that claims to model an exchange, and both matter more than any latency number.

The design as originally specified could do either. Three concrete holes:

**H1 — Consumer redelivery.** `apex-ingestion` reads an order from `orders.in`, publishes it into the Disruptor ring buffer, and crashes before committing the Kafka offset. On restart it re-reads the same offset and the order is matched a second time. **Double execution.**

**H2 — Fire-and-forget outputs.** Per ADR-008 the matching core emits trades to the outbound event bus and never waits. If the process dies between "matched in memory" and "trade published to `trades.out`", the match happened and no downstream system will ever know. **Lost execution.**

**H3 — Non-durable sequencing.** The sequencer assigns sequence numbers *after* Kafka, in `apex-ingestion`. Those numbers exist only in memory. After a crash, a different interleaving of redelivered records yields different sequence numbers, so replaying the log does not reproduce the original execution. **Recovery is not deterministic**, which means the Phase 4 snapshot/replay design in ADR-013 does not actually work as specified.

Forces at play:
- ADR-002 forbids I/O in `apex-core`, so the matching thread cannot itself write durably.
- ADR-008's fire-and-forget bus is what keeps a slow database off the matching thread — we do not want to give it up.
- The hot path must remain allocation-free (ADR-009) and the dedupe mechanism must not undo that.
- Single-VM deployment: one Redpanda broker, replication factor 1.

**Constraint:** true end-to-end exactly-once *delivery* is impossible across a network boundary. The achievable goal is **exactly-once effect**: the order is executed exactly once, and every downstream system converges on that single outcome regardless of retries or replays.

---

## Decision

**Make the durable input log the single source of truth, and the matching engine a deterministic state machine over it.**

Four changes:

1. **Partition `orders.in` by symbol, and use the Kafka partition offset as the sequence number.** Delete the separate sequencer. Kafka already provides a durable, gap-free, monotonic total order per partition — inventing a second one in memory was the cause of H3.

2. **`apex-core` becomes strictly deterministic.** Same inputs, same order ⟹ byte-identical outputs, always. This is enforced by rule (see Determinism Contract below) and verified by test.

3. **The engine carries `lastProcessedOffset` in its own state**, included in every snapshot. On replay, any record at or below it is skipped. This makes at-least-once redelivery from Kafka idempotent at the engine boundary, closing H1.

4. **Trade IDs become deterministic and reproducible:** `(symbol, inputOffset, tradeIndexWithinOrder)`. A replay re-emits *identical* events with *identical* IDs, so downstream systems dedupe naturally rather than needing coordination. This closes H2.

The key insight: **durable outputs are unnecessary when you have a durable, deterministic input log.** Reproducibility replaces output durability — which is precisely what lets us keep the fire-and-forget event bus from ADR-008 without sacrificing correctness.

### Offset commit protocol

Commit a Kafka offset only after a snapshot containing it has been durably written. On crash, replay resumes from the last snapshot; everything between snapshot and crash is re-executed, re-emitted with identical IDs, and deduped downstream.

```
snapshot(N) written durably  →  commit offset N  →  continue
crash at offset N+k          →  restore snapshot(N)  →  replay N+1..N+k
                             →  identical trades, identical IDs  →  downstream no-ops
```

Snapshot interval is a tuning knob: shorter means faster recovery and less replay, at the cost of more frequent (off-thread) serialization work.

### Determinism Contract for `apex-core`

These break replay and are therefore forbidden in the matching module:

| Forbidden | Why | Instead |
|---|---|---|
| `System.currentTimeMillis()` / `nanoTime()` | Replay produces different timestamps → different trade records | Timestamp assigned in `apex-ingestion`, carried in the input event, durable in the log |
| `HashMap` / `HashSet` iteration | Iteration order is not guaranteed stable across JVM runs | `LinkedHashMap`, or sorted/primitive collections (already required by ADR-006) |
| `Random`, `UUID.randomUUID()` | Self-evident | Derive IDs from `(symbol, offset, index)` |
| Wall-clock-driven logic (order expiry, session close) | Replay at a different wall time diverges | Explicit timer events written into the input log |
| Floating point where rounding could differ | Platform/JIT variance | Fixed-point `long` (already required by ADR-005) |
| `tryPublishEvent` on the ring buffer | Silently drops on a full buffer — data loss | Blocking `publishEvent`; let backpressure surface as Kafka consumer lag |

### Producer/consumer configuration

| Setting | Value | Reason |
|---|---|---|
| Gateway producer `enable.idempotence` | `true` | Dedupes producer-side retries within a partition |
| Gateway producer `acks` | `all` | Do not ack the client before the record is durable |
| Ingestion consumer `enable.auto.commit` | `false` | Offsets commit on the snapshot protocol above, not on a timer |
| `trades.out` producer `enable.idempotence` | `true` | Republished trades carry identical keys |
| Postgres trade insert | `ON CONFLICT (trade_id) DO NOTHING` | Deterministic trade ID as primary key |

---

## Options Considered

### Option A: Kafka offset as sequence number, deterministic replay (chosen)

| Dimension | Assessment |
|---|---|
| Complexity | Medium — determinism is a discipline, not a library |
| Latency cost | Near zero on the hot path; one `long` comparison per order |
| Scalability | Good — offsets are per-partition, partitions are per-symbol |
| Precedent | Strong — this is how LMAX and real exchanges do it |

**Pros**
- No distributed transaction, no coordinator, no two-phase commit anywhere.
- Preserves the fire-and-forget event bus (ADR-008) and the zero-I/O core (ADR-002) unchanged.
- Determinism is a *testable* property: replay the log twice, assert byte-identical output.
- Recovery and audit fall out of the same mechanism for free.
- Costs one `long` field in engine state and one comparison per order.

**Cons**
- Every contributor must respect the Determinism Contract; a single stray `currentTimeMillis()` silently breaks recovery.
- Downstream consumers must be idempotent — this pushes work to the edges.
- Replay window between snapshots re-emits duplicate events (harmless, but visible in metrics if not accounted for).

### Option B: Kafka transactions (exactly-once semantics, EOS)

| Dimension | Assessment |
|---|---|
| Complexity | High — transactional producer, consumer groups, fencing |
| Latency cost | Significant — transaction commit on the critical path |
| Scalability | Poor fit — transaction coordinator becomes a bottleneck |
| Precedent | Common, but built for Kafka-to-Kafka pipelines, not this shape |

**Pros:** Framework-provided guarantee; less hand-written discipline.
**Cons:** Transaction commits add milliseconds to a path we measured in nanoseconds; only covers Kafka-to-Kafka, so Postgres and WebSocket still need their own handling; directly contradicts the low-latency thesis of the whole project.

### Option C: Synchronous durable write before acking each match

| Dimension | Assessment |
|---|---|
| Complexity | Low conceptually |
| Latency cost | Catastrophic — fsync on the matching thread |
| Scalability | Poor |
| Precedent | Contradicts the latency goals of the system outright |

**Pros:** Trivially correct, easy to explain.
**Cons:** Puts blocking I/O directly on the hot path, violating ADR-002 and destroying the entire latency premise. Would turn ~250ns matches into millisecond matches.

### Option D: Accept at-least-once, dedupe only at the database

**Pros:** Minimal work.
**Cons:** The engine's in-memory book would still be corrupted by a double-matched order — the book state diverges from the trade record. Deduping at the database fixes the ledger while leaving the source of truth wrong. Rejected as incorrect, not merely weak.

---

## Trade-off Analysis

The central trade is **output durability vs. input durability**. Option C buys correctness with output durability and pays in latency. Option A buys the same correctness with input durability plus determinism, and pays in engineering discipline instead. For a system whose entire thesis is a nanosecond-scale hot path, paying in discipline is the only coherent choice.

Against Option B: Kafka transactions solve a broader problem (arbitrary non-deterministic consumers) that we do not have. Because our consumer *is* deterministic, we can use a far cheaper mechanism. Choosing EOS here would be paying for generality we deliberately do not need — and it would not cover the Postgres or WebSocket legs anyway.

The residual cost of Option A is real and should be stated plainly: duplicate *emissions* during the replay window are normal and expected. The guarantee is exactly-once **execution**, with **idempotent** downstream effects — not exactly-once delivery. Conflating the two is the most common way this gets explained badly.

---

## Consequences

**Easier**
- Recovery, audit, and time-travel debugging all become the same mechanism: replay the log.
- Correctness becomes testable rather than argued — determinism has a pass/fail assertion.
- Downstream consumers can be naively at-least-once and still converge.
- Snapshot interval becomes a single clean knob trading recovery time against background work.

**Harder**
- The Determinism Contract is a permanent constraint on every future change to `apex-core`. Adding a "quick" timestamp for logging silently breaks recovery with no immediate symptom.
- Every downstream sink now needs an idempotent write path (Postgres upsert, WebSocket resync).
- Debugging a determinism violation is painful: the symptom appears only at replay, far from the cause. Mitigate with a CI replay-equality test on every commit.

**To revisit**
- Snapshot interval, once real replay durations are measured.
- Whether `trades.out` should also be partitioned by symbol for ordered downstream consumption.
- Multi-instance deployment: this ADR assumes one matching process per symbol partition. Hot-standby with leader election is out of scope and would need its own ADR.

**Explicitly not solved by this ADR**
- Client-side retries producing genuinely distinct Kafka records → **ADR-016**.
- WebSocket delivery gaps → needs a sequence-numbered feed with client resync.
- Total data loss if the single Redpanda broker (RF=1) is lost. This is the hard durability ceiling of a single-VM deployment and should be stated in the README rather than papered over.

---

## Action Items

1. [ ] Repartition `orders.in` by symbol; remove the standalone sequencer from `apex-ingestion`
2. [ ] Add `lastProcessedOffset` to engine state and to the snapshot format
3. [ ] Implement deterministic trade ID: `(symbol, inputOffset, tradeIndex)`
4. [ ] Move timestamp assignment from `apex-core` into `apex-ingestion`
5. [ ] Switch ring buffer publication from `tryPublishEvent` to blocking `publishEvent`
6. [ ] Apply the producer/consumer configuration table above
7. [ ] Implement the snapshot-then-commit offset protocol
8. [ ] Add `ON CONFLICT (trade_id) DO NOTHING` to the persistence writer
9. [ ] **CI test: replay the same log twice, assert byte-identical trade output**
10. [ ] Write a static check (ArchUnit) forbidding `currentTimeMillis`, `Random`, and `HashMap` iteration in `apex-core`
11. [ ] Document the guarantee precisely in the README: *exactly-once execution, idempotent downstream effects, at-least-once emission*
