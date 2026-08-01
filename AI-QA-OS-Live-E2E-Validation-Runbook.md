# AI-QA-OS — Live E2E Validation Runbook

**Purpose.** The following roadmap items are **code-complete and green in the build (22 modules)** but their final validation needs live infrastructure (Docker/DB/Redis/Kafka/MinIO/browser) that the build sandbox doesn't have. Running the checks below flips them from **In Progress → Completed** (the SEC-1/ENT-1 precedent: code-complete + green build + a passing live E2E). Each section states what it proves, the exact opt-in flags, steps, and pass criteria.

> Conventions: run from `AI-QA-OS-Core/`. No Maven wrapper — use `mvn`. Runnable apps: **gateway** (`:8080` by default), **dashboard** (`:8090`), config. Run an app on the compose profile with `SPRING_PROFILES_ACTIVE=compose mvn -pl ai-qa-os-<app> spring-boot:run` (or build a jar and pass the env vars). All flags below are real and default to **off/local** — nothing here changes default behaviour.

---

## 0. Bring up the infrastructure (prerequisite for everything)

```bash
cd deployment/docker
cp .env.example .env            # adjust creds if desired (MinIO/Postgres have dev defaults)
docker compose up -d            # infra only: postgres, redis, qdrant, kafka (KRaft), minio(+bucket), otel
./verify-infra.sh               # T10 — checks each service is reachable/healthy
```

**Pass:** `verify-infra.sh` reports all services healthy; MinIO console at `http://localhost:9001` shows the `aiqaos-artifacts` bucket.

---

## 1. ENT-1 — Multi-tenant isolation  *(→ already Completed; regression-confirm)*

**Proves:** a request's tenant comes from the signed JWT; tenant A cannot see tenant B's data.

1. Run the gateway with security on + a bootstrap admin:
   ```bash
   SPRING_PROFILES_ACTIVE=compose \
   AIQAOS_SECURITY_ENABLED=true \
   AIQAOS_BOOTSTRAP_ADMIN_PASSWORD='<pick-a-dev-pw>' \
   JWT_SECRET='<32+ char dev secret>' \
   mvn -pl ai-qa-os-gateway spring-boot:run
   ```
2. `POST /api/auth/login` with the admin creds → capture the `accessToken`.
3. Create/seed data as tenant A (token A), then query the same endpoints with a token minted for tenant B.

**Pass:** tenant B's reads never return tenant A's rows; a token minted for tenant A resolving a tenant-B user → 401.

---

## 2. ENT-4 — Admin user-management write-ops  *(In Progress → Completed)*

**Proves:** `hasRole('ADMIN')` gates the write API (FI-ENT4-A/C, ADR-066/067); admin CRUD works tenant-scoped.

Run the gateway as in §1 (security on, bootstrap admin). With the admin `accessToken`:
```bash
# create a user
curl -X POST :8080/api/admin/users -H "Authorization: Bearer $TOK" -H 'Content-Type: application/json' \
  -d '{"username":"qa1","email":"qa1@acme.io","password":"pw","roles":["QA_MANAGER"]}'      # 201
# disable / enable
curl -X PATCH :8080/api/admin/users/<id>/enabled -H "Authorization: Bearer $TOK" -d '{"enabled":false}'
# assign roles
curl -X PUT   :8080/api/admin/users/<id>/roles   -H "Authorization: Bearer $TOK" -d '{"roles":["ADMIN"]}'
```
**Pass:** all succeed with the admin token; **without** a token or with a non-ADMIN token → **403**; disabling your own account or removing your own ADMIN → **400** (self-lockout/self-demotion guards). Read model: `GET /api/dashboard/admin/rbac`.

---

## 3. SCALE-1 — Distributed execution queue over Redis  *(In Progress → Completed)*

**Proves:** with Redis Streams, any worker instance picks up any job; results return to the submitter (ADR-065).

1. Start **two** gateway instances against the same Redis, queue enabled:
   ```bash
   SPRING_PROFILES_ACTIVE=compose AIQAOS_EXECUTION_QUEUE_ENABLED=true SERVER_PORT=8080 mvn -pl ai-qa-os-gateway spring-boot:run
   SPRING_PROFILES_ACTIVE=compose AIQAOS_EXECUTION_QUEUE_ENABLED=true SERVER_PORT=8081 mvn -pl ai-qa-os-gateway spring-boot:run
   ```
2. Submit executions to instance A; observe some jobs consumed on instance B (logs / `XPENDING execution:jobs execution-workers` in `redis-cli`).

