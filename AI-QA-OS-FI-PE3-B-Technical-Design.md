# FI-PE3-B — Prompt regression detection (dashboard read-model) · Technical Design

**Item:** PE-3 (Prompt quality dashboard) → sub-item **FI-PE3-B** — regression detection.
**Status:** design (awaiting decision + implement approval).

---

## 0. Grounding + scope

### 0.1 What exists
- **Read-model (PE-3/FI-PE3-A, done):** `PromptQualityService` (`ai-qa-os-dashboard`) reads persisted `eval_results` (`EvaluationResultEntity`), groups by `promptVersion` → mean score → ranked `LeaderboardEntry`s → pure `PromptQualityAssembler` → `PromptQualitySummary`. Endpoint `GET /api/dashboard/prompt-quality`. React `PromptQualityPage`. No benchmark is re-run (ADR-062).
- **Persisted signal:** `EvaluationResultEntity` carries `promptVersion`, `score`, **`createdTime`** (`LocalDateTime`), `suite`, `caseId`, `passed`, `@TenantId`.
- **CI-time regression harness (separate concern):** `PromptRegressionHarness` + per-**case** file `Baseline`/`BaselineStore` (`golden/<suite>.baseline.json`, ADR-013) compares a *fresh run* against the accepted baseline at benchmark/CI time. It is **not** a dashboard read-model and its baseline is per-case, a different granularity than the dashboard's per-version means — joining them would be a fabricated comparison (ADR-063). FI-PE3-B is a *read-model* signal, not a re-run.

### 0.2 What this slice delivers
A **faithful, read-only regression signal** over the already-persisted `eval_results`: which prompt versions have **gotten worse over time** — the same version's recent evaluation scores dropping materially below its earlier scores. Exposed as a new read-model endpoint alongside the existing summary; the React page gains a regressions panel.

### 0.3 Faithfulness (ADR-063)
The signal is derived **only** from persisted `score` + `createdTime` grouped by `promptVersion`. A version with too few results to split is reported as **insufficient data — skipped, never flagged** (no fabricated baseline). No new data source, no re-run, no cross-granularity join.

### 0.4 Deferred (out of scope)
- Per-execution history over `PromptExecutionEntity` (FI-PE3-C — separate).
- Alerting/notification on regression (a NotificationEvent hook — future FI).
- Reconciling the CI-time per-case `Baseline` into the dashboard (different granularity; intentionally not joined).

---

## 0.5 / Decision for approval — the regression reference model

Both options are faithful (only `eval_results`), both skip insufficient-data versions, both are pure/unit-tested. The fork is **what a version is compared against** — which changes what "regression" *means* on the dashboard.

- **Option A (recommended) — temporal within-version.**
  For each version, order its results by `createdTime`, split at the midpoint into an **earlier** and a **recent** window, and flag it if `recentMean < earlierMean − tolerance`. This is a *true* regression: the **same** prompt scoring worse now than it used to. Uses the `createdTime` signal we have; independent of version-string ordering; needs ≥ `minSamples` results (default 4 → 2 per window), else skipped.

- **Option B — champion-relative.**
  Flag every version whose mean score is below the **leading version's** score by more than `tolerance`. Simpler, but this is "underperforms the best version", **not** "regressed" — a brand-new weak version trips it while a genuinely-declining champion never does. Semantically weaker for a regression panel.

**Recommendation: Option A** — it is genuinely *regression* (temporal decline) and exploits the `createdTime` signal; Option B is a gap-to-best measure better served by the existing leaderboard spread. → **ADR-069.**

---

## 1. Technical Design (Option A)

### 1.1 New (eval module — `com.aiqaos.eval.benchmark`, pure)
- **`PromptRegressionSignal`** (record): `versionId`, `baselineScore` (earlier-window mean), `currentScore` (recent-window mean), `delta` (`current − baseline`, negative = worse), `sampleCount`.
- **`PromptRegressionReport`** (record): `tolerance`, `regressedCount`, `List<PromptRegressionSignal> regressions` (the read-model payload; empty list when none).
- **`PromptRegressionAnalyzer`** (`@Component`, pure — the assembler pattern): `analyze(Map<String versionId, List<Double> chronologicalScores>, double tolerance, int minSamples) → PromptRegressionReport`. Per version: skip if `size < minSamples`; else split at `size/2` (earlier = first half, recent = second half), mean each, emit a signal iff `recentMean < earlierMean − tolerance`. Signals sorted by `delta` ascending (worst first).

