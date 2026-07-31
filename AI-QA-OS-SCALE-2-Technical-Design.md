# SCALE-2 — Technical Design: Event Bus for Inter-Service Coordination

**Roadmap item:** SCALE-2 (Planned → In Progress). P1 · Effort L · Phase 2 · v1.5. Depends on SCALE-1.
**Status:** Draft — awaiting decision + go-ahead (no code until approved).
**Date:** 2026-07-30.

> **Scope discipline.** SCALE-2 introduces the *coordination seam* — a `core` `EventBus` contract over the existing `BaseEvent` hierarchy + an in-process default — mirroring SCALE-1's `ExecutionJobQueue`/`InProcessExecutionJobQueue`. The **distributed (Kafka) binding is deferred** (the container is provisioned, ADR-053, but no cross-service consumer exists yet). Consolidating today's fragmented publishers onto the seam is a follow-up, not a big-bang rip-out.

---

## 0. Grounding + scope

### 0.1 What exists (verified 2026-07-30)
- **Event contracts already in `core.event`:** `BaseEvent` (abstract, carries `BaseMetadata`) + typed events `Agent`/`Execution`/`Integration`/`Memory`/`Prompt`/`Reporting`/`Security`/`System`/`Workflow`. So SCALE-2's *"event contracts in core"* is **already satisfied** — no new contracts needed.
- **Fragmented publishing, no unifying seam:** `PlatformEventBus` (in `ai-qa-os-integration`) wraps Spring's `ApplicationEventPublisher` but keys on Spring's `ApplicationEvent`, not `core.BaseEvent`; plus per-module `GatewayEventPublisher`, `ObservabilityEventPublisher`, `WorkflowEventPublisher`, `IntegrationEventPublisher`. There is **no `core`-level `EventBus` abstraction** that publishers/consumers across modules can share without depending on a high-level module.
- **Pattern to mirror:** SCALE-1's `ExecutionJobQueue` (seam) + `InProcessExecutionJobQueue` (opt-in default) + deferred distributed target.

### 0.2 What SCALE-2 (this slice) delivers
A **`core.event.EventBus`** coordination contract over `BaseEvent` + a synchronous **`InProcessEventBus`** default (a thread-safe type→handlers registry), so any module can `publish`/`subscribe` depending only on `core`. The distributed (Kafka) `EventBus` is a thin deferred binding.

### 0.3 Deferred
- The **Kafka `EventBus` binding** (real cross-service coordination — container ready per ADR-053, no consumer yet).
- **Migrating** `PlatformEventBus` + the per-module publishers onto the seam (FI-SCALE2-A) — additive follow-up; today's publishers keep working.
- **Async / ordered / retried** dispatch (this slice is synchronous, deterministic).

### 0.4 / Decision for approval — in-process dispatch mechanism (ADR decision)

| Option | Mechanism | Trade-off |
|---|---|---|
| **A — Own `EventBus` seam + registry (recommended)** | `core.event.EventBus` interface (`publish(BaseEvent)` + `subscribe(Class<T>, Consumer<T>)`) + `InProcessEventBus` (a `ConcurrentHashMap<Class, handlers>`, synchronous type-assignable dispatch). | Mirrors SCALE-1 exactly; the contract is Spring-free so publishers couple only to `core`; the Kafka impl is a clean swap of the same interface. Slightly more code than reusing Spring events. |
| **B — Build on Spring `ApplicationEventPublisher`** | Lift `PlatformEventBus` to `core`; publish via Spring, consume via `@EventListener`. | Least dispatch code, but couples the coordination contract to Spring's in-JVM event model, and a distributed impl is a separate bridge anyway (Spring events don't cross a process). Harder to reason about "in-process now, distributed later" as one seam. |

**Recommendation: A.** SCALE-2 is explicitly about a *swappable* coordination bus (in-process → distributed). An own interface over `BaseEvent` is the seam that makes the Kafka binding a drop-in — exactly the SCALE-1 shape, and consistent with ADR-053.

> ✅ **Decision (confirmed 2026-07-30): Option A** — `core.event.EventBus` seam + synchronous `InProcessEventBus`; distributed (Kafka) binding deferred. To be recorded as **ADR-060** (number verified at implement).