**Pass:** jobs submitted on A are executed on either instance and the result returns to the submitter; killing one instance mid-run doesn't lose the other's jobs. *(Cross-instance `XAUTOCLAIM` reclaim of a dead worker's pending is a known follow-on, not required for this pass.)*

---

## 4. SCALE-2 — Distributed event bus over Kafka  *(In Progress → Completed)*

**Proves:** a `BaseEvent` published on one instance reaches all instances (ADR-064).

1. Two gateway instances, Kafka transport:
   ```bash
   SPRING_PROFILES_ACTIVE=compose AIQAOS_EVENTS_TRANSPORT=kafka SERVER_PORT=8080 mvn -pl ai-qa-os-gateway spring-boot:run
   SPRING_PROFILES_ACTIVE=compose AIQAOS_EVENTS_TRANSPORT=kafka SERVER_PORT=8081 mvn -pl ai-qa-os-gateway spring-boot:run
   ```
2. Trigger an event on A (e.g. run a workflow); confirm B's `@EventListener`/consumer fires (logs), and topic `aiqaos.events` shows the message.

**Pass:** an event published on A is observed on B; with the default (`AIQAOS_EVENTS_TRANSPORT` unset = in-process) it stays local — confirming the binding is opt-in.

---

## 5. ENT-5 — Durable artifacts: upload → serve round-trip  *(In Progress; core loop code-complete)*

**Proves:** execution artifacts land in object storage (FI-ENT5-A/ADR-071) and are servable from it (FI-ENT5-C/ADR-073) — cross-host reachable.

1. Run the gateway (producer) with durable storage + upload on:
   ```bash
   SPRING_PROFILES_ACTIVE=compose \
   AIQAOS_ARTIFACTS_STORE=object AIQAOS_ARTIFACTS_OBJECT_PROVIDER=s3 AIQAOS_ARTIFACTS_UPLOAD_ENABLED=true \
   mvn -pl ai-qa-os-gateway spring-boot:run
   ```
   (S3 creds default to the compose MinIO: endpoint `http://localhost:9000`, `minioadmin`/`minioadmin`, bucket `aiqaos-artifacts`.)
2. Run a test execution that produces artifacts (screenshot/video/trace).
3. In the MinIO console, confirm objects under `artifacts/<tenant>/executions/<id>/run-<n>/<browser>/<testCaseId>/<type>`.
4. Run the **dashboard** with the same flags; hit `GET /api/artifacts/store/<key>` (or view the artifact in the UI) — bytes stream from MinIO.

**Pass:** artifacts appear in MinIO after a run; the dashboard serves them from the store. *(Multi-tenant durable serve — binding the tenant at serve — is follow-on FI-ENT5-E; single-tenant/system serving works now.)*

---

## 6. PE-3 — Prompt quality + regression dashboard  *(In Progress → Completed candidate)*

**Proves:** the leaderboard (FI-PE3-A) and temporal regression detection (FI-PE3-B/ADR-069) read faithfully from persisted `eval_results`.

1. Populate `eval_results` by running the prompt-eval harness (multiple runs per prompt version, over time, so a version can show decline).
2. Run the dashboard (`mvn -pl ai-qa-os-dashboard spring-boot:run`, compose profile).
3. `GET /api/dashboard/prompt-quality` (leaderboard) and `GET /api/dashboard/prompt-quality/regressions` (regressed versions).

**Pass:** the leaderboard ranks versions by mean score; a version whose recent scores dropped below its earlier scores (beyond tolerance `0.05`, ≥`4` samples) appears in `regressions`; versions with too few samples are absent (skipped, not fabricated).

---

## 7. Dashboard UIs  *(ENT-4 / HEAL-3 / PE-3 pages)*

```bash
cd ai-qa-os-dashboard-ui
npm install
npm run dev            # Vite proxies /api → http://localhost:8090 (run the dashboard app too)
```
Visit `/admin` (ENT-4 — RBAC summary + create/enable-disable/role editor, ADMIN-gated), `/healing` (HEAL-3 analytics), `/prompt-quality` (PE-3 leaderboard + regressions panel).

**Pass:** each page loads its read-model; the `/admin` write actions work with an ADMIN session; the regressions panel renders (or shows "No regressions detected").

---

## Status impact

| Item | Flips to Completed when |
|---|---|
| SCALE-1 | §3 passes (cross-instance queue) |
| SCALE-2 | §4 passes (cross-instance events) |
| ENT-4 | §2 passes (admin write-ops authz) |
| PE-3 | §6 passes (dashboard + regressions over real eval_results) |
| ENT-5 | §5 passes; note the remaining FI-ENT5-B (backup CronJobs) + FI-ENT5-E (multi-tenant serve) |

**Honestly still deferred (no faithful data source — not E2E-able):** HEAL-3 locator drift (ADR-070/072), FI-PE3-C, LRN-3 — these await their producers being wired, not a live run.
