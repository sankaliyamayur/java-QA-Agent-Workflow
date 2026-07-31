# FI-ENT4-A — Admin write-ops (create/disable user, assign roles) · Technical Design

**Item:** ENT-4 (Admin & user-management surface) → sub-item **FI-ENT4-A** — the mutating half.
**Status:** design (awaiting decision + implement approval).
**Unblocked by:** FI-ENT4-C / ADR-066 (a real `ROLE_ADMIN` now exists to gate on).

---

## 0. Grounding + scope

### 0.1 What exists
- **Read-model (FI-ENT4-B, done):** `RbacAdminController` `GET /api/dashboard/admin/rbac` → secret-free `RbacAdminSummary` (`RbacAdminService` → pure `RbacAdminAssembler`, ADR-052). Read-only.
- **Roles (FI-ENT4-C, done):** `UserEntity.roles` `@ElementCollection<String>` (`security_user_roles`), `AuthorityMapper` → `ROLE_<name>`, `JwtAuthenticationFilter` grants them. Bootstrap admin has `["ADMIN"]`.
- **Tenancy (ENT-1):** `UserEntity` is `@TenantId`; username/email are **per-tenant unique** (FI-ENT1-D). `JwtAuthenticationFilter` binds the caller's tenant from the signed token (ADR-055/058).
- **Authz plumbing:** `SecurityConfig` is `@EnableMethodSecurity` (global, scanned into gateway **and** dashboard); `enforcedFilterChain` (JWT filter + deny-by-default) is active whenever `aiqaos.security.enabled=true` (dashboard default **true**, gateway default **true**).
- `UserEntity` write surface: `setUsername/Email/PasswordHash/Enabled/AccountLocked/Roles`; `BCryptPasswordEncoder` is the established hashing path (bootstrap). `RoleRepository` is the global role catalog (`RoleEntity.roleName` unique).

### 0.2 The security constraint that shapes this
`DashboardSecurityConfig` `@Order(1)` matches `/api/dashboard/**` (+ `/api/auth`, `/api/v1`, `/actuator`, `/swagger`, `/v3/api-docs`) and does **`permitAll()` with no JWT filter**. So **any write endpoint placed under `/api/dashboard/**` is unauthenticated** — unacceptable for user-management writes. The read-model living there is fine (secret-free); writes are not.

Paths **outside** that matcher fall onto `SecurityConfig.enforcedFilterChain` → JWT filter runs → `authenticated()` + method security applies. This is the seam FI-ENT4-A must land on.

### 0.3 What this slice delivers
A mutating admin API, **ADMIN-gated and tenant-scoped**:
1. **Create user** — username, email, password, roles → hashed, tenant auto-bound, roles validated.
2. **Enable/disable user** — flip `enabled` (soft-disable; never hard-delete here).
3. **Assign roles** — replace a user's `roles` set (validated against the global catalog).

All three reuse the FI-ENT4-B read DTOs for responses (return the updated `AdminUserView`).

### 0.4 Deferred (out of scope, logged honestly)
- Password reset / force-rotate, unlock-account, hard-delete, org/workspace assignment.
- Per-permission (fine-grained) authority editing (`RolePermissionEntity`).
- Audit-log emission for admin actions (candidate FI — see §7).
- Live ADMIN-gated E2E over a real DB (user-run).

---

## 0.5 / Decision for approval — where the write API is hosted + how it's authz-gated

Both options gate on `@PreAuthorize("hasRole('ADMIN')")` (method security is already global) and both fail closed. The fork is the **URL prefix + which filter chain serves it**, which also decides what the React admin UI calls.

- **Option A (recommended) — new `/api/admin/**` prefix on the enforced chain.**
  A new `AdminUserController` (`com.aiqaos.security.admin`) at `/api/admin/users`, class-level `@PreAuthorize("hasRole('ADMIN')")`. Because `/api/admin/**` is **not** in `DashboardSecurityConfig`'s matcher, it lands on `SecurityConfig.enforcedFilterChain` on both apps → JWT filter runs, ADMIN required. **No security-config change.** Read stays at `/api/dashboard/admin/rbac` (open, secret-free); writes at `/api/admin/users` (JWT+ADMIN). UI sends the bearer token for writes.

