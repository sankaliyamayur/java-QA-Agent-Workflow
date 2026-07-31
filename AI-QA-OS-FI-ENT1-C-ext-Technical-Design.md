# FI-ENT1-C (extension) — Technical Design: Tenant-Scope the Remaining Tenant-Owned Tables

**Parent item:** ENT-1 — **stays `In Progress`** after this slice.
**Follow-up:** FI-ENT1-C extension. Reuses the ADR-054 `@TenantId` mechanism; no new machinery.
**Status:** Draft — awaiting decision + go-ahead (no code until approved).
**Date:** 2026-07-30.

> **Nature of this slice.** Mechanical repetition of the proven `@TenantId` pattern across more entities. The only real design content is **classification** — which of the remaining entities are tenant-owned (get `@TenantId`) vs platform-global (a shared catalog) vs system telemetry/audit (never tenant-scoped). Once classified, the edits + migration are rote.

---

## 0. Grounding + classification

### 0.1 Already tenant-scoped (11, done in FI-ENT1-C/D/E)
`ModuleEntity`, `TestCaseEntity`, `ExecutionEntity`, `ExecutionStepEntity`, `ExecutionArtifactEntity`, `WorkflowExecutionEntity`, `HumanReviewEntity`, `UserEntity`, `MemoryNodeEntity`, `ConversationHistoryEntity`, `LLMCostEntity`.

### 0.2 Proposed classification of the remaining 33 entities

**A — Tenant-owned → add `@TenantId` (this slice, 16):** per-run / per-tenant operational + result data.
| Module | Entities |
|---|---|
| execution | `ExecutionAuditEntity` |
| orchestration | `WorkflowEntity`, `WorkflowStepEntity` |
| reporting | `ReportEntity`, `ReportArtifactEntity`, `FailureAnalysisEntity`, `TrendEntity` |
| intelligence | `PromptExecutionEntity` |
| agents | `AgentExecutionEntity` |
| agents-runtime | `AgentMessageEntity`, `AgentRuntimeEntity`, `AgentTaskEntity` |
| brain | `DecisionEntity`, `ReasoningTraceEntity`, `LearningEntity` |
| eval | `EvaluationResultEntity` |

