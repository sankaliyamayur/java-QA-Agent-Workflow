# FI-ENT1-E — Technical Design: Tenant-Scoped Memory, Cost & Artifact (enforcement slice)

**Parent item:** ENT-1 — **stays `In Progress`** after this slice.
**Follow-up ID:** FI-ENT1-E (tenant-scoped memory retrieval + cost isolation + artifact scoping). Builds on ADR-054 (persistence tenancy mechanism) and ADR-055 (RBAC).
**Status:** Draft — awaiting decision + go-ahead (no code until approved).
**Date:** 2026-07-30 · **Unblocked by:** Phase 1 infra (ADR-053) + FI-ENT1-C/D.

> **Scope discipline.** This slice tenant-isolates the **durable** memory, cost, and artifact surfaces — the security-relevant, validatable ones — by reusing the proven `@TenantId` mechanism and adding tenant key-namespacing to shared object storage. In-flight scoping backed by **stub/dev** components (vector-store search over the deferred Qdrant client; short-term Caffeine/Redis keys) is deferred, per the "no scaffolding without a real consumer" rule (ADR-053).

---

## 0. Grounding — current state, scope, decision

### 0.1 What exists (verified 2026-07-30)
- **Memory:** `MemoryNodeEntity` (`memory_nodes`, V5), `ConversationHistoryEntity` (`conversation_histories`, V5) — both extend `BaseEntity`, **no tenant column**. `MemoryMetadata` (a DTO, not an entity) carries a **`UUID tenantId`** with **no external callers** (only its own accessors).
- **Cost:** `LLMCostEntity` (`observability_llm_costs`, V3, extends `BaseEntity`, no tenant column), written by `CostTracker` (`ai-provider`).
- **Artifact:** `ObjectStorageArtifactStore` (active when `aiqaos.artifacts.store=object`) addresses blobs by `fullKey = staticPrefix + key` via an `ObjectStorageClient`. The **JPA row** (`ExecutionArtifactEntity`) got `@TenantId` in FI-ENT1-C, but the **blob bytes are still cross-tenant addressable** — no tenant in the key. (Default deployments use `LocalArtifactStore`; the object store is the shared, multi-tenant-relevant path.)
- **Mechanism available:** the ADR-054 `TenantIdentifierResolver` + `TenantHibernateCustomizer` live in `core` and are active in any app scanning `com.aiqaos` — so `@TenantId` on any entity "just works" in the apps.

### 0.2 What FI-ENT1-E (this slice) must achieve
Persisted **memory** and **cost** rows are written/read tenant-scoped (a tenant sees only its own memory nodes, conversation history, and LLM-cost records), and shared **artifact blobs** are namespaced per tenant so tenant A cannot read tenant B's bytes by key.

### 0.3 Scope — this slice vs. deferred
**In this slice:**
- `@TenantId` + `Tenanted` on `MemoryNodeEntity`, `ConversationHistoryEntity`, `LLMCostEntity`; migration **V19**.
- Reconcile `MemoryMetadata.tenantId` **UUID → String** (contract consistency; zero external callers).
- **Artifact blob tenant-namespacing** in `ObjectStorageArtifactStore` (key prefix from `TenantContextHolder`).
- Test wiring: supply the resolver to the observability persistence-unit tests (its `TestApplication`).

**Deferred:** vector-store **search** tenant-filtering (the real `QdrantStoreClient` is a stub — ADR-053); short-term **Caffeine/Redis** key namespacing (dev/secondary); `LocalArtifactStore` path scoping (dev-only, single-tenant); the memory *retrieval-service* filtering beyond what `@TenantId` gives the JPA layer.

### 0.4 / Decision for approval — artifact-blob isolation approach (ADR decision)

