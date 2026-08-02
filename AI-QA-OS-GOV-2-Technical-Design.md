# GOV-2 — Compliance frameworks & dashboard · Technical Design

**Item:** GOV-2 (un-deferred at user request 2026-08-02) — SOC 2 / ISO 27001 / GDPR posture.
**Status:** design (awaiting decision + implement approval).
**Depends on:** GOV-1 (AI audit trail) — done.

---

## 0. Grounding — is there a faithful source? (yes)

The governance stack is largely built and **produced**:
- **GOV-1 (done):** AI audit trail (`AiAuditService`/`AiAuditTrail`/`AiAuditEvent`, gateway).
- **GOV-3 (done):** policy engine & Responsible-AI rules (intelligence).
- **GOV-4 (done):** model/prompt version registry with rollback (intelligence).
- Real audit producers: `SecurityAuditLogger` → `SecurityAuditEntity`; `ExecutionAuditEntity`.
- Security/enterprise controls: SEC-1 auth, SEC-2 secrets, SEC-4 CSP/transport, **SEC-6 artifact signing**, ENT-1 tenancy, ENT-4 RBAC admin.

**A compliance framework is a control catalog** — a matrix mapping *implemented capabilities* to framework requirements, with a status. That is **declarative, curated attestation** — not a runtime read-model that needs a data producer (so it is *not* the HEAL-3/LRN-3 producerless trap). The one discipline: every control's status must be an **honest** statement of a *real* implemented capability (SATISFIED only when the capability genuinely exists), and the output is clearly **self-attestation, not a formal certification**.

---

## 0.1 / Decision for approval — GOV-2 scope

- **Option A (recommended) — compliance control catalog + report read-model + endpoint.**
  A curated `ComplianceControlCatalog` (code) mapping the platform's implemented controls → SOC 2 / ISO 27001 / GDPR requirements, each with an honest status + evidence pointer; a pure `ComplianceAssembler` → per-framework `ComplianceReport` (coverage); and `GET /api/governance/compliance`. The compliance-dashboard **backend**, faithful + fully unit-testable. React UI + runtime-evidence automation = FIs → **GOV-2 In Progress**. → **ADR-081.**

- **Option B — docs-only compliance matrix.**
  A markdown controls matrix, no code. Honest but ships no dashboard backend.

**Recommendation: Option A.** A real, testable compliance backend (control catalog + coverage report) curated from genuinely-implemented controls, with the UI as a follow-on.

---

## 1. Technical Design (Option A) — `ai-qa-os-gateway`, `com.aiqaos.gateway.compliance` (beside GOV-1's audit)

### 1.1 New
- **`ComplianceFramework`** (enum): `SOC2`, `ISO_27001`, `GDPR`.
- **`ControlStatus`** (enum): `SATISFIED`, `PARTIAL`, `NOT_IMPLEMENTED`.
- **`ComplianceControl`** (record): `id`, `framework`, `requirementRef` (e.g. "CC6.1"), `title`, `satisfiedBy` (the platform capability, e.g. "SEC-1 JWT auth enforcement"), `status`, `evidence` (a note/pointer).
- **`ComplianceControlCatalog`** (`@Component`): the curated control set — each entry maps a **real** implemented capability to a framework requirement (SEC-1→CC6.1 access control, SEC-2→secrets, SEC-6→integrity, ENT-1→tenant isolation, GOV-1→audit trail, GOV-3→AI policy, GOV-4→change mgmt, …), status honest. Controls with no implementation are listed `NOT_IMPLEMENTED`/`PARTIAL`, never falsely SATISFIED.
- **`ComplianceReport`** (record): `generatedNote` (self-attestation disclaimer), `frameworks` = list of **`ComplianceFrameworkSummary`** (framework, total, satisfied, partial, coveragePercent, controls).
- **`ComplianceAssembler`** (pure): groups the catalog by framework → `ComplianceReport` (coverage = satisfied / total). The GOV-1/HEAL-3 assembler pattern.
- **`ComplianceController`**: `GET /api/governance/compliance` → `ComplianceReport` (read-only).

### 1.2 Faithfulness discipline
The catalog is **authored attestation of implemented controls**, not scraped runtime evidence. The report header states it is self-attestation, not certification. Only genuinely-shipped capabilities are `SATISFIED`.

## 2. Testing (Mockito-free)
- **`ComplianceControlCatalogTest`** — catalog non-empty; every control has an id/framework/status; no control claims `SATISFIED` without a `satisfiedBy` capability.
- **`ComplianceAssemblerTest`** — per-framework grouping; coverage math (satisfied/total) for a hand-built catalog; SOC2/ISO/GDPR each summarised.
- Full reactor `mvn clean test` green (21 modules); additive.

## 3. What can't be validated here
A formal external audit — this is a self-attested control matrix. Runtime-evidence automation (deriving control status from live audit data) is a follow-on.

## 4. Implementation plan
1. Enums + `ComplianceControl` + `ComplianceControlCatalog` (curated).
2. `ComplianceReport`/`ComplianceFrameworkSummary` + `ComplianceAssembler` + `ComplianceController`.
3. `ComplianceControlCatalogTest`, `ComplianceAssemblerTest`.
4. Full reactor verify.
5. Docs: ADR-081, tracker GOV-2 (Deferred → In Progress) + counts, this doc's Implementation Outcome.

## 5. Follow-on
- FI-GOV2-A: React compliance dashboard page (coverage bars per framework + control table).
- FI-GOV2-B: runtime-evidence automation — derive/confirm control status from live audit trails (GOV-1 / `SecurityAuditLogger`), turning attestation into evidenced status.
- FI-GOV2-C: export (auditor-facing SOC 2 / ISO control report).

---

## Implementation Outcome

**Delivered 2026-08-02 (Option A / ADR-081). Full reactor green — 21 modules, 0 failures. GOV-2 → In Progress (backend done; UI + evidence-automation are FIs).**

Shipped as designed (`ai-qa-os-gateway`, `com.aiqaos.gateway.compliance`):
- **`ComplianceFramework`** (SOC2/ISO_27001/GDPR), **`ControlStatus`** (SATISFIED/PARTIAL/NOT_IMPLEMENTED), **`ComplianceControl`** (record).
- **`ComplianceControlCatalog`** (`@Component`) — 20 curated controls across the three frameworks, honestly mapping shipped capabilities; gaps recorded PARTIAL/NOT_IMPLEMENTED (not all-green).
- **`ComplianceReport`** + **`ComplianceFrameworkSummary`** + **`ComplianceAssembler`** (pure, coverage = satisfied÷total) + **`ComplianceController`** `GET /api/governance/compliance`.

**Tests:** `ComplianceAssemblerTest` 3/3 (coverage math, per-framework grouping, requirementRef sort, empty/null) + `ComplianceControlCatalogTest` 4/4 (well-formed; SATISFIED⇒names a capability; all three frameworks present; contains honest gaps). Full reactor green (21 modules); additive.

**Deviations:** none. Coverage counts only SATISFIED (partials reported separately) so the headline is honest.

**Honesty:** self-attestation, not certification; statuses are curated from real shipped items (no false SATISFIED — enforced by a test). Deriving status from live audit evidence is FI-GOV2-B.

**Follow-on:** FI-GOV2-A (React compliance page — coverage bars + control table), FI-GOV2-B (runtime-evidence automation from GOV-1 audit / `SecurityAuditLogger`), FI-GOV2-C (auditor-facing export).
