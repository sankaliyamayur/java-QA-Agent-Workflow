# MOD-5 — Make `ai-qa-os-data` real, or fold it in · Technical Design

**Item:** MOD-5 (un-deferred at user request 2026-08-02) — resolve the `ai-qa-os-data` module.
**Status:** design (awaiting decision + implement approval).

---

## 0. Grounding — what `ai-qa-os-data` actually is

An **orphan scaffolding module**:
- **5 completely empty stub classes** — `DatabaseConnectionManager`, `KnowledgeDatabase`, `ProjectDatabase` (classes), `TransactionService` (interface), `VectorDatabaseClient` — all `{}`, no methods, no fields.
- **`pom.xml` has no dependencies** (not even `core`) — the comment literally reads "Add module specific dependencies here."
- **Nothing depends on it** — only the parent reactor lists it as a `<module>`; no other module has it as a dependency.
- Builds in ~0.15s (no real compilation).

Every concern its class *names* imply is **already implemented in a real module**:
| Stub | Already real in |
|---|---|
| `VectorDatabaseClient` | `ai-qa-os-memory` — `VectorStoreClient` / `InMemoryVectorStoreClient` / `QdrantStoreClient` |
| `ProjectDatabase` / `KnowledgeDatabase` | JPA repositories across `core` / `security` / `observability` / `eval` |
| `DatabaseConnectionManager` | Spring datasource + Flyway (gateway-owned, ADR-024) |
| `TransactionService` | Spring `@Transactional` |

---

## 0.1 / Decision for approval — the MOD-5 fork

- **Option A (recommended) — fold in / remove.**
  Delete `ai-qa-os-data` and drop it from the parent `<modules>`. **Zero code is lost** (all five classes are empty), nothing imports it, and its concerns are already covered by real modules. The reactor goes **22 → 21 modules**. This removes dead scaffolding that misleads (a named "data" module implies a data layer that doesn't exist). → **ADR-077.**

- **Option B — make it real.**
  Fill it in as a unified data-access layer. Rejected: it would **duplicate** `memory` (vectors), the JPA repositories (project/knowledge data), and Spring (connections/transactions), and cut across the established boundaries (ADR-024 gateway-owned schema; `memory` owns vector storage). That is evolving the architecture to add redundancy — the opposite of what the frozen roadmap and module-boundary discipline call for.

**Recommendation: Option A (fold in / remove).** The evidence is one-sided — empty stubs, zero dependents, concerns already real. Making it "real" would build a redundant layer.

---

## 1. Technical Design (Option A — fold in / remove)

1. Delete the module directory `ai-qa-os-data/` (5 empty classes + its pom).
2. Remove `<module>ai-qa-os-data</module>` from the parent `pom.xml` `<modules>`.
3. No other change — nothing imports the module, no entity/migration/bean lives in it, so there is nothing to migrate.
4. Update the "22 modules" references in the docs to **21** where they describe the reactor going forward (historical notes stay as-written).

## 2. Verification
- Full reactor `mvn clean test` → **BUILD SUCCESS with 21 modules** (was 22), 0 failures — proving nothing depended on the removed module.
- No test asserts the module count; ArchUnit rules are per-module scope and unaffected.

## 3. What can't be validated here
Nothing — this is a self-contained reactor change, fully verified by the build.

## 4. Implementation plan
1. Remove the module from the parent `<modules>`; delete `ai-qa-os-data/`.
2. Full reactor verify (expect 21 modules green).
3. Docs: ADR-077, tracker MOD-5 (Deferred → Completed) + counts, this doc's Implementation Outcome; note the module count is now 21.

## 5. Follow-on
None. (If a genuine shared data-access need arises later, it belongs in the module that owns that data, not a cross-cutting "data" module — consistent with the boundaries this decision reinforces.)

---

## Implementation Outcome

**Delivered 2026-08-02 (Option A / ADR-077). Full reactor BUILD SUCCESS — 21 modules (was 22), 0 failures. MOD-5 → Completed.**

Done exactly as designed:
- Removed `<module>ai-qa-os-data</module>` from the parent `pom.xml`.
- Deleted the `ai-qa-os-data/` directory (5 empty classes + pom + build output).
- No other change — grep confirmed no residual `ai-qa-os-data` / `com.aiqaos.data` references in any pom or source; nothing imported it.

**Verification:** full reactor `mvn clean test` → **BUILD SUCCESS, 21 modules, 0 failures** (`ai-qa-os-data` absent from the reactor summary), proving nothing depended on the removed module.

**Deviations:** none.

**Note:** going-forward reactor builds report **21 modules**; historical ADR/tracker notes that say "22 modules" were accurate when written and are left as-is.

**Follow-on:** none.
