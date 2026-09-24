# ADR-016: Client Idempotency via Engine-Resident Keys

**Status:** Proposed
**Date:** 2026-09-18
**Deciders:** Rajenthar
**Related:** ADR-015 (exactly-once execution), ADR-002 (zero-I/O core), ADR-003 (single thread per symbol), ADR-009 (allocation-free hot path)

---

## Context

ADR-015 gives exactly-once execution for everything *inside* the system: redelivery, replay and recovery all converge on a single execution per input record. It does not cover the client boundary.

The uncovered case: a client submits an order, the gateway accepts it durably, and the HTTP response is lost in transit. The client times out and retries. This produces **two distinct records in `orders.in`, at two distinct offsets**, which ADR-015 correctly treats as two separate orders — because from the log's perspective, that is exactly what they are. The result is a genuine double execution that the offset mechanism cannot see.

`lastProcessedOffset` and a client idempotency key therefore solve different problems and both are required:

| Mechanism | Covers |
|---|---|
| `lastProcessedOffset` (ADR-015) | Internal redelivery and replay — same record, same offset |
| Client idempotency key (this ADR) | Client retries — same logical order, different offsets |

The same applies to **cancel and amend**, which are equally non-idempotent and are routinely forgotten.

Forces:
- The check sits on the order path, so it must be cheap and must not allocate (ADR-009).
- Whatever holds the dedupe state must be durable and must survive replay identically, or it reintroduces the non-determinism ADR-015 just eliminated.
- Unbounded key retention is not viable: at 10k orders/sec the system sees ~864M keys per day.
- A deduped retry must tell the client what actually happened to their order — otherwise the client cannot distinguish "already accepted" from "rejected" and will retry again.

---

## Decision

**Clients supply a `clientOrderId`. Deduplication happens inside the matching engine's own state, scoped per client, bounded by a published time window.**

The governing principle: **deduplicate at the point where the decision is made, inside the same durable state.** Any earlier and you split "key claimed" from "order executed" across two systems, creating a two-phase problem that needs its own recovery story.

Five specifics:

1. **Key scope is `(clientId, clientOrderId)`.** Uniqueness is a client-side obligation scoped to a session — the same contract real exchanges impose.

2. **Dedupe state lives in the engine and is part of the snapshot.** Because `apex-core` is single-threaded per symbol (ADR-003), the check is a lock-free O(1) lookup with no race. Because it is snapshotted, it is durable and replays deterministically, satisfying ADR-015.

3. **It is a map, not a set:** `clientOrderId → (orderId, status)`. A deduped retry returns the *original outcome*, not a bare rejection.

4. **Keys are hashed to `long` in `apex-ingestion`, never in `apex-core`.** The engine uses a `Long2ObjectOpenHashMap`. String handling stays out of the hot path entirely, preserving both the zero-allocation rule (ADR-009) and the no-framework rule (ADR-002).

5. **The window is bounded and published.** Retain one trading session (default 24h). Reject any order whose timestamp falls outside the window. The guarantee stated in the README: *idempotent within a 24-hour window, per client.*

### Request flow

```
Client  --(clientOrderId)-->  Gateway
                              └─ shape/auth validation only, no dedupe
                                 publish to orders.in (acks=all)
                                 ack client only after producer callback

Ingestion  ─ hash clientOrderId → long (off the hot path)
           ─ heavy validation
           └─ publish to ring buffer

Engine     ─ seen(clientId, hash)?
             ├─ yes → emit the stored (orderId, status); do not match
             └─ no  → record hash, match, store outcome
```

### Hash collision

A 64-bit hash within a bounded window of ~10M live keys gives a collision probability on the order of 10⁻¹². A collision would reject a legitimate order as a duplicate — a real but negligible failure. Two mitigations, in order of preference: require clients to send a numeric ID (no hashing needed), or store the full key alongside the hash and compare on hit. The second costs memory; take it only if a client contract mandating numeric IDs is not acceptable.

### Memory budget

At 10k orders/sec over 24h with per-client scoping, expect roughly 10M live entries. `Long2ObjectOpenHashMap` at ~48 bytes per entry (key, reference, value object, load factor) is **~480 MB** — larger than the entire 400 MB matching heap. Mitigations, applied together:

- Store the outcome as a packed `long` (orderId in the high bits, status in the low bits) in a `Long2LongOpenHashMap` → ~24 bytes/entry, ~240 MB.
- Shorten the default window to one trading session rather than a rolling 24h.
- Evict eagerly on session rollover rather than lazily per entry.

**This is the binding constraint on the feature**, and it must be measured rather than assumed before the window default is fixed.

---

## Options Considered

### Option A: Gateway-local in-memory cache

| Dimension | Assessment |
|---|---|
| Complexity | Low |
| Correctness | **Broken** |
| Latency cost | None |