### 1.2 Dashboard wiring (`ai-qa-os-dashboard`)
- **`PromptQualityService.getRegressions()`** — `findAll()` → group by `promptVersion` → within each version **sort by `createdTime`** (nulls last, stable) → `List<Double>` scores → `PromptRegressionAnalyzer.analyze(..., tolerance, minSamples)`. Tolerance/minSamples from `@Value` (`aiqaos.eval.regression.tolerance:0.05`, `aiqaos.eval.regression.min-samples:4`).
- **`PromptQualityController`** — add `GET /api/dashboard/prompt-quality/regressions` → `PromptRegressionReport` (read-only, on the existing open dashboard chain; the read-model carries no secrets).

### 1.3 React UI (ai-qa-os-dashboard-ui)
`promptQualityService.ts`: add `PromptRegressionSignal`/`PromptRegressionReport` types + `getPromptRegressions()`. `PromptQualityPage.tsx`: a "Regressions" panel — count headline + a table (version, baseline→current, Δ) with an empty state ("No regressions detected"). Convention-matched; `npm run lint`/`build` verified.

## 2. Testing (honest)
- **`PromptRegressionAnalyzerTest`** (pure): regression detected (recent < earlier − tol), no-regression (stable), improvement (recent > earlier → not flagged), tolerance boundary (drop == tol → not flagged), insufficient-data skip (< minSamples), multiple versions ranked worst-first.
- **`PromptQualityServiceTest`** gains `getRegressions_*`: proxy repo returns timestamped results across versions; asserts the declining version is flagged and the insufficient-data one is skipped; the existing `getSummary` tests are untouched.
- Full reactor `mvn clean test` green (22 modules). The existing `PromptQualitySummary`/`PromptQualityAssembler` and their tests are **unchanged** (new capability added alongside).

## 3. What can't be validated here (user-run)
The live page against a real `eval_results` table with multi-run history (`npm run dev`).

## 4. Implementation plan
1. `PromptRegressionSignal`, `PromptRegressionReport`, `PromptRegressionAnalyzer` (eval) + `PromptRegressionAnalyzerTest`.
2. `PromptQualityService.getRegressions()` + config; `PromptQualityController` endpoint; service test.
3. UI: service types/fn + page panel.
4. Full reactor verify + UI lint/build.
5. Docs: ADR-069, tracker PE-3 note, this doc's Implementation Outcome.

## 5. Follow-on (outside this slice)
- FI-PE3-C: per-execution history over `PromptExecutionEntity`.
- FI-PE3-D: emit a `NotificationEvent` (ENT-2 path) when a regression crosses the threshold.

---

## Implementation Outcome

**Delivered 2026-08-01 (Option A / ADR-069). Full reactor green — 22 modules, 0 failures; UI lint + build clean. PE-3 stays In Progress (FI-PE3-C remains).**

Shipped as designed:
- **`PromptRegressionAnalyzer`** (pure, `ai-qa-os-eval`) + **`PromptRegressionSignal`** / **`PromptRegressionReport`** records — per version, split chronological scores at the midpoint, flag if `recentMean < earlierMean − tolerance`; `< minSamples` versions skipped; signals worst-first.
- **`PromptQualityService.getRegressions()`** — groups `eval_results` by version, sorts by `createdTime` (nulls last), delegates; tolerance/minSamples via `@Value`. **`PromptQualityController`** `GET /api/dashboard/prompt-quality/regressions`.
- **React** `promptQualityService.ts` (+types + `getPromptRegressions()`) and `PromptQualityPage.tsx` (regressions panel: count + tolerance + worst-first table, "No regressions detected" empty state; loads summary + regressions via `Promise.all`).

**Tests:** `PromptRegressionAnalyzerTest` 7/7 (decline flagged, stable/improvement not flagged, tolerance boundary, insufficient-data skip, worst-first ranking, empty/null); `PromptQualityServiceTest` 3/3 (added a `getRegressions` case: declining version flagged, 2-sample version skipped) — the existing `getSummary` tests unchanged. CI-time `PromptRegressionHarnessTest`/`GateTest` untouched and green.

**Deviation:** the tolerance-boundary test exposed a floating-point edge — `0.75 − 0.80` is `-0.05000000000000004`, which tripped a naive `delta < -tolerance`. Fixed by comparing the drop against `tolerance + 1e-9` so a drop of *exactly* tolerance is not flagged. No other deviations.

**Decision confirmed:** Option A (temporal within-version) — a genuine regression signal from the `createdTime` we have; champion-relative rejected as gap-to-best.

**User-run (not validatable in sandbox):** the live page against a real `eval_results` table with multi-run history (`npm run dev`).

**Follow-on:** FI-PE3-C (per-execution history over `PromptExecutionEntity`), FI-PE3-D (emit an ENT-2 `NotificationEvent` when a regression crosses the threshold).