| Option | Approach | Trade-off |
|---|---|---|
| **A — Tenant key-prefix (recommended)** | `fullKey = staticPrefix + <tenant>/ + key`, tenant from `TenantContextHolder` (unbound ⇒ `__system__`). One shared bucket; isolation by key namespace. | Storage-agnostic (works with today's in-memory client and tomorrow's MinIO), tiny change, unit-validatable now. Isolation is by convention in the key, not a hard storage boundary. Legacy (pre-tenancy) blobs move under `__system__/` addressing. |
| **B — Bucket-per-tenant** | A distinct bucket per tenant. | A harder storage boundary, but MinIO-specific, needs dynamic bucket lifecycle/creation, and depends on the deferred real client — heavy for a first slice, unvalidatable here. |

**Recommendation: A.** Key-prefix namespacing gives tenant isolation now, storage-agnostically, and is validatable against the in-memory client. Bucket-per-tenant can layer on later if a hard boundary is required.

> ✅ **Decision (confirmed 2026-07-30): Option A** — tenant key-prefix in `ObjectStorageArtifactStore` (storage-agnostic, unit-validatable). To be recorded as **ADR-056** (number verified at implement).

---

## 1. Technical Design

### 1.1 Persistence tenancy (memory + cost) — reuse ADR-054
`MemoryNodeEntity`, `ConversationHistoryEntity`, `LLMCostEntity` each `implements Tenanted` + gain:
```
@TenantId
@Column(name = "tenant_id", length = 64, nullable = false, updatable = false)
private String tenantId;
```
Hibernate auto-stamps on insert and filters on read — no query changes. `MemoryMetadata.tenantId` becomes `String` for contract consistency.

### 1.2 Artifact blob tenant-namespacing
`ObjectStorageArtifactStore.fullKey(key)` prepends the current tenant:
```
String tenant = TenantContextHolder.current().map(TenantContext::getTenantId).orElse(SYSTEM_TENANT);
fullKey = staticPrefix + tenant + "/" + key;     // e.g. artifacts/acme/run-123/screenshot.png
```
`store/resolve/exists/delete/lastModified` all route through `fullKey` (already the case) → automatically tenant-scoped. `list(keyPrefix)` scopes within `staticPrefix + tenant + "/"` and strips that when returning keys. Unbound/system work uses `__system__/`.

### 1.3 Test wiring (the `@TenantId`-in-a-new-unit consequence)
Adding `@TenantId` to `LLMCostEntity` means the **observability** persistence unit now contains a tenant-discriminator entity, so its `@DataJpaTest`s need the resolver. Supply it once via `@Import(TenantHibernateCustomizer.class)` on observability's test `TestApplication` (covers all its slice tests) rather than per-test. Same check for the memory module at implement.

## 2. Folder / module changes
- `memory/entity/MemoryNodeEntity.java`, `.../ConversationHistoryEntity.java` — `@TenantId` + `Tenanted`.
- `memory/model/MemoryMetadata.java` — `tenantId` UUID→String.
- `observability/entity/LLMCostEntity.java` — `@TenantId` + `Tenanted`.
- `execution/artifact/ObjectStorageArtifactStore.java` — tenant key-prefix.
- `deployment/migration/db/migration/V19__tenant_id_memory_cost.sql` (gateway-owned).
- Test: observability `TestApplication` `@Import` the customizer.

## 3. Required classes / edits (key)
| Class | Change |
|---|---|
| `MemoryNodeEntity`, `ConversationHistoryEntity`, `LLMCostEntity` | `@TenantId` + `implements Tenanted` |
| `MemoryMetadata` | `tenantId` UUID→String |
| `ObjectStorageArtifactStore` | tenant key-prefix via `TenantContextHolder` |
| `V19__tenant_id_memory_cost.sql` | add `tenant_id` + index on the 3 tables |

## 4. Database changes — `V19__tenant_id_memory_cost.sql`
```
ALTER TABLE memory_nodes            ADD COLUMN tenant_id VARCHAR(64) NOT NULL DEFAULT '__system__';
ALTER TABLE conversation_histories  ADD COLUMN tenant_id VARCHAR(64) NOT NULL DEFAULT '__system__';
ALTER TABLE observability_llm_costs ADD COLUMN tenant_id VARCHAR(64) NOT NULL DEFAULT '__system__';
CREATE INDEX ix_memory_nodes_tenant       ON memory_nodes            (tenant_id);
CREATE INDEX ix_conversation_hist_tenant  ON conversation_histories  (tenant_id);
CREATE INDEX ix_llm_costs_tenant          ON observability_llm_costs (tenant_id);
```
Backfills legacy rows to the system tenant (no regression); `ddl-auto: validate` stays green.

