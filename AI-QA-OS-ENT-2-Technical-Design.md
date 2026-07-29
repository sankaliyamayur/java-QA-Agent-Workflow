# ENT-2 — Technical Design: Centralised Notifications (Event Routing over MOD-2)

**Version:** 1.0.0
**Document Type:** Implementation Technical Design
**Document Status:** Implemented — 2026-07-29 (§0.4 = A routing+templating; see [Implementation Outcome](#implementation-outcome))
**Last Updated:** 2026-07-29
**Roadmap item:** [`ENT-2`](./AI-QA-OS-Improvement-Roadmap.md#ent-2--centralised-notifications) (v2.2.0, frozen) — 🟡 P2 · Effort M · Owner Integration · Phase 4 · v2.1
**Modules:** `ai-qa-os-notification` (event routing + templating over MOD-2).
**Depends on:** MOD-2 (`NotificationService` egress — ✅) · AI-2 (approval flow this feeds — ✅).

> **Scope discipline.** ENT-2 gives stakeholders **run-complete, failure, and approval-request** notifications through the one governed egress point (MOD-2), with **templating**. It's the layer that turns a platform event into a templated, severity-ranked, channel-routed notification and dispatches it — the piece that lets AI-2's approval flow actually reach a human. The **event → notification routing + templating is fully validatable**; live `@EventListener` wiring to the real event publishers and delivery-retry guarantees are deferred (§0.3).

---

## 0. Roadmap Verification & Scope

### 0.1 What ENT-2 requires
> Outbound comms are scattered; stakeholders need **run-complete, failure, and approval-request** notifications through **one governed channel with templating and delivery guarantees**. **Where:** the `ai-qa-os-notification` module (MOD-2) consuming events, depending only on `core` + `integration`. **Impact:** prerequisite for AI-2's human-in-the-loop approval to actually reach a human.

### 0.2 Verified state
- MOD-2 delivered the egress point: `NotificationService` routes a `Notification` to a channel sender (Slack via the PLG-2 adapter; email/webhook/teams). ✅
- Event publishers exist (`WorkflowEventPublisher`, `SecurityEventPublisher`, `PlatformEventBus`, …) — but **nothing turns a platform event into a templated notification**.

### 0.3 Environment reality
- **Validatable now:** a `NotificationEvent` model (RUN_COMPLETE / RUN_FAILURE / APPROVAL_REQUEST), **per-type templating** (subject/body/severity/default-channel), and a `NotificationEventRouter` that builds the notification and dispatches it via MOD-2's `NotificationService`. Pure routing logic.
- **Deferred:** live `@EventListener` wiring to the real event publishers (cross-module event sources — FI-ENT2-A); delivery-retry/guarantee semantics beyond the `NotificationResult` MOD-2 returns (FI-ENT2-B); scheduled digests.

### 0.4 / Decision for approval — scope

| Option | Approach | Trade-off |
|---|---|---|
| **A — Event routing + templating over MOD-2 now; live event wiring deferred (recommended)** | `NotificationEvent` + `NotificationEventRouter` (template + severity + channel per type) dispatching through `NotificationService`. Live `@EventListener`s + retry deferred. | Delivers the centralised notification capability for the three named event types, fully validatable; respects MOD-2's egress. Doesn't yet fire from real events or guarantee redelivery. |
| **B — Also wire live `@EventListener`s + delivery retry now** | Subscribe to `WorkflowEventPublisher`/etc. and add a retry/outbox. | Truer end-to-end, but cross-module event wiring + an outbox need runtime, largely unvalidatable here. |

**Recommend A** — deliver the centralised routing + templating (the substance) now; subscribing to the real event streams + delivery guarantees are the runtime wiring (FI-ENT2-A/B).

> ✅ **Decision (confirmed 2026-07-29): Option A — `NotificationEvent` + `NotificationEventRouter` (per-type templating + severity + channel) dispatching through MOD-2's `NotificationService`; live `@EventListener` wiring + delivery retry deferred (FI-ENT2-A/B).** Recorded as ADR-049 (number verified at implement).

---

## 1. Technical Design (Option A)

`ai-qa-os-notification`, package `com.aiqaos.notification.event`:
- **`NotificationEventType`** — `RUN_COMPLETE`, `RUN_FAILURE`, `APPROVAL_REQUEST`.
- **`NotificationEvent`** — `type`, `subjectRef` (e.g. workflow id), `summary`, `recipient`, `channel` (nullable → the type's default).
- **`NotificationEventProperties`** — `aiqaos.notification.default-channel` (default `SLACK`).
- **`NotificationEventRouter`** (`@Service`) — `route(NotificationEvent)`: apply the per-type template → a MOD-2 `Notification` (subject/body, severity: `RUN_FAILURE`→CRITICAL, `APPROVAL_REQUEST`→WARNING, `RUN_COMPLETE`→INFO; channel = event's or the default), then dispatch via `NotificationService`; returns the `NotificationResult`.

**Defers:** live `@EventListener` subscriptions to real publishers (FI-ENT2-A) · delivery retry/outbox (FI-ENT2-B) · scheduled digests.

---

## 2. Folder Structure
```
ai-qa-os-notification/.../event/
    NotificationEventType.java     [N] RUN_COMPLETE / RUN_FAILURE / APPROVAL_REQUEST
    NotificationEvent.java         [N] type + subjectRef + summary + recipient + channel
    NotificationEventProperties.java [N] aiqaos.notification.default-channel
    NotificationEventRouter.java   [N] @Service: template + route → NotificationService
+ unit tests: router (each type → severity/subject; channel override vs default; delivered result).
```

---

## 3–5. Classes / DB / API
Key classes above. **DB:** none. **API:** none (fired by events; live subscription deferred).

---

## 6. Sequence
```mermaid
flowchart TD
    E["platform event (run complete / failed / approval needed)"] --> N["NotificationEvent(type, ref, summary, recipient)"]
    N --> R["NotificationEventRouter.route(...)"]
    R --> T["template → Notification (subject/body, severity, channel)"]
    T --> S["MOD-2 NotificationService.notify(...)"]
    S --> D["NotificationResult (delivered / failed)"]
    E -. deferred: live @EventListener on WorkflowEventPublisher/etc. (FI-ENT2-A) .-> N
```

---

## 7. Plan
1. **Model** — `NotificationEventType`, `NotificationEvent`, `NotificationEventProperties`.
2. **Router** — `NotificationEventRouter` (per-type template + severity + channel; dispatch via `NotificationService`).
3. **Tests** — each type → correct severity + subject; channel override vs type default; result propagates; unknown-channel graceful (via MOD-2). Recording `NotificationSender` doubles; no Mockito.
4. **Build & validate** — targeted `mvn -pl ai-qa-os-notification -am test -Dtest=… -Djacoco.skip=true -DargLine="-Xss40m"`; green; MOD-2 tests unaffected.
5. **Sync docs** — tracker `ENT-2` → Completed; **ADR-049** (centralised event→notification routing + templating over MOD-2; live event wiring deferred). Verify number at implement.

**DoD:** a run-complete/failure/approval-request event is templated (subject/body + severity), routed to a channel, and dispatched through MOD-2's egress point — unit-proven. **Deferred:** live event-source subscriptions, delivery retry, digests.

---

## Implementation Outcome

Implemented 2026-07-29 (§0.4 = A — event routing + templating over MOD-2). Recorded as **ADR-049**. **ENT-2 Completed.**

**Files (all new, `ai-qa-os-notification/.../event/`):**
- `NotificationEventType` (RUN_COMPLETE/RUN_FAILURE/APPROVAL_REQUEST), `NotificationEvent` (type/subjectRef/summary/recipient/channel), `NotificationEventProperties` (`aiqaos.notification.default-channel`=SLACK), `NotificationEventRouter` (`@Service`: per-type template + severity + channel resolution → dispatch via MOD-2 `NotificationService`).

**Validation (Maven):** `mvn -pl ai-qa-os-notification -am test` → **BUILD SUCCESS**; `NotificationEventRouterTest` **5/5** (RUN_FAILURE→CRITICAL/"FAILED", APPROVAL_REQUEST→WARNING/"Approval required", RUN_COMPLETE→INFO, event-channel overrides default, null→graceful failure). MOD-2's `NotificationServiceTest` 5/5 unaffected. Ran with `-Djacoco.skip=true -DargLine="-Xss40m"`. Single-constructor `@Service` — no wiring traps.

**Honest scope note:** the **event→notification routing + templating are unit-proven**. **Deferred:** live `@EventListener` subscriptions to the real publishers / AI-2 review events (FI-ENT2-A); delivery retry/outbox beyond MOD-2's synchronous result (FI-ENT2-B); scheduled digests.

---

## Appendix — Future Ideas
- **FI-ENT2-A** — Live `@EventListener` subscriptions to `WorkflowEventPublisher`/`SecurityEventPublisher`/AI-2's review events, mapping them to `NotificationEvent`s.
- **FI-ENT2-B** — Delivery guarantees (retry / outbox) beyond MOD-2's synchronous `NotificationResult`.

---

## Compliance Checklist
| Rule | Status |
|---|---|
| Roadmap not modified | ✅ ENT-2 metadata untouched |
| In `ai-qa-os-notification` (core + integration) | ✅ builds on MOD-2 |
| Reaches human for approvals (AI-2) | ✅ APPROVAL_REQUEST routing (live wiring deferred) |
| Non-breaking | ✅ additive; MOD-2 untouched |
| Spring-wiring sanity | ✅ single-constructor `@Service` (per 2026-07-29 lesson) |
| ADR discipline | ✅ ADR-049 to be recorded (number verified at implement) |

---

## Document Completion Status
**Status:** Implemented — 2026-07-29 (§0.4 = A). ENT-2 Completed. See [Implementation Outcome](#implementation-outcome). ADR-049.
**Implements:** `ENT-2` (roadmap v2.2.0, frozen) — centralised event→notification routing; live wiring deferred.
**Next step:** On approval + option choice, execute [§7](#7-step-by-step-implementation-plan) from Step 1.
