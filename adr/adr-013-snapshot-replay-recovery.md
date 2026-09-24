# ADR-013: Snapshot + Log Replay Recovery

**Status:** Accepted
**Date:** 2026-09-18
**Deciders:** Rajenthar
**Amended by:** ADR-015 (the sequencing mechanism specified here originally was non-deterministic and has been replaced)
**Related:** ADR-007 (Kafka before matching), ADR-015 (deterministic replay), ADR-016 (dedupe state)

---

## Context

The order book lives in memory. A process restart — crash, deploy, host failure — loses it entirely. Something must reconstruct it.

ADR-007 establishes that `orders.in` holds every order durably before execution, and ADR-015 establishes that `apex-core` is a deterministic state machine over that log. Together these make reconstruction possible in principle: replay the log from the beginning and the engine arrives at exactly the state it had.

In practice, replaying from the beginning does not scale. After a week at 10k orders/sec the log holds billions of records, and recovery would take hours. Something must bound the replay.

**Note on history:** this ADR originally specified a snapshot mechanism built on an in-memory sequencer in `apex-ingestion`. That was non-deterministic — a crash produced different sequence numbers on redelivery, so replay did not reproduce the original execution and recovery silently did not work. ADR-015 replaced that sequencer with the Kafka partition offset. This ADR is retained for the snapshot design; its sequencing content is superseded.

## Decision

**Periodic snapshots of engine state, plus replay of the log from the snapshot's offset forward.**

Snapshot contents:

| Component | Why |
|---|---|
| Full order book per symbol (all resting orders, all price levels) | The primary state |
| `lastProcessedOffset` per partition | Replay start point and the idempotence key (ADR-015) |
| Client dedupe map (ADR-016) | Must survive restart or idempotency guarantees break |
| Order ID counter (ADR-014) | Must not reissue IDs |

Recovery:

```
1. Load most recent valid snapshot        → state at offset N
2. Seek each partition to N+1
3. Replay to the log head                 → deterministic reconstruction
4. Resume live consumption
```

Operational rules:

- **Snapshots are written off the matching thread.** The engine hands a consistent view to a background writer; it does not serialize inline (ADR-002, ADR-008).
- **Snapshot interval is configuration, not a constant.** Start at every 5 minutes or 1M events, whichever first.
- **Offset commit follows snapshot durability** (ADR-015): snapshot written → offset committed.
- **Snapshots are validated on write** (checksum) and recovery falls back to the previous snapshot if the newest is corrupt. Retain at least three.
- **Replay re-emits outputs**, which downstream consumers dedupe on deterministic trade IDs (ADR-015). This is expected behaviour, not an anomaly.

## Options Considered

### Option A: No recovery — restart with an empty book

**Pros:** Nothing to build.
**Cons:** Every restart silently discards every resting order. Unacceptable for anything modelling an exchange.

### Option B: Replay the full log from the beginning

| Dimension | Assessment |
|---|---|
| Complexity | **Lowest** — no snapshot machinery at all |
| Recovery time | Grows without bound |
| Correctness | Perfect |

**Pros:** Simplest correct design. No snapshot format, no versioning, no corruption handling. Given ADR-015's determinism, it is exactly right — just slow.
**Cons:** Recovery time grows linearly with log length forever. Also requires infinite log retention, which a single-broker Redpanda cannot provide.

### Option C: Snapshot + replay from snapshot (chosen)

| Dimension | Assessment |
|---|---|
| Complexity | Medium — format, versioning, corruption handling |
| Recovery time | **Bounded** by snapshot interval |
| Correctness | Same as B, given determinism |

**Pros:** Recovery time is bounded and tunable by one knob. Permits finite log retention. Snapshots double as a debugging artifact — engine state at a known point. Builds directly on ADR-015's determinism.
**Cons:** Snapshot format needs versioning, or a schema change makes old snapshots unreadable. Corruption must be handled. Serialization is background work competing for two cores. State must be captured consistently without stalling matching.

### Option D: Continuous state replication to a hot standby

**Pros:** Near-zero recovery time; also provides HA.
**Cons:** A substantially larger problem — leader election, split-brain, replication lag, fencing. Out of scope, and would need its own ADR. It is the production next step.

## Trade-off Analysis

Options B and C are equally correct; the trade is purely **recovery time against snapshot machinery**. Given that the log cannot be retained indefinitely on this deployment, C is not really optional — B would eventually be unable to recover at all once retention truncated the log's head.

The design cost that matters most is not the snapshot format but **capturing state consistently without stalling the matching thread**. The clean approach, given ADR-003's single-threaded model, is for the matching thread to publish a snapshot-request marker into its own event stream; when it reaches that marker it hands a reference to the background writer at a known-consistent point, then continues. No lock, no pause, no copy on the hot thread.

Snapshot interval is a genuine tuning axis: shorter means faster recovery and shorter replay, at the cost of more background serialization on a 2-core host. It should be set from measured recovery time, not guessed.

## Consequences

**Easier**
- Recovery time is bounded and tunable.
- Log retention can be finite.
- Snapshots are useful for debugging and for seeding test environments.

**Harder**
- Snapshot format requires versioning from day one — retrofitting it is painful.
- Corruption handling and multi-snapshot retention are required, not optional.
- Serialization competes for CPU on a 2-core host.
- Recovery correctness depends entirely on ADR-015's Determinism Contract holding. A determinism violation shows up here, as silently wrong recovered state.

**To revisit**
- Snapshot interval, from measured recovery duration under realistic log rates.
- Option D (hot standby) if HA becomes a goal.
- Snapshot storage location — local disk is a single point of failure on a single-VM deployment.

## Action Items

1. [ ] Versioned snapshot format from the first commit; include a schema version field
2. [ ] Snapshot contents per the table above — book, offsets, dedupe map, ID counter
3. [ ] Marker-based consistent capture; serialization on a background thread
4. [ ] Checksum on write; validate on read; retain the last three snapshots
5. [ ] Recovery path: load → seek → replay → resume, with each stage logged and timed
6. [ ] Snapshot interval in configuration; default 5 minutes or 1M events
7. [ ] **Test: kill the process mid-flow, recover, assert the book matches a non-crashed control run exactly**
8. [ ] Measure and publish recovery time; feed it back into the interval default
9. [ ] Document snapshot storage location and its failure mode in the README
