# ADR-005: Fixed-Point `long` Pricing Over `BigDecimal`

**Status:** Accepted
**Date:** 2026-09-18
**Deciders:** Rajenthar
**Related:** ADR-009 (allocation-free), ADR-011 (cache-line layout), ADR-015 (determinism)

---

## Context

Money needs exact arithmetic. Binary floating point cannot represent 0.1 exactly, so `double` is disqualified for prices and quantities — this is not a performance question but a correctness one.

The conventional Java answer is `BigDecimal`: exact, arbitrary precision, explicit rounding. It is also, for this system, unusable on the hot path:

- **Every operation allocates.** `add`, `multiply`, `compareTo` on non-trivial values create new objects. At target throughput that is hundreds of megabytes per second of garbage (see ADR-009).
- **It is a boxed object.** Each `BigDecimal` is a pointer to heap memory holding an `int[]` or `long` plus a scale — a cache miss per price comparison, in an inner loop that does nothing but compare prices.
- **It is large.** A `BigDecimal` field makes the `Order` object far exceed the 64-byte cache line target of ADR-011.

Meanwhile the actual precision requirement is modest: instrument prices need four decimal places, not arbitrary precision.

## Decision

**All prices and quantities are `long`, in fixed-point with scale 10⁴.**

```
$150.25  →  1_502_500
$0.0001  →  1
```

- Conversion happens exactly once, at the system edge in `apex-gateway` / `apex-ingestion`. `apex-core` sees only `long`.
- Comparison is `<`, arithmetic is `+`/`-`. No allocation, no method dispatch, no cache miss.
- Scale is a single documented constant, `PRICE_SCALE = 10_000`.
- Overflow is bounded by validation at ingestion: maximum price and quantity are enforced there, keeping products well inside `long` range.

## Options Considered

### Option A: `double`

**Pros:** Primitive, fast, cache-friendly.
**Cons:** Cannot represent decimal fractions exactly. Accumulated rounding produces wrong balances. Also breaks ADR-015 determinism, since FP results can vary with JIT optimization. Disqualified on correctness.

### Option B: `BigDecimal`

| Dimension | Assessment |
|---|---|
| Complexity | Low — the conventional choice |
| Correctness | Excellent |
| Latency | **Poor** — allocation and indirection per operation |
| Cache behaviour | Poor — pointer chase per comparison |

**Pros:** Exact, arbitrary precision, explicit rounding control, familiar to every Java developer, the right default for most business software.
**Cons:** Allocates per operation; hundreds of MB/s of garbage at target throughput. Pointer indirection on every price comparison in the innermost loop. Inflates `Order` past one cache line.

### Option C: Fixed-point `long`, scale 10⁴ (chosen)

| Dimension | Assessment |
|---|---|
| Complexity | Medium — scale discipline required at boundaries |
| Correctness | Exact within the declared scale |
| Latency | **Excellent** — primitive comparison |
| Cache behaviour | Excellent — 8 bytes inline, no indirection |

**Pros:** Zero allocation. Price comparison compiles to a single machine instruction. Fits the 64-byte `Order` budget. Deterministic. This is what real exchange systems use.
**Cons:** Fixed precision must be decided upfront and is awkward to change later. Scale errors are a real hazard — a price off by 10⁴ is catastrophic and silent. Overflow must be bounded by validation. Display formatting needs explicit conversion.

### Option D: Fixed-point `long` with a wrapper type (`Price` value class)

**Pros:** Type safety against scale mistakes — cannot accidentally pass a quantity where a price belongs.
**Cons:** In Java 21, a wrapper class allocates unless the JIT reliably scalarises it, which cannot be assumed on the hot path. Worth revisiting when Valhalla value types ship; until then the type-safety gain is not worth the allocation risk.

## Trade-off Analysis

The trade is **arbitrary precision against performance**, and the key observation is that arbitrary precision is not actually required. Four decimal places covers every instrument this system will handle. `BigDecimal` pays for generality that is unused.

The genuine risk introduced is scale confusion — an unscaled value entering `apex-core`, or a scaled value displayed raw. This is a silent, high-consequence class of bug. It is mitigated structurally: conversion happens at exactly two places (gateway in, gateway out), `apex-core` has no notion of unscaled values at all, and property-based tests assert round-trip conversion.

Option D is the right answer once Java has value types. Noted for revisit rather than adopted now.

## Consequences

**Easier**
- Price comparison is one instruction; no allocation anywhere in arithmetic.
- `Order` fits one cache line (ADR-011).
- Deterministic arithmetic, satisfying ADR-015.

**Harder**
- Scale discipline is a permanent hazard requiring test coverage at every boundary.
- Changing precision later means migrating persisted data.
- Debugging shows `1502500` rather than `$150.25`; toString helpers needed for readability.
- Overflow bounds must be enforced by ingestion validation, not assumed.

**To revisit**
- A `Price` value type once Project Valhalla ships — it would give type safety at zero allocation cost.
- Whether scale 10⁴ suffices, if instruments with finer tick sizes (FX, crypto) are added.

## Action Items

1. [ ] `PRICE_SCALE = 10_000` as a single documented constant in `apex-common`
2. [ ] Conversion helpers at the gateway boundary only: `toScaled(BigDecimal)` / `toDisplay(long)`
3. [ ] Property-based test: round-trip conversion is lossless across the valid range
4. [ ] Ingestion validation enforcing max price and quantity, keeping products inside `long` range
5. [ ] Overflow test at boundary values
6. [ ] `toString` helper for readable debugging output
