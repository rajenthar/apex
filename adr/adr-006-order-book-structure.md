# ADR-006: Order Book Data Structure — fastutil Over `TreeMap`

**Status:** Accepted
**Date:** 2026-09-18
**Deciders:** Rajenthar
**Supersedes:** the original "TreeMap + ArrayDeque" sketch in the design doc
**Related:** ADR-005 (fixed-point `long`), ADR-009 (allocation-free), ADR-011 (cache layout)

---

## Context

The order book is the core data structure. It must support, per side:

- **Best price lookup** — every incoming order starts here. Hottest operation by far.
- **Ordered traversal** from best price outward, for orders that sweep multiple levels.
- **Insert / remove a price level** — when a level is created or fully consumed.
- **FIFO within a price level** — time priority (ADR-003 makes this well-defined).

The original design specified `TreeMap<Long, Queue<Order>>` with an `ArrayDeque` per level. That was wrong in a way that only became visible during the cache and allocation analysis:

**`TreeMap<Long, …>` boxes every key.** `Long` values outside the −128…127 cache are heap-allocated. Every price lookup either allocates a `Long` or, at minimum, chases a pointer to compare two boxed values. In the innermost loop of a system claiming zero allocation (ADR-009) and cache-line discipline (ADR-011), this directly contradicts both. The `ArrayDeque` choice was correct; the map was not.

## Decision

**`Long2ObjectRBTreeMap<PriceLevel>` (fastutil) per side; `ArrayDeque<Order>` per price level.**

- Bids use a descending comparator, asks ascending, so `firstKey()` is always the best price.
- Keys are primitive `long` — no boxing, no allocation, comparison inline.
- `PriceLevel` holds an `ArrayDeque<Order>` for strict FIFO time priority.
- `ArrayDeque` over `LinkedList`: contiguous backing array, no node allocation per element, far better locality.

**Start here, optimise only against benchmarks.** Faster structures exist (below); adopting them before measuring would be speculative.

## Options Considered

### Option A: `TreeMap<Long, Queue<Order>>` (original, rejected)

| Dimension | Assessment |
|---|---|
| Complexity | Low — standard library |
| Allocation | **Boxes every key** |
| Cache behaviour | Poor — pointer chase per comparison |

**Pros:** No dependency, familiar, correct.
**Cons:** Boxing violates ADR-009 outright. Each comparison dereferences two `Long` objects scattered on the heap. The single worst default in the original design.

### Option B: `Long2ObjectRBTreeMap` — fastutil (chosen)

| Dimension | Assessment |
|---|---|
| Complexity | Low — same API shape, one dependency |
| Allocation | **Zero on lookup** |
| Cache behaviour | Good — primitive keys inline in nodes |
| Performance | O(log n), no boxing |

**Pros:** Drop-in mental model for anyone who knows `TreeMap`. Primitive keys eliminate boxing entirely. Sorted, so ordered traversal from best price is natural. Mature, widely used library. One small dependency into an otherwise dependency-free module — acceptable, and declared explicitly in ADR-002.
**Cons:** Still a tree: O(log n) lookup with pointer chasing between nodes. Adds a dependency to `apex-core`, which is otherwise pure.

### Option C: Flat array indexed by price offset

| Dimension | Assessment |
|---|---|
| Complexity | Medium — requires bounded tick range |
| Allocation | Zero |
| Cache behaviour | **Excellent** — contiguous, prefetchable |
| Performance | **O(1)** lookup |

**Pros:** The fastest option by a clear margin. Contiguous memory, hardware prefetch works, no pointer chasing at all. Direct indexing rather than tree descent.
**Cons:** Requires a bounded, known tick range per symbol. Memory is proportional to price *range*, not to occupied levels — a wide range wastes memory, and a gapped book wastes most of it. Needs dynamic rebasing when price moves outside the window, which is real complexity and a source of bugs. Poor fit for instruments with wide or unpredictable ranges.

### Option D: Skip list

**Pros:** Lock-free variants exist.
**Cons:** ADR-003 removed the need for concurrency, so the lock-free property is worthless here. Worse constant factors and worse locality than a tree for this access pattern. No advantage.

## Trade-off Analysis

Option C is genuinely faster and is what the most aggressive production engines use. It is not the right *starting* choice, for two reasons: it requires committing to a bounded tick range before any real order flow has been observed, and the rebasing logic is a meaningful source of correctness risk in the component that must never be wrong.

Option B removes the actual defect — boxing — at near-zero complexity cost, and keeps the structure simple enough to verify exhaustively. If benchmarks later show tree descent is the bottleneck, migrating to Option C is a contained change behind the `OrderBook` interface.

This is the general principle worth stating: **fix the known defect now, optimise the suspected bottleneck after measuring.** Boxing was a known defect. Tree descent is a suspicion until a flame graph says otherwise.

The dependency cost is worth naming. ADR-002 declares `apex-core` framework-free; fastutil is a collections library with no I/O, no reflection and no lifecycle, so it does not compromise what that ADR protects.

## Consequences

**Easier**
- Zero allocation on price lookup, supporting the 0 B/op target.
- Same mental model as `TreeMap` — readable to any Java developer.
- Ordered traversal from best price is natural and correct by construction.

**Harder**
- One dependency in the otherwise pure core (justified above).
- Still O(log n) with pointer chasing; a future bottleneck if level counts grow large.
- `PriceLevel` objects are themselves allocations at level creation — acceptable, since level creation is far rarer than lookup, but worth watching in allocation profiling.

**To revisit**
- Migrate to a flat price-indexed array if `-prof perfnorm` shows tree descent dominating cache misses.
- `PriceLevel` pooling if level churn shows up in allocation profiles.

## Action Items

1. [ ] Add fastutil to `apex-core`; confirm it is the only non-`apex-common` dependency
2. [ ] `Long2ObjectRBTreeMap<PriceLevel>` per side, bids descending / asks ascending
3. [ ] `PriceLevel` wrapping `ArrayDeque<Order>`; ArchUnit ban on `LinkedList` in `apex-core`
4. [ ] Benchmark best-price lookup and multi-level sweep separately in `apex-benchmarks`
5. [ ] Run `-prof perfnorm` to record baseline cache misses per match
6. [ ] Keep the `OrderBook` public interface narrow enough that Option C remains a contained swap
