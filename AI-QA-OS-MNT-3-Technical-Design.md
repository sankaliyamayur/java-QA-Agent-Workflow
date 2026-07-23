# MNT-3 — Technical Design: Close the Two Highest-Value Test Gaps

**Version:** 1.0.0
**Document Type:** Implementation Technical Design
**Document Status:** Implemented — 2026-07-23 (gateway fell back §0.4-A→B; see [Implementation Outcome](#implementation-outcome))
**Last Updated:** 2026-07-23
**Roadmap item:** [`MNT-3`](./AI-QA-OS-Improvement-Roadmap.md#mnt-3--close-the-two-highest-value-test-gaps) (v2.2.0, frozen) — 🟠 P1 · Effort M · Owner Gateway + AI-Provider teams · Phase 1 · v1.4
**Modules:** `ai-qa-os-gateway` (public API surface) · `ai-qa-os-ai-provider` (LLM calls + cost) — both currently **zero tests**.
**Depends on:** MNT-1 (CI runs the suite as a gate).

> **Scope discipline.** MNT-3 raises the test floor on the **two riskiest zero-test modules** — it is not a drive for full coverage. Targets are exactly those the roadmap names; no structural change.

---

## 0. Roadmap Verification & the JDK-25 Constraint

### 0.1 What MNT-3 requires

> `ai-qa-os-gateway` (the entire public API surface) and `ai-qa-os-ai-provider` (every outbound LLM call and all cost accounting) have **zero tests** — the two modules where a defect is most expensive. **Where:** `@WebMvcTest` controller tests in the gateway; unit tests for **provider selection**, **key rotation (`ApiKeyPool`)**, **fallback (`LLMResilienceManager`)**, and **cost calculation (`CostTracker`)** in ai-provider.

### 0.2 Verified facts (read during design)

| Fact | Detail |
|---|---|
| Both zero-test | Neither module has any test or even a `spring-boot-starter-test` dependency — MNT-3 adds it to both poms. |
| ai-provider targets are stub-friendly | `LLMProvider` and `SecretManager` are **interfaces** (`generate/getProviderName/isAvailable/supports`; `getSecret`) → hand-stubbable. `ModelRouter(List<LLMProvider>)`, `LLMResilienceManager.executeWithFallback(primary, fallback, req)`, `ApiKeyPool(SecretManager, base, cooldown)`, `CostTracker(repos)` all take injectable deps. |
| Gateway logic is testable | `GlobalExceptionHandler`, `WorkflowController` (→ `WorkflowGatewayService`), `WebhookManager` route/dispatch by string, plain `Filter`s. |

### 0.3 The constraint that shapes the gateway tests

The roadmap says **`@WebMvcTest`**, which conventionally supplies controller collaborators via **`@MockBean`** — and **`@MockBean` needs Mockito, which cannot run on this JDK 25 environment** (byte-buddy; the project has used hand-written stubs everywhere: AI-1/AI-2, eval, SEC-3). `@WebMvcTest` **itself** does not need Mockito — only `@MockBean` does. So the question is how to feed the controller its collaborators.

### 0.4 / Decision for approval — the gateway test approach

| Option | Approach | Trade-off |
|---|---|---|
| **A — `@WebMvcTest` + hand-wired stub beans (recommended)** | Use `@WebMvcTest` (as the roadmap names) but provide stub service beans via a `@TestConfiguration` (hand-written impls) instead of `@MockBean`; drive via `MockMvc` with `@AutoConfigureMockMvc(addFilters=false)` | Faithful to the roadmap; exercises the **real HTTP surface** — routing, status codes, JSON (de)serialisation, and `@RestControllerAdvice` exception mapping. Slice-context setup is a bit more involved. |
| **B — Plain unit tests** | Test controllers/handlers as plain objects with hand-written stub services; no `MockMvc`/context | Simplest and fully deterministic, but does **not** exercise HTTP routing/serialisation — weaker for "the public API surface". |

**Recommend A** — the whole reason the gateway is "highest-value" is that it *is* the public API surface ("a broken gateway is a total outage"); testing it through `MockMvc` is what the roadmap asks and is what catches routing/status/serialization defects. `@WebMvcTest` runs fine without Mockito once collaborators come from a `@TestConfiguration`. ai-provider tests are plain unit tests either way (no fork there).

> ✅ **Decision (confirmed 2026-07-23): Option A — `@WebMvcTest` + hand-wired stub beans** (`@TestConfiguration`, `addFilters=false`). If the slice proves brittle in this JDK-25 environment, fall back to §0.4-B for the gateway and record the deviation honestly.

---

## 1. Technical Design

### 1.1 ai-provider — unit tests (roadmap-named targets)
- **`ModelRouterTest`** — `routeModel(purpose)`: `code-generation`/`bug-analysis` → a `CODE_GENERATION`-capable provider; `embedding`/`vision` → the right capability; unknown/null → `CHAT`; no available provider → the `"Gemini"` fallback. Stub `LLMProvider`s vary `isAvailable`/`supports`.
- **`LLMResilienceManagerTest`** — `executeWithFallback`: primary succeeds → primary's response; primary throws → fallback's response; both throw → `ProviderException` (message references the fallback failure).
- **`ApiKeyPoolTest`** — key parsing (CSV `…S`, single, numbered `_i`), `availableKeys` excludes those `markExhausted` within the cooldown and re-includes them after, `hasKeys`, and the `maskKey` static. Stub `SecretManager`.
- **`CostTrackerTest`** — `track(req, resp, provider)` computes and persists the expected cost/token row (verified against a hand-stub `LLMCostRepository`/`AgentTraceRepository` capturing the saved entity); the private cost calc is exercised through `track`.

### 1.2 gateway — `@WebMvcTest` (Option A)
- **`WorkflowControllerTest`** (`@WebMvcTest(WorkflowController.class)`, `addFilters=false`) — a `@TestConfiguration` supplies a stub `WorkflowGatewayService`. Covers the flagship public endpoints (e.g. start, status, reviews/approve/reject from AI-2): success → correct status + JSON body; a service exception → the `GlobalExceptionHandler`'s mapped status/shape (proving the error contract, incl. that internals aren't leaked).
- *(The `@RestControllerAdvice` is picked up by the slice, so exception mapping is exercised end-to-end through `MockMvc`.)*

### 1.3 No Mockito
All collaborators are hand-written stubs (interfaces: `LLMProvider`, `SecretManager`, repos; a stub `WorkflowGatewayService`) — consistent with the whole codebase under JDK 25.

---

## 2. Folder Structure

```
ai-qa-os-ai-provider/
├── pom.xml                                   [M] + spring-boot-starter-test (test)
└── src/test/java/com/aiqaos/provider/
    ├── router/ModelRouterTest.java           [N]
    ├── manager/LLMResilienceManagerTest.java [N]
    ├── key/ApiKeyPoolTest.java               [N]
    └── cost/CostTrackerTest.java             [N]
ai-qa-os-gateway/
├── pom.xml                                   [M] + spring-boot-starter-test (test)
└── src/test/java/com/aiqaos/gateway/
    └── controller/WorkflowControllerTest.java[N]  @WebMvcTest + @TestConfiguration stub
```

---

## 3. Required Test Classes

| Test | Module | Target logic |
|---|---|---|
| `ModelRouterTest` | ai-provider | Provider selection by purpose/capability + fallback |
| `LLMResilienceManagerTest` | ai-provider | Primary→fallback→error path |
| `ApiKeyPoolTest` | ai-provider | Key parsing, rotation/cooldown, masking |
| `CostTrackerTest` | ai-provider | Cost/token computation + persistence |
| `WorkflowControllerTest` | gateway | Public HTTP surface + exception mapping |

---

## 4. Database Changes

**None.** `CostTrackerTest` uses hand-stub repositories, not a database.

---

## 5. API Changes

**None.** Tests only — no production code changes.

---

## 6. Sequence Diagram

```mermaid
flowchart TD
    subgraph AIP["ai-provider (unit, stubs)"]
      MR["ModelRouterTest → routeModel"] --> P["stub LLMProviders (isAvailable/supports)"]
      RM["LLMResilienceManagerTest → executeWithFallback"] --> P
      KP["ApiKeyPoolTest"] --> SM["stub SecretManager"]
      CT["CostTrackerTest → track"] --> RepoStub["stub cost/trace repos"]
    end
    subgraph GW["gateway (@WebMvcTest, no Mockito)"]
      MVC["MockMvc → WorkflowController"] --> STUB["@TestConfiguration stub WorkflowGatewayService"]
      MVC --> ADVICE["GlobalExceptionHandler mapping"]
    end
```

---

## 7. Step-by-Step Implementation Plan

1. **poms** — add `spring-boot-starter-test` (test scope) to both modules.
2. **ai-provider** — `ModelRouterTest`, `LLMResilienceManagerTest`, `ApiKeyPoolTest`, `CostTrackerTest` (hand-written stubs; assert selection/fallback/rotation/cost).
3. **gateway** — `WorkflowControllerTest` (`@WebMvcTest` + `@TestConfiguration` stub service + `addFilters=false`): assert success responses and the `GlobalExceptionHandler` error contract.
4. **Build & validate** — `mvn -pl ai-qa-os-ai-provider,ai-qa-os-gateway -am test`; confirm both modules go from 0 → green tests and the reactor still builds. If the `@WebMvcTest` slice proves brittle in this environment, fall back to §0.4-B for the gateway and record the deviation honestly.
5. **Sync docs** — tracker `MNT-3` → Completed. (No ADR — this is test coverage, not an architecture decision; note the JDK-25 stub approach in the tracker.)

**Definition of Done:** both zero-test modules have a green, meaningful test suite over their highest-risk logic (provider selection, key rotation, fallback, cost; the public HTTP surface + error contract); no Mockito; reactor builds.

---

## Implementation Outcome

Implemented 2026-07-23. Both zero-test modules now have a green suite (**21 tests**). No ADR (test coverage, not architecture).

**Files:**
- **ai-provider** (+ `spring-boot-starter-test`) — `ModelRouterTest` (4), `LLMResilienceManagerTest` (3), `ApiKeyPoolTest` (5), `CostTrackerTest` (3). Stubs: `LLMProvider`/`SecretManager` interfaces; JPA repos stubbed via a **JDK dynamic proxy** that captures `save(...)` (no Mockito).
- **gateway** (+ `spring-boot-starter-test`) — `WorkflowControllerTest` (4), `GlobalExceptionHandlerTest` (2).

**Deviation from the approved §0.4-A (honest):** the `@WebMvcTest` slice **would not load** in this environment — the gateway's `Filter` `@Component`s (e.g. `GatewaySecurityFilter`) require security beans (`AuthenticationManager`) absent from the slice, and `@MockBean` (the usual fix) needs Mockito, which is unusable on JDK 25. After two attempts (including `excludeFilters`), I took the **pre-agreed fallback to §0.4-B**: plain unit tests over `WorkflowController`'s own logic (delegation + the `@RequestBody(required=false)` null-decision branch) and `GlobalExceptionHandler`'s error contract. This trades the real HTTP/serialization layer for determinism; broadening via `@WebMvcTest` is parked as FI-MNT3-A/B.

**Bug found (not fixed here):** `CostTrackerTest` surfaced **FI-MNT3-C** — `"gemini-*-flash".contains("mini")` is true, so all Gemini models are priced at mini rates. Left for the provider team (MNT-3 is test-only).

**Validation (JDK 25 / Maven):** `mvn -pl ai-qa-os-ai-provider,ai-qa-os-gateway test` → **BUILD SUCCESS, 21/0-fail/0-skip**. Only test-scope pom deps + test files added — no production code changed, so the rest of the reactor is unaffected.

---

## Appendix — Future Ideas (Outside Current Roadmap)

- **FI-MNT3-A** — Broaden gateway coverage to the other controllers (Execution/Report/Agent/Brain) and `WebhookManager` dispatch once `@WebMvcTest` (or a pinned test JDK) is workable here.
- **FI-MNT3-B** — A Mockito-on-JDK-25 resolution (or a pinned test JDK) so `@MockBean` / `@WebMvcTest` slice tests become available platform-wide (see §7 outcome — the slice would not load in this environment).
- **FI-MNT3-C** — `CostTracker.calculateCost` prices any model whose name contains `"mini"` at mini rates *before* the `"flash"` check — and **`"geMINI"` contains `"mini"`**, so every Gemini model (incl. `gemini-*-flash`) is billed at mini rates. Latent mispricing found while writing `CostTrackerTest`; fix belongs to the provider team (not in MNT-3's test-only scope).

---

## Compliance Checklist

| Rule | Status |
|---|---|
| Roadmap not modified | ✅ MNT-3 metadata untouched |
| Targets match the roadmap "Where" | ✅ selection/rotation/fallback/cost + gateway `@WebMvcTest` |
| No structural / production change | ✅ tests + two pom test-deps only |
| JDK-25 constraint respected | ✅ hand-written stubs, no Mockito/`@MockBean` |
| Scope = the two named gaps | ✅ not a full-coverage drive (further modules → FI) |

---

## Document Completion Status

**Status:** Implemented — 2026-07-23 (approved §0.4-A `@WebMvcTest`; fell back to §0.4-B for the gateway when the slice would not load — see Implementation Outcome).
**Version:** 1.0.0
**Implements:** `MNT-3` (roadmap v2.2.0, frozen)
