# AI-2 — Technical Design: Human-in-the-Loop Approval Workflow

**Version:** 1.0.0
**Document Type:** Implementation Technical Design
**Document Status:** Draft — Awaiting Approval (no code until approved)
**Last Updated:** 2026-07-22
**Roadmap item:** [`AI-2`](./AI-QA-OS-Improvement-Roadmap.md) (v2.2.0, frozen) — 🟠 P1 · Effort M · Owner AI/Brain + Orchestration · Phase 1 · v1.4
**Modules:** `ai-qa-os-orchestration`, `ai-qa-os-gateway`, `ai-qa-os-dashboard` (+ UI, + DB)
**Depends on:** [AI-1](./AI-QA-OS-AI-1-Technical-Design.md) (Completed) — consumes its `HUMAN_REVIEW` pause. **Governing ADRs:** ADR-004 (Gateway = single entry), ADR-010 (gate). Rule 15 (Human Approval Points).

> **Scope discipline.** Implements **only** AI-2: turn AI-1's `HUMAN_REVIEW` pause into a resumable, human-approvable workflow. It does **not** add durable cross-restart execution (SCALE-1/2), an event bus, notifications (ENT-2), or role-based approval authority beyond "authenticated" (roles are not user-wired — a SEC-1 finding).

---

## 0. Roadmap Verification & Grounded Findings

### 0.1 What AI-2 requires (from the finalized roadmap)

> A pause-for-approval step in `ai-qa-os-orchestration` (reusing the `WorkflowStateMachine` pause state), an **approval REST resource on the gateway**, and a **review/approve UI view in the dashboard**… Turns the existing pause capability into a governed workflow, connecting the state machine, gateway, and dashboard into a control loop.

### 0.2 Verified facts (read during design)

| Fact | Detail |
|---|---|
| Pipeline runs on the **gateway**, async in-process | `WorkflowClientImpl.start` runs `orchestrator.runPipeline` via `CompletableFuture.runAsync`; returns `STARTED`. **No registry** of running/paused runs. |
| Control endpoints are stubs | `WorkflowController` exposes `POST /{id}/{pause,resume,cancel}` and `GET /{id}`, but `WorkflowClientImpl.getStatus/pause/resume/cancel` all `return new WorkflowResponseDTO()` (no-ops). |
| AI-1 pause **finalizes** | On `HUMAN_REVIEW`, the orchestrator sets `PAUSED`, calls `completeExecution(...)`, and returns — the run is *finished*, not resumable, and the in-memory `WorkflowContext` is discarded. |
| Shared DB has execution records | `WorkflowExecutionEntity` (`workflow_executions`, shared DB `ai_qa_os_dashboard`): `status`, `currentStep`, step counts, timing. The **dashboard reads this** for its executions view. |
| State machine supports resume | `PAUSED → RESUMED → RUNNING` is a legal transition. |
| RBAC caveat (SEC-1) | Users have no role relationship, so approval can be gated to **authenticated** only, not to a `QA_MANAGER` role. |

### 0.3 The architectural tension (dictates the design)

The **paused run's `WorkflowContext`** (requirement, analysis, test cases, scripts, …) is **in-memory on the gateway JVM**. The **review UI** is served by the **dashboard JVM** (`:8090`), and the two share only the DB. Therefore:

- **Resume must happen where the paused context lives — the gateway.** A durable, any-JVM resume would require persisting the full workflow state (serialization + versioning) — that is SCALE-1/2 territory, **out of AI-2's Effort-M scope**.
- **Visibility must be durable (DB)** so the dashboard can list reviews across JVMs.

