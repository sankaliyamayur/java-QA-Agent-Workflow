# FI-ENT1-D — Technical Design: Tenant-Scoped RBAC (security enforcement slice)

**Parent item:** ENT-1 — **stays `In Progress`** after this slice.
**Follow-up ID:** FI-ENT1-D (tenant-scoped RBAC). Builds on ENT-1 foundation (ADR-041), the gateway filter (ADR-043), and the FI-ENT1-C persistence mechanism (ADR-054).
**Status:** Draft — awaiting decision + go-ahead (no code until approved).
**Date:** 2026-07-30 · **Unblocked by:** Phase 1 infra (ADR-053) + FI-ENT1-C (ADR-054).

> **Security-critical.** This slice decides which tenant an authenticated caller acts as. Get it wrong and a token minted for tenant A can read tenant B. The core rule here: **for an authenticated request the tenant comes from the signed JWT, never from the spoofable `X-Tenant-ID` header.**

---

## 0. Grounding — current state, scope, decision

### 0.1 What exists (verified 2026-07-30)
- **`UserEntity`** (`security_users`, extends `BaseEntity`): `username` + `email` are **globally unique**; `tenantId` is a **nullable `UUID`** (plus separate `organizationId`/`workspaceId` UUIDs — an org hierarchy, *not* the tenant discriminator). RBAC: `RoleEntity`, `PermissionEntity`, `RolePermissionEntity`; satellites `UserSessionEntity`, `ApiKeyEntity`, `PasswordHistoryEntity`.
- **Login** (`AuthenticationService.login`): `userRepository.findByUsername(username)` → BCrypt verify → `JwtTokenProvider.generateToken(user, …)`.
- **Token** (`JwtTokenProvider`): already carries a **`tenantId` claim** (from `user.getTenantId()`), plus `username`/`sessionId`/`tokenVersion` — but nothing consumes the tenant claim.
- **Per request** (`JwtAuthenticationFilter`): `userId = claims.getSubject()` → `userRepository.findById(UUID)` → sets Spring `SecurityContext`. **It never binds the `core` `TenantContext`**, and the user is loaded **globally**.
- **Gateway** (`TenantContextFilter`, ADR-043, `@Order(2)`): binds `TenantContext` from the **`X-Tenant-ID` header** — fine for the *login* request, but **unauthenticated/spoofable** for everything else.
- **Type mismatch:** `TenantContext.tenantId` and the FI-ENT1-C discriminator are **String** (`__system__` sentinel); `UserEntity.tenantId` is **UUID**. FI-ENT1-D reconciles this (deferred from FI-ENT1-C).

### 0.2 What FI-ENT1-D (this slice) must achieve
1. **Reconcile** `UserEntity.tenantId` to a String `@TenantId` discriminator, so user rows are tenant-isolated by the FI-ENT1-C mechanism (ADR-054).
2. **Per-tenant identity:** `username`/`email` unique **within a tenant**, not globally (acme/alice ≠ beta/alice).
3. **Token-authoritative tenancy:** on an authenticated request, bind the `TenantContext` from the **JWT's `tenantId` claim** (trusted) — overriding the header — *before* loading the user, so the user is loaded tenant-scoped and a cross-tenant token cannot resolve a foreign user.

### 0.3 Scope — this slice vs. deferred
**In this slice:** `UserEntity` tenancy (type reconcile + `@TenantId` + `Tenanted`) · per-tenant unique constraints · `JwtAuthenticationFilter` token→`TenantContext` binding · login lookup tenant-scoped · migration V18.
**Deferred (FI-ENT1-D slice 2 / FI-ENT1-E):** `@TenantId` on the user satellites (`UserSessionEntity`, `ApiKeyEntity`, `PasswordHistoryEntity`); tenant-scoped API-key / service-account auth; per-tenant bootstrap-admin seeding; the `SecurityAuditEntity` tenant attribution.

### 0.4 / Decision for approval — the RBAC tenancy model (ADR decision)

| Option | Model | Trade-off |
|---|---|---|
| **A — Users tenant-scoped; Roles/Permissions a global catalog (recommended)** | `UserEntity` gets `@TenantId`; `RoleEntity`/`PermissionEntity`/`RolePermissionEntity` stay **global** (platform-defined roles like `ADMIN`/`USER`, shared across tenants). Tenant users are assigned global roles. | Smallest, safest change; matches most SaaS ("platform roles, tenant users"); the role catalog is a single source of truth. Tenants can't define bespoke roles yet (a later enhancement). |
| **B — Fully tenant-scoped RBAC** | Roles **and** permissions also carry `@TenantId`; each tenant owns its own role/permission definitions and mappings. | Maximally flexible (bespoke roles per tenant), but much larger: every RBAC table gets a discriminator + per-tenant seeding, the bootstrap/role-manager flows all fork by tenant, and the blast radius is big for a first slice. |

**Recommendation: A.** Tenant-scope the *identity* (the actual isolation boundary) now; keep roles/permissions a shared catalog. Per-tenant custom roles, if ever needed, is a clean follow-up on top of A.

