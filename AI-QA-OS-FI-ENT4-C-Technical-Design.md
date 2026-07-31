# FI-ENT4-C — Technical Design: User↔Role Mapping + Role-Based Authorities

**Parent item:** ENT-4 (Admin & user-management surface) — **In Progress**. 🟡 P2.
**Unblocks:** proper role-based authorization (`hasRole('ADMIN')`) and the admin **write-ops** (FI-ENT4-A), which are currently un-buildable safely.
**Status:** Draft — awaiting decision + go-ahead (no code until approved).
**Date:** 2026-07-31.

> **Why this before the write-ops.** Grounding for FI-ENT4-A (create/disable user, assign role) found admin write-ops **cannot be built safely today**: `UserEntity` has **no roles**, `JwtAuthenticationFilter` grants only a hardcoded `ROLE_USER`, and there is **no `user_roles` mapping**. So a write endpoint would be either an ungated security hole or gated on an authority nobody holds. The missing foundation is FI-ENT4-C — persist which roles a user has, and turn them into Spring authorities so an ADMIN user actually has `ROLE_ADMIN`.

---

## 0. Grounding + scope

### 0.1 What exists / what's missing
- **Exists:** `UserEntity` (tenant-scoped, `@TenantId`); `RoleEntity` (global catalog, `role_name` unique — ADR-055); `RolePermissionEntity`; `RoleManager` (a stub — no user↔role logic).
- **Missing (the gap):** (1) a **user↔role mapping** (no `security_user_roles` table, no `UserEntity.roles`); (2) **role→authority wiring** — `JwtAuthenticationFilter` sets `List.of(new SimpleGrantedAuthority("ROLE_USER"))` for every principal (a `// no user→role model yet` note). So `@PreAuthorize("hasRole('ADMIN')")` can never be true, and the client-chosen UI role is unvalidated server-side (a known tech-debt item).

### 0.2 What this slice delivers
Persist a user's role assignments and derive Spring `GrantedAuthority`s from them at authentication, so an ADMIN-role user carries `ROLE_ADMIN` — enabling real role-based authorization and unblocking the ENT-4 write-ops.

### 0.3 Deferred
- The write-ops themselves (create/disable user, **assign role**) — FI-ENT4-A, the next slice once this foundation exists.
- Per-permission (fine-grained) authorities from `RolePermissionEntity` — roles are sufficient for admin gating now.

### 0.4 / Decision for approval — the mapping representation (ADR decision)

| Option | Representation | Trade-off |
|---|---|---|
| **A — `@ElementCollection<String>` role names on `UserEntity` (recommended)** | A `security_user_roles` collection table (`user_id`, `role_name`); `UserEntity.roles : List<String>`. Roles are referenced by their (unique) name — the global-catalog key. Mirrors the existing `backupCodes` `@ElementCollection`. | Simplest; no FK/`@ManyToMany` fetch complexity; matches "roles are a global catalog **by name**" (ADR-055). Names aren't referentially enforced against `RoleEntity` (acceptable — the catalog is small and admin-curated). |
| **B — `@ManyToMany<RoleEntity>`** | A `user_roles` join table with a `role_id` FK to `RoleEntity`. | Referential integrity to the catalog, but heavier (join-table entity, eager/lazy fetch decisions, cascade semantics) and couples user loading to `RoleEntity` — over-engineered when roles are used by name for authorities. |

**Recommendation: A.** Authorities are derived from role **names** (`ROLE_<name>`); an `@ElementCollection<String>` is the minimal faithful model and mirrors an existing pattern on the same entity. B's FK integrity isn't worth the join-table weight here.

> ✅ **Decision (confirmed 2026-07-31): Option A** — `@ElementCollection<String> roles` on `UserEntity` (`security_user_roles`), authorities derived as `ROLE_<name>`. To be recorded as **ADR-066** (number verified at implement).

---

## 1. Technical Design (Option A)

### 1.1 Persistence
- `UserEntity` gains `@ElementCollection(fetch = EAGER) @CollectionTable(name="security_user_roles", joinColumns=@JoinColumn(name="user_id")) @Column(name="role_name") private List<String> roles = new ArrayList<>();` + getter/setter. EAGER so authorities are available on load without a second round-trip.
- Migration **V22** (gateway-owned, ADR-024): `CREATE TABLE security_user_roles (user_id UUID NOT NULL, role_name VARCHAR(50) NOT NULL, PRIMARY KEY(user_id, role_name), FOREIGN KEY(user_id) REFERENCES security_users(id))`.
- Bootstrap admin (`BootstrapAdminInitializer`) seeds `roles=["ADMIN"]` on the created admin so there is at least one real admin.

