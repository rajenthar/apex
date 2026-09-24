# ADR-011: `Order` Fixed at 64 Bytes — One Cache Line

**Status:** Accepted
**Date:** 2026-09-18
**Deciders:** Rajenthar
**Related:** ADR-005 (fixed-point `long`), ADR-006 (order book), ADR-010 (ring buffer sizing)

---

## Context

`Order` is the most frequently touched object in the system. Every match reads several of them while walking price levels; every ring buffer slot holds one.

On Ampere Altra, as on essentially all modern hardware, memory moves between cache and RAM in **64-byte cache lines**. An object that fits within one line costs one fetch. An object of 65 bytes costs two — a 100% increase in memory traffic for a 1.5% increase in size. The cost is a cliff, not a slope.

With compressed oops (automatic for heaps under 32 GB), the current field set lands exactly on the boundary:

```
object header             12 bytes
long orderId               8
long price                 8
long quantity              8
long remainingQuantity     8
long timestamp             8
Side (reference)           4
OrderType (reference)      4
String symbol (reference)  4
                        ─────────
                          64 bytes   ← exactly one cache line
```

This is currently true by luck rather than by design. Adding one `long` field pushes it to 72 bytes, and every order access silently doubles its memory traffic — with no test failure and no obvious symptom.

## Decision

**`Order` is fixed at 64 bytes. The layout is a documented constraint, verified by test.**

- No field may be added without removing an equivalent number of bytes, or explicitly accepting and documenting the move to two cache lines.
- A JOL (Java Object Layout) test asserts `sizeOf(Order) == 64` and fails the build on regression.
- The ring buffer event type (ADR-010) is held to the same 64-byte budget, since slot size drives total buffer memory.
- Verified under `-XX:+UseCompressedOops` (the default below 32 GB) — the heap is 400 MB, so this holds.

Fields that would be natural to add and must instead live elsewhere: `clientOrderId` (ADR-016 keeps it in a side map keyed by hash), sequence number (derivable from Kafka offset per ADR-015), and any audit or diagnostic metadata (belongs on the emitted event, not the resident order).

## Options Considered

### Option A: Ignore layout, let the JVM decide

| Dimension | Assessment |
|---|---|
| Complexity | None |
| Cache behaviour | Unpredictable, silently degrading |

**Pros:** Zero effort; fields get added freely as features arrive.
**Cons:** The 64→72 byte cliff is invisible. Throughput regresses by a measurable margin with no failing test and no obvious cause, and the regression is attributed to whatever else changed that week.

### Option B: Fix at 64 bytes, enforced by test (chosen)

| Dimension | Assessment |
|---|---|
| Complexity | Low — one test, one documented rule |
| Cache behaviour | **One fetch per order, guaranteed** |

**Pros:** Guarantees single-fetch access. Makes an invisible property visible and enforceable. Forces genuine thought about whether a field belongs on the resident object or on the emitted event — which is usually the better design anyway. The constraint is cheap to state and cheap to check.
**Cons:** A real constraint on future features. Some fields must live in side structures, adding indirection where a field would have been simpler. Requires JOL as a test dependency.

### Option C: Pack aggressively into fewer bytes (bit-packing side/type into a status `long`)

**Pros:** Leaves headroom below 64 bytes for future fields.
**Cons:** Bit manipulation obscures the code in the one module that must be most obviously correct. The enum references cost 8 bytes total; packing them buys headroom at a real cost in readability. Premature — take this only if a field genuinely must be added later.

### Option D: Off-heap / `MemorySegment` layout (Project Panama)

**Pros:** Total layout control, no header overhead, no GC involvement at all.
**Cons:** Substantially more complex; manual lifetime management; loses ordinary debuggability and tooling. The 64-byte target is already achievable on-heap. Worth considering only if allocation or GC proves problematic after ADR-009 and ADR-012 are in place — which they should not.

## Trade-off Analysis

The trade is **future flexibility against guaranteed cache behaviour**. The constraint is genuinely restrictive, and it will at some point block a field someone wants to add.

That is largely a feature. The discipline it enforces — *does this data need to be resident on every order, or does it belong on the event emitted once?* — leads to better designs independently of cache concerns. Most fields people want to add to an order (audit metadata, diagnostic flags, denormalised client details) belong on the emitted trade event, where they are written once and never re-read in a hot loop.

The decisive argument for enforcement by test rather than by convention is the failure mode. An unenforced layout rule does not fail loudly; it degrades quietly and gets blamed on something else. A JOL assertion turns an invisible performance cliff into a red build.

## Consequences

**Easier**
- One cache fetch per order access, guaranteed rather than hoped for.
- Ring buffer memory is exactly predictable (ADR-010).
- An invisible performance property becomes a visible, testable one.

**Harder**
- Adding a field requires a deliberate decision and possibly a design change.
- Some data must live in side structures with the indirection that implies.
- JOL becomes a test dependency; the assertion is JVM- and flag-sensitive.
- The assertion holds for compressed oops specifically — it would need revisiting if the heap ever exceeded 32 GB, which it will not here.

**To revisit**
- If a field genuinely must be added, consider Option C (bit-packing) before accepting 128 bytes.
- Re-derive if the target hardware changes cache line size (unlikely; 64 bytes is near-universal).

## Action Items

1. [ ] JOL test asserting `sizeOf(Order) == 64` under compressed oops; build fails on regression
2. [ ] Same assertion for the ring buffer event type
3. [ ] Document the field budget in `Order`'s Javadoc, with the byte accounting shown above
4. [ ] Note explicitly which fields deliberately live elsewhere and why (`clientOrderId`, sequence, audit metadata)
5. [ ] Record `-XX:+UseCompressedOops` as an assumed JVM flag in the deployment configuration
