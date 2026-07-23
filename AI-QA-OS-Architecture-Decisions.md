# AI-QA-OS — Architecture Decision Records (ADRs)

**Version:** 1.0.0
**Document Type:** Architecture Decision Log
**Document Status:** Active
**Last Updated:** 2026-07-22
**Source of truth:** [`AI-QA-OS-Documentation.md`](./AI-QA-OS-Documentation.md) · [`AI-QA-OS-Improvement-Roadmap.md`](./AI-QA-OS-Improvement-Roadmap.md) (finalized)

> **Purpose.** Record the *why* behind AI-QA-OS's major architectural decisions so future contributors do not re-litigate settled questions or unknowingly violate them. These ADRs **document existing decisions** — they do not introduce new architecture. Where a decision is realised or extended by a roadmap item, the ID is linked; implementation status lives in [`AI-QA-OS-Implementation-Tracker.md`](./AI-QA-OS-Implementation-Tracker.md).

---

## ADR Format & Conventions

Each record uses a standard ADR structure:

- **Status** — Accepted (implemented) · Accepted — Planned (approved, delivered via a roadmap item) · Superseded
- **Context** — the forces and constraints that made a decision necessary
- **Decision** — what was chosen
- **Consequences** — positive, negative, and the rules the decision imposes
- **Related** — roadmap items and architecture rules

Two platform-wide rules recur throughout and are cited by ID:

- **Mediator Rules** (from `AI-QA-OS-Documentation.md` §2.9): agents never talk to each other, to the Brain, or to the Workflow Engine directly; `Agent → Agent Manager → Agent`, and all decisions pass through QA Brain.
- **Dependency Rule** (§2.14): every module depends inward toward `core`; nothing depends on the entry-point apps.

### Index

