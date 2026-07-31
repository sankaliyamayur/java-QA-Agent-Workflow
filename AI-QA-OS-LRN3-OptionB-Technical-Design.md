# LRN-3 (Option B) — Technical Design: Persist Learning Observations

**Item:** LRN-3 (Learning dashboard) — **In Progress**. This designs the data source deferred by ADR-062.
**Status:** Draft — awaiting decision + go-ahead (no code until approved).
**Date:** 2026-07-30.

> **What grounding revealed.** "Loop snapshot persistence" is a misnomer — there is **no loop** producing metrics and **no source** of the signal the read-model needs. `LearningMetricsCalculator.compute` has **zero callers**; `LearningObservation` has **zero producers**; and no persisted table carries a faithful **`success` (boolean) + `confidence` (double)** per run (`brain_learning` has neither; `brain_decisions` has confidence but not run-success). So Option B is really **"build the observation pipeline"** — a producer that captures success + confidence at run completion and persists it. The read-model (`LearningDashboardAssembler`) and calculator are already built and waiting.

---

## 0. Grounding + the honest scope
- **Built & waiting:** `LearningObservation` (sequence/success/confidence/label), `LearningMetricsCalculator.compute(observations) → LearningMetrics`, `LearningDashboardAssembler.assemble(metrics) → LearningDashboardView` (health + headline). All pure, unit-tested.
- **Missing (the whole gap):** (1) a **producer** that emits an observation when a QA run finishes — success from the run's pass/fail, confidence from the AI-1 gate; (2) a **persistent store** for observations; (3) the **read path** feeding the dashboard.
- **Seam-discipline note:** persistence + a recorder with **no producer** would be scaffolding that shows an empty dashboard forever. Option B is only worth building **with** the producer wired.

### 0.1 Where success + confidence converge
The one place both signals exist together is the **orchestration run pipeline** (`AutonomousQAPipelineOrchestrator`) — it runs the AI-1 confidence gate (a `confidence`) and drives execution (a pass/fail `success`) per run. That is the producer's hook point (exact call site verified at implement).

### 0.2 / Decision for approval — build the pipeline now, or keep LRN-3 deferred?

| Option | Scope | Trade-off |
|---|---|---|
| **B-full — build the observation pipeline (recommended only if LRN-3 is a priority)** | `LearningObservationEntity` + repo + migration; `LearningObservationRecorder` (learning); **wire the orchestration run-completion to record `(success, confidence)`**; dashboard read path (`dashboard → learning`) → calculator → assembler → view + `GET /api/dashboard/learning`; React UI. | Delivers a real, faithful learning dashboard fed by actual runs. But it's a **cross-module feature** (touches orchestration), and its value is only visible after real runs accumulate observations (user-run) — a larger effort than the other three dashboards. |
| **B-defer — keep LRN-3 In Progress; ship this design (recommended)** | Do **not** build a producerless half-pipeline. Record this design so the pipeline can be built when the learning loop is genuinely integrated (or when LRN-3 is prioritised). | Honest to the seam-discipline rule (no scaffolding without a producer); avoids an always-empty dashboard. LRN-3's read-model/assembler stay ready. Costs: the LRN-3 dashboard remains unshipped. |

**Recommendation: B-defer.** The read-model is ready; what's missing is a real *producer*, and wiring one into the orchestration run pipeline is a feature in its own right — best done when the learning loop is actually integrated, not as a dashboard afterthought that would otherwise show nothing. If LRN-3 is a priority, B-full is the concrete plan below.

> ✅ **Decision (confirmed 2026-07-30): B-defer** — keep LRN-3 In Progress with this design on record; do not build a producerless half-pipeline; build B-full when the learning loop is integrated / LRN-3 is prioritised. Recorded as **ADR-063**.

---

## 1. B-full design (if chosen)

### 1.1 Persistence (`learning`)
- `LearningObservationEntity` (`learning_observations`): `sequence` (long, epoch millis), `success` (boolean), `confidence` (double), `label` (String), `@TenantId tenant_id` (tenant-owned). `LearningObservationRepository extends JpaRepository`.
- Migration **V22** — create `learning_observations` (+ `tenant_id` index).

### 1.2 Producer (the missing piece)
- `LearningObservationRecorder` (`learning`, `@Component`): `record(boolean success, double confidence, String label)` → persists an observation stamped with the current tenant + sequence.
- **Wire the orchestration run pipeline** (`AutonomousQAPipelineOrchestrator`, after the AI-1 gate + execution) to call the recorder with the run's outcome + gate confidence. This is the crux — no producer, no data.

### 1.3 Read path (`dashboard → learning`)
- `LearningDashboardService` (dashboard): `LearningObservationRepository.findAll()` → `List<LearningObservation>` → `LearningMetricsCalculator.compute` → `LearningDashboardAssembler.assemble` → `LearningDashboardView`.
- `LearningDashboardController`: `GET /api/dashboard/learning`.
- React: `learningService.ts` + `LearningDashboardPage.tsx` (score/successRate/avgConfidence + confidence sparkline + trend/health headline + sampleCount) + route + nav.

### 1.4 Faithfulness
`success` = the run's actual pass/fail; `confidence` = the actual AI-1 gate value — **real signals, no fabrication** (the whole reason for B over aggregating `brain_learning`).

## 2–6. Changes / plan (B-full)
Entity + repo + V22 + recorder + orchestration hook + dashboard service/controller + UI + tests (recorder persists; read-path aggregation; reactor green). Cross-tenant + real-data behaviour is user-run.

---

## Implementation Outcome
*(filled at implementation — or left open if B-defer is chosen.)*
