# AI-6 — Context-window & cost budgeting per workflow · Technical Design

**Item:** AI-6 (un-deferred at user request 2026-08-02) — per-workflow token/context budgeting.
**Status:** design (awaiting decision + implement approval).
**Builds on:** ENT-3 (ADR-025) — the cost-quota foundation this mirrors for tokens.

---

## 0. Grounding + scope

### 0.1 What exists (ENT-3, the pattern to mirror)
- **Cost** budgeting is done: `CostBudgetProperties` (`aiqaos.cost.quota.*` — USD limits, `enabled`/`mode`, global/workflow/agent) → `CostBudgetEnforcer.check(LLMRequest)` → `BudgetVerdict` (soft cap: block once recorded spend ≥ limit) → thrown as `BudgetExceededException` (enforce) or logged (warn) in `LLMProviderManager.generate`.
- `SpendLedger` accumulates **cost** per global/workflow/agent (daily rollover); fed by `CostTracker.track(req, resp, provider)`.
- **The faithful token source already exists:** `LLMResponse.getUsage().getInputTokens()/getOutputTokens()` are the **actual** token counts, and `CostTracker` already sees them per request (keyed by `correlationId` = workflow, `agentType`). So a token budget can be enforced on **real** counts — no estimation, no fabrication (ADR-063).

### 0.2 What AI-6 adds
The cost half is ENT-3; AI-6's novel half is **token / context-window budgeting per workflow** (and per agent / global), the same soft-cap shape but counting **tokens** instead of dollars — so a workflow that consumes too much context/token budget is stopped, independent of dollar cost.

### 0.3 Deferred
- Cross-restart ledger seeding (the ENT-3 FI-ENT3-A gap applies equally).
- Per-tenant token budgets (ENT-1 scope, as with cost).

---

## 0.4 / Decision for approval — how far the context-window guard goes

- **Option A (recommended) — cumulative token budget, real counts only.**
  A `TokenLedger` (mirror of `SpendLedger`) accumulates **actual** tokens (`input+output` from `LLMResponse.usage`) per workflow/agent/global; a `TokenBudgetEnforcer` soft-caps each scope (block once recorded tokens ≥ limit). **Zero estimation** — identical faithfulness to ENT-3. This is the core "budget per workflow."

- **Option B — A + a per-request context-window pre-guard.**
  Also reject a single request whose **estimated input tokens** (≈ `prompt.length()/4`) exceed a configured `max-request-tokens`, pre-empting context-overflow before the call. Useful, but the pre-call figure is an **estimate** (the real input-token count is only known post-response) — clearly labelled as such, not a fabricated data signal, but not exact.

**Recommendation: Option A** — the faithful core enforced on real counts, mirroring ENT-3 one-to-one. The per-request estimated guard (B) is a genuine but fuzzier add; take it only if pre-emptive overflow protection is wanted. → **ADR-075.**

---

## 1. Technical Design (Option A) — all in `ai-qa-os-ai-provider`, `com.aiqaos.provider.cost`

### 1.1 New
- **`TokenLedger`** (`@Component`) — mirror of `SpendLedger`: `record(long tokens, String correlationId, String agentType)`, `workflow(id)`, `agent(type)`, `globalToday()`, daily rollover; package-private clock-supplier ctor as the test seam.
- **`TokenBudgetProperties`** (`@ConfigurationProperties("aiqaos.context.budget")`): `enabled` (default **false**), `mode` (`enforce`/`warn`, `isEnforce()`), `perWorkflowTokens` / `perAgentTokens` / `globalDailyTokens` (`Long`, null = unlimited for that scope). Non-breaking by default.
- **`TokenBudgetEnforcer`** (`@Component`) — mirror of `CostBudgetEnforcer`: `check(LLMRequest)` → `BudgetVerdict` (reused; scope labels `global-daily-tokens` / `per-workflow-tokens` / `per-agent-tokens`), soft cap on the `TokenLedger`; `isEnforce()`.
- **`TokenBudgetExceededException extends ProviderException`** — parallel to `BudgetExceededException`.

### 1.2 Wiring
- **`CostTracker.track`** — after cost, feed the token ledger (optional field, `@Autowired(required=false)`, mirrors `spendLedger`): `if (tokenLedger != null) tokenLedger.record(usage.getInputTokens() + usage.getOutputTokens(), req.getCorrelationId(), req.getAgentType());`.
- **`LLMProviderManager.generate`** — a parallel pre-flight block right after the ENT-3 cost check (optional `@Autowired(required=false) TokenBudgetEnforcer`): on `!allowed`, throw `TokenBudgetExceededException` in enforce mode, else `log.warn`. Same soft-cap semantics; a cache/simulator hit short-circuits above it, unchanged.