## 5. API changes
**None.** Purely persistence + storage-key behaviour.

## 6. Implementation plan — small, verifiable tasks
1. **Entities** — `@TenantId` + `Tenanted` on the 3; `MemoryMetadata` String. *Compiles here.*
2. **Artifact store** — tenant key-prefix + unit test against `InMemoryObjectStorageClient` (tenant A stores; tenant B cannot resolve the same key; `list` scoped). *Validatable here.*
3. **Migration V19** — authored; table/column names verified. *Applied by user.*
4. **Test wiring** — observability `TestApplication` imports the customizer; check memory module. *Validatable here.*
5. **Reactor green** — full `mvn clean test`. *Validatable here.*
6. **Docs** — tracker ENT-1 note, **ADR-056**, this doc's Implementation Outcome.
7. **Isolation E2E (user-run, needs DB):** under two tenants, confirm memory nodes / cost rows / artifacts written by one are invisible to the other.

**Definition of Done (this slice):** persisted memory + cost rows are tenant-isolated by `@TenantId`; shared artifact blobs are tenant-namespaced (unit-proven against the in-memory client); reactor green; migration authored. **ENT-1 stays In Progress** — vector-search/short-term/Local-artifact scoping and the FI-ENT1-C/D extensions remain.

**Honest boundary:** the mechanism (entity mapping, artifact key-prefix, migration) is provable here; **cross-tenant invisibility over a real DB is user-run**.

---

## Implementation Outcome

**Implemented 2026-07-30 (Option A — tenant key-prefix for blobs; `@TenantId` for memory + cost). Recorded as ADR-056. ENT-1 remains In Progress.**

**Delivered:**
- `@TenantId` + `implements Tenanted` on `MemoryNodeEntity`, `ConversationHistoryEntity` (memory), `LLMCostEntity` (observability).
- `MemoryMetadata.tenantId` reconciled **UUID → String** (no external callers).
- `ObjectStorageArtifactStore` — `fullKey`/`list` route through a `tenantPrefix()` (`staticPrefix + <tenant> + "/"`, tenant from `TenantContextHolder`, `__system__` fallback), so blobs are tenant-namespaced.
- `V19__tenant_id_memory_cost.sql` — `tenant_id` + index on `memory_nodes`, `conversation_histories`, `observability_llm_costs` (backfill `__system__`).
- Observability `TestApplication` `@Import(TenantHibernateCustomizer.class)` — supplies the resolver to that module's 6 `@DataJpaTest`s (LLMCostEntity joined their unit).

**Verified here:** `ObjectStorageArtifactStoreTenantTest` **2/2** (tenant A stores; tenant B cannot resolve/exists/list the same key; system fallback lands under `__system__/`); the existing `ObjectStorageArtifactStoreTest` updated for the tenant namespace (4/4); observability slice tests + `CostTrackerTest` green with the resolver wired; **full reactor `mvn clean test` BUILD SUCCESS, 22 modules, 0 failures** (8:58 min).

**One regression caught + fixed:** the pre-existing `ObjectStorageArtifactStoreTest` asserted the raw client key `artifacts/<key>`; with tenant-namespacing (no tenant bound → system) it's now `artifacts/__system__/<key>` — assertion updated.

**Deviations from design:** none material.

**Honest boundary:** the mechanism (entity mapping, artifact key-prefix, migration) is proven here; **cross-tenant invisibility over a real DB is user-run** (§6 step 7). Vector-store search filtering (deferred Qdrant client), short-term Caffeine/Redis keys, and `LocalArtifactStore` path scoping remain deferred.

**Deferred (ENT-1 stays In Progress):** FI-ENT1-E slice 2 (vector-search / short-term / Local-artifact scoping); plus the FI-ENT1-C extension to the remaining tenant-owned tables and FI-ENT1-D slice 2 (user satellites + audit).