### 1.2 Role→authority wiring
- `JwtAuthenticationFilter`: replace the hardcoded `ROLE_USER` with authorities derived from the loaded user — `ROLE_USER` (baseline) **plus** `ROLE_<name>` for each `user.getRoles()`. The user is already loaded per request (`findById`, tenant-scoped), so roles come free.
- Extract a small `AuthorityMapper.authorities(UserEntity) → List<GrantedAuthority>` (testable in isolation).

### 1.3 What it enables (not built here)
`@PreAuthorize("hasRole('ADMIN')")` / URL rules now work; the UI `RoleGuard` role is validated server-side; FI-ENT4-A write-ops can gate on `ROLE_ADMIN`.

## 2. Classes / migration
- `UserEntity` — `roles` element collection.
- `AuthorityMapper` (security) — role names → authorities.
- `JwtAuthenticationFilter` — use the mapper.
- `BootstrapAdminInitializer` — seed `["ADMIN"]`.
- `V22__user_roles.sql`.

## 3–5. API
No new endpoint. Behaviour: authenticated principals now carry role-derived authorities.

## 6. Implementation plan
1. `UserEntity.roles` `@ElementCollection` + `V22` migration. *Compiles here; migration applied by user.*
2. `AuthorityMapper` + **unit test** (roles `["ADMIN"]` → `ROLE_USER`+`ROLE_ADMIN`; empty → `ROLE_USER`). *Validatable here.*
3. `JwtAuthenticationFilter` uses the mapper (+ its existing tenant-binding test still green). *Validatable here.*
4. `BootstrapAdminInitializer` seeds `["ADMIN"]`. *Compiles here.*
5. Full `mvn clean verify` green (watch `security` `@DataJpaTest`s + the `security_user_roles` table under H2 `create-drop`). *Validatable here.*
6. Docs — tracker ENT-4 (FI-ENT4-C done), **ADR-066**, Outcome.

**Definition of Done:** users carry persisted roles and authenticated principals get role-derived authorities (`ROLE_ADMIN` for admins); unit-proven; reactor green; migration authored. ENT-4 stays **In Progress** — this unblocks **FI-ENT4-A** (write-ops) as the next slice. Live role-gated E2E is user-run (real DB + an ADMIN user).

**Honest boundary:** the mapping + authority derivation are provable here; end-to-end `hasRole('ADMIN')` enforcement over a real DB with a seeded admin is user-run.

---

## Implementation Outcome

**Delivered 2026-07-31 (Option A / ADR-066). Full reactor green — 22 modules, 0 failures, 6:27 min.**

Shipped exactly as designed:
- **`UserEntity.roles`** — `@ElementCollection(fetch = EAGER)` of `String` into `security_user_roles(user_id, role_name)` (mirrors the existing `backupCodes` collection), with `getRoles`/`setRoles`. Role names reference the global catalog by name (ADR-055).
- **`AuthorityMapper.authorities(user)`** (new, `com.aiqaos.security.authorization`) — baseline `ROLE_USER` + `ROLE_<NAME>` per role; upper-cased, `ROLE_`-prefix-aware, blanks/dupes skipped, null-user safe.
- **`JwtAuthenticationFilter`** — the user branch now grants `AuthorityMapper.authorities(user)` instead of the hardcoded `List.of(ROLE_USER)`. Tenant binding (FI-ENT1-D) unchanged.
- **`BootstrapAdminInitializer`** — seeds `roles = ["ADMIN"]` so there is a real ADMIN principal.
- **`V22__user_roles.sql`** (gateway-owned, ADR-024) — `security_user_roles` with FK to `security_users(id) ON DELETE CASCADE` + `ix_user_roles_user`.

**Tests:** `AuthorityMapperTest` 4/4 (ADMIN→ROLE_ADMIN, no-roles→baseline, null→baseline, case/prefix normalisation + dedup); `JwtAuthenticationFilterTenantTest` still 1/1; dashboard `@SpringBootTest` boots with the new mapping (`create-drop` generates `security_user_roles`).

**Deviations:** none.

**User-run (not validatable in sandbox):** end-to-end `hasRole('ADMIN')` over a real Postgres with a seeded bootstrap admin; the `/admin` UI page gated by the now-server-validated role.

**Unblocks:** FI-ENT4-A (admin write-ops — create/disable user, assign role) can now be authz-gated on a real `ROLE_ADMIN`.