### 1.3 Faithfulness
Enforcement reads **only recorded actual token counts** (from `LLMResponse.usage`). A brand-new workflow with no recorded usage is never blocked; the cap trips once its real accumulated tokens reach the limit — exactly ENT-3's soft-cap contract, in tokens.

## 2. Testing (Mockito-free)
- **`TokenLedgerTest`** — record/accumulate per workflow+agent+global; daily rollover via the clock seam; null-key safety.
- **`TokenBudgetEnforcerTest`** — disabled → allow; under limit → allow; workflow/agent/global at-or-over limit → exceeded with the right scope; unset scope → unlimited.
- Full reactor `mvn clean test` green (22 modules); default (`aiqaos.context.budget.enabled=false`) leaves `generate` behaviour unchanged, so no existing ai-provider test shifts.

## 3. What can't be validated here (user-run)
Live token-budget tripping across a real multi-call workflow against a real provider (the unit tests prove the ledger + enforcer logic deterministically).

## 4. Implementation plan
1. `TokenLedger`, `TokenBudgetProperties`, `TokenBudgetEnforcer`, `TokenBudgetExceededException`.
2. Feed `TokenLedger` from `CostTracker`; add the pre-flight block in `LLMProviderManager`.
3. `TokenLedgerTest`, `TokenBudgetEnforcerTest`.
4. Full reactor verify.
5. Docs: ADR-075, tracker AI-6 (un-deferred → done/In Progress), this doc's Implementation Outcome.

## 5. Follow-on
- FI-AI6-A: per-request context-window pre-guard (Option B — estimated input tokens).
- FI-AI6-B: per-tenant token budgets (ENT-1 scope); cross-restart ledger seeding (shared with FI-ENT3-A).

---

## Implementation Outcome

**Delivered 2026-08-02 (Option A / ADR-075). Full reactor green — 22 modules, 0 failures. AI-6 → Completed (un-deferred at user request).**

Shipped as designed (all `ai-qa-os-ai-provider`):
- **`TokenLedger`** (`com.aiqaos.provider.cost`) — mirror of `SpendLedger`; accumulates `long` tokens per workflow/agent/global with daily rollover, clock-supplier test seam.
- **`TokenBudgetProperties`** (`aiqaos.context.budget`) — `enabled` (default false), `mode`, `perWorkflowTokens`/`perAgentTokens`/`globalDailyTokens` (`Long`).
- **`TokenBudgetEnforcer.check(LLMRequest)`** — soft cap over the ledger, reusing `BudgetVerdict` (scopes `global-daily-tokens`/`per-workflow-tokens`/`per-agent-tokens`).
- **`TokenBudgetExceededException`** (parallel to `BudgetExceededException`).
- **`CostTracker`** feeds the `TokenLedger` with `input+output` from `LLMResponse.usage` (optional field, mirrors the `SpendLedger` feed); **`LLMProviderManager.generate`** adds the pre-flight token block right after the ENT-3 cost block (optional enforcer; throw on enforce, warn otherwise).

**Tests:** `TokenLedgerTest` 3/3 (per-scope accumulation, null-key safety, daily rollover via clock seam); `TokenBudgetEnforcerTest` 5/5 (disabled→allow, per-workflow/global/agent exceeded, under-limit + warn mode, unset scope = unlimited). Existing `SpendLedgerTest`/`CostBudgetEnforcerTest` still green. Full reactor green (22 modules); default (`enabled=false`) leaves `generate` unchanged, so no existing ai-provider test shifted.

**Deviations:** none. (`BudgetVerdict` reused for tokens — carries limit/used as doubles; token values fit exactly.)

**Decision confirmed:** Option A (cumulative token budget, real counts) — no estimation. The per-request context pre-guard (Option B) is logged as FI-AI6-A.

**User-run (not validatable in sandbox):** a real multi-call workflow tripping `perWorkflowTokens` against a live provider (the ledger + enforcer are deterministically unit-proven).

**Follow-on:** FI-AI6-A (per-request context-window pre-guard via estimated input tokens), FI-AI6-B (per-tenant token budgets; cross-restart ledger seeding, shared with FI-ENT3-A).