> ✅ **Decision (confirmed 2026-07-30): Option A** — users tenant-scoped (`@TenantId`), roles/permissions a global catalog; the JWT is the authoritative tenant source for authenticated requests. To be recorded as **ADR-055** (number verified at implement).

---

## 1. Technical Design (Option A)

### 1.1 `UserEntity` becomes tenant-owned
- `tenantId` **UUID → String**, `@TenantId @Column(name="tenant_id", length=64, nullable=false, updatable=false)`, `implements Tenanted`. Reuses the ADR-054 resolver/customizer already registered in `core` — no new Hibernate wiring.
- Unique constraints move from global to **per-tenant**: `@Table(uniqueConstraints = { UNIQUE(tenant_id, username), UNIQUE(tenant_id, email) })`.
- `organizationId`/`workspaceId` unchanged (separate hierarchy).

### 1.2 Token-authoritative tenant binding (the security core)
`JwtAuthenticationFilter`, once it has validated the token:
```
String tokenTenant = claims.get("tenantId", String.class);   // trusted — it was signed
TenantContext ctx = (tokenTenant != null) ? TenantContext.ofTenant(tokenTenant) : TenantContext.system();
TenantContextHolder.run(ctx, () -> {          // authoritative for the authenticated leg; restores prev after
    UserEntity user = userRepository.findById(userId).orElse(null);   // now tenant-filtered by @TenantId
    // → a token for tenant A cannot load a user unless the row's tenant_id == A
    ... set Spring SecurityContext ...
    chain.doFilter(req, res);
});
```
- Binding **from the token** overrides any `X-Tenant-ID` the caller sent — the header is only authoritative on the *unauthenticated* login path.
- `TenantContextHolder.run(...)` (the existing scoped API) restores the previous context afterwards, so the gateway filter's `finally`-clear stays correct.
- **Consistency check:** because `findById` is now `@TenantId`-filtered, a mismatched token/tenant simply yields "user not found" → 401. (Optional hardening: also assert `user.getTenantId().equals(tokenTenant)` for a precise audit signal.)

### 1.3 Login path
`AuthenticationService.login` keeps `findByUsername(username)` — now auto-scoped to the tenant bound by the gateway `X-Tenant-ID` filter (the login request *must* name its tenant). With per-tenant uniqueness the lookup is unambiguous. No-tenant login (e.g. the `__system__` bootstrap admin) resolves against `__system__`. Token minted carries the (now String) `tenantId` claim.

## 2. Folder / module changes
- `security/rbac/UserEntity.java` — type reconcile + `@TenantId` + `Tenanted` + per-tenant unique constraints.
- `security/jwt/JwtAuthenticationFilter.java` — token→`TenantContext` binding around user load + chain.
- `security/jwt/JwtTokenProvider.java` — trivial (tenantId claim now a String; drop the UUID `.toString()`).
- `security/auth/AuthenticationService.java` — no logic change (lookup auto-scopes); comment the tenant expectation.
- `deployment/migration/db/migration/V18__user_tenant_reconcile.sql` (gateway-owned).

## 3. Required classes / edits (key)
| Class | Change |
|---|---|
| `UserEntity` | UUID→String `tenant_id`, `@TenantId`, `implements Tenanted`, per-tenant unique constraints |
| `JwtAuthenticationFilter` | bind `TenantContext` from token claim via `TenantContextHolder.run` before loading the user |
| `JwtTokenProvider` | `tenantId` claim as String |
| `V18__user_tenant_reconcile.sql` | alter column type + backfill + swap unique indexes |

## 4. Database changes — `V18__user_tenant_reconcile.sql`
```
-- reconcile the tenant discriminator to String and enforce per-tenant identity
ALTER TABLE security_users ALTER COLUMN tenant_id TYPE VARCHAR(64) USING COALESCE(tenant_id::text, '__system__');
ALTER TABLE security_users ALTER COLUMN tenant_id SET DEFAULT '__system__';
UPDATE security_users SET tenant_id = '__system__' WHERE tenant_id IS NULL;
ALTER TABLE security_users ALTER COLUMN tenant_id SET NOT NULL;
-- swap global unique → per-tenant unique
ALTER TABLE security_users DROP CONSTRAINT IF EXISTS security_users_username_key;
ALTER TABLE security_users DROP CONSTRAINT IF EXISTS security_users_email_key;
CREATE UNIQUE INDEX ux_users_tenant_username ON security_users (tenant_id, username);
CREATE UNIQUE INDEX ux_users_tenant_email    ON security_users (tenant_id, email);
CREATE INDEX ix_users_tenant ON security_users (tenant_id);
```
Exact existing constraint names verified against V2/V8 at implement (Postgres auto-names may differ).

## 5. API changes
**None to the contract.** Login still `POST /auth/login` with `X-Tenant-ID` naming the tenant; the token already carries the tenant. Behaviour tightens (cross-tenant tokens stop resolving), the shape does not.