**B — Platform-global catalog → stay global (no `@TenantId`):** platform-authored *definitions*, shared across tenants (mirrors ADR-055's "roles are a global catalog").
- intelligence: `PromptTemplateEntity`, `PromptVersionEntity`, `VersionPinEntity` (the QA prompt catalog)
- agents: `AgentEntity` (the AGT-1 roster — platform capabilities)
- security: `RoleEntity`, `PermissionEntity`, `RolePermissionEntity` (already decided global, ADR-055)

**C — System telemetry / audit → stay system-scoped (no `@TenantId`):** platform-operator visibility must span tenants; a discriminator would *hide* cross-tenant rows from operators.
- observability: `AgentMetricEntity`, `AgentTraceEntity`, `AlertEntity`, `BugMetricEntity`, `EventEntity`, `HealingMetricEntity`, `ObservabilityMetricEntity`, `TimelineEventEntity`, `TraceEntity` (LLMCost was scoped in E because it's **billing**-relevant per tenant; the rest is operational telemetry)
- security: `SecurityAuditEntity` (cross-tenant security monitoring)

**Out of scope here (other slices):** security `UserSessionEntity`/`ApiKeyEntity`/`PasswordHistoryEntity` → **FI-ENT1-D slice 2** (RBAC satellites); `core.BaseEntity` is a mapped superclass, not a table.

### 0.3 / Decision for approval — the scoping boundary (ADR decision)

| Option | Boundary | Trade-off |
|---|---|---|
| **A — Conservative (recommended)** | `@TenantId` on the **16 operational/result** entities above; platform **catalogs** (prompts, agent defs, roles) stay global; **observability telemetry + audit** stay system-scoped. | Matches ADR-055 (roles global) and the conventional SaaS split — tenant owns its runtime/result data; platform owns definitions + monitoring. Platform operators keep cross-tenant telemetry visibility. |
| **B — Aggressive** | Also scope prompt/agent **definitions** and **observability telemetry** per tenant. | Maximal isolation, but each tenant must seed its own prompt/agent catalog, and per-tenant telemetry hides platform-wide operational views — a product decision with a large blast radius, much of it debatable. |

**Recommendation: A.** Scope the data a tenant *produces*; keep platform-authored *definitions* and operator *telemetry* shared. Definitions/telemetry can be tenant-scoped later if a product need appears — the reverse (un-scoping) is harder.

> ✅ **Decision (confirmed 2026-07-30): Option A (conservative)** — `@TenantId` on the 16 operational/result entities; platform catalogs global; observability telemetry + security audit system-scoped. To be recorded as **ADR-057** (number verified at implement).

---

## 1. Technical Design
Identical to ADR-054 per entity: `implements Tenanted` + `@TenantId @Column(name="tenant_id", length=64, nullable=false, updatable=false) private String tenantId;` + getter/setter. No query, service, or wiring changes — the `core` resolver/customizer already active in the apps handles all of them.

## 2. Database changes — `V20__tenant_id_extension.sql` (gateway-owned, ADR-024)
`ALTER TABLE <t> ADD COLUMN tenant_id VARCHAR(64) NOT NULL DEFAULT '__system__';` + `CREATE INDEX ix_<t>_tenant …` for each of the 16 tables (backfill legacy → `__system__`). Exact `@Table` names verified against each entity at implement.

## 3. Test wiring (the `@TenantId`/`@DataJpaTest` rule)
Per the standing rule (ADR-056): a module whose persistence unit gains a `@TenantId` entity must supply the resolver to its `@DataJpaTest`s. Footprint here is small — only **observability** (already handled, but untouched by this slice) and **orchestration** (`WorkflowExecutionRepositoryTest` already `@Import`s the customizer, which covers the new `WorkflowEntity`/`WorkflowStepEntity`) have `@DataJpaTest`s. The other modules (execution/reporting/intelligence/agents/agents-runtime/brain/eval) have none — verified at implement; add `TestApplication`/per-test `@Import` if any surfaces.

## 4. Implementation plan
1. Add `@TenantId` + `Tenanted` to the 16 entities. *Compiles here.*
2. `V20` migration — verify each `@Table` name. *Applied by user.*
3. Test wiring — grep each affected module for `@DataJpaTest`; supply the resolver where needed. *Validatable here.*
4. Full `mvn clean test` green. *Validatable here.*
5. Docs — tracker ENT-1 note, **ADR-057**, this doc's Implementation Outcome.

**Definition of Done:** the 16 operational tenant-owned tables are `@TenantId`-isolated; reactor green; migration authored. After this, the **only** durable tenant-owned data not yet scoped is the RBAC satellites (FI-ENT1-D slice 2); catalogs + telemetry are deliberately global/system. **ENT-1 stays In Progress** (D slice 2 + the E slice-2 in-flight scoping remain).

**Honest boundary:** mechanism provable here; cross-tenant invisibility over a real DB is user-run.

---

## Implementation Outcome

**Implemented 2026-07-30 (Option A — conservative). Recorded as ADR-057. ENT-1 remains In Progress.**

**Delivered:** `@TenantId` + `implements Tenanted` on all 16 operational/result entities (execution/orchestration/reporting/intelligence/agents/agents-runtime/brain/eval) + `V20__tenant_id_extension.sql` (add `tenant_id` + index on the 16 tables, backfill `__system__`). Platform catalogs and observability/audit deliberately untouched.

**Verified here:** **full reactor `mvn clean test` BUILD SUCCESS, 22 modules, 0 compile errors, 0 failures** (8:17 min). Test-wiring footprint was minimal — only orchestration has a `@DataJpaTest` (`WorkflowExecutionRepositoryTest`, already `@Import`s the customizer, covering the new `WorkflowEntity`/`WorkflowStepEntity`); the other 7 modules have no Hibernate-booting test in the normal run (eval's `@SpringBootTest` is env-gated). All 16 tables confirmed present in migrations (V5/V9/V15) before authoring V20.

**Deviations from design:** none. Pure mechanical reuse of the ADR-054 pattern.

**Honest boundary:** mechanism proven here; cross-tenant invisibility over a real DB is user-run. Borderline classifications (`LearningEntity`, `EvaluationResultEntity`, `ExecutionAuditEntity` → tenant-owned) are revisitable.

**Deferred (ENT-1 stays In Progress):** FI-ENT1-D slice 2 (RBAC satellites: `UserSessionEntity`/`ApiKeyEntity`/`PasswordHistoryEntity` + audit attribution) — the only durable tenant-owned data not yet scoped; FI-ENT1-E slice 2 (vector-search / short-term / Local-artifact scoping).
