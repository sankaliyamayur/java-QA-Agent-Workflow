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
| [ADR-021](#adr-021--vector-store-standardization-qdrant-production--in-memory-devtest) | Vector store standardization: Qdrant + In-Memory | Accepted |
| [ADR-022](#adr-022--semantic--prompt-cache-for-ai-invocations) | Semantic & prompt cache for AI invocations | Accepted |
| [ADR-023](#adr-023--sharded-cross-browser-execution-matrix-fan-out-over-virtual-threads) | Sharded cross-browser execution matrix (fan-out over virtual threads) | Accepted |
| [ADR-024](#adr-024--single-migration-owner--per-service-connection-pools-shared-schema-by-design) | Single migration owner + per-service connection pools (shared schema by design) | Accepted |
| [ADR-025](#adr-025--llm-cost-quota-enforcement-at-the-provider-choke-point-soft-cap) | LLM cost-quota enforcement at the provider choke point (soft cap) | Accepted |
| [ADR-026](#adr-026--unified-ai-audit-trail-by-aggregation-over-run-keyed-sources) | Unified AI audit trail by aggregation over run-keyed sources | Accepted |
| [ADR-027](#adr-027--deterministic-weighted-prompt-ab-routing--leaderboard-over-pe-1-scores) | Deterministic weighted prompt A/B routing + leaderboard over PE-1 scores | Accepted |

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

## ADR-021 — Vector store standardization: Qdrant (production) + In-Memory (dev/test)

**Status:** Accepted (SCALE-3 implemented 2026-07-23)

**Context.**
`ai-qa-os-memory` shipped with 5 `VectorStoreClient` implementations (`QdrantStoreClient`, `InMemoryVectorStoreClient`, `ChromaStoreClient`, `MilvusStoreClient`, `PgVectorStoreClient`). Maintaining five vector clients created unnecessary surface area and cognitive load. Qdrant (`qdrant:6333`) is natively provisioned by the deployment infrastructure.

**Decision.**
- **Standardise on Qdrant** as the single primary production vector store provider (`aiqaos.memory.vector.provider=qdrant`).
- **Standardise on InMemoryVectorStoreClient** as the supported dev/test fallback provider (`aiqaos.memory.vector.provider=in-memory` or when unset).
- **Deprecate and isolate unmaintained stubs** (`ChromaStoreClient`, `MilvusStoreClient`, `PgVectorStoreClient`) under `@Deprecated` and gate bean registration behind `aiqaos.memory.vector.experimental.enabled=true`.
- **Add unit testing**: unit test coverage added for `InMemoryVectorStoreClient` (vector save, cosine similarity search, update, delete, collection management).

**Consequences.**
- *Positive:* reduces maintenance surface, clarifies supported vector store path for production and testing environments.
- *Negative / trade-off:* experimental stubs require an explicit property to instantiate.
- *Imposed rule:* production deployments must set `aiqaos.memory.vector.provider=qdrant`; tests use in-memory vector store by default.

**Related:** SCALE-3; AI-4 (semantic cache); PERF-2 (batch embeddings).

---

## ADR-022 — Semantic & Prompt Cache for AI Invocations

**Status:** Accepted (AI-4 implemented 2026-07-23)

**Context.**
Pipeline steps invoke external LLMs via `LLMProviderManager.generate(LLMRequest)`. Semantically duplicate or identical prompts across test runs previously incurred full 2–15s network latency and token costs.

**Decision.**
- **Intercept LLM calls in `LLMProviderManager`**: check `PromptCacheManager` before executing provider calls.
- **Back cache by `VectorStoreClient`**: use vector similarity search (cosine similarity $\ge 0.95$) over the `"prompt_cache"` collection (using `Qdrant` in prod, `InMemory` in dev/test).
- **Return zero-token cached responses**: on hit, return cached `LLMResponse` with model=`"prompt-cache"`, 0 tokens used, and 1ms latency. On miss, delegate to provider and cache response.
- **Configurable controls**: gated by `aiqaos.ai.cache.enabled=true` and `aiqaos.ai.cache.similarity-threshold=0.95`.

**Consequences.**
- *Positive:* eliminates redundant LLM calls, cuts latency for repeated test scenarios from seconds to ~1ms, reduces API costs.
- *Negative / trade-off:* relies on vector store availability; non-deterministic LLM responses are cached statically per prompt.
- *Imposed rule:* prompt cache lookups must gracefully fallback to primary LLM provider on cache error or miss.

**Related:** AI-4; SCALE-3 (vector store standardization); PERF-2 (batch embeddings).

---

## ADR-023 — Sharded cross-browser execution matrix (fan-out over virtual threads)

**Status:** Accepted (implemented — WF-4, 2026-07-28)

**Context.**
Execution ran one browser, sequentially. WF-4 needs parallel, sharded, cross-browser runs. Playwright shards natively (`--shard=i/N`) and selects a browser via `--project`, but the platform never orchestrated a matrix. WF-4 is tagged as depending on SCALE-1, whose multi-host worker tier is deferred — but SCALE-1's `InProcessExecutionJobQueue` already runs on virtual threads, so single-host parallelism is free.

**Decision.**
Add a **matrix fan-out** in `ai-qa-os-execution`, decomposed into pure, testable pieces:
- `ExecutionConfiguration` gains `browsers[]` (matrix) + `shardCount`/`shardIndex`; unset = the pre-WF-4 single run (back-compat via existing `@JsonIgnoreProperties`).
- `ExecutionMatrixPlanner` expands config → one `ShardPlan` per `browser × shard`; `ExecutionResultAggregator` merges per-unit results into one suite result; `ShardedExecutionScheduler` dispatches units — `PARALLEL` over a virtual-thread executor, `SEQUENTIAL` inline — and a single-unit matrix runs the engine directly (zero overhead, unchanged default).
- `PlaywrightExecutionEngine` + `run-playwright.ps1` pass native `--shard=i/N` (omitted when unsharded); the browser already maps to `--project`.
- `ExecutionStep` calls the scheduler (null-safe) in the non-queue path.

**Consequences.**
- *Positive:* real parallel/sharded cross-browser execution on one host today; planner + aggregator + scheduler are unit-proven; default single-browser path is byte-for-byte unchanged.
- *Negative / trade-off:* cross-*host* distribution still awaits SCALE-1's containerised worker tier (a swap of the dispatch executor for the queue — no planner/aggregator change; FI-WF4-C). Live Playwright runs are unvalidated here (no Node), only the fan-out logic.
- *Imposed rule:* matrix execution flows planner → dispatch → aggregator; a new execution backend swaps the dispatch step, not the fan-out.

**Related:** WF-4; SCALE-1 + ADR-017 (the virtual-thread queue it borrows / defers multi-host to); native Playwright sharding; FI-WF4-A/B/C.

---

## ADR-024 — Single migration owner + per-service connection pools (shared schema by design)

**Status:** Accepted (implemented — SCALE-4, 2026-07-28)

**Context.**
The gateway and dashboard share one PostgreSQL database and both ran Flyway on startup — a race and a schema coupling. The two apps also **share data** (the AI-2 `human_reviews` queue is written by the gateway and read by the dashboard; execution/workflow rows likewise), so isolating each app in its own schema would fork that shared state and break the review/execution flows.

**Decision.**
- **Shared schema, by design** — SCALE-4 does *not* pursue schema-per-service; the shared data is intentional and stays in one shared schema.
- **Single migration owner** — the gateway owns Flyway (`enabled: true`); the dashboard runs with `flyway.enabled: false` and connects to the already-migrated DB (this ownership split was already established by ORG-2; SCALE-4 keeps it and makes it explicit). One migrator = no startup race, no lock contention.
- **Per-service connection pools** — explicit HikariCP sizing per app: the gateway a larger write-leaning pool (it fronts the workflow write path), the dashboard a smaller read-leaning pool. Values are env-overridable starting points.

**Consequences.**
- *Positive:* the dual-Flyway race is gone; each app's connection usage is sized to its role; the shared-data model is preserved and documented.
- *Negative / trade-off:* a startup **ordering expectation** (the schema must exist before the dashboard starts) — handled in deployment (gateway/init migrates first). A dedicated migration job that removes even that coupling is deferred (FI-SCALE4-A). True per-service/per-tenant schema separation belongs with ENT-1. The live effect (race removal, pool behaviour) is only observable against a running Postgres, not in the build.
- *Imposed rule:* exactly one component runs Flyway against the shared DB; new services connect Flyway-disabled with their own sized pool.

**Related:** SCALE-4; ORG-2 (single migration location — the owner split built on it); ENT-1 (tenancy / true schema separation); FI-SCALE4-A/B/C.

---

## ADR-025 — LLM cost-quota enforcement at the provider choke point (soft cap)

**Status:** Accepted (implemented — ENT-3, 2026-07-28)

**Context.**
`CostTracker` recorded LLM spend but nothing enforced a limit — an uncapped financial risk. Every LLM call funnels through `LLMProviderManager.generate`, and the request already carries the scope keys (`correlationId` = workflow, `agentType`). Token cost is known only *after* the call, so a precise pre-call cost is unavailable. The roadmap named `ai-qa-os-brain`, but **brain depends on ai-provider**, so a brain-resident enforcer consulted by the manager would invert the dependency.

**Decision.**
- **Enforce at the provider choke point** (`ai-provider`, beside `CostTracker`) — not `brain` (dependency direction; mirrors ADR-010's ConfidenceGate reasoning). Brain/dashboard *surface* limits later.
- **Soft cap** — a `CostBudgetEnforcer` blocks a call when the scope's **already-accumulated** spend (from a thread-safe in-memory `SpendLedger`, fed by `CostTracker` after each call) has reached its configured USD limit. No fragile pre-call token estimation; spend is bounded (≤ one in-flight overshoot per scope). The durable record stays in `LLMCostEntity`.
- **Scopes now:** global-daily, per-workflow, per-agent. **Per-tenant** is deferred to ENT-1.
- **Opt-in & non-breaking** — `aiqaos.cost.quota.enabled=false` by default, `enforce`/`warn` mode; the enforcer is injected optionally so direct-construction tests and the disabled default are no-ops.

**Consequences.**
- *Positive:* runaway LLM spend can be capped per workflow/agent/day; observability of cost becomes control of cost; zero behaviour change until switched on.
- *Negative / trade-off:* the in-memory ledger resets on restart (daily/global caps aren't cross-restart yet — FI-ENT3-A) and is per-instance; the soft cap can overshoot by one in-flight call; per-tenant caps await tenancy.
- *Imposed rule:* cost limits are enforced at the provider boundary via the ledger + enforcer; new cost backends record to the ledger and read budgets from `CostBudgetProperties`.

**Related:** ENT-3; `CostTracker`/`LLMCostEntity` (the spend it reads); ENT-1 (per-tenant), AI-6 (per-workflow budgeting); `AgentExecutionBudget`/`RateLimiter` (sibling enforcement axes); FI-ENT3-A/B/C.

---

## ADR-026 — Unified AI audit trail by aggregation over run-keyed sources

**Status:** Accepted (implemented — GOV-1, 2026-07-28)

**Context.**
"A complete record of every AI decision, prompt, model call, cost, and approval" already exists across per-domain immutable tables. GOV-1 must unify them into one queryable trail. Implementation grounding found the sources are **not consistently keyed to a run**: `AgentTraceEntity`/`LLMCostEntity` key on `correlationId`(`requestId`); `HumanReviewEntity`/`SecurityAuditEntity` key on `workflowId`; and **`DecisionEntity` has no run key at all**, while `PromptExecutionEntity` keys on `traceId`. Only the **gateway** depends on all four run-keyed source modules.

**Decision.**
- **Aggregate, don't re-capture** — no new audit table and no double-writes; the per-domain rows stay the immutable source of truth (the approved Option A).
- **Scope to the four run-keyed facets now:** approvals + model calls + cost + security events. A pure `AiAuditAssembler` orders the mapped `AiAuditEvent`s by time and rolls up cost/facet counts; `AiAuditService` (in the gateway) fans in the four repos — approvals/security by `workflowId`, model-calls/cost by `correlationId` — via additive `findBy…` methods; a read-only auth-gated `GET /api/v1/audit/{workflowId}?correlationId=` exposes it.
- **Decisions + prompt executions are excluded** until they carry a run key (they can't be joined today) — deferred, not silently dropped.

**Consequences.**
- *Positive:* a real, unified, time-ordered audit view per run over four facets, with zero data duplication; the assembler is fully unit-proven independent of a DB.
- *Negative / trade-off:* two of the six requested facets (decisions, prompt executions) are missing because their entities lack a run key — a genuine schema gap surfaced (FI-GOV1-D); the caller must supply both `workflowId` and `correlationId` (no bridge table is queried); the live cross-repo query needs a populated DB to observe.
- *Imposed rule:* the AI audit trail is a read-side aggregation keyed by the run identifiers; new auditable AI records must carry a run key (`correlationId`/`workflowId`) to appear in it.

**Related:** GOV-1; MNT-6 (`correlationId` threading); AI-2 (`HumanReviewEntity`); `AgentTraceEntity`/`LLMCostEntity`/`SecurityAuditEntity`; FI-GOV1-A (materialised table), FI-GOV1-D (run-key normalisation for Decision/PromptExecution).

---

## ADR-027 — Deterministic weighted prompt A/B routing + leaderboard over PE-1 scores

**Status:** Accepted (implemented — PE-2, 2026-07-28)

**Context.**
Improving prompts requires comparing versions on evidence. The eval spine already exists — PE-1's `PromptBenchmarkService`/`PromptScore` gives a version's measured outcome, and `PromptVersionEntity` gives the variants. PE-2 needed the two missing pieces: *routing* traffic across variants and a *leaderboard* ranking them — without rebuilding scoring, and knowing real-traffic scoring needs live LLM (deferred, like PE-1's live tier).

**Decision.**
- **Deterministic-by-key, weighted A/B** — `PromptExperimentRouter.assign(experiment, key)` hashes a stable key (e.g. workflow `correlationId`) into the variants' cumulative-weight ranges. The same key always resolves to the same variant, so a run stays coherent and any assignment is reproducible from its key, while the weighted split holds in aggregate. A disabled/empty experiment returns the default version. Pure routing (in `intelligence`).
- **Leaderboard over PE-1 scores** — `PromptLeaderboard.rank(versionIds, suites)` orders variants by `PromptBenchmarkService.benchmark(...).getOverall()` (best first) with 1-based ranks. Reuses PE-1; no new evaluation. (In `eval`.)
- **No new schema/API** — experiments are in-memory (`ExperimentRegistry`); scores come from PE-1's `eval_results`. Non-breaking (a disabled experiment = the default prompt).

**Consequences.**
- *Positive:* prompt versions can be compared and ranked on measured outcome; routing is reproducible and run-coherent; the eval spine (MOD-3→AI-3→PE-1→PE-2) is complete for offline experimentation.
- *Negative / trade-off:* scoring variants on **real traffic** and wiring the router into live prompt resolution are deferred (need the LLM-keyed environment — FI-PE2-A); experiments/assignments aren't durably stored yet (FI-PE2-B); no significance test / auto-promote (FI-PE2-C).
- *Imposed rule:* A/B assignment is deterministic from a stable key; leaderboards rank on PE-1 scores, not opinion.

**Related:** PE-2; PE-1 (`PromptBenchmarkService`/`PromptScore`), MOD-3/AI-3 (eval spine); `PromptVersionEntity`; PE-3 (dashboard leaderboard view); FI-PE2-A/B/C.

---

## ADR-028 — Durable version registry with pin/rollback behind a store seam (prompt now, generic for model)

**Status:** Accepted (implemented — GOV-4, 2026-07-28)

**Context.**
Prompt versions exist as history (`PromptVersionEntity` + `PromptVersionManager.getActiveVersion`) but with no *governed* promotion/rollback — "active" is set, not controlled. Model versions are ungoverned strings in `ai-provider`, which does **not** depend on `intelligence`, so a single registry cannot serve both without a shared placement (the GOV-1 cross-module shape). GOV-4 needed real pin/rollback control now, without inventing model-version governance or forcing a premature cross-module contract, and a rollback must survive restarts.

**Decision.**
- **A generic version registry in `intelligence`** — `VersionRegistry` keyed by a generic `registryKey` + `VersionKind` (`PROMPT`/`MODEL`): `pin` (promote, deactivating the prior active), `rollback` (re-pin the immediately-previous *distinct* version as a fresh, audited pin), `activeVersion` ("what runs now"), `history` (newest-first timeline). `pin` is idempotent on the already-active version; `rollback` with no earlier distinct version is a safe no-op. Grounded on prompt versions today; the registry already *models* model versions.
- **Durability behind a store seam** — a governance rollback must persist, so pins are durable (`version_pins`, migration `V16`), but the logic sits behind a `VersionPinStore` seam: `InMemoryVersionPinStore` (default, insertion-order timeline → deterministic, the unit-test reference) and `JpaVersionPinStore` (activated by `aiqaos.version-registry.store=jpa`). The store enforces the single-active-pin-per-key invariant. Same pattern as ENT-5's `ObjectStorageClient` and ENT-3's `SpendLedger`.
- **The active-pin flag is `active_pin`**, deliberately distinct from `BaseEntity`'s soft-delete `active` column (different meaning).

**Consequences.**
- *Positive:* an approved version can be pinned and instantly rolled back to last-known-good with a durable, auditable history; the pin/rollback/history logic is fully unit-proven against the in-memory store; the generic key/kind means model-version governance drops in behind the same registry with no rework.
- *Negative / trade-off:* model-version governance (a `ModelVersion` object + a shared/`core` placement so `ai-provider` participates) is deferred (FI-GOV4-A); wiring `PromptManager`/`brain` selection to consult the pinned version is deferred (FI-GOV4-B); the durable JPA store's live behaviour needs a running Postgres (unvalidated here, like every DB item); policy-gated promotion (GOV-3) and the dashboard surface (GOV-2) are separate items.
- *Imposed rule:* the pinned version is the authority on "what runs"; a rollback is a new audited pin, never an edit to history — mirroring the ADR immutability discipline (ADR-009).

**Related:** GOV-4; `PromptVersionEntity`/`PromptVersionManager` (grounded on); GOV-1 (cross-module placement lesson); ENT-5/ENT-3 (store-seam pattern); ADR-010/015 (contract-in-core only when a second consumer is real); GOV-2/GOV-3; FI-GOV4-A/B/C.

---

## ADR-029 — Responsible-AI policy as a config-driven guardrail at the SEC-3 boundary (OPA deferred)

**Status:** Accepted (implemented — GOV-3, 2026-07-28)

**Context.**
GOV-3 asks to govern *AI behaviour* with policy-as-code ("no production URLs", "no PII in prompts", "human approval for destructive actions") at the confidence-gate (AI-1) and guardrail (SEC-3) boundaries, and the roadmap says to extend the existing `OpaSecurityPolicyEngine` in `ai-qa-os-security`. But `ai-qa-os-security` depends on **`core` only**, and `intelligence`/`brain`/`orchestration` do **not** depend on `security` — so the AI boundary cannot call the security OPA engine, nor vice-versa. Same cross-module wall as GOV-1 and the ConfidenceGate/Guardrail promotion (ADR-010/015). Also, a live OPA/Rego backend needs a running OPA server (unvalidatable in this environment).

**Decision.**
- **Enforce at the SEC-3 `Guardrail` boundary** — a deterministic, config-driven `ResponsibleAiPolicyEngine` (reference impl `RuleBasedResponsibleAiPolicyEngine`) in `intelligence`, surfaced as a `core`-`Guardrail` (`ResponsibleAiPolicyGuardrail`). It is discovered as a `Guardrail` bean alongside the SEC-3 guards, so it runs where guardrails already run — no new wiring, no dependency inversion.
- **Declarative rule model** — `AiPolicyRule` (compiled pattern + `PolicyEffect` + optional phase) collected in an `AiPolicyRuleSet` that returns the **most-severe** matching effect (`ALLOW`<`WARN`<`REQUIRE_REVIEW`<`BLOCK`). Default rules (`no-production-urls`, `no-pii-in-prompts`, `destructive-requires-review`) are all overridable via `aiqaos.governance.policy.*` (toggle PII/destructive, supply production hosts, add block patterns); enabled by default in `enforce` mode, mirroring SEC-3.
- **Fail-safe mapping** — `BLOCK` **and** `REQUIRE_REVIEW` both map to `GuardrailVerdict.block`, so a review-required action is never silently allowed; `warn` mode logs and allows; `enabled=false` allows everything.
- **OPA as a deferred backend** — the roadmap's OPA/Rego engine becomes a drop-in alternative `ResponsibleAiPolicyEngine` behind the seam (FI-GOV3-A), added when an OPA server exists — without touching the boundary.

**Consequences.**
- *Positive:* Responsible-AI rules govern AI content at the real boundary today, deterministic and fully unit-proven (13 tests); config-driven and non-breaking (additive `@Component`, gated, no schema/API); OPA slots in later behind the seam.
- *Negative / trade-off:* no live OPA/Rego yet (FI-GOV3-A); `REQUIRE_REVIEW` is enforced as a conservative block rather than routed into the AI-1/AI-2 human-review path (FI-GOV3-B); the policy is applied at the prompt/requirement boundary — the `execution` generated-script surface and `orchestration` output boundary are adjacent follow-ups (FI-GOV3-C); GOV-4 promotion gating + auditing decisions into the GOV-1 trail are deferred (FI-GOV3-D).
- *Imposed rule:* Responsible-AI policy is enforced fail-safe at the guardrail boundary; a review-required action blocks until the escalation path exists.

**Related:** GOV-3; SEC-3 (`Guardrail` spine reused), AI-1 (`ConfidenceGate` boundary); ADR-010/015 (contract-in-core, swappable impl); GOV-1 (cross-module wall); `OpaSecurityPolicyEngine` (deferred backend); GOV-2 (compliance surface), GOV-4 (promotion gating); FI-GOV3-A/B/C/D.

---

## ADR-030 — Real PII masking in `ai-qa-os-testdata`: classification-driven, pluggable strategies, format-preserving default

**Status:** Accepted (implemented — MOD-4, 2026-07-28)

**Context.**
`ai-qa-os-testdata` shipped as ~16 empty stubs, including its named flagship `MaskingEngine`. WF-1 had made `SyntheticGenerator` real (it emits fixtures containing card numbers and emails), so there was live PII to mask, but no masking existed. PII masking is a genuine GDPR/enterprise requirement feeding Category N (GOV-2) and complements GOV-3's PII policy. The roadmap said to fill the existing module (not create one), and named `core`/`intelligence`/`ai-provider` as anticipated deps — but the module actually depends on `core` only, and masking is self-contained.

**Decision.**
- **Fill the flagship, defer the scaffolds (§0.4-A)** — implement `PiiDetector`, `MaskingEngine`/`MaskingService`/`SecureData`/`MaskingProperties`, and a thin `TestDataOrchestrator` (generate→mask) + `DataValidator` (residual-PII check). Leave the optimizer/pipeline/repository scaffolds empty until a consumer exists (FI-MOD4-A) — no speculative code.
- **Classification-driven masking with an auto-detect path** — `maskRecord(record, field→PiiType)` masks only classified fields (precise, no false positives on structured data); `maskText(freeText)` uses the shared `PiiDetector` (email/SSN/card/phone) for unstructured content. `NAME` is masked only by classification (not reliably regex-detectable).
- **Pluggable strategies, `PARTIAL` default** — `REDACT` / `PARTIAL` (format-preserving) / `HASH` (deterministic SHA-256 → referential integrity survives) / `FAKE`, configurable per `PiiType` via `aiqaos.testdata.masking.*`.
- **Stay on `core` only** — no module dependency added; the small duplication of PII regexes vs. reusing GOV-3's (in `intelligence`) is the right trade against inverting the module graph for a self-contained feature.
- **The card detector requires 15–16 digits** so 13-digit epoch timestamps aren't misread as card numbers.

**Consequences.**
- *Positive:* the module masks PII (classification + auto-detect) with pluggable strategies and asserts no raw PII survives — fully unit-proven (16 tests) over WF-1's real generator; the GDPR feature that feeds GOV-2 exists; non-breaking (stubs filled, no schema/API, no new dep).
- *Negative / trade-off:* the optimizer/pipeline/repository scaffolds remain empty (FI-MOD4-A); masking isn't yet a first-class orchestration-pipeline stage (FI-MOD4-B); no reversible tokenization vault (FI-MOD4-C); `FAKE` returns fixed placeholders, not a real faker library (FI-MOD4-D); PII regexes are duplicated with GOV-3 by deliberate choice.
- *Imposed rule:* synthesised/handled test data is masked before it leaves the boundary; masking correctness is asserted by re-detection, not assumed.

**Related:** MOD-4; WF-1 (`SyntheticGenerator` reused); GOV-3 (PII policy — sibling PII patterns), GOV-2 (compliance surface fed by masking); ADR-010/015 (don't invert the graph for a self-contained feature); FI-MOD4-A/B/C/D.

---

## ADR-031 — Close the continuous-learning loop via recorded improvement proposals; adoption gated on LRN-4

**Status:** Accepted (implemented — LRN-1, 2026-07-28)

**Context.**
The learning loop (Execution History → Failure Analysis → Root Cause → Prompt/Scenario/Automation Improvement → Memory → Future Runs) was half-built: `FailurePatternAnalyzer` (root cause) and `LearningMemoryStore` (memory) are real, but the **improvement stage** — turning root-caused `FailurePattern`s into concrete changes — was an empty stub (`ReflectionService`/`ReflectionResult`). Closing the loop by *adopting* changes is dangerous without a safe-adoption gate: the roadmap explicitly warns that an unconditional loop "risks compounding mistakes." That gate is **LRN-4, which is not yet implemented**.

**Decision.**
- **Fill the improvement stage as deterministic reflection** — `DefaultReflectionService.reflect(patterns)` maps each `FailurePattern` to a typed `ImprovementProposal` (`PROMPT` / `SCENARIO` / `AUTOMATION`) by error signature (locator/element→automation, model/prompt→prompt, coverage/assertion→scenario, else→automation). Recurring (`occurrenceCount ≥ 3`) or high-confidence (`≥ 0.8`) patterns are raised to `HIGH` priority. Pure and unit-testable.
- **Record, do not adopt** — proposals are persisted via `LearningMemoryStore.storeImprovementProposals` (new, key `learning:improvement_proposals`, mirrors the existing pattern store) and surfaced as a `LearningEvent`. `LearningEngineImpl` calls the reflector (optional, null-safe field injection) in its deterministic heuristic path — additive; the agent path is untouched.
- **Adoption is deferred behind LRN-4** — actually applying proposals (writing prompt versions in `intelligence`, scenario/automation changes in `orchestration`/`agents`, routed through `brain`) waits for the safe-adoption gate (FI-LRN1-A), honouring the roadmap's own safety argument.

**Consequences.**
- *Positive:* the loop's missing improvement→memory arc now runs deterministically and safely; each run's failures become prioritised, recorded improvement proposals; fully unit-proven (9 tests) with existing learning tests green; non-breaking (additive, no schema/API, optional bean).
- *Negative / trade-off:* proposals are not yet acted on — the loop informs rather than self-modifies until LRN-4 exists (FI-LRN1-A); brain-routing (`DecisionService`) remains a stub (FI-LRN1-B); no outcome feedback yet (did an adopted proposal help? — FI-LRN1-C, ties to LRN-2).
- *Imposed rule:* the learning loop may **propose and record** improvements autonomously but must **not adopt** them without passing the LRN-4 safe-adoption gate — learning stays monotonic (quality can rise, not silently fall).

**Related:** LRN-1; OBS-1 (data), AI-1 (trust scoring) — deps; LRN-4 (safe-adoption gate — adoption prerequisite), LRN-2 (metrics), LRN-3 (dashboard); reuses `FailurePatternAnalyzer` + `LearningMemoryStore`; FI-LRN1-A/B/C.

---

## ADR-032 — Safe-adoption gate: core contract + Brain impl reusing the confidence gate + an eval threshold

**Status:** Accepted (implemented — LRN-4, 2026-07-28)

**Context.**
LRN-1 records improvement proposals but must not adopt them unconditionally (self-degradation risk). LRN-4 is the gate: admit a learned improvement only if it passes eval (MOD-3) and clears the confidence gate (AI-1), else reject-for-review. The roadmap places the gate in `ai-qa-os-brain` (Rule 2). But `brain` depends on neither `ai-qa-os-learning` (which produces the proposals) nor `ai-qa-os-eval` (which scores them) — the same cross-module wall as GOV-1/3/4 and AI-1's own `ConfidenceGate`.

**Decision.**
- **Gate contract in `core`, impl in `brain`** — `SafeAdoptionGate` + `AdoptionCandidate` + `AdoptionDecision` + `AdoptionKind`/`AdoptionVerdict` live in `core` (like `ConfidenceGate`, ADR-010), so the Brain gate operates on a core-level candidate rather than learning's concrete `ImprovementProposal`. Justified by a real second consumer (LRN-1 + the gate).
- **Decide over supplied scores; reuse `ConfidenceGate` (§0.4-A)** — the `AdoptionCandidate` carries `evalScore` (from MOD-3/PE-1, computed upstream) and `confidence`. `LearningAdoptionGate` admits iff `evalScore ≥ aiqaos.brain.learning.eval-threshold` (default 0.75) **and** the reused `ConfidenceGate` returns `PROCEED`/`PROCEED_WITH_VALIDATION`. The gate does not *run* eval (would force a `brain → eval` edge + live LLM); it enforces the threshold, exactly as AI-1 decides over a supplied confidence.
- **Fail-safe on unreported confidence** — `UNGATED` (`c ≤ 0`) is **rejected** for adoption, the opposite of AI-1's pipeline safeguard: gating a self-modification requires positive trust, so absence of a confidence signal must not admit. Rejections are logged (WARN) for human review, never dropped.

**Consequences.**
- *Positive:* learning is now monotonic — an improvement is adopted only when measured-good and trusted; the decision is a pure, reusable Brain policy over a `core` contract, unit-proven (8 tests); AI-1's gate is reused unchanged; non-breaking (additive contract + bean, no schema/API).
- *Negative / trade-off:* mapping LRN-1 `ImprovementProposal` → `AdoptionCandidate` and wiring admitted candidates into a real adoption path are deferred (FI-LRN4-A); the candidate's `evalScore` is supplied, not computed live (FI-LRN4-B); rejections are logged, not yet queued durably for review (FI-LRN4-C).
- *Imposed rule:* no learned change is adopted without passing the eval threshold and clearing the confidence gate; unreported confidence fails safe.

**Related:** LRN-4; AI-1 (`ConfidenceGate` reused), MOD-3/PE-1 (eval score), LRN-1 (produces candidates); ADR-010/015 (core contract when a second consumer is real); ADR-031 (LRN-1 recorded-not-adopted — this gate is its adoption prerequisite); AI-2 (human review surface); FI-LRN4-A/B/C.

---

## ADR-033 — Autonomous locator healing: deterministic heuristic healer behind a seam + confidence-gated auto-apply

**Status:** Accepted (implemented — HEAL-1, 2026-07-28)

**Context.**
The healing loop's mechanics exist (`SelfHealingEngineImpl` does loop-protection, retry via `ExecutionEngine`, memory), but its *brain* did not: `RecoveryStrategyResolver` maps `LOCATOR_UPDATE` to a strategy *name string* with no component that actually finds a better locator. The roadmap wants "AI finds a better locator" using `ai-provider` and governed by AI-1 — but `ai-qa-os-healing` depends on neither `ai-provider` nor a live browser/LLM in this environment.

**Decision.**
- **Deterministic heuristic healer behind a seam (§0.4-A)** — a `LocatorHealer` interface with a `HeuristicLocatorHealer` that generates candidate locators from the element's attributes, preferring the most robust strategy (`data-testid` 0.95 → `id` 0.90 → `name` 0.85 → `role` 0.80 → `text` 0.70 → `css` 0.55 → relaxed `xpath` 0.35), plus a relaxed fallback parsed from the broken locator (positional index stripped). De-duplicated, ranked best-first. This is how real self-healing frameworks primarily work; an LLM-backed healer implements the same seam later (FI-HEAL1-B) with no `ai-provider` edge forced now.
- **Confidence-gated auto-apply, reusing AI-1** — `LocatorHealingService` gates the best candidate through the reused `ConfidenceGate` (optional, `@Autowired(required=false)`): `PROCEED`/`PROCEED_WITH_VALIDATION` auto-apply; anything else is surfaced for review (HEAL-2), never silently applied. When no gate is present, a local `aiqaos.healing.locator.min-confidence` (0.70) decides — same monotonic-safety stance as LRN-4.
- **Additive** — the engine is untouched; wiring the healer into `SelfHealingEngineImpl` is deferred (FI-HEAL1-A).

**Consequences.**
- *Positive:* the loop's missing locator-reasoning + governance now exist, deterministic and unit-proven (11 tests); reuses AI-1; no new module/dependency edge; non-breaking (additive — existing healing tests unaffected).
- *Negative / trade-off:* no LLM reasoning yet (FI-HEAL1-B); the healer isn't wired into `SelfHealingEngineImpl` (needs a failure→broken-locator extraction step — FI-HEAL1-A); no live validation/retry of the healed locator or script rewrite (needs a browser — FI-HEAL1-C); cross-run healed-locator memory is HEAL-4.
- *Imposed rule:* a healed locator is auto-applied only when it clears the confidence gate; low/unreported confidence is surfaced for review, never silently applied.

**Related:** HEAL-1; AI-1 (`ConfidenceGate` reused), LRN-1 (durable improvement); ADR-032/LRN-4 (confidence-gated monotonic safety — sibling stance); HEAL-2 (approval workflow), HEAL-4 (healing memory); `ai-provider` (deferred LLM backend); FI-HEAL1-A/B/C.

---

## ADR-034 — Learning metrics computed in-module over a supplied observation series

**Status:** Accepted (implemented — LRN-2, 2026-07-28)

**Context.**
LRN-2 must make "is the platform getting better?" measurable — Learning Score, Success Rate, Confidence History. The roadmap computes them in `ai-qa-os-learning` from execution history + `ReasoningTraceEntity`/confidence and stores them as observability metrics. But `ai-qa-os-learning` depends on `core`/`memory`/`agents` — **not `brain`** (where `ReasoningTraceEntity` lives) **nor `observability`** (where metrics are stored). The metric *definitions* are pure arithmetic; the sourcing and storage are cross-module wiring.

**Decision.**
- **Compute in-module over supplied observations (§0.4-A)** — a `LearningMetricsCalculator` computes `LearningMetrics` (Learning Score, Success Rate, Confidence History, `LearningTrend`) from a time-ordered `List<LearningObservation>` (each: outcome + confidence). Pure, deterministic; no `brain`/`observability` edge.
- **Trend by half-split** — the second half's success rate vs the first half's; `> +epsilon` IMPROVING, `< −epsilon` REGRESSING, else STABLE (epsilon default 0.05).
- **Composite Learning Score** — `successWeight·successRate + confidenceWeight·avgConfidence + trendWeight·trendBonus`, clamped [0,1] (defaults 0.5/0.3/0.2; trendBonus improving 1.0 / stable 0.5 / regressing 0.0). All weights + epsilon config-overridable (`aiqaos.learning.metrics.*`).
- **Sourcing + storage deferred** — feeding observations from real execution history/`ReasoningTraceEntity` (FI-LRN2-A) and persisting to observability (FI-LRN2-B) belong with the LRN-3 dashboard and a history-read seam.

**Consequences.**
- *Positive:* the three metrics + a trend signal are defined and unit-proven (7 tests); deterministic, non-breaking, no module-graph change; the `REGRESSING` trend is a ready stop-loss signal for LRN-4.
- *Negative / trade-off:* metrics compute over *supplied* observations, not yet real traces (FI-LRN2-A); nothing is persisted to observability yet (FI-LRN2-B); no dashboard (LRN-3) or regression alerting (FI-LRN2-C).
- *Imposed rule:* learning improvement is measured (Learning Score + trend), not asserted.

**Related:** LRN-2; LRN-1 (the loop measured); LRN-3 (dashboard), LRN-4 (can consume the trend as stop-loss); `ReasoningTraceEntity` (brain) / observability (deferred sources/sink); FI-LRN2-A/B/C.

---

## ADR-035 — Healing approval: strict confidence tier for script edits + in-module approval lifecycle

**Status:** Accepted (implemented — HEAL-2, 2026-07-28)

**Context.**
Auto-editing a test script (HEAL-1's endpoint) is the platform's highest-risk autonomous action — a wrong heal can mask a real bug. HEAL-2 adds a healing-confidence tier + an approval workflow for anything short of unambiguous confidence, reusing AI-1's gate and AI-2's review workflow ("composition of existing gates"). But `ai-qa-os-healing` depends on `core` (so AI-1's `ConfidenceGate` is reusable) — **not `orchestration`**, where AI-2's `HumanReviewService`/`PausedWorkflowRegistry` live.

**Decision.**
- **Strict tier (§0.4-A)** — `HealingApprovalService` maps the reused AI-1 verdict: `PROCEED` → `AUTO_APPROVED`; `PROCEED_WITH_VALIDATION`/`HUMAN_REVIEW` → `PENDING_APPROVAL` (a human must approve before the script is edited); `UNGATED` (unreported) → `REJECTED` (fail-safe). For the highest-risk action, "proceed *with validation*" means "a human validates", not "apply anyway" — deliberately stricter than the generic pipeline gate.
- **In-module approval lifecycle over a store seam** — pending heals are recorded in a `HealingApprovalStore` (`InMemoryHealingApprovalStore` reference, `@ConditionalOnMissingBean`); `approve`/`reject` transition a pending request to `APPROVED`/`REJECTED`; `pending()` lists outstanding ones.
- **Gate-absent fallback** — when the `ConfidenceGate` bean is absent, local `aiqaos.healing.approval.auto-approve` (0.90) / `review-floor` (0.50) thresholds decide.
- **AI-2 routing deferred** — backing the store with AI-2's durable `HumanReviewService` (`orchestration`) is a seam impl (FI-HEAL2-A), so `healing` needn't depend on `orchestration`.

**Consequences.**
- *Positive:* the highest-risk action is now gated + human-approvable, unit-proven (8 tests); reuses AI-1; non-breaking (additive, no schema/API, no new module edge); the store seam lets AI-2's durable store drop in later.
- *Negative / trade-off:* pending heals aren't yet routed to AI-2's durable store/dashboard (FI-HEAL2-A/B); applying an approved heal to the script is HEAL-1's deferred FI-HEAL1-C; no reviewer notification (ENT-2).
- *Imposed rule:* a script is auto-edited only on unambiguous confidence; anything softer waits for human approval; unreported confidence is rejected.

**Related:** HEAL-2; HEAL-1 (produces the heals), AI-1 (`ConfidenceGate` reused), AI-2 (`HumanReviewService` — deferred durable sink); ADR-032/033 (confidence-gated monotonic safety — sibling stance); FI-HEAL2-A/B.

---

## ADR-036 — Plugin architecture: in-process contract + lifecycle registry with semver + permission governance

**Status:** Accepted (implemented — PLG-1, 2026-07-28)

**Context.**
The platform had plugin-shaped pieces (`ExecutionPlugin` in `execution`, `PluginStep`/`GithubPlugin`/… in `orchestration`) but no common contract, lifecycle, registry, versioning, or permission model — every extension was a core change. PLG-1 introduces the controlled extension seam. The roadmap places the plugin runtime in `ai-qa-os-integration` (the cross-system module).

**Decision.**
- **In-process registry + managed lifecycle (§0.4-A)** — a `Plugin` SPI (id + `initialize`/`onEnable`/`onDisable` default hooks), a `PluginManifest` (id/version/sdkApiVersion/capabilities/requiredPermissions), and a `PluginRegistry` driving `REGISTERED → INITIALIZED → ENABLED ⇄ DISABLED` (invalid transitions refused; `FAILED` on a throwing hook). No dynamic class-loading/signing/sandbox.
- **Governance at register** — the registry refuses a duplicate id, an id/manifest mismatch, an **incompatible SDK version** (`SemanticVersion`: same major and runtime minor ≥ plugin minor), or a **required permission not in the granted set** (`aiqaos.plugins.*`).
- **Contract stays in `integration`, not `core`** — the only consumers now are the runtime + the future SDK (DX-5); per the ConfidenceGate lesson (ADR-010/015), don't promote to `core` until a cross-module implementer is real. Migrating the existing plugins (in `orchestration`/`execution`) is deferred (FI-PLG1-C).

**Consequences.**
- *Positive:* the extension contract + lifecycle + versioning + permission governance the SDK/PLG-2/marketplace build on now exist, unit-proven (13 tests); non-breaking (additive; existing plugins untouched).
- *Negative / trade-off:* no dynamic JAR discovery/loading/unload (FI-PLG1-A); no signature verification (SEC-6) or sandboxing (FI-PLG1-B); the existing plugins aren't migrated onto the contract yet (FI-PLG1-C); no durable registry/marketplace (PLG-4).
- *Imposed rule:* a plugin is admitted only when semver-compatible and its required permissions are granted; lifecycle transitions are validated, not implicit.

**Related:** PLG-1; existing `PluginStep`/`ExecutionPlugin` (deferred migration); DX-5 (SDK — re-exposes the contract), PLG-2 (integration plugins), PLG-4 (marketplace), SEC-6 (signing); ADR-010/015 (core-contract only when a cross-module implementer is real); FI-PLG1-A/B/C.

---

## ADR-037 — Integration plugins normalised on the PLG-1 contract via delegating adapters + governed registrar

**Status:** Accepted (implemented — PLG-2, 2026-07-29)

**Context.**
Integrations (GitHub/Jira/Slack existing as bespoke `PluginStep`s in `orchestration`; GitLab/Azure DevOps/Jenkins/Teams absent) needed to become uniform plugins on the PLG-1 contract. `ai-qa-os-integration` depends on `orchestration`, so the existing plugins can be reused by delegation. Today's plugins already return simulated strings (no live API), consistent with the environment.

**Decision.**
- **Uniform `IntegrationPlugin` on PLG-1 (§0.4-A)** — `IntegrationPlugin extends Plugin` adds a `PluginManifest`, an `IntegrationCategory` (SCM/ALM/CI/CHAT), and `execute(IntegrationRequest) → IntegrationResponse`. `AbstractIntegrationPlugin` holds the manifest. Seven plugins: GitHub/Jira/Slack **delegate to the existing `PluginStep` beans** (migration by reuse), GitLab/Azure DevOps/Jenkins/Teams are new (simulated).
- **Governed registrar** — `IntegrationPluginRegistrar.registerAll()` admits each into the `PluginRegistry` (register → initialize → enable), **gracefully skipping** (log, non-fatal) any whose required permission (`integration.<id>`) isn't granted or whose version is incompatible — integrations are off-by-default until an operator grants `aiqaos.plugins.granted-permissions`.
- **Live API deferred** — real REST clients per integration ride each integration's own follow-up (FI-PLG2-A) under the SEC-2 credential model; unchanged from today's simulated plugins.

**Consequences.**
- *Positive:* seven integrations sit behind one governed contract, unit-proven (7 tests); GitHub/Jira/Slack reuse existing logic; new systems are additive; safe default (off until permitted). Non-breaking (existing `PluginStep` beans untouched).
- *Negative / trade-off:* actions are simulated — no live external API yet (FI-PLG2-A); inbound webhook expansion for new CI systems beyond WF-2 is deferred (FI-PLG2-B); no plugin-admin surface.
- *Imposed rule:* an integration is admitted only under PLG-1 governance (version + granted permission); integrations stay off until explicitly permitted.

**Related:** PLG-2; PLG-1 (`Plugin`/`PluginRegistry` reused), existing `GithubPlugin`/`JiraPlugin`/`SlackPlugin` (delegated), WF-2 (CI webhook handlers), SEC-2 (credential model); DX-5 (SDK), PLG-4 (marketplace); FI-PLG2-A/B.

---

## ADR-038 — Learning loop closed end-to-end: LRN-1 proposals gated through the LRN-4 safe-adoption gate

**Status:** Accepted (implemented — FI-LRN4-A, 2026-07-29)

**Context.**
LRN-1 records typed `ImprovementProposal`s but deliberately does not adopt them; LRN-4 provides a `SafeAdoptionGate` (in `core`) that admits a candidate only when it passes eval + confidence. The two were built as separate, safe halves (ADR-031, ADR-032) with the connection deferred as FI-LRN4-A. This wire connects them so the learning loop is governed end-to-end without a cross-module dependency (the gate contract is in `core`, which `learning` already depends on — not `brain`).

**Decision.**
- **`ProposalAdoptionCoordinator` (in `learning`)** maps an `ImprovementProposal` → core `AdoptionCandidate` (`ProposalType` → `AdoptionKind` by name; `evalScore` supplied by the caller; the proposal's confidence as the trust score) and runs it through the injected `SafeAdoptionGate` (optional, `@Autowired(required=false)`).
- **Fail-safe when ungated** — with no `SafeAdoptionGate` on the classpath, proposals are `REJECTED_FOR_REVIEW`, never adopted ungated (the ADR-031/032 monotonic stance).
- **evalScore stays supplied** — running real MOD-3/PE-1 eval to produce it remains FI-LRN4-B; the *wiring* is now real, the *score source* is the remaining deferral.

**Consequences.**
- *Positive:* the loop is now end-to-end — failure patterns → reflection → proposals → **governed adoption decision** — unit-proven (5 tests); no `brain`/`eval` dependency added (the `core` gate contract carries it); realizes FI-LRN4-A.
- *Negative / trade-off:* the `evalScore` is still supplied, not computed live (FI-LRN4-B); admitted proposals aren't yet *applied* to prompts/scenarios/scripts (that adoption action remains FI-LRN4-A's downstream half / per-target wiring).
- *Imposed rule:* a recorded improvement reaches adoption only through the safe-adoption gate; ungated contexts reject.

**Related:** FI-LRN4-A; LRN-1/ADR-031 (records proposals), LRN-4/ADR-032 (`SafeAdoptionGate`); ADR-010/015 (core contract enabling the cross-module connection); FI-LRN4-B (live eval score).

---

## ADR-039 — Self-healing loop closed end-to-end: HEAL-1 locator proposal governed by the HEAL-2 approval workflow

**Status:** Accepted (implemented — HEAL-1 × HEAL-2 bridge, 2026-07-29)

**Context.**
HEAL-1 proposes a better locator (heuristic healer + candidate confidence); HEAL-2 provides the confidence-tiered approval workflow (auto-approve / pending / reject). They were built as separate halves (ADR-033, ADR-035) in the same module. Connecting them turns "find a locator" + "govern a script edit" into one flow.

**Decision.**
- **`LocatorHealCoordinator` (in `healing`)** proposes the best candidate via HEAL-1's `LocatorHealer`, then routes it through HEAL-2's `HealingApprovalService.decide(...)` using the candidate's confidence — one call yields a governed `LocatorHealResult` (chosen candidate + approval decision). A strong locator (`data-testid`) auto-approves; a brittle one (bare `xpath`) is routed to human approval; an unfindable one yields no candidate.
- **HEAL-2's strict tier is the authority** — the coordinator uses the approval workflow (3-tier + lifecycle), not HEAL-1's binary gate, as the single decision point, so a script is auto-edited only on unambiguous confidence.

**Consequences.**
- *Positive:* the self-healing loop is end-to-end — broken locator → proposed replacement → governed decision — unit-proven (3 tests); both halves reused unchanged; non-breaking (additive).
- *Negative / trade-off:* actually applying an auto-approved locator to the persisted script + live validation remain deferred (FI-HEAL1-C); wiring into `SelfHealingEngineImpl`'s failure path (failure→locator extraction) remains FI-HEAL1-A; routing pending heals to AI-2's durable store remains FI-HEAL2-A.
- *Imposed rule:* a proposed locator reaches a script only through the HEAL-2 approval tier.

**Related:** HEAL-1/ADR-033 (locator healer), HEAL-2/ADR-035 (approval workflow); mirrors ADR-038 (learning loop closed end-to-end); FI-HEAL1-A/C, FI-HEAL2-A.

---

## ADR-040 — Governance loop closed: version promotion policy-gated by the Responsible-AI policy

**Status:** Accepted (implemented — GOV-3 × GOV-4 bridge, 2026-07-29)

**Context.**
GOV-4 pins/rolls-back versions (`VersionRegistry`); GOV-3 evaluates AI content against Responsible-AI rules (`ResponsibleAiPolicyEngine`). Both live in `ai-qa-os-intelligence`. A governed version registry should not be able to promote content the policy forbids — the two were built separately (ADR-028, ADR-029) with the connection deferred (FI-GOV3-D / FI-GOV4-C).

**Decision.**
- **`GovernedVersionPromoter` (in `intelligence`)** — before `VersionRegistry.pin` activates a version, its content is evaluated by the `ResponsibleAiPolicyEngine`. If the policy permits (`ALLOW`/`WARN`) the version is pinned (`PROMOTED`); if it violates (`BLOCK`/`REQUIRE_REVIEW`) the promotion is **blocked — no pin** — returning the offending `PolicyDecision`.
- **In-module connection** — both halves are in `intelligence`, so no new dependency edge; the promoter composes the two existing services.

**Consequences.**
- *Positive:* a governed pin can never activate policy-forbidden content (PII, production URLs, destructive text); unit-proven (4 tests, incl. "a blocked promotion doesn't change the active version"); realizes FI-GOV3-D / FI-GOV4-C; non-breaking (additive — `VersionRegistry.pin` still available directly for ungoverned/system pins).
- *Negative / trade-off:* the promoter must be *given* the version content to check (callers that pin via the raw registry bypass the gate — promotion-through-the-promoter is the governed path by convention, not yet enforced); policy-decision auditing into GOV-1 remains open.
- *Imposed rule:* a version promotion that must be governed goes through `GovernedVersionPromoter`; policy-violating content is never pinned.

**Related:** GOV-4/ADR-028 (`VersionRegistry`), GOV-3/ADR-029 (`ResponsibleAiPolicyEngine`); mirrors ADR-038/039 (loops closed end-to-end); FI-GOV3-D, FI-GOV4-C; GOV-1 (audit — open).

---

## ADR-041 — Multi-tenancy foundation: tenant-context contract in `core` with thread-local + MDC propagation

**Status:** Accepted (implemented — ENT-1 foundation, 2026-07-29) · ENT-1 remains In Progress

**Context.**
ENT-1 (multi-tenancy / project isolation) is Effort XL — it threads a tenant boundary through persistence, security, memory, cost, and execution; today both apps share one database with no tenant boundary. The roadmap is explicit: *"Best done early conceptually (as a context contract in `core`) even if enforcement lands incrementally."* MOD-1 likewise specifies the tenant context contract should live in `core` so every module references it without a cycle.

**Decision.**
- **Tenant-context contract in `core` (§0.4-A)** — `TenantContext` (immutable `tenantId`/`projectId`, with a `system()` context for platform-internal work), `Tenanted` (the marker tenant-owned data implements — the persistence dimension), and `TenantContextHolder` (a `ThreadLocal` current-tenant binding mirrored into `MDC` for tenant-attributed logs, with scoped `run`/`call` that restore the previous context — the same set-and-clear-in-`finally` discipline as `CorrelationIdFilter`).
- **Enforcement lands incrementally** — the `ai-qa-os-tenant` module, gateway tenant-resolution filter, row-level persistence tenancy, tenant-scoped RBAC/memory/cost, and artifact scoping are each their own follow-up (FI-ENT1-A…E). **ENT-1 stays In Progress** — this is the foundation, not the whole XL item.

**Consequences.**
- *Positive:* the tenancy dimension the whole platform lacked now has a `core` home every module can reference without a cycle; the holder is unit-proven (8 tests: bind/clear, require, nested scoped-restore, MDC attribution); zero blast radius (additive, no schema/API, nothing else changed); also satisfies MOD-1's core-contract half.
- *Negative / trade-off:* no data is actually isolated yet — all enforcement is deferred; ENT-1 is not Completed. *(Resolved 2026-07-29: `TenantContextException` was placed in `core.tenant`, violating the ArchUnit "exceptions live in `..exception..`" rule — moved to `core.exception`. The full reactor `mvn clean test` is BUILD SUCCESS across all 22 modules with `-DargLine="-Xss40m"` for ArchUnit's importer on Java 25.)*
- *Imposed rule:* the tenant boundary is a first-class `core` context, bound and propagated (MDC-attributed) with safe scoped restore; enforcement layers key on `Tenanted`.

**Related:** ENT-1 (foundation), MOD-1 (core contract satisfied; module deferred); `CorrelationIdFilter` (propagation pattern mirrored); ADR-010/015 (core contract for a cross-cutting dimension); FI-ENT1-A/B/C/D/E.

---

## ADR-042 — `ai-qa-os-tenant` module: tenant registry seam + active-only resolver over the `core` context

**Status:** Accepted (implemented — MOD-1, 2026-07-29)

**Context.**
ENT-1/ADR-041 put the tenant *context contract* in `core`, but no module owned tenant *identity* — the roadmap's "Project Manager" and the multi-tenancy foundation (MOD-1). A new module was needed to register tenants and resolve a tenant key into a `core` `TenantContext`.

**Decision.**
- **New `ai-qa-os-tenant` module (22nd)** depending on `core` only, registered in the parent reactor.
- **Registry behind a seam (§0.4-A)** — `Tenant` (id/name/projectIds/`TenantStatus`), a `TenantRegistry` interface with an `InMemoryTenantRegistry` reference (`@ConditionalOnMissingBean`; register/find/all/activate/suspend). A durable JPA registry is a drop-in seam impl later (FI-MOD1-A) — consistent with every other seam this session.
- **Active-only resolver** — `TenantResolver.resolve(tenantId, projectId)` returns a `core` `TenantContext` only for a known, `ACTIVE` tenant that owns the requested project; otherwise `TenantResolutionException`. This is the bridge a gateway filter will call per request (deferred, FI-ENT1-B).

**Consequences.**
- *Positive:* the platform finally has an owner for tenant identity + lifecycle, and a governed path from a tenant key to a bindable `core` context; unit-proven (9 tests); additive new module, no blast radius. Advances ENT-1 (FI-ENT1-A realized).
- *Negative / trade-off:* the registry is in-memory (durable JPA deferred — FI-MOD1-A); nothing binds the resolved context per request yet (gateway filter — FI-ENT1-B); enforcement (persistence/RBAC/memory) remains ENT-1's deferred work. *Validated with a constrained heap; the machine-memory ArchUnit caveat from ADR-041 still applies to the full core gate.*
- *Imposed rule:* a tenant key resolves to a usable context only when the tenant is known and active.

**Related:** MOD-1; ENT-1/ADR-041 (`core` `TenantContext` reused); FI-MOD1-A (durable registry), FI-ENT1-B (gateway filter), FI-ENT1-C/D/E (enforcement).

---

## ADR-043 — Gateway tenant-resolution filter: request-scoped tenant binding (ENT-1 FI-ENT1-B)

**Status:** Accepted (implemented — FI-ENT1-B, 2026-07-29) · ENT-1 remains In Progress

**Context.**
ENT-1's `core` `TenantContext` (ADR-041) and MOD-1's `TenantResolver` (ADR-042) existed, but nothing bound a tenant per request — the runtime half of multi-tenancy. The gateway is the single public entry point (ADR-004) and already has the `CorrelationIdFilter` set-and-clear pattern to mirror.

**Decision.**
- **`TenantContextFilter` (gateway, `@Order(2)`)** — reads `X-Tenant-ID` (+ optional `X-Project-ID`), resolves it via MOD-1's `TenantResolver` to a `core` `TenantContext`, and binds it through `TenantContextHolder` with a `finally`-clear, exactly like `CorrelationIdFilter` (`@Order(1)`, runs first).
- **No header → proceed unbound** (tenancy optional at this layer; code needing a tenant calls `require()`); **unknown/suspended tenant → 400** before the chain runs (a bad tenant never reaches downstream handlers).
- **Wiring** — `ai-qa-os-gateway` now depends on `ai-qa-os-tenant`; the gateway's `com.aiqaos` component scan picks up `TenantResolver`/`InMemoryTenantRegistry`.

**Consequences.**
- *Positive:* a request now binds its tenant for the whole call (MDC-attributed via the holder), and bad tenants are rejected at the edge; unit-proven (4 tests via `MockHttpServletRequest` + a capturing chain, no Mockito). The tenancy path is now end-to-end at the request boundary.
- *Negative / trade-off:* binding a context does not yet *isolate* anything — persistence row-scoping, tenant-scoped RBAC/memory/cost still key on `Tenanted` and remain deferred (FI-ENT1-C/D/E). Live end-to-end through a running gateway is unvalidated here (no server); the filter logic is unit-proven. ENT-1 stays In Progress.
- *Imposed rule:* the gateway binds the request's tenant (or rejects a bad one) before downstream handlers run.

**Related:** FI-ENT1-B; ENT-1/ADR-041 (`TenantContextHolder`), MOD-1/ADR-042 (`TenantResolver`), ADR-004 (gateway as single entry), `CorrelationIdFilter` (mirrored); FI-ENT1-C/D/E (enforcement).

---

## Document Completion Status

**Status:** Active — new ADRs appended as decisions are made (per MNT-5)
**Version:** 1.0.0
**Convention:** ADRs are immutable once Accepted; a changed decision is recorded as a new ADR that *Supersedes* the old one, never by editing history
**Related documents:** [`AI-QA-OS-Documentation.md`](./AI-QA-OS-Documentation.md) · [`AI-QA-OS-Improvement-Roadmap.md`](./AI-QA-OS-Improvement-Roadmap.md) · [`AI-QA-OS-Implementation-Tracker.md`](./AI-QA-OS-Implementation-Tracker.md)
