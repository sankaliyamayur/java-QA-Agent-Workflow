# BRAIN-1 — Staged QA Brain evolution · Technical Design

**Item:** BRAIN-1 (un-deferred at user request 2026-08-02) — the QA Brain's staged autonomy model.
**Status:** design (awaiting decision + implement approval).
**Roadmap framing:** "Six stages → Autonomous Brain · **Vision (2027→2030) · v2.x→v3.x**".

---

## 0. Grounding — honest scope

The QA Brain is real and substantial (`ai-qa-os-brain`): `DecisionEngine` + `RuleBasedDecisionStrategy` / `LlmBasedDecisionStrategy` / `HybridDecisionStrategy`, `ConfidencePolicyManager`, `ReasoningEngine`, `IntentAnalyzer`, `LearningEngine` / `FeedbackProcessor`, `QAPlanner` / `TestStrategyPlanner`, `BrainManagerImpl`. What BRAIN-1 asks for is not a new brain feature — it's a **staged evolution model**: a maturity ladder ("six stages → autonomous") describing how the brain progresses toward autonomy, and where it stands today. This is a **maturity-framework / architecture** deliverable (like PLG-4's "marketplace architecture"), tiered Vision 2027→2030 — the multi-year evolution itself is out of scope for a bounded slice.

What's faithful and buildable now: a **maturity model** (the six stages + the capability that characterises each) and a **current-stage assessor** that inventories which brain capabilities genuinely exist and reports the attained stage — plus recording the **evolution architecture** in the ADR. Like GOV-2, this is **self-assessment** (a factual capability inventory), clearly labelled, not a claim of autonomy.

---

## 0.1 / Decision for approval — how much of BRAIN-1

- **Option A (recommended) — maturity model + stage assessor + architecture ADR.**
  A `BrainMaturityStage` ladder (0–5: Assisted → Advisory → Supervised → Adaptive → Orchestrated → Autonomous) with per-stage capability criteria; a `BrainStageAssessor` that reports each capability's presence and the **attained stage** (highest stage whose required capabilities all exist); and ADR-082 recording the six-stage evolution architecture. Fully unit-testable; honest self-assessment. → **BRAIN-1 In Progress.** → **ADR-082.**

- **Option B — architecture ADR only (no code).**
  Just the six-stage evolution design. Honest, but no tangible artifact; BRAIN-1 In Progress on design.

**Recommendation: Option A** — a tangible, testable maturity model + honest current-stage assessment, plus the architecture, while being explicit that the standalone multi-year evolution (and true autonomy) is v3.x and **not** built (BRAIN-1 **In Progress**, not Completed).

---

## 1. Technical Design (Option A) — `ai-qa-os-brain`, `com.aiqaos.brain.maturity`

### 1.1 The six-stage ladder (`BrainMaturityStage` enum)
| Stage | Name | Characterising capability |
|---|---|---|
| 0 | ASSISTED | rule-based decisions (`RuleBasedDecisionStrategy`) |
| 1 | ADVISORY | LLM-assisted recommendations behind a confidence gate (`LlmBasedDecisionStrategy` + `ConfidencePolicyManager`) |
| 2 | SUPERVISED | auto-apply within confidence thresholds, humans review exceptions (`HybridDecisionStrategy` + HEAL-2 approval) |
| 3 | ADAPTIVE | learns from feedback (`LearningEngine` / `FeedbackProcessor`, LRN-1) |
| 4 | ORCHESTRATED | self-directed planning across workflows (`QAPlanner` / `TestStrategyPlanner`) |
| 5 | AUTONOMOUS | fully self-governing brain under governance guardrails (vision) |

### 1.2 `BrainMaturityStageDescriptor` (record)
`stage`, `name`, `summary`, `requiredCapabilities` (`Set<String>` of capability keys).

### 1.3 `BrainStageAssessor` (`@Component`)
- Takes the **present capabilities** (a `Set<String>` of capability keys the runtime reports — supplied at assessment; the default set reflects the shipped brain components).
- `assess(Set<String> present)` → `BrainMaturityReport`: for each stage, whether its `requiredCapabilities` are all present; the **attained stage** = the highest *contiguous* stage fully satisfied from 0; a self-assessment disclaimer; per-stage satisfied flags + missing capabilities.
- Contiguous rule: a later stage doesn't count as attained if an earlier one is unmet (maturity is cumulative) — honest, avoids "we skipped to stage 5".

### 1.4 Architecture (ADR-082, recorded — not built)
The multi-year path: each stage's graduation criteria, the governance guardrails (GOV-3 policy, ENT-3 budgets, confidence gates) that must tighten as autonomy rises, and the human-oversight model that must persist through stage 5. This slice delivers the model + honest current assessment it all hangs on.

## 2. Testing (Mockito-free)
- **`BrainStageAssessorTest`** — full capability set → attained AUTONOMOUS; only rule-based present → ASSISTED; a gap in an early stage caps the attained stage (contiguity) even if a later stage's capabilities exist; missing-capabilities reported; empty set → below stage 0 (nothing attained).
- **`BrainMaturityStageTest`** — the ladder is ordered 0–5 and every stage has criteria (stage 5 may be aspirational).
- Full reactor `mvn clean test` green (21 modules); additive.

## 3. What can't be validated here
The actual multi-year evolution and true autonomy (v3.x). The maturity model + assessment logic is fully unit-verified; the *attained stage* is a self-assessment over declared capabilities.

## 4. Implementation plan
1. `BrainMaturityStage`, `BrainMaturityStageDescriptor`, `BrainMaturityModel` (the curated ladder), `BrainMaturityReport`, `BrainStageAssessor`.
2. `BrainStageAssessorTest`, `BrainMaturityStageTest`.
3. Full reactor verify.
4. Docs: ADR-082 (model + evolution architecture), tracker BRAIN-1 (Deferred → In Progress) + counts, this doc's Implementation Outcome.

## 5. Follow-on
- FI-BRAIN1-A: wire the assessor to a real capability inventory (which brain beans/flags are active) instead of a supplied set; a `GET /api/governance/brain-maturity` endpoint + dashboard tile.
- FI-BRAIN1-B: per-stage graduation gates enforced (a stage can't be "entered" until its governance guardrails are configured).

---

## Implementation Outcome

**Delivered 2026-08-02 (Option A / ADR-082). Full reactor green — 21 modules, 0 failures. BRAIN-1 → In Progress (model + architecture; multi-year evolution is v3.x).**

Shipped as designed (`ai-qa-os-brain`, `com.aiqaos.brain.maturity`):
- **`BrainMaturityStage`** (enum, 6 cumulative stages 0–5), **`BrainMaturityModel`** (`@Component`, stage→capability keys), **`BrainMaturityReport`** + `StageAssessment`, **`BrainStageAssessor`** (`@Component`, cumulative attained-stage + gaps, self-assessment disclaimer).

**Tests:** `BrainStageAssessorTest` 6/6 — full caps → AUTONOMOUS; rule-only → ASSISTED; empty/null → none attained; **early-stage gap caps the attained stage despite later capabilities present** (contiguity); missing capabilities reported per stage; six ordered stages. Full reactor green (21 modules); additive.

**Deviations:** none. `attainedStage` is nullable (null = below ASSISTED).

**Scope honesty:** the attained stage is self-assessment over declared capabilities (present = the component exists); true autonomy + the multi-year evolution are v3.x. So **BRAIN-1 stays In Progress**.

**Follow-on:** FI-BRAIN1-A (wire the assessor to a real runtime capability inventory + a `/api/governance/brain-maturity` endpoint & dashboard tile), FI-BRAIN1-B (enforced per-stage graduation gates).
