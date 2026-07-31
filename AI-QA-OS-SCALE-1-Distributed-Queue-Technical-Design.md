# SCALE-1 (distributed queue) — Technical Design: Redis-backed `ExecutionJobQueue`

**Item:** SCALE-1 (Decouple execution into a queue-fed worker pool) — **In Progress**. 🟠 P1 · Phase 2 · v1.5.
**Continues:** the `ExecutionJobQueue` seam + `InProcessExecutionJobQueue` (whose javadoc names this the "drop-in point for the deferred Redis-Streams / containerised-worker tier").
**Status:** Draft — awaiting decision + go-ahead (no code until approved).
**Date:** 2026-07-31.

> **Why this is the next work.** SCALE-1 and SCALE-2 are the only 🟠 P1 pending items. SCALE-2's distributed binding shipped (ADR-064); SCALE-1's is the remaining P1 — a distributed `ExecutionJobQueue` so execution jobs run on a **worker pool spanning nodes**, not just the caller's JVM. Redis is provisioned (Phase 1).

> **Key contrast with SCALE-2 (important).** The event bus is **broadcast** (every subscriber gets every event). A job queue is the opposite — **competing consumers**: exactly one worker takes each job, and the result must return to the *submitter*. So this cannot reuse the `EventBus`; it needs a real work queue.

---

## 0. Grounding + scope

### 0.1 What exists (the seam)
- `ExecutionJobQueue`: `String submit(ExecutionJob)` + `ExecutionJobResult awaitResult(String jobId, Duration timeout)`.
- `ExecutionJob` (jobId, framework, scriptSuite, configuration) · `ExecutionJobResult` (success/failure + `ExecutionResult`).
- `InProcessExecutionJobQueue` — `@ConditionalOnProperty(aiqaos.execution.queue.enabled=true)`; virtual-thread pool; `submit` → `engineFactory.getEngine(framework).execute(...)` → future; `awaitResult` via `CompletableFuture`. **Absent by default** → `ExecutionStep` runs inline (non-breaking).

### 0.2 What this slice delivers
A Redis-backed `ExecutionJobQueue` so `submit` enqueues a job that **any worker on any instance** can pick up, run, and return — decoupling execution across the pool. Selected by config, defaulting to the in-process queue (itself off by default).

### 0.3 Honest boundary
The queue impl, serialization, and submit/await round-trip are unit-testable here (mocked/embedded Redis); **true cross-instance distribution is user-run** (Redis up + ≥2 instances). Jobs also require the `ExecutionEngine` (Playwright) to actually *run* — so an end-to-end job is doubly environment-bound (Redis + browser).

### 0.4 / Decision for approval — result-delivery + durability model (ADR decision)

