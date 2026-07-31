# LRN-3 / PE-3 — Technical Design: Exposing the Dashboard Read-Models

**Items:** LRN-3 (Learning dashboard) + PE-3 (Prompt quality dashboard) — both **In Progress** (backend read-model + assembler done; UI + endpoint deferred).
**Status:** Draft — awaiting decision + go-ahead (no code until approved).
**Date:** 2026-07-30.

> **The task reframed by grounding.** "Endpoint hosting" turned out to be the *small* part. Unlike ENT-4 (user repo) and HEAL-3 (healing-metric repo) — whose read-models are a simple `repo.findAll() → assembler` — **LRN-3 and PE-3 have no directly-queryable source**: their assemblers consume *computed* inputs. The real design decision is **where the data comes from**.

---

## 0. Grounding — why these two are different

| | Assembler input | Source today | On-demand queryable? |
|---|---|---|---|
| **ENT-4** | `List<UserEntity>` | `UserRepository.findAll()` | ✅ yes (done) |
| **HEAL-3** | `List<HealingMetricEntity>` | `HealingMetricRepository` | ✅ yes (done) |
| **LRN-3** | `LearningMetrics` ← `LearningMetricsCalculator.compute(List<LearningObservation>)` | **`LearningObservation` has no store** — observations are transient, fed by the live learning loop | ❌ no |
| **PE-3** | `List<LeaderboardEntry>` ← `PromptLeaderboard.rank(versionIds, suites)` | ranking **runs benchmarks** (LLM calls) — there is no persisted leaderboard | ❌ not directly |

**But persisted data that could be *aggregated* into the inputs does exist:**
- **PE-3:** `EvaluationResultRepository` (`eval_results`) holds per-version/suite scores (MOD-3 persists them). A leaderboard can be **aggregated from persisted results** — no benchmark re-run.
- **LRN-3:** `LearningRepository` (`brain_learning`) holds recorded learning entries. `LearningObservation`s can be **derived from those entries** (score/success/timestamp), then fed to the calculator.

### 0.1 Hosting (the easy, settled part)
Whichever option, the endpoint lives in the **dashboard app** (consistent with ENT-4/HEAL-3, keeps the `/api/dashboard/*` convention the UI already targets). This requires adding **`dashboard → eval`** (PE-3) and **`dashboard → brain`** (LRN-3) dependencies — apps depending on feature modules is allowed (only the reverse is forbidden). Thin `@RestController` + a small dashboard service that does the aggregation → assembler.

### 0.2 / Decision for approval — the data-source strategy (ADR decision)

| Option | Approach | Trade-off |
|---|---|---|
| **A — Aggregate from existing persisted data (recommended)** | PE-3: dashboard service aggregates `EvaluationResultRepository` → per-version scores → `LeaderboardEntry`s → `PromptQualityAssembler`. LRN-3: aggregates `LearningRepository` entries → `LearningObservation`s → `LearningMetricsCalculator` → `LearningDashboardAssembler`. | **No loop/harness changes; validatable now** (the aggregation is pure, unit-testable). Reuses data already persisted (and now tenant-scoped). The new work is the two aggregation mappings (results→leaderboard, learning-entries→observations), which must be defined faithfully. |
| **B — Snapshot-persistence in the loops** | The eval harness writes a ready **leaderboard snapshot** after each run; the learning loop writes a **metrics snapshot** each cycle. Dashboard reads the latest snapshot. | The clean write-model/read-model split (dashboard reads pre-computed rows), but **modifies the loops**, adds two snapshot tables + migrations, and only populates when the loops actually run — larger, and the dashboard shows nothing until a loop has executed. |

**Recommendation: A.** Aggregate from the data that already exists — it needs no loop changes, is unit-testable here, and gives the dashboard something real to show immediately. Snapshot-persistence (B) is a reasonable later optimisation if the aggregation proves expensive.

> ✅ **Decision (confirmed 2026-07-30): Option A** — dashboard-hosted endpoints that aggregate persisted `eval_results` (PE-3) and `brain_learning` (LRN-3) into the assembler inputs; no loop changes. To be recorded as **ADR-062** (number verified at implement).

---

## 1. Technical Design (Option A)

