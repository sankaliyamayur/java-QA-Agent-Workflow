# SCALE-2 (Kafka binding) — Technical Design: Distributed `EventBus` over Kafka

**Item:** SCALE-2 (Introduce an event bus for inter-service coordination) — **In Progress**. 🟠 P1 · Phase 2 · v1.5.
**Continues:** ADR-060 (the `core` `EventBus` seam + `InProcessEventBus`) and ADR-061 (the Spring bridge).
**Status:** Draft — awaiting decision + go-ahead (no code until approved).
**Date:** 2026-07-31.

> **Why this is the next highest-priority pending work.** Of the 7 In-Progress items, only SCALE-1 and SCALE-2 are 🟠 P1. SCALE-2's seam already exists (ADR-060); the remaining piece is the **distributed binding** — a `KafkaEventBus` that drops into the same `EventBus` interface. It is **not scaffolding without a consumer**: publishers and subscribers already use the in-process bus, so this simply makes the *same* coordination work **across instances**. Its Kafka container is already provisioned (Phase 1 / ADR-053).

---

## 0. Grounding + scope

### 0.1 What exists
- `core.event.EventBus` (`publish(BaseEvent)` + `subscribe(Class<T>, Consumer<T>)`); `InProcessEventBus` (plain `@Component`, synchronous in-JVM dispatch); `SpringEventBridge` (seam → Spring `@EventListener`). `BaseEvent` is `Serializable` (JSON-friendly).
- Kafka **provisioned** (compose: `apache/kafka:3.8.0` KRaft, `localhost:9092`; `.env` `KAFKA_BOOTSTRAP`). **No `spring-kafka` Java dependency yet** (ADR-053 deferred it).

### 0.2 What this slice delivers
A `KafkaEventBus implements EventBus` so a `BaseEvent` published on any instance is **delivered to every instance's local subscribers** (cross-JVM coordination) — selected by config, defaulting to the in-process bus so nothing changes unless Kafka is turned on.

### 0.3 Honest validation boundary
The **wiring + serialization + selection** are unit-testable in this sandbox; **true cross-instance delivery is user-run** (needs the Kafka container up + two app instances). No live Kafka here.

### 0.4 / Decision for approval — where the Kafka binding lives (ADR decision)