---

## 1. Technical Design (Option A)

### 1.1 The seam (`core.event`)
```java
public interface EventBus {
    void publish(BaseEvent event);
    <T extends BaseEvent> void subscribe(Class<T> type, java.util.function.Consumer<T> handler);
}
```
- `publish` dispatches to every handler registered for the event's type **or any supertype** (so a `Consumer<BaseEvent>` sees all events; a `Consumer<WorkflowEvent>` sees only workflow events).
- Lives in `core.event` beside `BaseEvent`, so every module already depending on `core` can publish/subscribe with no new cross-module edge.

### 1.2 In-process default (`core.event.InProcessEventBus`)
- `@Component` (plain — sole reference impl; a real bus overrides by being the injected `EventBus` where present).
- `ConcurrentHashMap<Class<?>, List<Consumer>>` (CopyOnWriteArrayList values); `publish` walks the event's class hierarchy and invokes matching handlers **synchronously** on the caller thread; a throwing handler is isolated (logged, others still run).
- Deterministic and unit-testable with no infrastructure.

### 1.3 Distributed binding (deferred)
A future `KafkaEventBus implements EventBus` serialises `BaseEvent` to a topic and drives subscribers from a consumer group — the container is provisioned (ADR-053); built when a real cross-service consumer exists (no scaffolding without a consumer).

## 2. Folder / class changes
- `core/event/EventBus.java` `[N]`, `core/event/InProcessEventBus.java` `[N]`.
- No migration, no API change.

## 3. Required classes
| Class | Role |
|---|---|
| `EventBus` | coordination contract over `BaseEvent` |
| `InProcessEventBus` | synchronous in-process default (registry) |

## 4–5. Database / API changes
**None.**

## 6. Implementation plan
1. `EventBus` + `InProcessEventBus`. *Compiles here.*
2. **Unit test** — subscribe by exact type + by `BaseEvent` supertype; publish dispatches to matching handlers only; multiple handlers; a throwing handler doesn't stop the rest. *Validatable here.*
3. Full `mvn clean test` green (additive `@Component` in core — watch context wiring). *Validatable here.*
4. Docs — tracker SCALE-2 → In Progress, **ADR-060**, this doc's Implementation Outcome.

**Definition of Done (this slice):** a `core` `EventBus` seam + in-process default lets any module publish/subscribe `BaseEvent`s, unit-proven, reactor green. **SCALE-2 → In Progress** (the Kafka binding + publisher consolidation are follow-ups). Cleanly validatable here — no infra needed for the seam.

---

## Implementation Outcome

**Implemented 2026-07-30 (Option A — own `EventBus` seam + `InProcessEventBus`). Recorded as ADR-060. SCALE-2 → In Progress.**

**Delivered:** `core.event.EventBus` (interface) + `core.event.InProcessEventBus` (`@Component`, synchronous type-hierarchy dispatch over a `ConcurrentHashMap` registry, throwing-handler isolation). No migration, no API change.

**Verified here:** `InProcessEventBusTest` **3/3** (dispatch by type + supertype only; all handlers run + a throwing one is isolated; null ignored); **full reactor `mvn clean test` BUILD SUCCESS, 22 modules, 0 failures** (7:23 min).

**Deviations from design:** none.

**FI-SCALE2-A (bridge + entry point) — done 2026-07-30 (ADR-061):** `PlatformEventBus.publish(BaseEvent)` routes through the seam; `SpringEventBridge` forwards seam events to Spring so existing `@EventListener` consumers keep firing; `WorkflowEventPublisher` migrated as the representative proof. `SpringEventBridgeTest` 1/1, reactor green. The remaining publishers migrate opportunistically.

**Deferred (SCALE-2 stays In Progress):** the distributed (Kafka) `EventBus` binding (container ready, ADR-053; no cross-service consumer yet); migrating the remaining per-module publishers (`Gateway`/`Observability`/`Integration`) onto the seam; retiring the Spring `ApplicationEvent` path; async/ordered/retried dispatch.