- **Option B — authenticated admin sub-chain under `/api/dashboard/admin/**`.**
  Keep one base path by adding an `@Order(0)` chain to `DashboardSecurityConfig` matching `/api/dashboard/admin/**` that *does* add `JwtAuthenticationFilter` and requires `hasRole('ADMIN')`. Cleaner URL story, but it re-opens the dashboard security config, adds a third chain to reason about, and risks ordering mistakes — more surface for a security-critical change.

**Recommendation: Option A** — least-privilege by construction, zero change to the security chains, fails closed even if a chain is later misconfigured. → **ADR-067.**

---

## 1. Technical Design (Option A)

### 1.1 New classes (all in `ai-qa-os-security`, `com.aiqaos.security.admin`)
- **`AdminUserController`** `@RestController @RequestMapping("/api/admin/users")` `@PreAuthorize("hasRole('ADMIN')")` `@ConditionalOnProperty(aiqaos.security.database-enabled=true, matchIfMissing=true)` (mirrors `AuthenticationController`):
  - `POST /` `CreateUserRequest` → 201 `AdminUserView`
  - `PATCH /{id}/enabled` `{enabled:boolean}` → 200 `AdminUserView`
  - `PUT /{id}/roles` `{roles:List<String>}` → 200 `AdminUserView`
- **`AdminUserService`** `@Service` — the write logic + validation; depends on `UserRepository`, `RoleRepository`, `BCryptPasswordEncoder` (or `new` per call, as bootstrap does).
- **Request records:** `CreateUserRequest(username,email,password,roles)`, `EnabledRequest(boolean enabled)`, `RolesRequest(List<String> roles)`.
- **Response:** reuse `AdminUserView` — but it lives in `ai-qa-os-dashboard`. To avoid a security→dashboard dependency (wrong direction), the controller returns a **security-local `AdminUserView`** record (id, username, email, enabled, accountLocked, roles) — secret-free, mirrors the dashboard DTO's fields. (The dashboard read-model keeps its own copy; no cross-module coupling.)