| ADR | Title | Status |
|---|---|---|
| [ADR-001](#adr-001--the-qa-brain-as-central-decision-authority) | QA Brain as central decision authority | Accepted |
| [ADR-002](#adr-002--the-workflow-engine-as-pipeline-orchestrator) | Workflow Engine as pipeline orchestrator | Accepted |
| [ADR-003](#adr-003--the-memory-engine-as-a-tiered-experience-store) | Memory Engine as a tiered experience store | Accepted |
| [ADR-004](#adr-004--the-gateway-as-the-single-public-entry-point) | Gateway as the single public entry point | Accepted |
| [ADR-005](#adr-005--the-agent-manager-as-mandatory-mediator) | Agent Manager as mandatory mediator | Accepted |
| [ADR-006](#adr-006--self-healing-as-a-first-class-autonomous-capability) | Self-Healing as a first-class autonomous capability | Accepted — Planned |
| [ADR-007](#adr-007--ai-governance-as-a-cross-cutting-control-plane) | AI Governance as a cross-cutting control plane | Accepted — Planned |
| [ADR-008](#adr-008--multi-tenancy-as-a-core-context-dimension) | Multi-Tenancy as a core context dimension | Accepted — Planned |
| [ADR-009](#adr-009--decision--roadmap-honesty-discipline) | Decision & roadmap-honesty discipline | Accepted |
| [ADR-010](#adr-010--confidence-gate-contract-in-core-implementation-in-the-brain) | Confidence gate: contract in core, impl in the Brain | Accepted |
| [ADR-011](#adr-011--human-review-in-memory-resume-on-the-gateway-durable-queue-in-the-db) | Human review: in-memory resume (gateway) + durable queue (DB) | Accepted |
| [ADR-012](#adr-012--a-dedicated-evaluation--guardrails-module-with-contract-seams) | A dedicated evaluation & guardrails module with contract seams | Accepted |
| [ADR-013](#adr-013--prompt-regression-baseline-as-a-committed-file) | Prompt regression baseline as a committed file | Accepted |
| [ADR-014](#adr-014--two-tier-prompt-eval-ci-gate-deterministic-always-on--live-key-gated) | Two-tier prompt-eval CI gate: deterministic always-on + live key-gated | Accepted |
| [ADR-015](#adr-015--guardrail-contract-promoted-to-core-the-ai-boundary-control-point) | Guardrail contract promoted to core; the AI-boundary control point | Accepted |
| [ADR-016](#adr-016--centralised-hardened-security-headers--download-only-html-artifacts) | Centralised hardened security headers + download-only HTML artifacts | Accepted |
| [ADR-017](#adr-017--execution-decoupled-behind-a-job-queue--artifact-store-seam) | Execution decoupled behind a job-queue + artifact-store seam | Accepted — Partial |
| [ADR-018](#adr-018--artifact-object-storage-via-a-client-seam--opt-in-age-retention) | Artifact object storage via a client seam + opt-in age retention | Accepted — Partial |
| [ADR-019](#adr-019--modulepackage-naming-convention--mdc-correlation-id-propagation) | Module↔package naming convention + MDC correlation-id propagation | Accepted |
| [ADR-020](#adr-020--raw-otel-tracing-config-gated-export--span-instrumentation) | Raw-OTel tracing: config-gated export + span instrumentation | Accepted — Partial |

---

## ADR-001 — The QA Brain as central decision authority

**Status:** Accepted (implemented — `ai-qa-os-brain`)

**Context.**
An autonomous QA platform makes many judgment calls per run: which strategy to use, which agents to invoke, whether an output is trustworthy, when to stop. If each agent or workflow step made those calls locally, decision logic would scatter across the codebase, become impossible to audit, and drift between agents. The platform's "AI First" principle requires that major decisions be *reasoned*, and reasoned decisions need one accountable place.

**Decision.**
Introduce a single central intelligence — the **QA Brain** (`ai-qa-os-brain`) — through which **all** AI decisions pass (Mediator Rule 2). The Brain owns strategy selection, agent selection, execution context, and (per the roadmap) confidence scoring and cost/quality trade-offs. Agents and steps request decisions; they do not make platform-level decisions themselves.

**Consequences.**
- *Positive:* one auditable decision point; consistent behaviour across agents; a natural home for the confidence gate, learning governance, and future reasoning sophistication.
- *Positive:* enables the staged Brain evolution ([BRAIN-1](./AI-QA-OS-Improvement-Roadmap.md)) without touching agents.
- *Negative / trade-off:* the Brain is a hot path and a potential bottleneck; it must stay stateless-per-request and horizontally scalable. It is currently skeletal, so the decision is real but under-realised.
- *Imposed rule:* no component may embed platform decision logic that belongs in the Brain.

**Related:** AI-1 (confidence gate), AI-6 (cost budgeting), LRN-4 (safe-adoption gate), ENT-3 (cost governance), BRAIN-1 (evolution). Mediator Rule 2.

---

## ADR-002 — The Workflow Engine as pipeline orchestrator

**Status:** Accepted (implemented — `ai-qa-os-orchestration`, package `com.aiqaos.workflow`)

**Context.**
Turning a requirement into a tested, reported outcome is a long, multi-stage process (read → analyse → generate cases → generate scripts → execute → heal → analyse bugs → report → learn). Something must sequence these stages, carry state between them, support pause/resume/cancel for long-running autonomy, and remain declarative enough to change without rewriting agents.

**Decision.**
Provide a dedicated **Workflow Engine** (`AutonomousQAPipelineOrchestrator` + `WorkflowStateMachine` + `WorkflowDslParser`) that owns step sequencing and run lifecycle. Each step delegates to exactly one agent; the engine — not the agents — controls order and state.

**Consequences.**
- *Positive:* the pipeline is reconfigurable; new steps slot in without agent changes; pause/resume enables the human-in-the-loop and approval workflows.
- *Positive:* a single place to enforce "validate before proceeding" (Mediator Rule 3).
- *Negative / trade-off:* synchronous, in-process orchestration limits scale and resilience until the event bus lands.
- *Imposed rule:* agents must not sequence or invoke other agents (Mediator Rule 1); orchestration is the engine's job.

**Related:** AI-2 (human-in-the-loop via pause), WF-1/WF-2 (new workflows), SCALE-2 (event bus), MNT-6 (package rename to match module). Mediator Rules 1 & 3.

---

## ADR-003 — The Memory Engine as a tiered experience store

**Status:** Accepted (implemented — `ai-qa-os-memory`)

**Context.**
The platform's value compounds only if it remembers. Every execution produces context (what worked, what failed, which locators, which prompts) that should inform future runs. This needs fast working memory, durable long-term memory, and semantic retrieval over unstructured experience — and it must be reusable across agents without each re-implementing storage.

**Decision.**
Provide a shared **Memory Engine** (`ai-qa-os-memory`) with tiered stores — Caffeine (short-term), Redis/relational (long-term), and a vector store (semantic) — plus chunking, ranking, retrieval, and ingestion. All learning and retrieval funnel through it (Mediator Rule 4: all execution data stored for future learning).

**Consequences.**
- *Positive:* one substrate powers RAG, the learning loop, the semantic cache, and healing memory.
- *Positive:* agents depend on `memory` (an inward dependency) rather than owning persistence.
- *Negative / trade-off:* five vector-store clients are more surface than needed (addressed by SCALE-3); retrieval quality is unmeasured until the eval work lands.
- *Imposed rule:* execution outcomes must be written back to memory, not discarded.

**Related:** AI-4 (semantic cache), LRN-1 (learning loop), HEAL-4 (AI healing memory), SCALE-3 (standardise vector store), PERF-2 (batch embeddings). Mediator Rule 4.

---

## ADR-004 — The Gateway as the single public entry point

**Status:** Accepted (implemented — `ai-qa-os-gateway`, :8082)

**Context.**
Multiple clients (dashboard UI, CLI, PowerShell scripts, webhooks, CI) need to start workflows, query status, and control runs. Exposing every internal service directly would multiply the attack surface, scatter cross-cutting concerns (auth, rate limiting, validation), and couple clients to internal structure.

**Decision.**
Front the platform with a single **Gateway** (`ai-qa-os-gateway`) that owns the public REST API (`/api/v1/**`), WebSocket streaming, webhook intake, and the CLI runner, and applies cross-cutting filters (JWT auth, rate limiting) at the edge. Internal services are not publicly exposed.

**Consequences.**
- *Positive:* one place to enforce authentication, rate limiting, and API versioning; clients decouple from internal topology.
- *Positive:* a clear seam for the approval API and CI-triggered workflows.
- *Negative / trade-off:* the Gateway is a single point of failure and must scale horizontally; today its permit-all/ignore configuration bypasses the very controls it exists to enforce (the reason SEC-1 is a P0 blocker).
- *Imposed rule:* new public capability is exposed through the Gateway, not by opening internal services.

**Related:** SEC-1 (restore auth — critical), SEC-4 (CSP), AI-2 (approval endpoint), WF-2 (CI triggers), DX-1 (CLI). Dependency Rule (nothing depends on the Gateway).

---

## ADR-005 — The Agent Manager as mandatory mediator

**Status:** Accepted (design mandate; enforcement pending)

**Context.**
The platform is designed as a team of single-responsibility agents ("one agent = one responsibility"). If agents could call each other directly, the system would degenerate into a tangle of hidden couplings, circular dependencies, and un-auditable AI-to-AI conversations — the opposite of the modular, replaceable, observable design the Blueprint requires.

**Decision.**
Mandate that agents communicate **only** through an **Agent Manager** mediator: `Agent → Agent Manager → Agent`. Direct `Agent → Agent`, `Agent → QA Brain`, and `Agent → Workflow Engine` calls are forbidden (Mediator Rule 1). The agent runtime (`ai-qa-os-agents-runtime`) models the lifecycle and messaging that make this possible.

**Consequences.**
- *Positive:* agents stay independent and replaceable; collaboration is observable and governable; no dependency cycles.
- *Positive:* enables a future multi-agent "organisation" and an agent marketplace without relaxing coupling rules.
- *Negative / trade-off:* the mediator is currently a convention, not a compiler-enforced boundary — a real gap until the architecture validator lands.
- *Imposed rule:* the mediator path is non-negotiable; violations are build failures once DX-6 is in place.

**Related:** DX-6 (architecture validator to enforce), AGT-2 (agent collaboration/marketplace), AGT-1 (expanding roster). Mediator Rule 1.

---

## ADR-006 — Self-Healing as a first-class autonomous capability

**Status:** Accepted — Planned (baseline `ai-qa-os-healing` exists; full loop via roadmap Category O)

**Context.**
UI locator drift is the dominant cause of flaky, high-maintenance automation and the biggest recurring human cost in traditional frameworks. A platform claiming autonomous QA cannot treat healing as a mere retry wrapper — the differentiator is *recovering* from drift, *learning* the fix, and *preventing* recurrence.

**Decision.**
Treat self-healing as a first-class, governed capability: a closed loop (detect → AI-analyse → find better locator → validate → retry → update script → update memory → future auto-recovery), gated by a healing confidence score and, when low-confidence, a human approval workflow, with a dedicated AI Healing Memory for cross-run recovery. It reuses the Brain's confidence gate and the approval workflow rather than inventing parallel machinery.

**Consequences.**
- *Positive:* converts test maintenance from a human cost into an autonomous capability — a flagship selling point.
- *Positive:* healing knowledge compounds via memory, exactly like the general learning loop.
- *Negative / trade-off:* auto-editing test scripts is high-risk; without the confidence gate and approval flow it can silently mask real bugs. Hence the strict dependency on AI-1/AI-2.
- *Imposed rule:* no autonomous script edit without a confidence score and, below threshold, human approval.

**Related:** HEAL-1/2/3/4, AI-1 (confidence), AI-2 (approval), LRN-1 (learning loop), ENT-1 (tenant-scoped healing memory).

---

## ADR-007 — AI Governance as a cross-cutting control plane

**Status:** Accepted — Planned (roadmap Category N)

**Context.**
An autonomous system that makes decisions, spends money on LLM calls, and touches potentially sensitive data must be auditable and policy-governed to be trusted — and to be sellable into regulated enterprises. Per-domain logging (security events, prompt executions, cost records) exists but is not unified, and there is no policy enforcement over *AI behaviour*.

**Decision.**
Establish a cross-cutting **AI Governance** control plane: a unified AI audit trail (prompt/model/cost/approval + version history), a policy engine enforcing Responsible AI rules at the confidence-gate and guardrail boundaries (reusing the existing OPA engine), a model/prompt version registry with rollback, and compliance reporting (GDPR/SOC 2/ISO 27001). Governance composes over existing controls rather than replacing them.

**Consequences.**
- *Positive:* every autonomous action becomes reconstructable and policy-bounded; unblocks enterprise/regulated adoption.
- *Positive:* policy-as-code lets the platform say "never target production," "require approval for destructive actions," "block PII prompts."
- *Negative / trade-off:* governance touches many modules (security, intelligence, ai-provider, brain); it must be assembled carefully to avoid becoming a bottleneck. Full compliance mapping is deferred (GOV-2).
- *Imposed rule:* autonomous actions are subject to policy evaluation and are recorded in the audit trail.

**Related:** GOV-1/2/3/4, AI-1 (gate), SEC-3 (guardrails), MOD-4 (PII masking), AI-6/ENT-3 (cost governance).

---

## ADR-008 — Multi-Tenancy as a core context dimension

**Status:** Accepted — Planned (roadmap ENT-1 / MOD-1)

**Context.**
The founding philosophy is "Build Once, Use Everywhere — multiple projects consume the same platform." Today both runnable apps share one database with no tenant boundary; one project's data, secrets, cost, and executions are not isolated from another's. Retrofitting isolation after data accumulates is far more expensive than designing it in.

**Decision.**
Model tenancy as a first-class **context dimension** carried on every request. The tenant/project *context contract* lives in `core` (so every module can reference it without a dependency cycle); a dedicated `ai-qa-os-tenant` module (`MOD-1`) owns tenant identity; enforcement threads through persistence, security (tenant-scoped RBAC), memory (tenant-scoped retrieval), cost, and the artifact store.

**Consequences.**
- *Positive:* realises the platform's core premise; enables per-tenant cost quotas, isolated memory/healing, and a marketplace.
- *Positive:* placing the contract in `core` keeps the dependency graph acyclic while letting every module honour tenancy.
- *Negative / trade-off:* the largest structural change on the roadmap; enforcement must land incrementally, and every persistence and retrieval path must be tenant-aware — a broad, cross-cutting effort (XL).
- *Imposed rule:* new persistent or retrievable data must carry tenant scope once the contract exists.

**Related:** ENT-1, MOD-1, ENT-3 (per-tenant quotas), SCALE-4 (DB separation), HEAL-4 (tenant-scoped healing memory), PLG-4 (multi-tenant marketplace). Dependency Rule (contract in `core`).

---

## ADR-009 — Decision & roadmap-honesty discipline

**Status:** Accepted (process; established by MNT-5, 2026-07-22)

**Context.**
Design intent and actual state drifted badly: ~20 modules were implemented while the docs-repo build roadmap (`AI-QA-OS-Docs/docs/roadmap/00-Project-Roadmap.md`) still marked every phase and step "Not Started," and `docs/decisions/` sat empty. Without a recorded discipline, every future divergence is invisible and the documentation stops being trustworthy — the exact failure MNT-5 was raised to fix.

**Decision.**
Adopt a single, explicit decision-and-honesty discipline:
- **ADRs** record significant, hard-to-reverse decisions in **one canonical log** (`AI-QA-OS-Architecture-Decisions.md`); `docs/decisions/` indexes and points to it. ADRs are immutable-append — a changed decision is a new *superseding* ADR, never an edit to history.
- **Live status** is `AI-QA-OS-Implementation-Tracker.md`, mirroring the frozen `AI-QA-OS-Improvement-Roadmap.md`'s per-item `Status`, updated as work completes.
- **The build roadmap is reconciled, not fabricated** — where per-step statuses were never maintained, it is marked stale and points to the authoritative sources rather than inventing Done/Not-Started markers.

**Consequences.**
- *Positive:* drift stays visible; the doc set (platform doc, improvement roadmap, tracker, release plan, ADRs, build roadmap) stays mutually consistent and trustworthy.
- *Positive:* new contributors get one honest source per concern instead of contradictory artifacts.
- *Negative / trade-off:* requires discipline on every change — the tracker and ADR log are part of "done," not an afterthought.
- *Imposed rule:* completing a roadmap item includes updating the tracker; an architecture-affecting change includes an ADR.

**Related:** MNT-5; `AI-QA-OS-Implementation-Tracker.md`; the improvement roadmap's `Status` fields; `AI-QA-OS-Docs/docs/decisions/README.md` (index).

---

## ADR-010 — Confidence gate: contract in core, implementation in the Brain

**Status:** Accepted (implemented — AI-1, 2026-07-22)

**Context.**
AI-1 requires a confidence gate **owned by the QA Brain** (Rule 2 / ADR-001) but **evaluated after each agent output in the orchestration pipeline**. The module graph runs `ai-qa-os-brain → ai-qa-os-orchestration` (brain depends on orchestration), not the reverse — so an orchestration step cannot compile-time reference a brain class without a dependency cycle.

**Decision.**
Split the gate across the dependency boundary:
- The **contract** (`ConfidenceGate`, `ConfidenceVerdict`, `ConfidenceDecisionContext`) lives in `ai-qa-os-core`, which both modules already depend on.
- The **implementation** (`ConfidencePolicyManager`) lives in `ai-qa-os-brain` — the Brain owns the decision and persists it (`DecisionEntity`).
- The **orchestrator** references only the core interface and receives the brain impl at runtime via Spring `ObjectProvider`, applying a **permissive default** (no gating) when no implementation is present (e.g. orchestration's own test module).

**Consequences.**
- *Positive:* honours "the Brain owns the decision" without inverting or cycling the module graph; orchestration stays decoupled from brain at compile time; orchestration tests run without the brain on the classpath.
- *Positive:* the pattern generalises to any future Brain-owned decision invoked from a lower layer.
- *Negative / trade-off:* the gate is silently permissive if the brain impl is absent at runtime — acceptable for a gate, but "no gate bean" must be a deliberate deployment choice, not an accident.
- *Imposed rule:* Brain-owned decisions invoked by lower layers are expressed as `core` contracts with the impl in `brain`, injected optionally.

**Related:** AI-1; ADR-001 (Brain as decision authority); Rule 2; prerequisite for AI-2.

---

## ADR-011 — Human review: in-memory resume on the gateway, durable queue in the DB

**Status:** Accepted (implemented — AI-2, 2026-07-22)

**Context.**
AI-2 turns AI-1's `HUMAN_REVIEW` pause into an approvable, resumable workflow. But the pipeline runs **async in-process on the gateway JVM** (the paused `WorkflowContext` is in memory there), while the review UI is served by the **dashboard JVM**; the two share only the database. There is no event bus (SCALE-2 is future), so a durable any-JVM resume would require serializing the entire workflow state.

**Decision.**
Split responsibilities across the JVM boundary:
- **Resume is gateway-local and in-memory** — a `PausedWorkflowRegistry` on the gateway retains the paused run (context + next step index); resume continues the pipeline there.
- **The review queue is durable (shared DB)** — a `HumanReviewEntity` records each pause (PENDING) and decision (APPROVED/REJECTED), so the dashboard (a different JVM) can list it.
- **The dashboard proxies approve/reject to the gateway** (forwarding the bearer token), because only the gateway can resume its in-memory run.

**Consequences.**
- *Positive:* delivers a working approval loop on the current in-process architecture, with no event bus or state serialization.
- *Negative / trade-off:* paused runs do **not** survive a gateway restart, and the model assumes a **single gateway instance** (the registry is per-JVM). Durable, multi-instance resume converges with SCALE-1/2 (FI-AI2-A).
- *Imposed rule:* until durable execution exists, resumable in-flight state lives in the JVM that runs the pipeline; cross-JVM visibility goes through the shared DB.

**Related:** AI-2; ADR-010 (the gate that produces `HUMAN_REVIEW`); ADR-004 (gateway as entry); SCALE-1/2 (durable execution).

---

## ADR-012 — A dedicated evaluation & guardrails module with contract seams

**Status:** Accepted (implemented — MOD-3, 2026-07-22)

**Context.**
No module owned the question "is the AI actually good and safe?" Prompt-quality evaluation (AI-3, PE-1/2/3) and AI guardrails (SEC-3) were scattered roadmap items with no home. Placing them inside `intelligence` or `brain` would bind evaluation to the thing being evaluated and force every future eval/guardrail item to widen those modules.

**Decision.**
Introduce a new leaf module **`ai-qa-os-eval`** (depends on `core`, `intelligence`, `memory`, `ai-provider`; nothing depends on it yet) whose primary product is a small, stable **contract set** — `Evaluator`, `EvaluationCase`/`EvaluationResult`/`EvaluationReport`, `Guardrail`, `GoldenDatasetProvider`, `RetrievalQualityMetric`. MOD-3 ships **working reference implementations** (deterministic evaluators + a minimal LLM-judge behind an injectable seam, a reference guardrail, an in-memory dataset provider, a `PromptEvaluationService`, durable `EvaluationResultEntity`), so downstream items **extend seams rather than fill an empty shell**. The LLM-judge's model call is an interface (`JudgeLlm`) so scoring logic is testable without a live LLM.

**Consequences.**
- *Positive:* AI-3 (regression harness), PE-1/2 (scoring/benchmarking), PE-3 (dashboard), and SEC-3 (guardrails) each plug into a concrete contract; the platform gains a real quality/safety feedback loop, not a third stub module (after `testdata`/`data`).
- *Negative / trade-off:* the module is not yet wired into a running app (no dependant), so its `@Component`s/entity are dormant and the `V15` eval table is unused until AI-3/PE-1 consume it; the LLM-judge is deliberately basic.
- *Imposed rule:* evaluation and guardrail capabilities are added as implementations of the `eval` contracts in this module — not inlined into `intelligence`/`brain`/`execution`.

**Related:** MOD-3; AI-3, PE-1/PE-2/PE-3 (Category Q), SEC-3 (all build on these seams); ADR-010 (confidence gate — a different, runtime AI-safety control).

---

## ADR-013 — Prompt regression baseline as a committed file

**Status:** Accepted (implemented — AI-3, 2026-07-22)

**Context.**
AI-3's regression harness needs a reference point — the scores a prompt version achieved before a change — to decide whether a new run regressed. Two homes were credible: a committed baseline file per suite, or querying the prior scores MOD-3 already persists to `eval_results`.

**Decision.**
The regression baseline is a **committed JSON file per suite** (`golden/<suite>.baseline.json`), read/written by `FileBaselineStore`. The harness compares each case's current aggregate score against this file (a case regresses when it falls more than `tolerance`, default 0.05, below baseline). MOD-3's DB-persisted results are retained for history/dashboards, but are **not** the regression reference.

**Consequences.**
- *Positive:* "prompts as source code" — the baseline is git-diffable and reviewed in the same PR as the prompt change, and the harness **runs in CI with no database** (which is where PE-1 will wire the gate). Deterministic and reproducible.
- *Negative / trade-off:* requires an explicit "update baseline" step (like snapshot tests) and a committed file per suite; a stale baseline is a review responsibility, not an automatic one.
- *Imposed rule:* regression decisions read the committed baseline, not live DB state; DB-persisted eval results serve history and PE-3 dashboards, not gating.

**Related:** AI-3; MOD-3 (the evaluators/persistence it builds on); PE-1 (turns the harness verdict into a CI merge-gate); PE-3 (dashboards over the DB history).

---

## ADR-014 — Two-tier prompt-eval CI gate: deterministic always-on + live key-gated

**Status:** Accepted (implemented — PE-1, 2026-07-23)

**Context.**
PE-1 must wire prompt regression testing into CI (MNT-1's `mvn verify`, run on every PR). But a prompt-quality benchmark needs LLM outputs, and the PR CI job runs with **no API key** and must stay deterministic and free. A single gate cannot be both "runs everywhere" and "measures real model output."

**Decision.**
Split the gate in two:
- **Always-on gate** (`PromptRegressionGateTest`, in `mvn verify`, no key) — asserts golden-suite ↔ baseline **integrity** (every committed suite has a baseline covering all case ids) and that the benchmark→score→regression **machinery** runs over a deterministic stub runner. Catches baseline drift and broken eval wiring on every PR.
- **Live gate** (`PromptBenchmarkLiveTest`, `@EnabledIfEnvironmentVariable(AIQAOS_PROMPT_EVAL_LIVE)`, plus a key-conditional `deploy.yml` step) — runs the real benchmark through the live LLM and fails on regression, only where a provider key is configured.

**Consequences.**
- *Positive:* PR CI is deterministic, free, and green without keys, yet still blocks baseline/machinery regressions; real prompt-quality gating runs where a model exists. Honest about what each tier can prove.
- *Negative / trade-off:* the real quality gate is not universal — a repo/fork without an `LLM_API_KEY` secret never runs it; the live test's enabled path depends on a provider-configured environment (JPA/DataSource auto-config excluded in its boot config since persistence is optional).
- *Imposed rule:* a prompt *regression* (a change in model output) is proven only by the live tier; key-less CI asserts integrity and wiring, not quality.

**Related:** PE-1; AI-3 (the harness it gates) + ADR-013 (the committed baseline it reads); MOD-3 (evaluators/persistence); MNT-1 (the CI job it extends); PE-3 (dashboards over the persisted results).

---

## ADR-015 — Guardrail contract promoted to `core`; the AI-boundary control point

**Status:** Accepted (implemented — SEC-3, 2026-07-23)

**Context.**
SEC-3 adds a security control at the AI boundary across three modules — input sanitisation (`intelligence`), output grounding (`orchestration`), and an execution surface guard (`execution`) — and the roadmap requires it to be "consolidated with the guardrails home in MOD-3." But MOD-3 placed the `Guardrail` contract in `ai-qa-os-eval`, and `eval` **depends on** `intelligence` (nothing depends on `eval`). So those modules could not implement `eval`'s `Guardrail` without a dependency cycle.

**Decision.**
Promote the `Guardrail`/`GuardrailVerdict`/`GuardrailContext` contract from `eval` to **`com.aiqaos.core.guardrail`**. Every module (intelligence, orchestration, execution, eval) now implements the **same** seam. This is the ConfidenceGate pattern (ADR-010) applied to guardrails: a cross-cutting AI-safety **contract in `core`**, implementations in the modules. SEC-3's three guards, MOD-3's `NonEmptyOutputGuardrail`, and future guards all share it. Enforcement is governed by `aiqaos.security.guardrails.enabled` (default true) and `mode` (enforce/report); guards are injected null-safely so direct construction (tests) degrades gracefully.

**Consequences.**
- *Positive:* one guardrail model platform-wide; SEC-3 genuinely consolidates on MOD-3's home; no dependency cycle; consistent with ADR-010. The platform moves from "runs AI output" to "governs AI output".
- *Negative / trade-off:* the contract moved out of the module that introduced it (a mechanical re-home of three files + one import); MOD-3's design doc now points at `core` for the seam.
- *Imposed rule:* new guardrails implement `com.aiqaos.core.guardrail.Guardrail`; the security-focused guards live at their boundary (input/output/execution), not in a central class.

**Related:** SEC-3; MOD-3 + ADR-012 (the guardrails home whose contract this re-homes); ADR-010 (the ConfidenceGate precedent it mirrors); SEC-1 (the `aiqaos.security.*` enablement pattern).

---

## ADR-016 — Centralised hardened security headers + download-only HTML artifacts

**Status:** Accepted (implemented — SEC-4, 2026-07-23)

**Context.**
The CSP was `default-src * 'unsafe-inline' 'unsafe-eval'` with frame-options disabled — effectively no CSP — an XSS/clickjacking surface. Two independent filter chains exist: the shared `SecurityConfig` and the dashboard's `@Order(1)` chain, and the latter (which serves `/api/artifacts/**`, the actual XSS surface) set **no** headers at all. Artifact HTML reports (interactive JS apps) were served `inline` as `text/html` from the app origin.

**Decision.**
- **One header policy, both chains.** A `SecurityHeaders.apply(http, csp)` helper in `ai-qa-os-security` defines the hardened set — strict CSP (`script-src 'self'`, no `unsafe-inline`/`unsafe-eval`; `frame-ancestors 'none'`; `object-src 'none'`), `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`, `Permissions-Policy`, HSTS. Both `SecurityConfig` and `DashboardSecurityConfig` call it, so the artifact surface is finally covered.
- **CSP is a tunable property** (`aiqaos.security.csp`, strict default) so ops can relax it per environment (e.g. for Swagger UI) without a rebuild.
- **HTML artifacts download, not render.** `ArtifactController` serves `text/html` with `Content-Disposition: attachment` (media stays `inline`), plus a real-path/symlink traversal check and `nosniff` + `default-src 'none'; sandbox` on every artifact response. A report can no longer execute in the app origin.

**Consequences.**
- *Positive:* removes the XSS/clickjacking surface across both chains and the artifact endpoint; one place to reason about headers; CSP tunable without redeploy.
- *Negative / trade-off:* HTML reports now download instead of rendering in a dashboard tab; a strict `script-src` may require tuning the CSP property for Swagger UI (not verifiable without a running app).
- *Imposed rule:* response-security headers are defined once in `SecurityHeaders`; new chains call it rather than hand-rolling headers.

**Related:** SEC-4; SEC-1 (the chains it hardens + the `aiqaos.security.*` convention); ADR-004 (gateway as public entry); FI-SEC4-A (the still-permissive dashboard CORS, deferred).

---

## ADR-017 — Execution decoupled behind a job-queue + artifact-store seam

**Status:** Accepted — Partial (SCALE-1 seam implemented 2026-07-23; distributed tier gated on ENT-5)

**Context.**
`ExecutionStep` ran the Playwright engine synchronously in the orchestration JVM (`engine.execute(...)`) — Windows-bound, single-host. SCALE-1 aims to convert this into a horizontally scalable worker tier. But the distributed goal needs a message broker, containerised workers, and shared **object storage for artifacts (ENT-5, currently Deferred)** — none of which are available/buildable in the current environment.

**Decision.**
Split SCALE-1 into a **code-level decoupling seam (built now)** and a **distributed tier (deferred)**:
- An `ExecutionJobQueue` seam (`submit` + `awaitResult`) with `ExecutionJob`/`ExecutionJobResult`, plus an `ArtifactStore` seam (results address artifacts by **key**, not host path). These live in **`ai-qa-os-execution`, not `core`** — because `ExecutionJob` references `ExecutionConfiguration` (an execution-module type) and `core` cannot depend on `execution`; orchestration (the producer) already depends on `execution`.
- An in-JVM reference `InProcessExecutionJobQueue` (a worker pool) + `LocalArtifactStore`, **opt-in** via `aiqaos.execution.queue.enabled`.
- `ExecutionStep` submits-and-awaits when the queue bean is present, and otherwise keeps the in-process path (non-breaking default).
- The **deferred tier** — a Redis-Streams binding (Redis is already in the topology; chosen over the roadmap's Kafka/RabbitMQ), containerised Playwright-native workers, and an object-storage `ArtifactStore` (ENT-5) — is what actually removes the single-host ceiling.

**Consequences.**
- *Positive:* the pipeline↔engine in-process coupling is broken; the worker tier is a drop-in behind stable seams; no behaviour change by default.
- *Negative / trade-off:* the headline outcome (horizontal, OS-agnostic execution) is **not delivered** — it is blocked on ENT-5 + infra; SCALE-1 remains **In Progress**. Submit-and-await decouples the host but not the pipeline's control flow (fully event-driven is SCALE-2).
- *Imposed rule:* execution work crosses the pipeline↔worker boundary as an `ExecutionJob` addressed by artifact **key**; new execution backends implement `ExecutionJobQueue`/`ArtifactStore`, not a direct engine call.

**Related:** SCALE-1; ENT-5 (object storage — the gating dependency, Deferred); SCALE-2 (event bus — the fully-async follow-on); ADR-005 (Agent Manager mediation); the existing in-memory `ExecutionQueue` this builds on.

---

## ADR-018 — Artifact object storage via a client seam + opt-in age retention

**Status:** Accepted — Partial (ENT-5 seam implemented 2026-07-23; real cloud binding + CronJobs deferred)

**Context.**
The `playwright-output/` artifact tree grows unbounded on local disk, and cross-host execution (SCALE-1) needs artifacts in shared object storage. ENT-5 must supply object storage + retention + backup/DR — but there is no cloud bucket, credentials, or cluster available in this environment to build/validate a real S3/GCS integration or Kubernetes backup jobs.

**Decision.**
- **Object storage behind a client seam.** `ObjectStorageArtifactStore` implements SCALE-1's `ArtifactStore` on top of an injectable `ObjectStorageClient` (put/get/exists/list/delete/lastModified). `InMemoryObjectStorageClient` is the validatable reference; a real S3/GCS client is a thin deferred adapter overriding it via `@ConditionalOnMissingBean`. The active `ArtifactStore` is chosen by `aiqaos.artifacts.store` (`local` default). This is the **code-level SCALE-1 unblock** — artifacts addressed by key in shared storage.
- **`ArtifactStore` gains `list`/`delete`/`lastModified`** so the tree can be enumerated and age-purged.
- **Retention is opt-in and age-based.** `ArtifactRetentionService.purgeExpired()` deletes artifacts older than `aiqaos.artifacts.retention.max-age-days`; disabled by default (never deletes unless switched on).
- **Backup/DR are ops CronJobs** (`deployment/kubernetes/backup/`) — pg_dump, Qdrant snapshot, artifact sync — authored as templates, not run here.

**Consequences.**
- *Positive:* a unit-proven object-storage `ArtifactStore` exists for SCALE-1 to wire; artifact growth is controllable; the cloud adapter is a small drop-in.
- *Negative / trade-off:* the **real** object storage (S3/GCS creds) and the **execution worker uploading** artifacts to it are deferred, as is running the CronJobs — so this ENT-5 slice does **not** by itself move production artifacts off local disk or remove SCALE-1's ceiling. ENT-5 remains **In Progress**.
- *Imposed rule:* artifact backends implement `ObjectStorageClient`/`ArtifactStore`; retention is opt-in; new object-storage adapters override the in-memory reference via `@ConditionalOnMissingBean`.

**Related:** ENT-5; SCALE-1 + ADR-017 (the `ArtifactStore` seam this fills); FI-ENT5-A/B/C (count/size retention, restore drills, bucket lifecycle rules).

---

## ADR-019 — Module↔package naming convention + MDC correlation-id propagation

**Status:** Accepted (implemented — MNT-6, 2026-07-23)

**Context.**
`ai-qa-os-orchestration`'s Java package was `com.aiqaos.workflow` — a module/package mismatch that surprised every contributor. Separately, a `correlationId` arrived on requests but was not threaded through logs, so tracing one run end-to-end was manual. Both are prerequisites for observability maturity (OBS-1 / Category K).

**Decision.**
- **Convention: a module's root package matches its artifact.** `com.aiqaos.workflow` → `com.aiqaos.orchestration` (a mechanical global rename — 84 orchestration files + 9 dashboard/gateway importers). Safe because every component/entity/repository scan uses the broad `com.aiqaos` and no config/string referenced the old package.
- **Correlation via MDC.** `correlationId` is placed in SLF4J MDC at each entry boundary and cleared in `finally`: the gateway `CorrelationIdFilter` (per request) and `AutonomousQAPipelineOrchestrator` (per run — which, because agents and the execution engine run in-process on the same thread, thread-locally covers the whole orchestration → agents → execution chain). The log pattern surfaces `%X{correlationId}` on every line.

**Consequences.**
- *Positive:* naming is consistent; a run is greppable across all logs by its correlationId — the foundation OBS-1 builds on. No API/DB/behaviour change.
- *Negative / trade-off:* MDC is thread-local, so the opt-in SCALE-1 worker-pool boundary needs an explicit context-copy (FI-MNT6-A). A package rename requires a **clean** build — stale `.class` files from before the move otherwise duplicate beans.
- *Imposed rule:* new modules name their root package after their artifact; entry boundaries set/clear `correlationId` in MDC.

**Related:** MNT-6; OBS-1 (distributed tracing — this unblocks it); the existing `CorrelationIdFilter` / `LogContextManager`; FI-MNT6-A/B (worker-thread MDC propagation, structured JSON logs).

---

## ADR-020 — Raw-OTel tracing: config-gated export + span instrumentation

**Status:** Accepted (OBS-1 implemented 2026-07-23 — instrumentation, then the boundary wiring later the same day; only operational enablement of the collector export remains)

**Context.**
OTel was present but inert: `TelemetryConfig` built an `SdkTracerProvider` with **no span processor or exporter**, and the home-grown helpers (`SpanManager`, `TraceManager`, `TraceContextPropagator`, `CorrelationTraceBridge`) were never called. An OTel Collector exists in the deployment topology but cannot run in this environment. Note the gateway runs the pipeline **in-process**, so gateway → orchestration → agents → execution share a JVM (context propagates thread-locally); the genuine cross-JVM hop is dashboard ↔ gateway.

**Decision.**
- **Stay on the raw OTel SDK** (not Micrometer Tracing) — consistent with the existing helpers.
- **Export is config-gated, no-op by default.** A `BatchSpanProcessor(OtlpGrpcSpanExporter)` is added only when `aiqaos.otel.exporter=otlp` (+ `aiqaos.otel.endpoint`); otherwise no processor is registered. Spans are attributed to `service.name=ai-qa-os`. Tracing is therefore inert and near-free until a collector is configured.
- **Instrument the workflow**: the orchestrator emits a `workflow.run` span with `workflow.id` + `correlationId`, and a `workflow.step.<name>` child span per step. The tracing beans are **field-injected and optional**, so direct construction (tests) and any context without observability degrade to no spans.
- **Correlate logs with traces**: `CorrelationTraceBridge` binds `correlationId` (MDC + span attribute) and mirrors `traceId` into MDC; the log pattern surfaces both.

**Consequences.**
- *Positive:* a workflow run now produces a real span tree linked to its logs; enabling the collector is a single config flag; zero behaviour change by default.
- *Negative / trade-off:* **live export to the collector is not yet proven** — no collector/cluster exists in the build environment, so enabling `aiqaos.otel.exporter=otlp` and eyeballing a real trace remains an operational step. The boundary wiring itself is now in place (gateway inbound `TracingFilter`, dashboard outbound inject) and each half is unit-proven — extract via a `traceparent` request test, inject via the round-trip test — but the two halves have not been exercised over a real network hop. MDC/OTel context are thread-local, so the SCALE-1 worker pool needs explicit context copying (FI-OBS1-A).

> **Update (2026-07-23, same day):** the deferred boundary wiring was completed — `TracingFilter` (gateway, `@Order(0)`, lower-cased carrier keys so W3C lookup matches) and the dashboard→gateway trace-context inject in `ReviewController`. Status moved Partial → Accepted; OBS-1 → Completed, with only operational collector enablement outstanding.
- *Imposed rule:* new spans go through `TraceManager`/`SpanManager`; correlation ids are bound via `CorrelationTraceBridge`; exporters are selected by configuration, never hard-wired.

**Related:** OBS-1; MNT-6 + ADR-019 (the correlationId-in-MDC foundation); OBS-2/OBS-3 (dashboards over this data); FI-OBS1-A/B/C.

---

## Document Completion Status

**Status:** Active — new ADRs appended as decisions are made (per MNT-5)
**Version:** 1.0.0
**Convention:** ADRs are immutable once Accepted; a changed decision is recorded as a new ADR that *Supersedes* the old one, never by editing history
**Related documents:** [`AI-QA-OS-Documentation.md`](./AI-QA-OS-Documentation.md) · [`AI-QA-OS-Improvement-Roadmap.md`](./AI-QA-OS-Improvement-Roadmap.md) · [`AI-QA-OS-Implementation-Tracker.md`](./AI-QA-OS-Implementation-Tracker.md)
