# AGT-2 — Agent collaboration, lifecycle & marketplace · Technical Design

**Item:** AGT-2 (un-deferred at user request 2026-08-02) — multi-agent org under mediator rules.
**Status:** design (scoped to the collaboration-mediator foundation + architecture) · **Vision · v3.0**.

---

## 0. Grounding — what exists, what's novel

The agent runtime is substantial (`ai-qa-os-agents-runtime`):
- **Lifecycle:** `AgentLifecycleManager`, `AgentLifecycleEvent`, `AgentRuntimeManager`, `AgentHealthMonitor` — **exists**.
- **Communication:** `AgentMessageBus`, `AgentMessageEntity` — **exists**.
- **Security:** `AgentPermissionMatrix`, `AgentExecutionBudget`, `RuntimeSecurityManager` — **exists**.
- **Marketplace/catalog:** AGT-1's `AgentRoster` (all/byCategory/byStatus/find/coverage) already catalogues agents; PLG-4's `PluginCatalog` is the discovery pattern.

So of AGT-2's three parts, **lifecycle and marketplace largely exist**. The genuinely-missing, novel piece is **collaboration "under mediator rules"** — a governed decision on whether one agent may collaborate with (delegate to) another. That is the bounded, faithful slice; the full multi-agent org is Vision v3.0.

**Scope decision (consistent with PLG-4/BRAIN-1):** build the **collaboration mediator foundation** + record the multi-agent-org architecture. AGT-2 → In Progress. → **ADR-083.**

---

## 1. Technical Design — `ai-qa-os-agents-runtime`, `com.aiqaos.runtime.collaboration`

### 1.1 New
- **`CollaborationRequest`** (record): `requesterId`, `requesterCapabilities` (`Set<String>`), `targetId`, `targetCapabilities` (`Set<String>`), `requestedCapability`.
- **`CollaborationDecision`** (record): `allowed`, `reason` + `allow()` / `deny(reason)`.
- **`CollaborationPolicy`** (`@Component`): the **mediator rules** — a set of **privileged** capabilities (e.g. `execute.privileged`, `modify.governance`) that require the requester to hold a granting `supervisor` capability. `isPrivileged(cap)`.
- **`CollaborationMediator`** (`@Component`): `mediate(CollaborationRequest)` → `CollaborationDecision`, applying, in order:
  1. **Well-formedness** — blank ids / capability → deny.
  2. **No self-collaboration** — requester == target → deny.
  3. **Capability availability** — `requestedCapability` must be in `targetCapabilities` → else deny ("target does not provide …").
  4. **Privilege gate** — if the capability is privileged, the requester must hold the `supervisor` capability → else deny (prevents privilege escalation across agents).
  5. Otherwise **allow**.

This is the "mediator rules" the item names: a governed, stateless collaboration decision reusing the platform's capability + permission vocabulary. In the runtime, `AgentMessageBus` / delegation would consult the mediator before routing a cross-agent request (FI-AGT2-A).

### 1.2 Architecture (ADR-083, recorded — not built = v3.0)
The multi-agent org: an **agent marketplace** (extend `AgentRoster`/`PluginCatalog` with published agents), **collaboration protocols** over `AgentMessageBus` gated by the mediator, **lifecycle** graduation (via `AgentLifecycleManager`), governance guardrails (permission matrix, execution budgets), and an **org topology** (mediator-coordinated roles). This slice delivers the mediator the org hangs on.

## 2. Testing (Mockito-free)
- **`CollaborationMediatorTest`** — allow (target provides a non-privileged capability); deny self-collaboration; deny when the target doesn't provide the capability; deny a privileged capability without `supervisor`; **allow** a privileged capability **with** `supervisor`; deny blank fields.
- Full reactor `mvn clean test` green (21 modules); additive.

## 3. What can't be validated here
The full multi-agent org / marketplace (v3.0). The mediator decision logic is fully unit-verified; wiring it into live agent delegation is FI-AGT2-A.

## 4. Implementation plan
1. `CollaborationRequest`, `CollaborationDecision`, `CollaborationPolicy`, `CollaborationMediator`.
2. `CollaborationMediatorTest`.
3. Full reactor verify.
4. Docs: ADR-083 (mediator + org architecture), tracker AGT-2 (Deferred → In Progress) + counts, this doc's Implementation Outcome.

## 5. Follow-on (the v3.0 org)
- FI-AGT2-A: wire the mediator into `AgentMessageBus` delegation (consult before routing a cross-agent request).
- FI-AGT2-B: agent marketplace (published agents) extending `AgentRoster`/`PluginCatalog`.
- FI-AGT2-C: collaboration protocols + org topology (mediator-coordinated multi-agent roles).

---

## Implementation Outcome

**Delivered 2026-08-03 (ADR-083). Full reactor green — 21 modules, 0 failures. AGT-2 → In Progress (mediator + architecture; full org is v3.0).**

Shipped as designed (`ai-qa-os-agents-runtime`, `com.aiqaos.runtime.collaboration`):
- **`CollaborationRequest`** / **`CollaborationDecision`** (records), **`CollaborationPolicy`** (`@Component`, privileged capabilities + `supervisor` grant), **`CollaborationMediator`** (`@Component`, ordered rules → decision).

**Tests:** `CollaborationMediatorTest` 6/6 — allow non-privileged; deny self-collaboration; deny missing-capability; deny privileged-without-supervisor; **allow privileged-with-supervisor**; deny blank/null. Full reactor green (21 modules); additive.

**Deviations:** `agents-runtime` had **no test dependency** — added `spring-boot-starter-test` (test scope) so the mediator test compiles (its first unit test).

**Scope honesty:** the mediator is an **unwired governance seam** (nothing consults it in live delegation yet); the agent marketplace, collaboration protocols, and org topology are v3.0. So **AGT-2 stays In Progress**.

**Follow-on (the v3.0 org):** FI-AGT2-A (consult the mediator in `AgentMessageBus` delegation), FI-AGT2-B (agent marketplace extending `AgentRoster`/`PluginCatalog`), FI-AGT2-C (collaboration protocols + mediator-coordinated org topology).