**Resolution:** a **gateway-held in-memory registry** of paused runs (resume happens there) **plus a DB-persisted `HumanReviewEntity`** (so the dashboard can display the review queue and status). Approve/reject actions route to the gateway (which owns the resume); the dashboard reads the queue from the shared DB. This is [Fork 1 = in-memory](#05--decisions-for-approval).

---

## 1. Technical Design

### 1.1 Orchestration — make the pipeline resumable

- Extract the step loop of `AutonomousQAPipelineOrchestrator` into `runFrom(request, context, startIndex, executionRecord…)`. `runPipeline` = `runFrom(…, 0)`.
- **On `HUMAN_REVIEW` (refines AI-1):** instead of `completeExecution` (finalize), set the execution record `status=PAUSED`, register the paused run in a new **`PausedWorkflowRegistry`** (in-memory: `workflowId → {request, context, nextStepIndex, executionId, stepName, confidence}`), write a **PENDING `HumanReviewEntity`** to the DB, and return the `PAUSED` response. The context is **retained** by the registry (not discarded).
- **`resume(workflowId, reviewer, comment)`:** look up the paused run; `WorkflowStateMachine` `PAUSED→RESUMED→RUNNING`; call `runFrom(nextStepIndex)`; mark the `HumanReviewEntity` `APPROVED`; remove from the registry on completion.
- **`reject(workflowId, reviewer, comment)`:** mark the execution `CANCELLED`/`FAILED`; mark the `HumanReviewEntity` `REJECTED`; remove from the registry.

### 1.2 Persistence — the review queue

- **`HumanReviewEntity`** (new, `ai-qa-os-orchestration`, `human_reviews` table): `reviewId`, `workflowId`, `executionId`, `stepName`, `confidence`, `status` (PENDING/APPROVED/REJECTED), `reviewer`, `decisionComment`, `createdTime`, `decidedTime`. `HumanReviewRepository`.
- **Flyway `V14__human_reviews.sql`** creates the table. Reuses `WorkflowStatus.PAUSED` — no enum change.

### 1.3 Gateway — the approval REST resource

- `WorkflowController` (+`WorkflowGatewayService`/`WorkflowClientImpl` — implement the stubs):
  - `GET /api/v1/workflows/reviews` — list PENDING reviews (from `HumanReviewRepository`).
  - `POST /api/v1/workflows/{id}/approve` — body `{reviewer, comment}` → `orchestrator.resume(...)`.
  - `POST /api/v1/workflows/{id}/reject` — body `{reviewer, comment}` → `orchestrator.reject(...)`.
  - Implement `GET /{id}` (status from the registry/execution record). (`/pause` stays a stub / future.)
- All under the SEC-1 auth surface (authenticated; RBAC-authenticated only per §0.2).

### 1.4 Dashboard — the review view

- **Backend (`ai-qa-os-dashboard`):** `GET /api/dashboard/reviews` reads `HumanReviewEntity` (shared DB) for the UI list. Approve/Reject: **[Fork 2](#05--decisions-for-approval)** — either the dashboard backend **proxies** `POST /api/dashboard/reviews/{id}/{approve,reject}` to the gateway, or the UI calls the gateway directly.
- **UI (`ai-qa-os-dashboard-ui`):** a **Human Review** page (route `/reviews`, replacing/hosting the `agent-traces` placeholder) — a table of pending reviews (workflow, step, confidence) with **Approve**/**Reject** actions (+ optional comment).

### 1.5 Control loop (end to end)

```
run (gateway) → AI-1 gate: HUMAN_REVIEW → register paused run (gateway memory) + PENDING review (DB)
   → dashboard lists pending reviews (reads DB) → human approves/rejects (→ gateway)
   → approve: gateway resumes from paused step; reject: gateway cancels → review marked decided (DB)
```

### 1.6 Explicit non-goals

Durable cross-restart resume (paused runs are lost if the gateway restarts — SCALE-1/2); role-based approver authority (needs user↔role wiring); push notifications to reviewers (ENT-2 — reviews are pull/poll for now); multi-gateway coordination (single-instance assumption for the in-memory registry).

### 0.5 / Decisions for approval

| # | Decision | Options | Recommendation |
|---|---|---|---|
| **F1 — Resumability** | How a paused run resumes | **(a)** in-memory registry on the gateway; **(b)** persist the full `WorkflowContext`/state, resume from any JVM | **(a)** — matches the current in-process async model; durable resume is SCALE-1/2. Accept: paused runs don't survive a gateway restart |
| **F2 — UI approve/reject routing** | How the dashboard UI triggers a gateway-owned resume | **(a)** dashboard backend **proxies** to the gateway (UI stays single-origin on `:8090`); **(b)** UI calls the gateway (`:8082`) directly (needs a proxy/env) | **(a)** — keeps the UI's single-origin model; the one cross-service HTTP hop is centralized in the dashboard backend |

*(Everything else — gateway-owned resume, DB review queue, reuse of `PAUSED`, authenticated-only RBAC — is fixed by §0.3/§0.2.)*

> ✅ **Decisions (confirmed 2026-07-22):**
> - **F1 = (a) in-memory registry** on the gateway; resume continues from the paused step in the same JVM. Paused runs do not survive a gateway restart (durable resume = FI-AI2-A / SCALE-1/2).
> - **F2 = (a) dashboard backend proxies** approve/reject to the gateway; the UI stays single-origin on `:8090`.

---

## 2. Folder Structure

`[N]` new, `[M]` modified. `†` = Fork-dependent.

```
AI-QA-OS-Core/
├── ai-qa-os-orchestration/…/workflow/
│   ├── pipeline/AutonomousQAPipelineOrchestrator.java   [M] runFrom(startIndex); register on HUMAN_REVIEW; resume(); reject()
│   ├── review/PausedWorkflowRegistry.java               [N] in-memory paused-run registry (F1a)
│   ├── entity/HumanReviewEntity.java                    [N] human_reviews
│   └── repository/HumanReviewRepository.java            [N]
├── ai-qa-os-gateway/…/gateway/
│   ├── controller/WorkflowController.java               [M] GET /reviews, POST /{id}/approve|reject, GET /{id}
│   ├── service/WorkflowGatewayService.java              [M] wire review ops
│   └── client/WorkflowClientImpl.java                   [M] implement resume/cancel/getStatus/listReviews
├── ai-qa-os-dashboard/…/dashboard/
│   └── controller/ReviewController.java                 [N] GET /api/dashboard/reviews (+ proxy approve/reject †F2a)
├── deployment/migration/db/migration/
│   └── V14__human_reviews.sql                           [N]
└── tests: orchestration resume/reject unit + gateway review controller slice   [N]

ai-qa-os-dashboard-ui/src/
├── pages/HumanReviewPage.tsx                            [N] pending reviews + approve/reject
├── services/reviewService.ts                            [N] GET/POST via apiClient
└── routes/AppRoutes.tsx + DashboardLayout.tsx           [M] add /reviews route + nav
```

---

## 3. Required Classes (key)

| Class | Type | Module | Responsibility |
|---|---|---|---|
| `AutonomousQAPipelineOrchestrator` | Modified | orchestration | `runFrom(startIndex)`; register paused run + PENDING review on HUMAN_REVIEW (no finalize); `resume()`/`reject()` |
| `PausedWorkflowRegistry` | New | orchestration | In-memory `ConcurrentHashMap<UUID,PausedRun>` (gateway JVM) |
| `HumanReviewEntity` / `HumanReviewRepository` | New | orchestration | Durable review queue (shared DB) |
| `WorkflowController` / `WorkflowGatewayService` / `WorkflowClientImpl` | Modified | gateway | Approval REST resource; implement the resume/cancel/status stubs |
| `ReviewController` | New | dashboard | `GET /api/dashboard/reviews` (+ approve/reject proxy, F2a) |
| `HumanReviewPage` / `reviewService` | New | dashboard-ui | Review + approve/reject surface |

---

## 4. Database Changes

**One migration:** `V14__human_reviews.sql` — create `human_reviews` (id, review_id, workflow_id, execution_id, step_name, confidence, status, reviewer, decision_comment, created_time, decided_time). `ddl-auto: validate` unchanged; entity matches the migration. No change to `workflow_executions` (reuse `status=PAUSED`).

---

## 5. API Changes

**New (gateway):** `GET /api/v1/workflows/reviews`, `POST /api/v1/workflows/{id}/approve`, `POST /api/v1/workflows/{id}/reject`; `GET /api/v1/workflows/{id}` now returns real status. **New (dashboard):** `GET /api/dashboard/reviews` (+ approve/reject proxy, F2a). All authenticated (SEC-1). Existing `/start` unchanged; a run may now end `PAUSED` awaiting approval.

---

## 6. Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant ORC as Orchestrator (gateway JVM)
    participant REG as PausedWorkflowRegistry (memory)
    participant DB as human_reviews (shared DB)
    participant DASH as Dashboard backend (:8090)
    participant UI as Review UI
    participant GW as Gateway (:8082)
    ORC->>ORC: step succeeds, AI-1 gate = HUMAN_REVIEW
    ORC->>REG: register paused run (context, nextStepIndex)
    ORC->>DB: insert PENDING review
    Note over ORC: return PAUSED (run held, not finalized)
    UI->>DASH: GET /api/dashboard/reviews
    DASH->>DB: read PENDING
    DASH-->>UI: pending reviews
    UI->>DASH: POST /reviews/{id}/approve (F2a proxy)
    DASH->>GW: POST /api/v1/workflows/{id}/approve
    GW->>REG: look up paused run
    GW->>ORC: resume(nextStepIndex)
    ORC->>DB: mark review APPROVED; continue pipeline
    ORC-->>GW: run continues to completion
```

---

## 7. Step-by-Step Implementation Plan

1. **Verify (read-only).** Confirm `WorkflowExecutionService` API (updateStatus vs completeExecution), `WorkflowContext`/`AutonomousQAWorkflowState` retention (mutable object held by registry), and gateway DTOs.
2. **Orchestration:** `PausedWorkflowRegistry`; refactor to `runFrom`; register + PENDING review on HUMAN_REVIEW (not finalize); `resume()`/`reject()`; `HumanReviewEntity`/repo; `V14` migration.
3. **Gateway:** implement `WorkflowClientImpl` resume/cancel/getStatus/listReviews; add controller endpoints + service wiring.
4. **Dashboard:** `ReviewController` (`GET /reviews`; approve/reject per F2).
5. **UI:** `HumanReviewPage` + `reviewService` + route/nav.
6. **Tests:** orchestration `resume`/`reject` unit tests (paused run resumes from the right step; reject cancels); gateway review-controller `@WebMvcTest`; keep the quarantined pipeline tests + others green.
7. **Build & validate:** `mvn -pl ai-qa-os-gateway,ai-qa-os-dashboard -am test-compile` + module tests; UI `npm run build`. Report honestly — a full pause→approve→resume loop needs a live multi-service env.
8. **Sync docs:** tracker `AI-2` → status; ADR only if a new architectural decision emerges (the gateway-in-memory-resume + DB-queue split is a candidate — likely **ADR-011**).

**Definition of Done:** a `HUMAN_REVIEW`-paused run is retained (not finalized) with a PENDING DB review; the gateway approval resource lists/approves/rejects; approve resumes from the paused step, reject cancels; the dashboard lists and actions reviews; builds/tests green; tracker updated.

---

## Implementation Outcome

Implemented 2026-07-22 (F1 in-memory registry, F2 dashboard proxy). Recorded as **ADR-011**.

**Files:**
- **orchestration** — `review/PausedRun`, `review/PausedWorkflowRegistry`, `review/HumanReviewService`, `entity/HumanReviewEntity`, `repository/HumanReviewRepository` [N]; `WorkflowExecutionService.updateStatus` [M]; `AutonomousQAPipelineOrchestrator` [M] — extracted `executeFrom(startIndex)`, register-on-`HUMAN_REVIEW` (no finalize) + PENDING review, added `resume()`/`reject()`; `AutonomousQAPipelineTest` constructor updated.
- **gateway** — `dto/HumanReviewDTO`, `dto/ReviewDecisionDTO` [N]; `WorkflowClient`/`WorkflowClientImpl` [M] — implemented `resume`/`cancel`/`getStatus` + `approve`/`reject`/`listReviews`; `WorkflowController` [M] — `GET /reviews`, `POST /{id}/approve|reject`; `WorkflowGatewayService` [M].
- **dashboard** — `controller/ReviewController` [N] — `GET /api/dashboard/reviews` (reads the shared-DB queue) + proxies approve/reject to the gateway, forwarding the bearer token.
- **UI** — `services/reviewService.ts`, `pages/HumanReviewPage.tsx` [N]; `AppRoutes.tsx` (+`/reviews`) and `DashboardLayout.tsx` (+ nav) [M].
- **DB** — `V14__human_reviews.sql` [N]. Reuses `WorkflowStatus.PAUSED`; no enum change.

**Validation (JDK 25 / Maven / Node):**
- orchestration compiles + installs; **orchestration tests 35 run / 0 fail / 4 quarantined-skipped** (refactor no regression).
- gateway and dashboard **compile** with the new resume/approval wiring.
- UI **`npm run build`** (tsc+vite) green — after fixing a `verbatimModuleSyntax` type-only-import error on `HumanReview`; lint only pre-existing warnings; **tests 2/2**.
- **Not executed here:** the full live loop (start → pause → dashboard list → approve → gateway resume) — needs both apps running against Postgres plus an agent reporting a genuinely low confidence (all report 0.95/0.0 today, so a pause won't trigger without FI-AI1-A). The dashboard→gateway proxy is a real inter-service HTTP call only exercisable with both services up.

**Known limitations (by design):** paused runs are lost on gateway restart (in-memory); single-gateway assumption; approval authenticated-only (no role gate); pull/poll (no notification). See FI-AI2-A..D.

---

## Appendix — Future Ideas (Outside Current Roadmap)

- **FI-AI2-A — Durable resume** (persist workflow state) so paused runs survive restarts / any JVM — converges with SCALE-1/2.
- **FI-AI2-B — Role-based approver authority** once user↔role RBAC exists (post SEC-1 follow-up).
- **FI-AI2-C — Notify reviewers** on pause (ENT-2 / Category M) instead of polling.
- **FI-AI2-D — Approval audit** feeding GOV-1 (the `HumanReviewEntity` is already an audit seed).

---

## Compliance Checklist

| Rule | Status |
|---|---|
| Roadmap not modified | ✅ AI-2 metadata untouched |
| No new roadmap items/categories/modules | ✅ fills stubs + a review entity/UI |
| Dependency rules | ✅ gateway/dashboard depend on orchestration; no new upward deps |
| ADR-004 (edge) / ADR-010 (gate) | ✅ approval on the gateway; consumes the gate's HUMAN_REVIEW |
| Out-of-scope untouched | ✅ durable resume / notifications / role-RBAC deferred |

---

## Document Completion Status

**Status:** Implemented — 2026-07-22 (F1 in-memory, F2 dashboard-proxy; see [Implementation Outcome](#implementation-outcome))
**Version:** 1.0.0
**Implements:** `AI-2` (roadmap v2.2.0, frozen)
**Next step:** On approval + F1/F2 choices, execute [§7](#7-step-by-step-implementation-plan) from Step 1. No code until approved.
