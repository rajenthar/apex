# ADR-002: Zero-I/O, Zero-Framework Matching Core

**Status:** Accepted
**Date:** 2026-09-18
**Deciders:** Rajenthar
**Related:** ADR-001 (module layout), ADR-008 (outbound event bus), ADR-009 (allocation-free match), ADR-015 (deterministic replay)

---

## Context

This is the load-bearing decision of the entire project. Everything else either supports it or follows from it.

Apex claims nanosecond-scale matching. That claim is only meaningful — and only *measurable* — if the matching logic is isolated from everything that takes milliseconds. A matching engine that calls a database, publishes to Kafka, or resolves a Spring bean mid-match cannot be benchmarked honestly, because the benchmark measures the I/O, not the algorithm.

The deeper problem is contamination by degrees. Nobody adds a blocking JDBC call to a matching loop on purpose. It arrives as a logging statement that resolves a config property, an `@Autowired` collaborator that turns out to hit a cache, a metrics counter that allocates. Each is individually defensible; collectively they turn a 250ns function into a 2ms one, and no single commit looks responsible.

ADR-015 later adds a second reason: the core must be a **deterministic state machine** for replay-based recovery to work. I/O and frameworks are the primary sources of non-determinism.

## Decision

**`apex-core` depends on `apex-common` and fastutil. Nothing else. Ever.**

Specifically forbidden in `apex-core`:
- Any Spring, Jakarta, or DI framework annotation or type
- Any Kafka, JDBC, HTTP, file, or socket API
- Any logging framework (SLF4J, Logback, Log4j)
- Any metrics library (Micrometer, Dropwizard)
- `System.currentTimeMillis()`, `Random`, `UUID.randomUUID()` (per ADR-015)

`apex-core` exposes pure functions over primitives. Matching results leave through a callback (ADR-009), not a return-allocated collection, and not a side effect.

Enforced three ways: module classpath (ADR-001), an ArchUnit test in CI, and a `maven-enforcer-plugin` banned-dependencies rule.

## Options Considered

### Option A: Spring-managed matching service

| Dimension | Assessment |
|---|---|
| Complexity | Low — conventional Java service |
| Latency | Poor — proxies, reflection, unpredictable allocation |
| Benchmarkability | Poor — cannot isolate from context |
| Team familiarity | High |

**Pros:** Conventional, fast to write, DI for testing, everything wired for free.
**Cons:** CGLIB/JDK proxies on method calls, reflection at startup, unpredictable allocation from framework internals, and no way to construct the engine in a JMH benchmark without a Spring context. Kills both the latency claim and the ability to measure it.

### Option B: Plain Java core with I/O allowed

| Dimension | Assessment |
|---|---|
| Complexity | Low |
| Latency | Unpredictable |
| Benchmarkability | Poor |

**Pros:** No framework overhead; still simple to write.
**Cons:** A single blocking call anywhere in the match path introduces millisecond stalls and multi-second GC-adjacent tail latency. Benchmarks measure I/O, not matching. Non-deterministic, so ADR-015 replay recovery cannot work.

### Option C: Zero-I/O, zero-framework core (chosen)

| Dimension | Assessment |
|---|---|
| Complexity | Medium — requires discipline and explicit wiring |
| Latency | Predictable, measurable, nanosecond-scale |
| Benchmarkability | Excellent — JMH constructs it in one line |
| Team familiarity | Lower — unconventional for typical Spring shops |

**Pros:** Latency is a property of the algorithm alone. Unit tests need no context and run in milliseconds. Fuzz testing and property-based testing become trivial. Deterministic, enabling ADR-015. JMH numbers are honest.
**Cons:** Wiring is manual and explicit. Some duplication at module boundaries. Unfamiliar to developers used to annotating everything.

## Trade-off Analysis

The trade is **convenience against measurability**. Spring makes writing the engine faster; it makes proving anything about the engine impossible. For a system whose entire purpose is to demonstrate and measure low-latency behaviour, measurability is not negotiable.

The manual-wiring cost is smaller than it looks. `apex-core` has no configuration, no lifecycle, no collaborators to inject — it is a data structure and a function over it. DI solves a problem this module does not have.

The discipline cost is real and permanent. Mitigate with automated enforcement rather than review vigilance: ArchUnit fails the build, a code reviewer might not.

## Consequences

**Easier**
- Honest benchmarking: JMH constructs an `OrderBook` directly, no context, no mocks.
- Unit and property-based tests are fast and deterministic.
- ADR-015's replay recovery becomes possible.
- Reasoning about latency: if it is slow, it is the algorithm.

**Harder**
- Explicit wiring at every boundary.
- Metrics and logging must be gathered *outside* the core, from the events it emits.
- Onboarding: the pattern is unfamiliar to most Spring-native developers.

**To revisit**
- Nothing. This decision is foundational; changing it invalidates ADRs 009, 010, 011 and 015.

## Action Items

1. [ ] Confirm `apex-core` POM declares only `apex-common` and fastutil
2. [ ] ArchUnit test: no `org.springframework`, `java.sql`, `java.net`, `java.io`, `org.slf4j`, `io.micrometer` in `apex-core`
3. [ ] ArchUnit test: no `System.currentTimeMillis`, `Random`, `UUID.randomUUID` in `apex-core`
4. [ ] `maven-enforcer-plugin` banned-dependencies rule as a build-level backstop
5. [ ] Document the rule prominently in `apex-core/README.md` — it is the first thing a reader should see