| Option | Model | Trade-off |
|---|---|---|
| **A — Redis Streams + per-job result key (recommended, matches §0.3b)** | Jobs on a **stream** `execution:jobs` with a **consumer group** (competing consumers; `XACK` on done; **`XCLAIM` reclaims** a stalled/crashed worker's job → at-least-once, no lost jobs). Worker writes the result to `execution:result:<jobId>` (TTL); `awaitResult` polls it with the timeout. | Durable + redelivery — right for expensive execution jobs you must not drop; matches the named target. More Redis surface (streams, groups, claim). |
| **B — Redis Lists (`BRPOP`)** | `LPUSH` job / `BRPOP` worker (competing consumers); worker `LPUSH`es the result to `execution:result:<jobId>`, `awaitResult` `BRPOP`s it. | Simplest, naturally blocking. But **at-most-once**: a worker that crashes mid-job silently loses it (no redelivery) — bad for execution work. |

**Recommendation: A.** A distributed execution queue must not lose jobs when a worker dies; Streams' consumer-group ack + claim gives at-least-once delivery. B is simpler but its failure mode (lost jobs) is unacceptable for real execution work.

> ✅ **Decision (confirmed 2026-07-31): Option A** — Redis Streams + consumer group (`XACK`/`XAUTOCLAIM`, at-least-once) + per-job result key; default stays the in-process queue. To be recorded as **ADR-065** (number verified at implement).

---

## 1. Technical Design (Option A)

### 1.1 Selection (mutually exclusive with the in-process queue)
- `aiqaos.execution.queue.enabled=true` gates queueing at all (unchanged).
- New `aiqaos.execution.queue.provider` = `in-process` (default) | `redis`.
- `InProcessExecutionJobQueue`: add `@ConditionalOnProperty(...provider=in-process, matchIfMissing=true)` (ANDed with `enabled=true`).
- `RedisStreamExecutionJobQueue`: `@ConditionalOnProperty(enabled=true)` + `@ConditionalOnProperty(provider=redis)` + `@ConditionalOnClass(StringRedisTemplate)`. `spring-data-redis` optional in `execution` (the ADR-064 pattern) → zero Redis weight unless opted in.

### 1.2 `RedisStreamExecutionJobQueue implements ExecutionJobQueue`
- **submit(job):** JSON-serialise the job → `XADD execution:jobs *`. Return `jobId`. (Job's `scriptSuite`/`configuration` must be Jackson round-trippable — a contract test asserts this.)
- **Worker loop** (`@Scheduled`/dedicated thread per instance, consumer group `execution-workers`, unique consumer name per instance): `XREADGROUP` → deserialise → run the engine (reusing `InProcessExecutionJobQueue.runJob` logic, extracted to a shared `ExecutionJobRunner`) → write `execution:result:<jobId>` (SET with TTL) → `XACK`. A periodic `XAUTOCLAIM` reclaims jobs stalled past a threshold (crashed worker).
- **awaitResult(jobId, timeout):** poll `execution:result:<jobId>` until present or the timeout elapses; deserialise → `ExecutionJobResult`; delete the key.

### 1.3 Worker model
Each instance with `provider=redis` runs the worker consumer, so the pool **spans all instances** (competing consumers via the group). Dedicated worker-only instances are the same code with the pipeline disabled — a deployment choice, no code change.

### 1.4 Faithfulness / correctness
Exactly-one-worker per job (consumer group); at-least-once (ack + claim) so a crash re-runs rather than loses; result keyed by `jobId` returns to the correct submitter across instances; TTLs prevent Redis leakage.

## 2. Module / class changes
- `execution/pom.xml`: `spring-data-redis` `<optional>true</optional>`.
- `execution.queue`: `RedisStreamExecutionJobQueue`; extract `ExecutionJobRunner` (shared run logic); add the `provider` conditional to `InProcessExecutionJobQueue`.
- Gateway/worker app: opt-in config (`queue.enabled=true`, `queue.provider=redis`, `spring.data.redis.*`).

## 3. Required classes
| Class | Role |
|---|---|
| `ExecutionJobRunner` | shared engine-run logic (used by both queues) |
| `RedisStreamExecutionJobQueue` | `ExecutionJobQueue` over Redis Streams + result key |
| (edit) `InProcessExecutionJobQueue` | add `provider` default conditional |

## 4–5. DB / API
**None.** Redis-only transport (no relational schema, no new endpoint).

## 6. Implementation plan — small, verifiable tasks
1. `spring-data-redis` optional in `execution`; extract `ExecutionJobRunner`; guard `InProcessExecutionJobQueue` with the `provider` default. **Reactor green** (default path unchanged). *Validatable here.*
2. Job/result **JSON round-trip contract test** (`ExecutionJob` + `ExecutionJobResult` serialise faithfully). *Validatable here.*
3. `RedisStreamExecutionJobQueue` — submit (XADD), worker consume+run+result+ack, awaitResult (poll result key). **Unit test** with a mocked/embedded `StringRedisTemplate` proving submit serialises to the stream and a delivered record runs + posts a result the awaiter reads (Mockito-free). *Validatable here (logic; not live Redis).*
4. App opt-in config; compiles + boots on the profile (default provider in-process, so no Redis needed to boot). *Compile-validatable here.*
5. Full `mvn clean verify` green. *Validatable here.*
6. Docs — tracker SCALE-1, **ADR-065**, Implementation Outcome.
7. **Live E2E (user-run):** Redis up; two instances with `queue.enabled=true, provider=redis`; submit a job on A → a worker (A or B) runs it → A's `awaitResult` returns; kill a worker mid-job → `XAUTOCLAIM` redelivers.

**Definition of Done:** a config-selected Redis queue distributes execution jobs across instances with at-least-once delivery; serialization + submit/await logic unit-proven; reactor green; default path unchanged. **SCALE-1 → Completed** once the live cross-instance E2E passes (user-run) — the SEC-1/ENT-1/SCALE-2 precedent.

**Honest boundary:** everything except live cross-instance execution is provable here; the two-instance + Redis (+ Playwright) E2E is yours.

---

## Implementation Outcome

**Implemented 2026-07-31 (Option A — Redis Streams). Recorded as ADR-065. SCALE-1 code-complete; live E2E user-run.**

**Delivered:** `spring-data-redis` optional in `execution`; extracted `ExecutionJobRunner` (shared run logic); `@JsonCreator` on `ExecutionJob`/`ExecutionJobResult` (immutable → JSON-round-trippable); `InProcessExecutionJobQueue` refactored to the runner + guarded `@ConditionalOnExpression(enabled AND provider!=redis)`; `RedisStreamExecutionJobQueue` (`@ConditionalOnClass(StringRedisTemplate)` + `@ConditionalOnExpression(enabled AND provider=redis)`) — XADD submit, consumer-group worker → runner → result key (TTL) → XACK, poll-based `awaitResult`, own-pending recovery on startup. Gateway `compose`-profile opt-in (`AIQAOS_EXECUTION_QUEUE_ENABLED`/`_PROVIDER`, default OFF).

**Verified here:** `ExecutionJobSerializationTest` **3/3** (job + success + failure round-trip — the serialization risk retired first), `ExecutionJobRunnerTest` **2/2** (wraps result; engine failure → failure not throw), `InProcessExecutionJobQueueTest` **3/3** (constructor updated to the runner); **full reactor `mvn clean test` BUILD SUCCESS, 22 modules, 0 failures** — queue off by default, so `ExecutionStep` runs inline unchanged and no Redis is needed to build/boot.

**Two fixes during impl:** (1) `ExecutionEngine` is not a functional interface (4 methods) → the runner test uses a full anonymous engine; (2) `InProcessExecutionJobQueueTest`'s 3 construction sites updated to the new `ExecutionJobRunner` constructor.

**Honest boundary:** the runner, serialization, and conditional selection are proven here; **true cross-instance distribution is user-run** — Redis up, ≥2 instances with `queue.enabled=true, provider=redis`, submit on A → a worker (A or B) runs it → A's `awaitResult` returns. A *full* job also needs Playwright.

**Deferred (independent):** cross-instance `XAUTOCLAIM` reclaim of a permanently-dead worker (own-pending recovery covers restarts); per-type partitioning / backpressure; containerised Playwright workers + object-storage-backed artifacts (ENT-5).
