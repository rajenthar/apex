# ADR-014: Order ID Generation

**Status:** Accepted
**Date:** 2026-09-18
**Deciders:** Rajenthar
**Related:** ADR-003 (single thread per symbol), ADR-013 (snapshot), ADR-015 (determinism), ADR-016 (client idempotency)

---

## Context

Every accepted order needs a server-assigned identifier: returned to the client, referenced by cancel and amend, recorded on every trade, and stored in Postgres.

The requirements are narrower than they first appear:

- **Unique** across the system's lifetime.
- **Deterministic** — replay must reproduce the same IDs, or recovered state disagrees with already-persisted trades (ADR-015). This is the constraint that eliminates most conventional options.
- **Cheap** — assigned on the order path, so no allocation and no coordination.
- **`long`** — it occupies 8 bytes of the 64-byte `Order` budget (ADR-011), so a `String` or `UUID` is out.

Determinism is the decisive requirement and the one most easily overlooked. Anything drawing on wall-clock time, randomness, or cross-thread coordination produces different IDs on replay, which quietly corrupts recovery.

## Decision

**A per-symbol counter, incremented on the matching thread, included in the snapshot.**

- Because ADR-003 guarantees one thread per symbol, this is a plain `long` field — not an `AtomicLong`. There is no contention, so no atomic is required. Increment is a single instruction.
- The counter is part of engine state and therefore part of every snapshot (ADR-013), so it never reissues an ID after restart.
- Replay reproduces the identical sequence, satisfying ADR-015.
- The externally visible ID packs the symbol shard with the counter, keeping it globally unique without coordination:

```
orderId = (symbolShardId << 48) | counter
```

16 bits of shard (65,536 symbols) and 48 bits of counter (~281 trillion orders per symbol) is comfortable headroom for any realistic lifetime.

Trade IDs are handled separately and derive from `(symbol, inputOffset, tradeIndex)` per ADR-015 — deterministic by construction without any counter at all.

## Options Considered

### Option A: Per-symbol counter on the matching thread (chosen)

| Dimension | Assessment |
|---|---|
| Complexity | **Lowest** |
| Determinism | **Perfect** |
| Cost | One increment, no atomic |

**Pros:** Trivial. Deterministic. No coordination, no allocation, no atomic instruction — ADR-003's threading model makes the simple thing correct. Snapshot inclusion handles restart. IDs are sequential per symbol, which is convenient for debugging and for index locality in Postgres.
**Cons:** Sequential IDs leak order volume to anyone who sees two of them — a genuine information leak in a real exchange, out of scope here but worth naming. Requires shard-ID stability: symbol-to-shard assignment must not change without care.

### Option B: `AtomicLong` shared across symbols

**Pros:** Globally unique without packing; conceptually simple.
**Cons:** A CAS on the hot path for no benefit, since ADR-003 already removed contention. Worse, it is **non-deterministic**: the interleaving of increments across symbol threads varies between runs, so replay produces different IDs. Breaks ADR-015. The original design specified this; it was wrong.

### Option C: Snowflake-style (timestamp + node + sequence)

| Dimension | Assessment |
|---|---|
| Complexity | Medium |
| Determinism | **None** — embeds wall clock |
| Multi-node coordination | Solves a problem this system does not have |

**Pros:** Globally unique across nodes without coordination; roughly time-ordered; a recognisable distributed-systems pattern.
**Cons:** Embeds wall-clock time, so replay at a different moment produces different IDs — a direct violation of ADR-015's Determinism Contract. Solves multi-node coordination the system does not have. Clock skew and leap seconds add failure modes for no gain here.

### Option D: `UUID`

**Pros:** Universally unique, no coordination at all.
**Cons:** 128 bits — blows the ADR-011 byte budget on its own. Random, so non-deterministic. Allocates. Poor index locality in Postgres. Wrong on four independent counts.

## Trade-off Analysis

This looks like a small decision and mostly is, but it contains one genuine trap: **the conventional answers are non-deterministic.** `AtomicLong` across threads, Snowflake, and `UUID` are all perfectly reasonable in an ordinary service and all break replay recovery here. The original design reached for `AtomicLong` precisely because it is the reflexive Java answer.

The chosen option is simpler than all of them *because* of ADR-003. Single-threaded access removes the need for atomics entirely — a good illustration of how a threading decision made for latency reasons pays out as simplicity elsewhere.

The sequential-ID information leak is real and worth acknowledging rather than ignoring. In a production exchange it would matter; the mitigation would be a separate public-facing opaque identifier mapped to the internal `long`. Out of scope here, but a good answer to have ready.

## Consequences

**Easier**
- ID generation costs one increment with no atomic and no allocation.
- Deterministic, so replay recovery is consistent with persisted data.
- Sequential per symbol gives good Postgres index locality and readable debugging.

**Harder**
- Symbol-to-shard mapping must be stable across restarts, or packed IDs collide. This must be persisted, not derived from iteration order.
- Sequential IDs leak volume information.
- The counter is one more piece of state that must be in the snapshot — omitting it silently reissues IDs after restart.

**To revisit**
- An opaque external identifier if the volume leak ever matters.
- Shard bit-width if symbol count ever approaches 65,536.

## Action Items

1. [ ] Plain `long` counter per `OrderBook` — explicitly **not** `AtomicLong`
2. [ ] Pack as `(symbolShardId << 48) | counter`
3. [ ] Include the counter in the snapshot (ADR-013 action item 2)
4. [ ] Persist the symbol-to-shard mapping; never derive it from map iteration order
5. [ ] Test: restart from snapshot, assert no ID is reissued
6. [ ] Test: replay the same log twice, assert identical order IDs (ADR-015 compliance)
7. [ ] Document the sequential-ID volume leak in the README limitations section