| Option | Placement | Trade-off |
|---|---|---|
| **A — `core.event`, `spring-kafka` as an *optional* dependency + conditional bean (recommended)** | `KafkaEventBus` sits beside `InProcessEventBus`; `spring-kafka` is `<optional>true</optional>` in `core`'s pom; the bean is `@ConditionalOnClass(KafkaTemplate)` + `@ConditionalOnProperty(aiqaos.events.transport=kafka)`. An app opts in by adding `spring-kafka` to *its* pom + setting the property. | Cohesive with the seam; **zero Kafka weight** on modules that don't opt in (optional deps aren't transitive); the standard Spring-Boot optional-integration pattern. Slightly subtle (optional dep + conditionals). |
| **B — a new `ai-qa-os-messaging` module** | `KafkaEventBus` + Kafka config in a dedicated module that runnable apps depend on. | Clear separation, but a whole new module + every app that wants distributed events takes a hard `spring-kafka` dependency; heavier for a one-class binding. |

**Recommendation: A.** One class, opt-in, no forced Kafka weight — the seam stays the abstraction and Kafka is a swappable, conditionally-loaded binding.

> ✅ **Decision (confirmed 2026-07-31): Option A** — `KafkaEventBus` in `core.event`; `spring-kafka` optional in `core`; conditional-on-class + conditional-on-property; default stays in-process. To be recorded as **ADR-064** (number verified at implement).

---

## 1. Technical Design (Option A)

### 1.1 Selection (make the transport pluggable)
- Add `@ConditionalOnProperty(name="aiqaos.events.transport", havingValue="in-process", matchIfMissing=true)` to **`InProcessEventBus`** (today it is unconditional — this prevents two `EventBus` beans when Kafka is on).
- `KafkaEventBus`: `@ConditionalOnProperty(name="aiqaos.events.transport", havingValue="kafka")` + `@ConditionalOnClass(KafkaTemplate.class)`.
- Default (`matchIfMissing`) = in-process, so existing behaviour is unchanged.

### 1.2 `KafkaEventBus` (implements `EventBus`)
- **Local registry** (same shape as `InProcessEventBus`) for `subscribe`/dispatch-on-receive.
- **publish(BaseEvent):** wrap in an envelope `{ type: <FQN>, payload: <json> }`, serialise with Jackson, `KafkaTemplate.send(TOPIC, key, bytes)`. Key = event type (for partition affinity/ordering per type).
- **Consume:** a `@KafkaListener(topics = TOPIC)` deserialises the envelope, loads the class by FQN, maps the payload to that `BaseEvent` subtype, and **dispatches to local subscribers** via the same type-hierarchy walk as `InProcessEventBus` (extract that dispatch into a small shared helper so both buses reuse it).
- **A throwing handler is isolated** (logged), as in-process.

### 1.3 Topic + consumer-group strategy
- **Single topic** `aiqaos.events` (event type in the envelope drives routing) — simplest; per-type topics are a later optimisation.
- **Per-instance consumer group** (`aiqaos-events-<instanceId>`) so **every** instance receives every event (broadcast coordination), not a competing consumer group where only one instance would. `instanceId` from `${HOSTNAME}`/a UUID.

### 1.4 Serialization faithfulness
The envelope carries the concrete class FQN so the consumer reconstructs the exact `BaseEvent` subtype (Agent/Workflow/… ). Events must be Jackson-round-trippable (they are POJOs over `BaseMetadata`); a contract test asserts round-trip for each typed event.

## 2. Module / config changes
- `core/pom.xml`: `spring-kafka` `<optional>true</optional>`.
- `core.event`: `KafkaEventBus`, `EventEnvelope`, extract `EventDispatch` helper; add the conditional to `InProcessEventBus`.
- Gateway (the distributed entry point): add `spring-kafka` (compile) + `application-compose.yml` → `aiqaos.events.transport=kafka`, `spring.kafka.bootstrap-servers=${KAFKA_BOOTSTRAP:localhost:9092}`, consumer group. Other apps stay in-process unless opted in.

## 3. Required classes
| Class | Role |
|---|---|
| `EventEnvelope` | `{ type (FQN), payload }` JSON wrapper |
| `EventDispatch` | shared type-hierarchy dispatch (used by both buses) |
| `KafkaEventBus` | `EventBus` over `KafkaTemplate` + `@KafkaListener` |
| (edit) `InProcessEventBus` | add `@ConditionalOnProperty` default guard |

## 4–5. DB / API
**None.** Pure messaging transport.

## 6. Implementation plan — small, verifiable tasks
1. `spring-kafka` optional dep in `core`; extract `EventDispatch`; guard `InProcessEventBus` with the default conditional. **Reactor stays green (default still in-process).** *Validatable here.*
2. `EventEnvelope` + Jackson round-trip **contract test** for every typed `BaseEvent`. *Validatable here.*
3. `KafkaEventBus` (publish → template; `@KafkaListener` → dispatch) + a **unit test** with an in-memory/mocked `KafkaTemplate` proving publish serialises and an injected record dispatches to local subscribers (Mockito-free). *Validatable here.*
4. Gateway opt-in: `spring-kafka` dep + `compose`-profile config. Compiles + boots on `compose` (default profile untouched). *Compile-validatable here.*
5. Full `mvn clean verify` green (default transport unchanged; Kafka path conditionally absent unless enabled). *Validatable here.*
6. Docs — tracker SCALE-2 (Kafka binding done, distributed), **ADR-064**, Implementation Outcome.
7. **Live E2E (user-run):** `docker compose up -d kafka`; run two gateway instances on `compose` with `transport=kafka`; publish on one → both instances' subscribers fire; confirm on the topic via `kafka-ui`.

**Definition of Done:** a config-selected `KafkaEventBus` delivers `BaseEvent`s across instances; default stays in-process; serialization + dispatch unit-proven; reactor green. **SCALE-2 → Completed** once the live cross-instance E2E passes (user-run). Remaining publisher migration (FI-SCALE2-A) is independent.

**Honest boundary:** everything except live cross-instance delivery is provable here; the two-instance Kafka E2E is yours (the container is provisioned).

---

## Implementation Outcome

**Implemented 2026-07-31 (Option A). Recorded as ADR-064. SCALE-2 code-complete; live E2E user-run.**

**Delivered:** `spring-kafka` optional in `core`; extracted `EventDispatch` (shared registry/dispatch); `InProcessEventBus` refactored + guarded `@ConditionalOnProperty(...=in-process, matchIfMissing=true)`; `EventEnvelope`; `KafkaEventBus` (`@ConditionalOnClass(KafkaTemplate)` + `@ConditionalOnProperty(...=kafka)`) — `serialize`→envelope→topic, `@KafkaListener`→`receive`→reconstruct FQN→dispatch, per-instance consumer group. Gateway: `spring-kafka` dep + `compose`-profile `aiqaos.events.transport` (`AIQAOS_EVENTS_TRANSPORT`, default `in-process`) + `spring.kafka.bootstrap-servers`.

**Verified here:** `KafkaEventBusTest` **3/3** (serialize→receive round-trips + reconstructs the exact `WorkflowEvent`; `BaseEvent` supertype dispatch; malformed/non-`BaseEvent` messages ignored); `InProcessEventBusTest` 3/3 + `SpringEventBridgeTest` 1/1 unaffected; **full reactor `mvn clean test` BUILD SUCCESS, 22 modules, 0 failures** (default transport unchanged — every `@SpringBootTest` boots `InProcessEventBus`).

**One fix during impl:** the test's proxy `ProducerFactory` initially returned `null` for `transactionCapable()` (a `boolean`) → NPE in `KafkaTemplate`'s constructor; fixed to return `false` for boolean methods.

**Honest boundary:** wiring, serialization, dispatch, and transport selection are proven here; **true cross-instance delivery is user-run** — `docker compose up -d kafka`, run two gateway instances with `AIQAOS_EVENTS_TRANSPORT=kafka`, publish on one → both instances' subscribers fire.

**Deferred:** live 2-instance E2E (user-run) closes SCALE-2; per-type topics, delivery/ordering guarantees, and FI-SCALE2-A (remaining publisher migration) are independent follow-ups.
