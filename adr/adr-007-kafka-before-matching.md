# ADR-007: Kafka Before Matching, Not After

**Status:** Accepted
**Date:** 2026-09-18
**Deciders:** Rajenthar
**Related:** ADR-008 (outbound event bus), ADR-015 (deterministic replay), ADR-013 (recovery)

---

## Context

Orders arrive over HTTP/WebSocket at `apex-gateway`. They must reach the matching engine. The question is whether a durable log sits *before* matching or *after* it.

**Kafka before matching:** Gateway → Kafka → Ingestion → Matching. The order is durable before it is executed.

**Kafka after matching:** Gateway → Matching (in-memory) → Kafka. Matching happens immediately; the log records results.

The second is faster by roughly 200–500µs — one network round trip plus a broker write removed from the path to execution. For a system built around latency, that is not a trivial saving.

It is also the wrong choice here, and the reason only becomes fully clear in ADR-015: if matching happens before durability, then a crash between "matched in memory" and "written to Kafka" loses an executed trade with no way to reconstruct it. There is no log to replay, because the log is downstream of the event it would need to reproduce.

## Decision

**Kafka sits before matching. `orders.in` is the durable input log and the system's source of truth.**

- `apex-gateway` publishes with `acks=all` and `enable.idempotence=true`, and acks the client only after the producer callback confirms durability.
- The client ack means *"durably accepted"*, explicitly not *"matched"*. This distinction is published in the API contract.
- Per ADR-015, `orders.in` is partitioned by symbol and the partition offset serves as the sequence number.

## Options Considered

### Option A: Kafka before matching (chosen)

| Dimension | Assessment |
|---|---|
| Latency | +200–500µs to execution |
| Durability | **Order is durable before it executes** |
| Recovery | Full replay possible |
| Complexity | Low |

**Pros:** Nothing can be executed without first being recorded. Replay-based recovery (ADR-015) becomes possible, and with it deterministic reconstruction of engine state. Provides a natural audit trail. Backpressure has an obvious home: consumer lag.
**Cons:** Adds a broker round trip before execution. The client waits on durability before receiving an ack.

### Option B: Kafka after matching

| Dimension | Assessment |
|---|---|
| Latency | **Fastest** |
| Durability | Executed trades can be lost |
| Recovery | **Impossible to reconstruct** |
| Complexity | Low to build, high to make correct |

**Pros:** Removes 200–500µs from the path to execution — the genuinely fastest design.
**Cons:** A crash after matching and before publishing loses an executed trade permanently. No input log means no replay, so ADR-015's entire recovery model is unavailable. Would require durable output writes to be correct, which puts I/O back on the hot path — contradicting ADR-002.

### Option C: Both — in-memory match with async durable input log

**Pros:** Fast execution plus an eventual audit trail.
**Cons:** The log can lag the execution, so on crash the log and the engine's actual state disagree with no way to reconcile. Worse than either pure option: it has the complexity of both and the guarantees of neither.

### Option D: Local durable write-ahead log before matching, Kafka after

| Dimension | Assessment |
|---|---|
| Latency | Better than Kafka-before (local fsync ~50–100µs) |
| Durability | Good, but local to one node |

**Pros:** Meaningfully faster than a broker round trip while still durable-before-execute. This is closer to what the most latency-sensitive exchanges actually do.
**Cons:** Durability is local — losing the VM loses the log, and the single-VM deployment makes that a real scenario. Adds a second durable store to operate, snapshot and reason about. The complexity is justified at microsecond-sensitive scale; it is not justified at this project's scale.

## Trade-off Analysis

The trade is **200–500µs against the existence of a recovery story**. Stated that way the answer is clear, but the reasoning deserves to be explicit, because the latency cost is real and the choice should not look accidental.

The saving is real and it is the largest single latency cost in the design. It is worth paying because it is the *only* thing that makes ADR-015 possible, and ADR-015 is what makes the system correct under failure. A faster system that cannot recover from a crash without losing executed trades is not a better system.

Worth separating clearly: this cost is paid on the path to *execution*, not on the matching hot path itself. The core still matches in ~250ns. The 200–500µs is infrastructure latency, which is exactly the distinction the benchmark strategy reports separately (core vs end-to-end).

Option D is the honest "what would you do with more time" answer, and is the most likely future direction.

## Consequences

**Easier**
- ADR-015's replay recovery and ADR-013's snapshot model both become possible.
- Audit trail is free — the input log *is* the audit.
- Backpressure surfaces naturally as consumer lag.
- Gateway and matching scale and deploy independently.

**Harder**
- 200–500µs added to end-to-end latency; must be disclosed, not hidden.
- Ack semantics need careful definition and documentation at the API boundary.
- Kafka becomes a hard dependency on the order path — if the broker is down, orders are rejected rather than queued.

**To revisit**
- Option D (local WAL + async Kafka) if end-to-end latency becomes the binding constraint and multi-node durability is addressed separately.

## Action Items

1. [ ] Gateway producer: `acks=all`, `enable.idempotence=true`
2. [ ] Ack the client only inside the producer callback, never before
3. [ ] Partition `orders.in` by symbol (ADR-015)
4. [ ] Document ack semantics in the API contract: *durably accepted ≠ matched*
5. [ ] Define and document behaviour when the broker is unavailable (reject with a clear error; do not buffer silently)
6. [ ] Measure the actual gateway→ingestion latency and publish it as a named component of end-to-end p99