### 1.1 PE-3 (dashboard, `→ eval`)
- `PromptQualityService` (dashboard): inject `EvaluationResultRepository` + `PromptQualityAssembler`. `getSummary()` = aggregate results → group by version → mean score → ranked `LeaderboardEntry`s → `assembler.summarize(entries)`.
- `PromptQualityController`: `GET /api/dashboard/prompt-quality` → `PromptQualitySummary`.

### 1.2 LRN-3 (dashboard, `→ brain`)
- `LearningDashboardService` (dashboard): inject `LearningRepository` + `LearningMetricsCalculator` + `LearningDashboardAssembler`. `getView()` = map `LearningEntity` rows → `LearningObservation`s → `calculator.compute(...)` → `assembler.assemble(metrics)`.
- `LearningDashboardController`: `GET /api/dashboard/learning` → `LearningDashboardView`.

### 1.3 Faithfulness caveat
The two mappings (eval-results → per-version leaderboard; learning-entries → observations) must reproduce what the live sources would compute. Where a field the assembler expects isn't persisted, the aggregation approximates it — called out in the ADR + tests, and revisited under Option B if it drifts.

## 2. Module / class changes
- `ai-qa-os-dashboard/pom.xml` — add `ai-qa-os-eval`, `ai-qa-os-brain` deps.
- `dashboard`: `PromptQualityService` + `PromptQualityController`; `LearningDashboardService` + `LearningDashboardController`.
- React (not runnable here): `promptQualityService.ts` + `PromptQualityPage.tsx`; `learningService.ts` + `LearningDashboardPage.tsx`; routes + nav.

## 3–5. DB / API
No migration (reads existing tables). New GETs: `/api/dashboard/prompt-quality`, `/api/dashboard/learning`.

## 6. Implementation plan
1. PE-3 service (aggregation) + controller + unit test (aggregation faithfulness). *Validatable here.*
2. LRN-3 service (mapping) + controller + unit test. *Validatable here.*
3. Full reactor green (new dashboard deps — watch context wiring). *Validatable here.*
4. React pages + routes + nav (convention-matched; `npm run build`/`dev` on user side).
5. Docs — tracker LRN-3/PE-3 notes, **ADR-062**, this doc's Outcome.

**Definition of Done:** both read-models are served from persisted data via dashboard endpoints + scaffolded UIs; aggregation unit-proven; reactor green. LRN-3/PE-3 stay **In Progress** only for the live-data E2E (needs a real DB with loop/harness output).

**Honest boundary:** aggregation logic is provable here; whether it matches live-loop output is confirmed only against a real DB with recorded learning/eval data (user-run). If the aggregation proves unfaithful, Option B (snapshot-persistence) is the fallback.

---

## Implementation Outcome

**Implemented 2026-07-30 (Option A). Recorded as ADR-062. PE-3 done; LRN-3 deferred (data gap).**

**PE-3 — done + validated:** `dashboard → eval` dependency; `PromptQualityService` aggregates `EvaluationResultRepository` (`eval_results`) → mean `score` per `promptVersion` → ranked `LeaderboardEntry`s → `PromptQualityAssembler`; `PromptQualityController` `GET /api/dashboard/prompt-quality`. `PromptQualityServiceTest` **2/2** (mean-per-version + ranking; empty case), dashboard `@SpringBootTest` green, **build success**. Read-only React UI: `promptQualityService.ts` + `PromptQualityPage.tsx` (5 MetricCards + leaderboard table) + `/prompt-quality` route + nav link (not compiled here — `npm run build`/`dev` to verify).

**LRN-3 — deferred, honestly:** Option A is **not faithful** for LRN-3. `LearningEntity` (`brain_learning`) carries only `pattern`/`previousDecision`/`result`/`improvement` strings — it **lacks the `success` (boolean) and `confidence` (double)** that `LearningObservation` requires. Deriving them would fabricate the confidence signal → a misleading dashboard. So LRN-3 needs **Option B** (the learning loop persisting a faithful metrics/observation snapshot) and was **not built**. Recorded in ADR-062's imposed rule: never fabricate a missing signal to fill a read-model.

**Honest boundary:** PE-3 aggregation is unit-proven; whether it matches live eval output is confirmed only against a real DB with recorded results (user-run).
