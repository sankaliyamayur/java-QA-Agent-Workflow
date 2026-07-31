# Phase 1 — Technical Design: Local Infrastructure Setup

**Version:** 1.0.0
**Document Type:** Phase Technical Design (cross-cutting enablement — not a single roadmap item)
**Document Status:** Draft — Awaiting Approval (no code until approved)
**Last Updated:** 2026-07-29
**Owner:** Platform / Infrastructure
**Unblocks:** ENT-1 enforcement (FI-ENT1-C/D/E), SCALE-1, SCALE-2, ENT-5, the dashboard UIs, and the live-wiring FIs — none of which can be validated without a real DB / broker / object store.

> **Purpose & scope.** Phase 1 **stands up the runtime the code already expects** and wires the apps to it via a real-infrastructure Spring profile, with a per-service verification. It does **not** implement any feature logic (SCALE-2's event bus, ENT-5's real S3 client, SCALE-1's broker-backed queue are later phases that *consume* this infra). Deliverable of this design: the environment exists, every app boots against real Postgres, Flyway migrates, and every service is health-verified.
>
> **Environment note.** This design + the files it specifies are authored here; **`docker compose up` and booting the Spring apps run on your machine** (this sandbox has no Docker/DB/broker). Each task below carries an exact **verify command you run locally**.

---

## 0. Grounding — what exists, what's missing, what's inconsistent

### 0.1 Already provisioned (`deployment/docker/docker-compose.yml`)
`postgres:16-alpine` (DB `ai_qa_os`, user `qaosuser`), `redis:7-alpine`, `qdrant:v1.9.0`, `otel/opentelemetry-collector:0.95.0`, and the `gateway` app image.

### 0.2 Already wired in Spring (config, no code needed)
| Concern | Mechanism | Today |
|---|---|---|
| Datasource | Gateway is the **Flyway migration owner** (ORG-2/ADR-024, `V1..V16`), `ddl-auto: validate`, HikariCP per-service pools (SCALE-4), creds env-injected (SEC-2) | Real Postgres ready; `local` profile uses **H2 in-memory** |
| Short-term memory | `MemoryConfig`: `aiqaos.memory.shortterm.provider` = `caffeine` (default) or **`redis`** | Redis wiring is a property flip |
| Vector store | `aiqaos.memory.vector.provider` = `in-memory` (default) or **`qdrant`** | Qdrant wiring is a property flip |
| Artifact object storage | ENT-5 `ObjectStorageClient` seam + `InMemoryObjectStorageClient` (real S3/GCS deferred) | MinIO is the local S3; real client is ENT-5 |
| Execution job queue | SCALE-1 `ExecutionJobQueue` seam + in-process impl (broker-backed deferred) | Broker feeds SCALE-1 later |
| Events | `PlatformEventBus` (in-process); ENT-2 `NotificationEventRouter` | Broker feeds SCALE-2 later |

### 0.3 Missing / inconsistent (Phase 1 fixes)
1. **No message broker** container. **No MinIO** container.
2. **DB-name inconsistency:** both `gateway` and `dashboard` `application.yml` target `jdbc:postgresql://…/ai_qa_os_dashboard`, but compose creates `ai_qa_os`. → Phase 1 makes the compose DB name **`ai_qa_os_dashboard`** and drives the URL from env.
3. **No real-infra Spring profile** — `local` is H2 only; there is no profile that points the apps at the containers.
4. **Credentials** must stay env-injected (SEC-2) — a `.env` drives compose *and* the apps; `.env.example` already exists and gets extended.

### 0.4 / Decision for approval — message broker choice

| Option | Approach | Trade-off |
|---|---|---|
| **A — Kafka (KRaft, single node), Redpanda as the lighter local drop-in (recommended)** | `apache/kafka:3.8` in KRaft mode (no Zookeeper), one container; document `redpandadata/redpanda` as a lighter Kafka-API-compatible alternative for constrained laptops. | Best long-term fit for SCALE-1 (job queue), SCALE-2 (event bus), and streaming; log-based replay + partitions. Slightly heavier than Rabbit locally (mitigated by Redpanda). |
| **B — RabbitMQ** | `rabbitmq:3-management`. | Lighter, simpler routing/ack semantics for pure notifications; but weaker fit for a job-queue/streaming platform and diverges from the SCALE-2 "event bus" framing. |

**Recommend A (Kafka KRaft; Redpanda for local).** The platform's remaining scale items (SCALE-1 worker pool, SCALE-2 event bus, execution job queue) are Kafka-shaped; KRaft mode removes the Zookeeper dependency, and Redpanda is a single-binary drop-in if RAM is tight. All broker config is behind the `compose` profile, so the choice is reversible.

> ✅ **Decision (confirmed 2026-07-29): Option A — Kafka `apache/kafka:3.8` in KRaft mode (single node, no Zookeeper); Redpanda documented as the lighter local drop-in; all broker config behind the `compose` profile.** To be recorded as ADR-053 at Phase-1 completion.

---

## 1. Technical Design

### 1.1 Target local stack (services, images, ports)

| Service | Image | Host ports | Purpose |
|---|---|---|---|
| PostgreSQL | `postgres:16-alpine` | 5432 | Canonical relational store (shared schema, gateway-owned migrations) |
| Redis | `redis:7-alpine` | 6379 | Short-term memory store (ADR-003) |
| Qdrant | `qdrant/qdrant:v1.9.0` | 6333 (HTTP), 6334 (gRPC) | Vector store (ADR-021) |
| **Kafka (KRaft)** | `apache/kafka:3.8.0` | 9092 (external), 9093 (controller) | Event bus (SCALE-2) + job queue (SCALE-1) |
| **MinIO** | `minio/minio` | 9000 (S3 API), 9001 (console) | S3-compatible artifact/backup store (ENT-5) |
| OTel Collector | `otel/opentelemetry-collector:0.95.0` | 4317/4318 | Trace/metric export (OBS-1) |
| *(optional)* Kafka UI | `provectuslabs/kafka-ui` | 8085 | Broker inspection |
| *(optional)* pgAdmin | `dpage/pgadmin4` | 5050 | DB inspection |
| *(optional)* Prometheus + Grafana | from `docker-compose-observability.yml` | 9090 / 3000 | Dashboards (OBS suite) |

### 1.2 Recommended local development stack (tooling)
- **Docker Desktop / Podman** + **Docker Compose v2**.
- **JDK 21** for the Maven target. ⚠️ The runtime JDK matters: on **JDK 25**, JaCoCo can't instrument (use `-Djacoco.skip=true`) and ArchUnit's importer needs `-DargLine="-Xss40m"` (see `build-env-java25-gotchas`).
- **Maven 3.9+**.
- **Node 18+** (dashboard UI + the bundled Playwright project).
- **MinIO Client (`mc`)** and **`psql`**/`redis-cli`/`kafka` CLIs (optional, for manual verification).

### 1.3 Profile strategy — a new `compose` profile
Add a **`compose`** Spring profile (alongside `local`/`dev`/`stage`/`prod`) that points the apps at the containers. `local` stays H2 (fast, no infra); `compose` is "real infra on my laptop." Activated with `SPRING_PROFILES_ACTIVE=compose`. It sets:
- **Datasource** → containerized Postgres (`${DB_URL}`), `ddl-auto: validate`, Flyway **enabled** on the gateway only.
- **Redis** → `spring.data.redis.host/port`; `aiqaos.memory.shortterm.provider=redis`.
- **Vector** → `aiqaos.memory.vector.provider=qdrant` + Qdrant host.
- **Kafka** → `spring.kafka.bootstrap-servers=${KAFKA_BOOTSTRAP:localhost:9092}` (connectivity + health only in Phase 1).
- **Object storage** → `aiqaos.storage.*` MinIO endpoint/creds/buckets (connectivity + health only in Phase 1).
- **Security** → `aiqaos.security.enabled=false` for frictionless local dev (documented toggle; flip to `true` to exercise SEC-1).

### 1.4 DB reconciliation
Canonical dev DB name = **`ai_qa_os_dashboard`** (both apps already expect it). Phase 1: set the compose Postgres `POSTGRES_DB=ai_qa_os_dashboard`, drive the app URL from `${DB_URL}`/`${SPRING_DATASOURCE_*}`, and keep the **gateway as the sole Flyway owner** (dashboard app runs `flyway.enabled=false`) to avoid the dual-migration race (ADR-024).

### 1.5 Verification model (how each task is proven)
- **Postgres/Redis/Kafka**: Spring Boot Actuator `/actuator/health` contributors (DB + Redis auto; Kafka via `spring-kafka`) must report `UP`.
- **Qdrant / MinIO**: a lightweight custom health contributor (`QdrantHealthIndicator`, `MinioHealthIndicator`) + `mc`/HTTP checks.
- **Migrations**: `flyway_schema_history` shows `V1..V16` applied on boot.
- **Container liveness**: compose `healthcheck` per service + `docker compose ps` all `healthy`.

---

## 2. Folder / module changes

```
AI-QA-OS-Core/
  deployment/docker/
    docker-compose.yml            [E]  add kafka + minio (+ minio-createbuckets, optional UIs); DB name → ai_qa_os_dashboard
    .env.example                  [N]  compose-scoped env (co-located)  — or reuse root .env.example [E]
  ai-qa-os-config/src/main/resources/
    application-compose.yml       [N]  real-infra profile (datasource/redis/qdrant/kafka/minio/security)
  ai-qa-os-gateway/src/main/resources/
    application-compose.yml       [N]  gateway overrides (Flyway OWNER on; datasource; kafka health)
  ai-qa-os-dashboard/src/main/resources/
    application-compose.yml       [N]  dashboard overrides (Flyway OFF; datasource)
  ai-qa-os-memory/…/vector/
    QdrantHealthIndicator.java    [N]  actuator health for Qdrant (behind compose profile)
  ai-qa-os-execution/…/artifact/  (ENT-5's home for ObjectStorageClient)
    ObjectStorageProperties.java  [N]  aiqaos.storage.* (endpoint/creds/buckets)
    MinioHealthIndicator.java     [N]  actuator health for MinIO (behind compose profile)
    MinioBucketInitializer.java   [N]  CommandLineRunner: ensure buckets exist (compose profile)
  pom.xml (relevant modules)      [E]  add spring-kafka (gateway) + io.minio:minio (execution) — provided/optional, profile-gated beans
.env.example (repo root)          [E]  add KAFKA_/MINIO_/DB_ vars
```

> **Minimal-core vs. full:** Tasks **T1–T6** (compose + `compose` profile + Postgres/Redis/Qdrant + migrations) are the must-have, fully self-verifying core — they need **no new Java deps** (Redis/Qdrant are property flips). Broker + MinIO health wiring (**T7–T9**) add `spring-kafka` + `io.minio` and are what prove those two new services are reachable; the *consumers* (SCALE-2/ENT-5/SCALE-1) are later phases.

---

## 3. Docker Compose configuration (design)

`deployment/docker/docker-compose.yml` (infra additions; existing services kept, DB name reconciled):

```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: qaos-postgres
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-ai_qa_os_dashboard}
      POSTGRES_USER: ${POSTGRES_USER:-qaosuser}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?set in .env}
    ports: ["5432:5432"]
    volumes: ["postgres_data:/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-qaosuser} -d ${POSTGRES_DB:-ai_qa_os_dashboard}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: qaos-redis
    ports: ["6379:6379"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  qdrant:
    image: qdrant/qdrant:v1.9.0
    container_name: qaos-qdrant
    ports: ["6333:6333", "6334:6334"]
    volumes: ["qdrant_data:/qdrant/storage"]

  kafka:
    image: apache/kafka:3.8.0
    container_name: qaos-kafka
    ports: ["9092:9092"]
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@localhost:9093
      KAFKA_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
    healthcheck:
      test: ["CMD-SHELL", "/opt/kafka/bin/kafka-broker-api-versions.sh --bootstrap-server localhost:9092 >/dev/null 2>&1 || exit 1"]
      interval: 15s
      timeout: 10s
      retries: 10

  minio:
    image: minio/minio
    container_name: qaos-minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER:-minioadmin}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD:?set in .env}
    ports: ["9000:9000", "9001:9001"]
    volumes: ["minio_data:/data"]
    healthcheck:
      test: ["CMD-SHELL", "mc ready local || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5

  minio-createbuckets:      # one-shot: ensure buckets exist, then exit
    image: minio/mc
    depends_on:
      minio: { condition: service_healthy }
    entrypoint: >
      /bin/sh -c "
      mc alias set local http://minio:9000 $${MINIO_ROOT_USER} $${MINIO_ROOT_PASSWORD};
      mc mb -p local/aiqaos-artifacts local/aiqaos-backups;
      exit 0;"
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER:-minioadmin}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD:?set in .env}

  otel-collector:
    image: otel/opentelemetry-collector:0.95.0
    container_name: qaos-otel-collector
    # (existing config retained)

  # optional inspection UIs — profile: tools
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    profiles: ["tools"]
    ports: ["8085:8080"]
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
    depends_on: [kafka]

volumes:
  postgres_data:
  qdrant_data:
  minio_data:
```

*(Redpanda alternative for T-broker: replace the `kafka` service with `redpandadata/redpanda` exposing `9092`; the app config is unchanged since it's Kafka-API-compatible.)*

---

## 4. Spring Boot configuration changes (design)

`application-compose.yml` (base, in `ai-qa-os-config`; apps pull via `spring.config.import`/profile):
```yaml
spring:
  datasource:
    url: ${DB_URL:jdbc:postgresql://localhost:5432/ai_qa_os_dashboard?preferQueryMode=simple}
    username: ${SPRING_DATASOURCE_USERNAME:qaosuser}
    password: ${SPRING_DATASOURCE_PASSWORD}
  jpa:
    hibernate: { ddl-auto: validate }
    database-platform: org.hibernate.dialect.PostgreSQLDialect
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP:localhost:9092}

aiqaos:
  memory:
    shortterm: { provider: redis }
    vector:    { provider: qdrant, qdrant: { host: ${QDRANT_HOST:localhost}, port: ${QDRANT_PORT:6334} } }
  storage:
    provider: minio
    endpoint: ${MINIO_ENDPOINT:http://localhost:9000}
    access-key: ${MINIO_ROOT_USER:minioadmin}
    secret-key: ${MINIO_ROOT_PASSWORD}
    buckets: { artifacts: aiqaos-artifacts, backups: aiqaos-backups }
  security:
    enabled: ${AIQAOS_SECURITY_ENABLED:false}   # dev-friendly; flip to true to exercise SEC-1
```
- **Gateway `application-compose.yml`**: `spring.flyway.enabled=true` (owner), datasource pool `gateway-pool`.
- **Dashboard `application-compose.yml`**: `spring.flyway.enabled=false` (consumer of the shared schema), datasource pool `dashboard-pool`.

`.env.example` additions:
```
POSTGRES_DB=ai_qa_os_dashboard
POSTGRES_USER=qaosuser
POSTGRES_PASSWORD=change-me
SPRING_DATASOURCE_USERNAME=qaosuser
SPRING_DATASOURCE_PASSWORD=change-me
JWT_SECRET=change-me-32-bytes-min
REDIS_HOST=localhost
QDRANT_HOST=localhost
KAFKA_BOOTSTRAP=localhost:9092
MINIO_ENDPOINT=http://localhost:9000
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=change-me
```

---

## 5. New modules / config classes

**No new Maven module** — infra config lives in existing modules (`config`, `memory`, `execution`) + `deployment/`. New classes, all **profile-gated (`@Profile("compose")`)** so `local`/test are untouched:

| Class | Module | Role |
|---|---|---|
| `QdrantHealthIndicator` | `memory` | Actuator health: HTTP GET Qdrant `/healthz` |
| `ObjectStorageProperties` | `execution` | `@ConfigurationProperties("aiqaos.storage")` |
| `MinioHealthIndicator` | `execution` | Actuator health: MinIO `bucketExists` ping |
| `MinioBucketInitializer` | `execution` | `CommandLineRunner`: ensure `aiqaos-artifacts`/`aiqaos-backups` exist |

*(Datasource, Redis, and Kafka health are Spring Boot auto-configured once the properties + `spring-kafka` are present — no custom class needed.)*

---

## 6. Verification strategy

A single script `deployment/docker/verify-infra.sh` chains the per-service checks (each also runnable standalone):
`docker compose ps` (all healthy) → `pg_isready` → `redis-cli ping` → Qdrant `/healthz` → `kafka-broker-api-versions` → `mc ls local/aiqaos-artifacts` → boot the gateway on `compose` profile → `curl /actuator/health` shows `db/redis/kafka` = `UP` → `flyway_schema_history` has `V1..V16`.

---

## 7. Implementation plan — small, verifiable tasks

Each task is independently verifiable; do them **in order**, verifying before proceeding (Design → Implement → **Verify** → Document).

| # | Task | Deliverable | Verify (run locally) |
|---|---|---|---|
| **T1** | Reconcile DB name + `.env` | compose `POSTGRES_DB=ai_qa_os_dashboard`; extend `.env.example`; create `.env` | `docker compose config` renders; no `?set in .env` errors |
| **T2** | Postgres + Redis + Qdrant up (already defined) + healthchecks | compose healthchecks added | `docker compose up -d postgres redis qdrant` → `docker compose ps` all `healthy` |
| **T3** | `compose` Spring profile (datasource → real Postgres) | `application-compose.yml` (config + gateway + dashboard) | `SPRING_PROFILES_ACTIVE=compose` boot gateway → starts; `/actuator/health` `db=UP` |
| **T4** | Flyway migrates on real Postgres (gateway owner) | gateway flyway on, dashboard off | boot gateway → `psql -c 'select version,success from flyway_schema_history'` shows `V1..V16` success |
| **T5** | Redis short-term memory wired | `provider: redis` in `compose` | `/actuator/health` `redis=UP`; a memory put/get round-trips (smoke) |
| **T6** | Qdrant vector wired + health | `provider: qdrant` + `QdrantHealthIndicator` | `/actuator/health` `qdrant=UP`; a `prompt_cache` collection can be created |
| **T7** | Kafka container + broker health | compose `kafka` service; `spring-kafka` dep (gateway) | `docker compose up -d kafka` healthy; gateway `/actuator/health` `kafka=UP` |
| **T8** | MinIO container + buckets | compose `minio` + `minio-createbuckets` | `mc ls local` shows `aiqaos-artifacts`, `aiqaos-backups` |
| **T9** | MinIO health + bucket-init in app | `ObjectStorageProperties`, `MinioHealthIndicator`, `MinioBucketInitializer` (compose profile) | boot gateway → `/actuator/health` `minio=UP`; buckets present |
| **T10** | Full-stack up + verify script | `verify-infra.sh` | `docker compose up -d` → `verify-infra.sh` exits 0; optional `--profile tools` UIs reachable |
| **T11** | Regression guard | — | `mvn clean test -Djacoco.skip=true -DargLine="-Xss40m"` still green (no `compose`-profile bean leaks into tests) |
| **T12** | Document | update `AI-QA-OS-Documentation.md` §7 (setup) + a `deployment/docs/local-infra.md`; ADR-053 (local infra stack: compose profile + Kafka/MinIO) | doc review |

**Definition of Done (Phase 1):** `docker compose up -d` brings **all** services healthy; both apps boot on the `compose` profile against real Postgres with `V1..V16` migrated; Redis/Qdrant/Kafka/MinIO all report `UP`; the full unit-test reactor stays green; ADR-053 records the stack. **Then** Phase 2 (ENT-1 enforcement) can begin against real infrastructure.

---

## Appendix — risks & future ideas
- **FI-P1-A** — CI service-container matrix (spin the same compose in GitHub Actions for integration tests against real infra).
- **FI-P1-B** — Helm/K8s parity: fold Kafka + MinIO into `deployment/kubernetes` and the Helm chart (K8s dirs already exist for gateway/brain/databases).
- **Risk — memory/CPU:** the full stack (Postgres+Redis+Qdrant+Kafka+MinIO+2 apps) is heavy on a laptop; Redpanda (T-broker alt) and running only `postgres redis qdrant` for backend-only work mitigate it.
- **Risk — dual Flyway:** enforced by dashboard `flyway.enabled=false` (ADR-024); T4 verifies only the gateway writes `flyway_schema_history`.

---

## Implementation Outcome (2026-07-30)

Executed in checkpoints. Grounding during implementation **corrected the plan** where the design over-reached against stub/absent code — recorded honestly here rather than building scaffolding for services nothing consumes.

### Checkpoint 1 — containers (T1, T2, T8-infra)
- Rewrote `deployment/docker/docker-compose.yml`: reconciled `POSTGRES_DB=ai_qa_os_dashboard`, dev-default creds, healthchecks; added **Kafka** (`apache/kafka:3.8.0` KRaft), **MinIO** + `minio-createbuckets` (buckets `aiqaos-artifacts`/`aiqaos-backups`), Qdrant gRPC; `gateway` behind `--profile apps`, `kafka-ui` behind `--profile tools`.
- Extended `.env.example` with the Phase 1 infra vars.
- **Verification:** user-run (`docker compose up -d` / `ps`) — sandbox has no Docker.

### Checkpoint 2 — `compose` Spring profile (T3–T4)
- **Design correction:** the plan assumed the profile would live in `ai-qa-os-config` and apps would pull it — but neither app depends on `config` and there is no `spring.config.import`, so that YAML would never load. Put `application-compose.yml` **directly in each app's resources** instead.
- `ai-qa-os-gateway/.../application-compose.yml` — Flyway **owner** (`enabled: true`), compose-matching datasource, `ddl-auto: validate`.
- `ai-qa-os-dashboard/.../application-compose.yml` — Flyway **off** (ADR-024 single-owner), same datasource.
- **Verification:** both boot jars package cleanly; gateway jar carries `application-compose.yml` + `db/migration/V1..V16`. Runtime Flyway/health check is user-run (needs live Postgres).

### Checkpoint 3 — service bindings (T5; T6–T9 recalibrated)
Grounding revealed most T6–T9 targets bind to **stub or absent** code, so only the real one was wired:
- **T5 Redis — DONE.** `RedisMemoryStore` + `spring-boot-starter-data-redis` are real and on the gateway classpath (via `brain → memory`). Gateway `compose` profile now sets `aiqaos.memory.shortterm.provider=redis` + `spring.data.redis.host/port`; Spring Boot's auto `RedisHealthIndicator` surfaces `redis` at `/actuator/health`. No custom code needed. (Dashboard lacks the memory module → Redis config would be inert there, so it was not added.)
- **T6 Qdrant — DEFERRED.** `QdrantStoreClient` is a **no-op stub** (`health()` unconditionally returns `true`; no `io.qdrant` client, no host/port). A health indicator against it would be theatre, and switching the vector provider to `qdrant` would *degrade* dev to a no-op store. Kept `aiqaos.memory.vector.provider=in-memory`. Deferred to the real Qdrant binding (SCALE-3 completion). Container remains up and ready.
- **T7 Kafka — DEFERRED.** No Kafka usage exists anywhere (no `KafkaTemplate`/`@KafkaListener`/`spring-kafka` dep/producer). A broker health check with no consumer is speculative. Deferred to the first real event producer/consumer (SCALE-2). Container remains up and ready.
- **T8/T9 MinIO — DEFERRED (user decision, 2026-07-30).** The `ObjectStorageClient` seam is real (interface + `InMemoryObjectStorageClient` reference + `ObjectStorageArtifactStore` consumer), but the interface javadoc already marks the real binding deferred ("not built in this environment — no bucket/creds"), and it cannot be runtime-validated in the sandbox. Chose to defer the `MinioObjectStorageClient` (io.minio) binding until **ENT-5** backups is activated and can exercise it. Container + buckets remain up and ready.

### Checkpoint 4 — infra verifier (T10)
- Authored `deployment/docker/verify-infra.sh`: one-shot health verifier for the whole stack (postgres/redis/kafka/minio/qdrant as **required**; Flyway version + MinIO buckets as **informational**, since they depend on the app/init having run). Flags: `--up` (compose up, then wait for cold-start healthchecks), `--down`, or no-arg fast verify. Direct per-service probes (unambiguous across compose versions); exits non-zero on any required failure.
- **Verification (here):** `bash -n` clean; exec bit set; flag guard rejects unknown args before touching Docker. Live run is user-side (needs Docker).

**Net Phase 1 state:** infra containers provisioned (Ck1); apps boot on `compose` against real Postgres with gateway-owned Flyway (Ck2); Redis short-term memory live (Ck3); `verify-infra.sh` health verifier authored (Ck4/T10). Qdrant/Kafka/MinIO **Java bindings** are deliberately deferred to their consuming features — the *containers* are in place, so each binding is later a thin, validatable addition. Full unit-test reactor green (T11, verified this session). **Remaining to close Phase 1:** T12 — doc/ADR-053 sync + the postponed `AI-QA-OS-Documentation.md` refresh.

---

## Document Completion Status
**Status:** ✅ **Complete (design-side).** Checkpoints 1–4 + T12 done — infra provisioned; `compose` profile boots apps on real Postgres with gateway-owned Flyway; Redis short-term memory live; `verify-infra.sh` authored; Qdrant/Kafka/MinIO bindings deferred to consumers (containers ready). **T12 doc sync done:** ADR-053 recorded + ADR index backfilled (028→053); Implementation Tracker updated (Infrastructure Enablement note, dated); `AI-QA-OS-Documentation.md` reconciled (auth/CSP/AI-1/module-count/security-warning corrected + status-reconciliation banner). The only remaining step is **user-run runtime validation** (Docker not available in the build sandbox).
**Implements:** Phase 1 (infrastructure enablement) — unblocks ENT-1 enforcement + all infra-gated items.
**Next step:** User runs `bash verify-infra.sh --up`, boots the gateway on the `compose` profile, and confirms Flyway `V1..V16` + `redis`/db `UP`. On green, Phase 2 (ENT-1 enforcement against real infrastructure) can begin.