**Pros:** Trivial; no hot-path cost.
**Cons:** Multiple gateway instances do not share the cache, so a retry routed to a different instance slips straight through. A gateway restart loses the state entirely. Two concurrent retries race. Rejected as incorrect, not merely weak.

### Option B: Redis at the gateway (`SET NX`)

| Dimension | Assessment |
|---|---|
| Complexity | Medium |
| Correctness | Conditionally correct |
| Latency cost | A network hop on the order path |

**Pros:** Shared across gateway instances; survives restarts; `SET NX` is atomic.
**Cons:** Introduces a two-phase problem — the key is claimed, then the gateway dies before publishing to Kafka. The key now says "seen" while no order exists, and the client's retry is permanently rejected. Fixing it requires a `reserved → committed` state machine with TTL-based reclaim, plus its own recovery story. Adds a network round trip to a path measured in nanoseconds, to solve a problem that need not be created.

### Option C: Dedupe inside the matching engine's state (chosen)

| Dimension | Assessment |
|---|---|
| Complexity | Medium — memory management is the real work |
| Correctness | Correct by construction |
| Latency cost | One primitive hash lookup, zero allocation |

**Pros:** Single-threaded, so no race and no locking. Durable and replay-deterministic via the snapshot. Authoritative, because it is co-located with the component that actually executes. No new infrastructure. No network hop.
**Cons:** Consumes matching-process heap — and per the budget above, this is the binding constraint. Couples client-protocol concerns into `apex-core`, which is a genuine purity cost against ADR-002 (mitigated by keeping all string handling in `apex-ingestion`). Key eviction becomes engine state that must itself replay deterministically.

### Option D: Dedupe in Postgres on write

**Pros:** Simple; the database is already the system of record for trades.
**Cons:** Too late. By the time the write happens the order has already matched and mutated the in-memory book. The ledger would be corrected while the source of truth stayed wrong — the same failure mode rejected as Option D in ADR-015.

---

## Trade-off Analysis

The decision is **where** to dedupe, and it reduces to a single question: can the dedupe record and the execution it guards be made durable atomically? Options A, B and D all split them, and every split demands its own reconciliation protocol. Option C keeps them in one place — the engine's snapshot — so they commit together by construction.

The price is heap in the matching process, which is the one resource the deployment is tightest on. That is a fair trade: memory is measurable, tunable, and fails loudly, whereas a distributed two-phase claim fails silently and intermittently, at 3am, under retry storms.

The purity objection — client-protocol state inside a module ADR-002 declared framework-free — is worth naming honestly. The mitigation is that `apex-core` sees only `long` keys and packed `long` values; every string, header and protocol concern stays in `apex-ingestion`. The core remains a pure function over primitives, which is what ADR-002 was actually protecting.

---

## Consequences

**Easier**
- The client boundary gains the same exactly-once guarantee the internals already have under ADR-015.
- No new infrastructure: no Redis, no extra hop, no additional failure mode in the deployment.
- Retry handling becomes uniform across submit, cancel and amend.
- A deduped retry returns a useful answer, so a well-behaved client converges after one retry instead of looping.

**Harder**
- Matching-process memory becomes a first-class operational concern with a hard ceiling.
- Key eviction is now engine state and must replay deterministically — eviction driven by wall clock would violate the ADR-015 Determinism Contract. Evict on explicit session-rollover events in the log, never on a timer.
- `apex-core` carries client-protocol awareness it would not otherwise have.

**To revisit**
- Window length and memory footprint, once real key-cardinality is measured rather than estimated.
- Whether to store full keys alongside hashes, if numeric client IDs cannot be mandated.
- Whether cancel/amend need a separate key namespace from submit.

**Not solved**
- A client that generates a *fresh* key on retry. That is a genuinely new order and matching it is correct behaviour; no server-side mechanism can fix a broken client contract.
- Idempotency across a client's own session boundaries, once keys age out of the window.

---

## Action Items

1. [ ] Add required `clientOrderId` to the REST/WS order schema; reject requests without one
2. [ ] Hash `(clientId, clientOrderId)` → `long` in `apex-ingestion`; keep all string handling out of `apex-core`
3. [ ] Add `Long2LongOpenHashMap` dedupe state to the engine, with packed `(orderId, status)` values
4. [ ] Include dedupe state in the snapshot format (coordinate with ADR-015 item 2)
5. [ ] Implement session-rollover eviction driven by an explicit log event, **not** by wall clock
6. [ ] Extend the same path to cancel and amend
7. [ ] Return the original `(orderId, status)` on a deduped retry, not a bare rejection
8. [ ] **Measure real memory footprint under Gatling load before fixing the window default**
9. [ ] Test: submit the same `clientOrderId` twice, assert one execution and two identical responses
10. [ ] Test: replay a log containing duplicate `clientOrderId`s, assert byte-identical output (ADR-015 compliance)
11. [ ] Publish the guarantee in the README: *idempotent within a 24-hour window, per client*