## 6. Sequence (authenticated request)
```mermaid
sequenceDiagram
  participant C as Client (Bearer token)
  participant GF as Gateway TenantContextFilter (header)
  participant JF as JwtAuthenticationFilter
  participant H as TenantContextHolder
  participant DB as Postgres (@TenantId)
  C->>GF: request + X-Tenant-ID (untrusted)
  GF->>H: set(header tenant)  // provisional
  GF->>JF: continue
  JF->>JF: validate token → tenantId claim (trusted)
  JF->>H: run(token tenant) { … }   // authoritative, overrides header
  JF->>DB: findById(userId)  // filtered WHERE tenant_id = token tenant
  DB-->>JF: user (only if it belongs to the token's tenant)
  JF->>C: 401 if not found; else authenticated, tenant = token tenant
```

## 7. Implementation plan — small, verifiable tasks
1. **`UserEntity`** — type reconcile + `@TenantId` + `Tenanted` + per-tenant constraints. *Compiles here.*
2. **`JwtTokenProvider`** — String tenant claim. *Compiles here.*
3. **`JwtAuthenticationFilter`** — token→`TenantContext` binding via `TenantContextHolder.run`; **unit test** (token with tenant claim binds it; user loaded within it; context restored after). *Validatable here.*
4. **Migration V18** — authored; constraint names verified. *Applied by user.*
5. **Reactor green** — `mvn clean test -Djacoco.skip=true -DargLine="-Xss40m"`; watch `@DataJpaTest`/security tests (import the tenancy customizer if any slice touches `UserEntity`). *Validatable here.*
6. **Docs** — tracker ENT-1 note, **ADR-055**, this doc's Implementation Outcome.
7. **Isolation E2E (user-run, needs DB):** create `alice` in tenant `acme` and a different `alice` in `beta`; log into each (`X-Tenant-ID`), confirm each token only ever resolves its own user; a token from `acme` used against `beta`'s data → 401/empty.

**Definition of Done (this slice):** user identity is tenant-isolated and authentication is tenant-authoritative from the signed token; filter unit-proven; reactor green; migration authored. **ENT-1 stays In Progress** — user satellites, per-tenant roles (if ever), and FI-ENT1-E remain deferred.

**Honest boundary:** the mechanism (entity mapping, token→context binding, migration) is provable here; **the multi-tenant login/isolation E2E is user-run** (needs real Postgres with users in two tenants).

---

## Implementation Outcome

**Implemented 2026-07-30 (Option A — users tenant-scoped, roles global; JWT-authoritative tenant). Recorded as ADR-055. ENT-1 remains In Progress.**

**Delivered:**
- `UserEntity` — `tenantId` **UUID → String** + `@TenantId` + `implements Tenanted`; `@Table` per-tenant unique constraints (`ux_users_tenant_username`/`_email`); getter/setter → String.
- `JwtAuthenticationFilter` — binds `TenantContext` from the signed token's `tenantId` claim (`TenantContextHolder.set` → previous restored in `finally`) around the user load + downstream chain; auth logic factored into `authenticate(claims, request)`.
- `TenantContextFilter` (gateway) — yields to an already-bound tenant (`if current().isPresent() → proceed`), so the token tenant wins regardless of filter ordering and the untrusted header is ignored for authenticated calls.
- `JwtTokenProvider` — `tenantId` claim now a plain String.
- `V18__user_tenant_reconcile.sql` — column type reconcile (UUID→VARCHAR, NULL→`__system__`, NOT NULL + default), drop global-unique, add per-tenant unique + index.

**Verified here:** `JwtAuthenticationFilterTenantTest` **1/1** — the token's tenant is bound during the user load *and* the downstream chain, then restored; `JwtTokenProviderTest` 2/2 (String tenant); `TenantContextFilterTest` 4/4 (guard inert on the clean path); `RbacAdminAssemblerTest` 3/3 (type change safe); **full reactor `mvn clean test` BUILD SUCCESS, 22 modules, 0 failures** (8:02 min).

**Deviations from design:** added the `TenantContextFilter` "yield to bound tenant" guard (design named only `JwtAuthenticationFilter`). It makes correctness **independent of the servlet-filter ordering** between the security chain and the gateway filters — which is not guaranteed — so the token tenant is authoritative either way. No API/contract change.

**Honest boundary:** the mechanism (entity mapping, token→context binding, restore, migration) is proven here; **the multi-tenant login/isolation E2E is user-run** — two `alice`s in `acme` vs `beta`, each token resolving only its own user, a cross-tenant token → 401 (§7 step 7). Needs real Postgres.

**Deferred (ENT-1 stays In Progress):** user satellites (`UserSessionEntity`/`ApiKeyEntity`/`PasswordHistoryEntity`) + `SecurityAuditEntity` attribution (FI-ENT1-D slice 2); per-tenant custom roles (not in Option A); FI-ENT1-E (tenant-scoped memory/cost/artifact); FI-ENT1-C extension to the remaining tenant-owned tables.
