# ADR-001: Multi-Module Maven Layout

**Status:** Accepted
**Date:** 2026-09-18
**Deciders:** Rajenthar
**Related:** ADR-002 (zero-I/O core)

---

## Context

Apex needs a repository structure before any code is written. The defining constraint comes from ADR-002: the matching core must have zero I/O and zero framework dependencies, and that claim has to be *enforceable* rather than aspirational.

A single-module project cannot enforce it. Nothing stops a developer — including a future me, or an AI assistant generating code — from importing `org.springframework` into the matching engine at 2am. The compiler would accept it, the tests would pass, and the project's central architectural claim would quietly become false.

The other forces: JMH benchmarks need a build step (annotation processing, shaded uber-jar) that should not apply to production modules, and the deployment topology already calls for four separate containers with different dependency sets.

## Decision

**Six Maven modules under one root POM**, with dependency direction enforced by the build:

```
apex/
├── apex-common/       # DTOs, events, enums — depends on nothing
├── apex-core/          # Matching engine — depends ONLY on apex-common + fastutil
├── apex-ingestion/     # Kafka consumers, validation, Disruptor wiring
├── apex-persistence/   # Async batched Postgres writer
├── apex-gateway/       # Spring Boot REST + WebSocket
└── apex-benchmarks/    # JMH — depends only on apex-core
```

The root POM manages versions via `dependencyManagement` (Java 21, Disruptor 4.0, fastutil, JMH 1.37, Spring Boot 3.3.4). Spring Boot is declared **only** in `apex-gateway`.

## Options Considered

### Option A: Single module

| Dimension | Assessment |
|---|---|
| Complexity | Low |
| Enforcement of ADR-002 | **None** |
| Build time | Fast |

**Pros:** Simplest possible setup; no inter-module version juggling; fastest to start.
**Cons:** Cannot enforce the core's isolation. Every dependency is on every class's classpath. The project's headline architectural property becomes a convention rather than a constraint.

### Option B: Multi-module Maven (chosen)

| Dimension | Assessment |
|---|---|
| Complexity | Medium |
| Enforcement of ADR-002 | **Compile-time** |
| Build time | Slightly slower |

**Pros:** Spring literally cannot be imported into `apex-core` — it is not on that module's classpath. Benchmarks get their own build configuration. Each container ships only what it needs. Module boundaries document the architecture without a diagram.
**Cons:** More POM ceremony; a change spanning modules needs a full reactor build.

### Option C: Gradle multi-project

**Pros:** Faster incremental builds; nicer DSL.
**Cons:** No advantage that matters here, and Maven is the more common convention in this domain. Familiarity to the reader is a feature.

## Trade-off Analysis

The decision turns on a single question: should ADR-002 be a rule people follow, or a rule the build enforces? Every other consideration is minor by comparison. Multi-module costs some POM boilerplate and buys compile-time enforcement of the project's central claim — an easy trade.

The secondary benefit is communicative. A reviewer opening the repo sees six directories and immediately understands the layering, before reading a line of code or documentation.

## Consequences

**Easier**
- ADR-002 is enforced by the compiler, not by discipline.
- `apex-benchmarks` gets JMH's shade plugin without polluting production builds.
- Container images ship minimal dependency sets.

**Harder**
- Cross-module refactors require a reactor build.
- Version management lives in the root POM and must stay disciplined.

**To revisit**
- Whether `apex-persistence` and `apex-ingestion` should merge if they stay thin.

## Action Items

1. [ ] Root POM with `dependencyManagement` for Java 21, Disruptor, fastutil, JMH, Spring Boot
2. [ ] Six module POMs with explicit, minimal dependency declarations
3. [ ] Verify `apex-core`'s dependency tree contains only `apex-common` and fastutil (`mvn dependency:tree`)
4. [ ] Add ArchUnit or `maven-enforcer-plugin` banned-dependencies rule as a second line of defence