### 1.2 Rules / validation (in `AdminUserService`)
- **Tenant:** never set explicitly — `@TenantId` auto-binds from the JWT-bound `TenantContext` on insert, and `findAll`/`findById` are tenant-filtered. An admin only ever sees/mutates users in their own tenant (ENT-1 invariant).
- **Create:** reject blank username/email/password (400); reject duplicate username/email **within tenant** (409, catch the unique-constraint path / pre-check via `findByUsername`/`findByEmail`); BCrypt-hash the password; `enabled=true`.
- **Roles (create + assign):** each requested role name must exist in the global catalog (`RoleRepository`, matched case-insensitively on `roleName`) → else 400. Store normalized names. (ADR-066 left names un-FK'd; this write path adds the catalog *check* so admins can't invent roles.)
- **Enable/disable:** flip `enabled`; **self-lockout guard** — an admin cannot disable their own account (compare to the authenticated principal's id) → 400.
- **Assign roles:** **self-demotion guard** — an admin cannot remove `ADMIN` from their own account → 400. (Prevents the last admin locking everyone out via the UI.)

### 1.3 Error handling
Reuse the existing REST error contract (`RestAccessDeniedHandler` 403 / `RestAuthenticationEntryPoint` 401 handle authz). Validation failures throw `IllegalArgumentException`/a small `AdminUserException` → mapped to 400/409 via a `@RestControllerAdvice` in security (add one if none exists; otherwise `ResponseStatusException`).

## 2. Migration
**None.** `security_user_roles` (V22) and all columns already exist. Pure behavior over existing schema.

## 3. React UI (ai-qa-os-dashboard-ui)
Extend `AdminPage.tsx` (already `RoleGuard('ADMIN')`): a "Create user" form + per-row enable/disable toggle + role multi-select, calling `/api/admin/users*` **with the bearer token**. `adminService.ts` gains `createUser/setEnabled/setRoles`. (Written to convention; `npm run build`/`dev` is user-run — no Node build guarantee needed for the Java slice.)

## 4. Testing (honest, JDK-25 / Mockito-free)
- **`AdminUserServiceTest`** (unit, hand-stubbed `UserRepository`/`RoleRepository`): create happy-path (hashes, roles kept), blank-field rejection, duplicate username/email rejection, unknown-role rejection, self-disable guard, self-ADMIN-removal guard, enable/disable flip, role replace.
- Controller authz (`hasRole('ADMIN')`) is Spring-config — **not** unit-tested here (tests run permissive); it's covered by the enforced-chain design + user-run E2E.
- Full reactor `mvn clean test -Djacoco.skip=true` must stay green (22 modules).

## 5. What can't be validated here (user-run)
Live ADMIN-gated calls over a real Postgres with a seeded bootstrap admin + JWT; the `/admin` UI create/disable/assign flows via `npm run dev`.

## 6. Implementation plan
1. `AdminUserView`, request records, `AdminUserException` (security).
2. `AdminUserService` + validation/guards.
3. `AdminUserController` (`@PreAuthorize` class-level).
4. `@RestControllerAdvice` (if absent) for 400/409 mapping.
5. `AdminUserServiceTest`.
6. UI extension (`adminService.ts`, `AdminPage.tsx`).
7. Full reactor verify.
8. Docs: ADR-067, tracker ENT-4 note, this doc's Implementation Outcome.

## 7. Follow-on ideas (outside current roadmap — log as FI)
- **FI-ENT4-D:** audit-log emission for admin mutations (who/what/when).
- **FI-ENT4-E:** unlock-account + password force-reset admin ops.

---

## Implementation Outcome

**Delivered 2026-07-31 (Option A / ADR-067). Full reactor green — 22 modules, 0 failures; UI lint + build clean. ENT-4 → Completed.**

Shipped as designed:
- **`AdminUserController`** (`com.aiqaos.security.admin`, `/api/admin/users`) — `POST /` (201), `PATCH /{id}/enabled`, `PUT /{id}/roles`; class-level `@PreAuthorize("hasRole('ADMIN')")`; `@ConditionalOnProperty(database-enabled)`. Caller id for the guards comes from `@AuthenticationPrincipal UserEntity` (null-safe).
- **`AdminUserService`** — create (blank-field 400, duplicate username/email 409, BCrypt hash, roles validated → catalog-canonical), enable/disable (self-disable 400), assign roles (unknown-role 400, self-ADMIN-removal 400). Tenant is never set explicitly — `@TenantId` stamps/filters (ENT-1).
- **`AdminUserRequests`** (create/enabled/roles records), **`AdminUserView`** (security-local, secret-free — no dashboard dependency).
- **Read-model** `dashboard.dto.AdminUserView` extended with secret-free `id` + `roles`; `RbacAdminAssembler` passes them; `RbacAdminAssemblerTest` updated (asserts id/roles).
- **React** `adminService.ts` (+`createUser`/`setUserEnabled`/`setUserRoles`, `id`/`roles` on the view) and `AdminPage.tsx` (create form, per-row enable/disable, role editor; refetches summary after each write; `apiClient` injects the bearer token).

**Tests:** `AdminUserServiceTest` 8/8 (Mockito-free — JDK-proxy repos over an in-memory store); `AuthorityMapperTest` 4/4; `RbacAdminAssemblerTest` 3/3; dashboard/gateway `@SpringBootTest`s boot with the new controller/DTO. UI `npm run lint` (no new warnings) + `npm run build` (`tsc -b` + vite) green.

**Deviations:** none of substance. Errors are surfaced via `ResponseStatusException` (no new `@RestControllerAdvice` — Spring's default `/error` path, which both chains permit, handles 400/409); the enforced chain's `RestAuthenticationEntryPoint`/`RestAccessDeniedHandler` handle 401/403.

**No migration** — `security_user_roles` (V22) and all columns pre-exist; pure behavior over existing schema.

**User-run (not validatable in sandbox):** live ADMIN-gated calls over a real Postgres with a seeded bootstrap admin + JWT (create → login → disable → assign); the `/admin` UI flows via `npm run dev`.

**Follow-on ideas logged (outside ENT-4 core):** FI-ENT4-D (admin-action audit log), FI-ENT4-E (unlock-account + password force-reset).
