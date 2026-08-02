# AI-QA-OS — Architecture Decision Records (ADRs)

**Version:** 1.0.0
**Document Type:** Architecture Decision Log
**Document Status:** Active
**Last Updated:** 2026-07-30
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
| [ADR-028](#adr-028--durable-version-registry-with-pinrollback-behind-a-store-seam-prompt-now-generic-for-model) | Durable version registry with pin/rollback behind a store seam (prompt now, generic for model) | Accepted |
| [ADR-029](#adr-029--responsible-ai-policy-as-a-config-driven-guardrail-at-the-sec-3-boundary-opa-deferred) | Responsible-AI policy as a config-driven guardrail at the SEC-3 boundary (OPA deferred) | Accepted |
| [ADR-030](#adr-030--real-pii-masking-in-ai-qa-os-testdata-classification-driven-pluggable-strategies-format-preserving-default) | Real PII masking in `ai-qa-os-testdata`: classification-driven, pluggable strategies, format-preserving default | Accepted |
| [ADR-031](#adr-031--close-the-continuous-learning-loop-via-recorded-improvement-proposals-adoption-gated-on-lrn-4) | Close the continuous-learning loop via recorded improvement proposals; adoption gated on LRN-4 | Accepted |
| [ADR-032](#adr-032--safe-adoption-gate-core-contract--brain-impl-reusing-the-confidence-gate--an-eval-threshold) | Safe-adoption gate: core contract + Brain impl reusing the confidence gate + an eval threshold | Accepted |
| [ADR-033](#adr-033--autonomous-locator-healing-deterministic-heuristic-healer-behind-a-seam--confidence-gated-auto-apply) | Autonomous locator healing: deterministic heuristic healer behind a seam + confidence-gated auto-apply | Accepted |
| [ADR-034](#adr-034--learning-metrics-computed-in-module-over-a-supplied-observation-series) | Learning metrics computed in-module over a supplied observation series | Accepted |
| [ADR-035](#adr-035--healing-approval-strict-confidence-tier-for-script-edits--in-module-approval-lifecycle) | Healing approval: strict confidence tier for script edits + in-module approval lifecycle | Accepted |
| [ADR-036](#adr-036--plugin-architecture-in-process-contract--lifecycle-registry-with-semver--permission-governance) | Plugin architecture: in-process contract + lifecycle registry with semver + permission governance | Accepted |
| [ADR-037](#adr-037--integration-plugins-normalised-on-the-plg-1-contract-via-delegating-adapters--governed-registrar) | Integration plugins normalised on the PLG-1 contract via delegating adapters + governed registrar | Accepted |
| [ADR-038](#adr-038--learning-loop-closed-end-to-end-lrn-1-proposals-gated-through-the-lrn-4-safe-adoption-gate) | Learning loop closed end-to-end: LRN-1 proposals gated through the LRN-4 safe-adoption gate | Accepted |
| [ADR-039](#adr-039--self-healing-loop-closed-end-to-end-heal-1-locator-proposal-governed-by-the-heal-2-approval-workflow) | Self-healing loop closed end-to-end: HEAL-1 locator proposal governed by the HEAL-2 approval workflow | Accepted |
| [ADR-040](#adr-040--governance-loop-closed-version-promotion-policy-gated-by-the-responsible-ai-policy) | Governance loop closed: version promotion policy-gated by the Responsible-AI policy | Accepted |
| [ADR-041](#adr-041--multi-tenancy-foundation-tenant-context-contract-in-core-with-thread-local--mdc-propagation) | Multi-tenancy foundation: tenant-context contract in `core` with thread-local + MDC propagation | Accepted |
| [ADR-042](#adr-042--ai-qa-os-tenant-module-tenant-registry-seam--active-only-resolver-over-the-core-context) | `ai-qa-os-tenant` module: tenant registry seam + active-only resolver over the `core` context | Accepted |
| [ADR-043](#adr-043--gateway-tenant-resolution-filter-request-scoped-tenant-binding-ent-1-fi-ent1-b) | Gateway tenant-resolution filter: request-scoped tenant binding (ENT-1 FI-ENT1-B) | Accepted |
| [ADR-044](#adr-044--ai-qa-os-notification-module-single-governed-egress-point-via-a-channel-sender-spi) | `ai-qa-os-notification` module: single governed egress point via a channel-sender SPI | Accepted |
| [ADR-045](#adr-045--first-class-agent-roster-catalog-agt-1-increment) | First-class agent roster catalog (AGT-1 increment) | Accepted |
| [ADR-046](#adr-046--tenant-scoped-cross-run-healing-memory-over-memorystore) | Tenant-scoped cross-run healing memory over `MemoryStore` | Accepted |
| [ADR-047](#adr-047--healing-analytics-read-model-via-a-pure-assembler-heal-3-backend-increment) | Healing analytics read-model via a pure assembler (HEAL-3 backend increment) | Accepted |
| [ADR-048](#adr-048--extension-sdk-uniform-extension-spi-in-core--governed-registry-in-integration) | Extension SDK: uniform `Extension` SPI in `core` + governed registry in `integration` | Accepted |
| [ADR-049](#adr-049--centralised-eventnotification-routing--templating-over-mod-2) | Centralised event→notification routing + templating over MOD-2 | Accepted |
| [ADR-050](#adr-050--learning-dashboard-read-model--health-signal-in-learning-lrn-3-backend-increment) | Learning-dashboard read-model + health signal in `learning` (LRN-3 backend increment) | Accepted |
| [ADR-051](#adr-051--prompt-quality-read-model-over-the-pe-2-leaderboard-pe-3-backend-increment) | Prompt-quality read-model over the PE-2 leaderboard (PE-3 backend increment) | Accepted |
| [ADR-052](#adr-052--rbac-admin-read-model-over-the-security-entities-ent-4-backend-increment) | RBAC admin read-model over the security entities (ENT-4 backend increment) | Accepted |
| [ADR-053](#adr-053--local-infrastructure-stack-a-compose-spring-profile-over-provisioned-containers-with-real-service-bindings-deferred-to-their-consumers) | Local infrastructure stack: a `compose` Spring profile over provisioned containers, with real service bindings deferred to their consumers | Accepted |
| [ADR-054](#adr-054--row-level-persistence-tenancy-via-hibernate-tenantid-discriminator-fi-ent1-c-pilot) | Row-level persistence tenancy via Hibernate `@TenantId` discriminator (FI-ENT1-C pilot) | Accepted |
| [ADR-055](#adr-055--tenant-scoped-rbac-users-tenant-owned-roles-a-global-catalog-jwt-authoritative-tenant-fi-ent1-d) | Tenant-scoped RBAC: users tenant-owned, roles a global catalog, JWT-authoritative tenant (FI-ENT1-D) | Accepted |
| [ADR-056](#adr-056--tenant-scoped-memory-cost--artifact-tenantid-on-the-durable-rows--tenant-key-prefix-for-blobs-fi-ent1-e) | Tenant-scoped memory, cost & artifact: `@TenantId` on the durable rows + tenant key-prefix for blobs (FI-ENT1-E) | Accepted |
| [ADR-057](#adr-057--tenant-scope-the-remaining-operational-tables-catalogs-global-telemetry-system-scoped-fi-ent1-c-extension) | Tenant-scope the remaining operational tables; catalogs global, telemetry system-scoped (FI-ENT1-C extension) | Accepted |
| [ADR-058](#adr-058--credential-entities-session-api-key-are-tenant-attributed-not-tenantid-discriminated-fi-ent1-d-slice-2) | Credential entities (session, API key) are tenant-attributed, not `@TenantId`-discriminated (FI-ENT1-D slice 2) | Accepted |
| [ADR-059](#adr-059--runtime-tenant-scoping-of-short-term-memory-vector-search-and-local-artifacts-fi-ent1-e-slice-2) | Runtime tenant-scoping of short-term memory, vector search, and local artifacts (FI-ENT1-E slice 2) | Accepted |
| [ADR-060](#adr-060--in-process-eventbus-coordination-seam-over-core-baseevent-distributed-binding-deferred-scale-2) | In-process `EventBus` coordination seam over `core` `BaseEvent`; distributed binding deferred (SCALE-2) | Accepted |
| [ADR-061](#adr-061--bridge-the-core-eventbus-seam-to-spring-events-during-publisher-migration-fi-scale2-a) | Bridge the `core` `EventBus` seam to Spring events during publisher migration (FI-SCALE2-A) | Accepted |
| [ADR-062](#adr-062--serve-dashboard-read-models-by-aggregating-persisted-data-pe-3-from-eval_results-lrn-3-deferred-no-faithful-source) | Serve dashboard read-models by aggregating persisted data; PE-3 from `eval_results`, LRN-3 deferred (no faithful source) | Accepted |
| [ADR-063](#adr-063--lrn-3-dashboard-deferred-its-observation-pipeline-is-unbuilt-no-empty-dashboard-scaffolding) | LRN-3 dashboard deferred: its observation pipeline is unbuilt; no empty-dashboard scaffolding | Accepted |
| [ADR-064](#adr-064--distributed-eventbus-over-kafka-kafkaeventbus-behind-an-optional-dependency-scale-2-kafka-binding) | Distributed `EventBus` over Kafka: `KafkaEventBus` behind an optional dependency (SCALE-2 Kafka binding) | Accepted |
| [ADR-065](#adr-065--distributed-executionjobqueue-over-redis-streams-redisstreamexecutionjobqueue-behind-an-optional-dependency-scale-1) | Distributed `ExecutionJobQueue` over Redis Streams: `RedisStreamExecutionJobQueue` behind an optional dependency (SCALE-1) | Accepted |
| [ADR-066](#adr-066--userrole-mapping-via-elementcollection--role-derived-authorities-fi-ent4-c) | User↔role mapping via `@ElementCollection` + role-derived authorities (FI-ENT4-C) | Accepted |
| [ADR-067](#adr-067--admin-write-ops-api-at-apiadmin-on-the-enforced-chain-admin-gated-fi-ent4-a) | Admin write-ops API at `/api/admin/**` on the enforced chain, ADMIN-gated (FI-ENT4-A) | Accepted |
| [ADR-068](#adr-068--real-object-storage-binding-over-s3-s3objectstorageclient-behind-an-optional-dependency-ent-5) | Real object-storage binding over S3: `S3ObjectStorageClient` behind an optional dependency (ENT-5) | Accepted |
| [ADR-069](#adr-069--prompt-regression-detection-via-temporal-within-version-score-decline-fi-pe3-b) | Prompt regression detection via temporal within-version score decline (FI-PE3-B) | Accepted |
| [ADR-070](#adr-070--heal-3-locator-drift-ranking-fi-heal3-b-deferred-no-faithful-enumerable-drift-source) | HEAL-3 locator-drift ranking (FI-HEAL3-B) deferred: no faithful, enumerable drift source | Accepted |
| [ADR-071](#adr-071--execution-worker-artifact-upload-into-artifactstore-via-a-deterministic-key-fi-ent5-a) | Execution-worker artifact upload into `ArtifactStore` via a deterministic key (FI-ENT5-A) | Accepted |
| [ADR-072](#adr-072--heal-3-persisted-locator-store-fi-heal3-a-deferred-the-locator-healing-subsystem-is-unwired-end-to-end) | HEAL-3 persisted locator store (FI-HEAL3-A) deferred: the locator-healing subsystem is unwired end-to-end | Accepted |
| [ADR-073](#adr-073--serve-execution-artifacts-from-artifactstore-via-an-additive-key-endpoint-fi-ent5-c) | Serve execution artifacts from `ArtifactStore` via an additive key endpoint (FI-ENT5-C) | Accepted |
| [ADR-074](#adr-074--runnable-backup-cronjobs-to-object-storage-fi-ent5-b) | Runnable backup CronJobs to object storage (FI-ENT5-B) | Accepted |
| [ADR-075](#adr-075--per-workflow-tokencontext-budgeting-mirroring-the-ent-3-cost-soft-cap-ai-6) | Per-workflow token/context budgeting mirroring the ENT-3 cost soft-cap (AI-6) | Accepted |
| [ADR-076](#adr-076--artifact-content-signing-hmac-sha256-for-tamper-evidence-sec-6) | Artifact content signing (HMAC-SHA256) for tamper-evidence (SEC-6) | Accepted |
| [ADR-077](#adr-077--remove-the-orphan-ai-qa-os-data-module-mod-5-fold-in) | Remove the orphan `ai-qa-os-data` module (MOD-5: fold in) | Accepted |

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

## ADR-044 — `ai-qa-os-notification` module: single governed egress point via a channel-sender SPI

**Status:** Accepted (implemented — MOD-2, 2026-07-29)

**Context.**
Outbound comms were a cross-cutting smear — `reporting`'s `NotificationFramework`/Slack/Teams senders, `orchestration`'s `SlackPlugin`, `observability`'s `NotificationAdapter`, and PLG-2's chat plugins — with no single egress point. MOD-2 centralises them; the roadmap says the existing `SlackPlugin` becomes an adapter, and the module depends on `core` + `integration`.

**Decision.**
- **New `ai-qa-os-notification` module (23rd)** depending on `core` + `integration`, registered in the reactor.
- **Channel-sender SPI + dispatcher (§0.4-A)** — a `NotificationSender` SPI (`channel()` + `send()`), per-channel senders (SLACK/EMAIL/WEBHOOK/TEAMS), and a `NotificationService` that routes a `Notification` to the matching sender; **a channel with no sender yields a failed `NotificationResult`, never an exception**.
- **Slack via the PLG-2 adapter** — `SlackNotificationSender` delegates to the PLG-2 `SlackIntegrationPlugin`, realising "SlackPlugin becomes an adapter"; Email/Webhook/Teams are simulated. Real transports (SMTP/live Slack/HTTP) are deferred behind the SPI (FI-MOD2-B).

**Consequences.**
- *Positive:* one governed egress point exists, unit-proven (5 tests); the Slack path reuses the PLG-2 plugin; additive module, no blast radius; all beans single-constructor plain `@Component`s (no repeat of the 2026-07-29 wiring bugs).
- *Negative / trade-off:* transports are simulated (FI-MOD2-B); the scattered `reporting`/`orchestration`/`observability` senders aren't migrated to route through it yet (FI-MOD2-A); event-driven triggering is ENT-2.
- *Imposed rule:* outbound comms go through the `NotificationService` egress point; an unconfigured channel fails gracefully, never breaks the caller.

**Related:** MOD-2; PLG-2/ADR-037 (`SlackIntegrationPlugin` reused as adapter), ENT-2 (event-driven notifications — consumer), SEC-2 (transport credentials); FI-MOD2-A/B.

---

## ADR-045 — First-class agent roster catalog (AGT-1 increment)

**Status:** Accepted (implemented — AGT-1 roster increment, 2026-07-29) · AGT-1 stays incremental/rolling

**Context.**
AGT-1 ("grow the agent roster incrementally", Effort XL, v2.1→v3.x) had no first-class *roster*: the runtime `AgentRegistry` tracks live `Agent` beans, but the roadmap's catalog of 18 agents (implemented + designed + future) with categories/status lived only in a roadmap table. Coverage and gaps weren't programmatic.

**Decision.**
- **`AgentRoster` catalog in `ai-qa-os-agents` (§0.4-A)** — `AgentDescriptor` (name, `AgentCategory`, `AgentStatus`, implementing class) seeded with the 18 roster entries; queries `byCategory`/`byStatus`/`find`/`categories` + a `coverageRatio` (implemented/total). Pure data + queries.
- **Specialist agents deferred** — implementing the 10 Future agents (API/Mobile/Performance/…) is the ongoing incremental work behind DX-2 scaffolding + PLG-3 execution SDKs (FI-AGT1-A); as each lands, its descriptor flips to `IMPLEMENTED`.
- **AGT-1 stays incremental/rolling** — the roster *mechanism* is done; the roster itself grows over releases.

**Consequences.**
- *Positive:* agent coverage (6/18) + gaps by category are now programmatic and unit-proven (7 tests); additive plain `@Component`, no blast radius; a stable place to track new agents as they're built.
- *Negative / trade-off:* the catalog is hand-seeded (not reconciled against live `AgentRegistry` beans — FI-AGT1-B); no new agents were built (that's the deferred incremental work).
- *Imposed rule:* new specialist agents are registered in the roster as they're scaffolded, keeping the coverage view honest.

**Related:** AGT-1; `AgentRegistry`/`AgentType` (runtime agents), DX-2 (scaffolding), PLG-3 (execution SDKs); FI-AGT1-A (build Future agents), FI-AGT1-B (reconcile with live beans).

---

## ADR-046 — Tenant-scoped cross-run healing memory over `MemoryStore`

**Status:** Accepted (implemented — HEAL-4, 2026-07-29)

**Context.**
HEAL-1 heals a locator per failure, but nothing remembered a validated heal for next time — so the loop's "Future Auto Recovery" was aspirational. HEAL-4 makes healing cross-run: recall a previously-validated locator when the same element drifts again (even in a different test) and flag fragile locators. It reuses `memory` (healing already depends on it) and should be tenant-scoped once ENT-1 lands — which it has (ADR-041).

**Decision.**
- **`HealingMemory` over `MemoryStore` (§0.4-A)** — `remember(brokenLocator, healedLocator, strategy, confidence)` stores a `HealedLocatorRecord` keyed by the broken locator; `recall(brokenLocator)` returns the previously-validated heal; `isKnownFragile` flags a locator that has drifted more than once. Reuses the `memory` infrastructure — no new module.
- **Tenant-scoped keys** — `healing:heal:<tenantId>:<brokenLocator>`, with `tenantId` from ENT-1's `TenantContextHolder.current()` (system tenant when unbound) — one project's heals never recall under another's, realising the roadmap's tenant-isolation requirement.
- **Fragility from repeat drift** — re-remembering the same broken locator increments `reuseCount` and marks it `fragile` (pre-emptive hardening signal).
- **Exact-key match; vector-similarity deferred** — fuzzy retrieval of *similar* elements/contexts via `VectorStoreClient` is the enrichment (FI-HEAL4-A).

**Consequences.**
- *Positive:* a validated heal is remembered and recalled instantly on recurrence, tenant-isolated; repeat drift flags fragility; unit-proven (5 tests incl. tenant isolation); additive single-constructor `@Component`, reuses `memory`, no new dep. First real consumer of ENT-1's tenant context beyond the gateway filter.
- *Negative / trade-off:* exact broken-locator key only — doesn't yet match a *similar* element in a different test (FI-HEAL4-A); recall-before-propose isn't wired into the heal flow yet (FI-HEAL4-B); the HEAL-3 locator-history surface is separate.
- *Imposed rule:* healing knowledge is cross-run and tenant-isolated; a locator that drifts twice is treated as fragile.

**Related:** HEAL-4; HEAL-1/ADR-033 (locator heals), ENT-1/ADR-041 (`TenantContext` — reused for scoping), `RecoveryHistoryStore`/`MemoryStore` (memory infra); mirrors LRN-1 (cross-run learning); HEAL-3 (locator history), FI-HEAL4-A/B.

---

## ADR-047 — Healing analytics read-model via a pure assembler (HEAL-3 backend increment)

**Status:** Accepted (implemented — HEAL-3 read-model, 2026-07-29) · HEAL-3 remains In Progress

**Context.**
HEAL-3 makes autonomous editing trustworthy via a healing dashboard + analytics + locator version history. The existing `HealingAnalyticsService` was thin (per-execution list + action-type breakdown) with no success-rate/summary read-model. In a no-frontend environment, the validatable substance is the analytics aggregation, not the UI.

**Decision.**
- **Pure analytics assembler (§0.4-A)** — `HealingAnalyticsAssembler.summarize(List<HealingMetricEntity>) → HealingAnalyticsSummary` (total, applied, successful, **success rate**, avg improvement score, action-type/recovery-status/failure-category breakdowns). No I/O — the GOV-1 `AiAuditAssembler` pattern — so it's trivially unit-testable. Exposed via `HealingAnalyticsService.getSummary()` (the assembler injected into the existing single-constructor service).
- **UI + versioned locator store deferred** — the dashboard UI is frontend (not verifiable here); a versioned locator store for locator version history needs persistence (FI-HEAL3-A); "most-drifting locators" needs a list-queryable index over HEAL-4's key-value `HealingMemory` (FI-HEAL3-B). **HEAL-3 stays In Progress.**

**Consequences.**
- *Positive:* healing is now auditable at a glance (success rate + breakdowns), unit-proven (4 tests); additive; the constructor change wired cleanly (dashboard `@SpringBootTest`/`@WebMvcTest` context green — no repeat of the earlier wiring bugs).
- *Negative / trade-off:* no UI rendered, no locator version history, no drift ranking yet — HEAL-3 is not Completed.
- *Imposed rule:* healing analytics are computed by a pure assembler over the metric records; presentation layers consume the read-model.

**Related:** HEAL-3; `HealingMetricEntity`/`HealingMetricRepository` (observability), HEAL-1/2/4 (the healed data), GOV-1/ADR-026 (assembler pattern), OBS-3 (dashboards suite); FI-HEAL3-A/B/C.

---

## ADR-048 — Extension SDK: uniform `Extension` SPI in `core` + governed registry in `integration`

**Status:** Accepted (implemented — PLG-3, 2026-07-29)

**Context.**
The platform's deepest extension points — custom agents, execution engines (only Playwright built; Selenium/REST-Assured/Appium designed), reporters, browsers — each had a per-type contract but no uniform, discoverable extension seam or SDK-version governance. PLG-3 exposes them as SDK contracts so teams add capability without touching core.

**Decision.**
- **`Extension` SPI in `core` (§0.4-A)** — `Extension` (`id`/`kind`/`extensionPoint`/`sdkApiVersion`) + `ExtensionKind` (AGENT/EXECUTION_ENGINE/REPORTER/BROWSER). Placed in `core` because the genuine implementers live in `agents`/`execution`/`reporting` (or third-party modules depending on them) — none depend on `integration` — the real cross-module implementer case (ADR-010/015), unlike PLG-1's plugins which live in `integration`.
- **`ExtensionRegistry` in `integration`** — governed registration: an extension's id must be unique within its kind, and its declared SDK API version must be compatible with the runtime, reusing **PLG-1's `SemanticVersion`**. Discovery by kind (`byKind`/`find`/`kinds`/`all`); `ExtensionSdkProperties` (`aiqaos.sdk.api-version`).
- **Building extensions deferred** — actual new engines/reporters/browsers are the ongoing work behind DX-2 + real frameworks (FI-PLG3-A); wiring the registry into selection paths is FI-PLG3-B.

**Consequences.**
- *Positive:* the four deep extension points are now a uniform, governed, discoverable seam, unit-proven (5 tests); reuses PLG-1's versioning; additive single-constructor `@Component`, no blast radius. This is how the platform gains API/mobile/performance testing without core rewrites.
- *Negative / trade-off:* no new extension built yet (FI-PLG3-A); the registry isn't wired into the execution/agent/reporter selection paths (FI-PLG3-B); no DX-2 scaffolding.
- *Imposed rule:* a deep extension implements the one `Extension` SPI and registers under SDK-version governance; ids are unique per kind.

**Related:** PLG-3; PLG-1/ADR-036 (`SemanticVersion` + registry pattern reused), `Agent`/`ExecutionEngine`/`ReportGenerator` (the per-type contracts unified), AGT-1 (agent extenders), DX-2 (scaffolding), PLG-4 (marketplace); FI-PLG3-A/B.

---

## ADR-049 — Centralised event→notification routing + templating over MOD-2

**Status:** Accepted (implemented — ENT-2, 2026-07-29)

**Context.**
Outbound comms were scattered; stakeholders need run-complete, failure, and approval-request notifications through one governed channel with templating — and AI-2's approval flow needs a way to actually reach a human. MOD-2 delivered the egress point (`NotificationService`), but nothing turned a platform event into a templated notification.

**Decision.**
- **`NotificationEventRouter` in `ai-qa-os-notification` (§0.4-A)** — turns a `NotificationEvent` (`RUN_COMPLETE`/`RUN_FAILURE`/`APPROVAL_REQUEST`) into a templated, severity-ranked `Notification` (`RUN_FAILURE`→CRITICAL, `APPROVAL_REQUEST`→WARNING, `RUN_COMPLETE`→INFO; per-type subject/body) on a resolved channel (event's own, else the configured `aiqaos.notification.default-channel`), then dispatches it through MOD-2's `NotificationService`. All run/approval comms flow through one governed path.
- **Live event wiring deferred** — `@EventListener` subscriptions to the real publishers (`WorkflowEventPublisher`/AI-2 review events, cross-module) and delivery-retry/outbox guarantees are deferred (FI-ENT2-A/B).

**Consequences.**
- *Positive:* the three named notification types are centralised, templated, and severity-ranked over the one egress point, unit-proven (5 tests); reuses MOD-2 (the Slack channel already delegates to the PLG-2 adapter); additive single-constructor `@Service`, no blast radius. Provides the concrete path for AI-2's approval to reach a human.
- *Negative / trade-off:* nothing fires it from real events yet (FI-ENT2-A); delivery is MOD-2's synchronous result — no retry/outbox (FI-ENT2-B); no scheduled digests.
- *Imposed rule:* run/approval notifications are built by templating an event and dispatched through the single `NotificationService` egress point.

**Related:** ENT-2; MOD-2/ADR-044 (`NotificationService` egress), AI-2 (approval flow this feeds), event publishers (`WorkflowEventPublisher`/etc. — deferred sources); FI-ENT2-A/B.

---

## ADR-050 — Learning-dashboard read-model + health signal in `learning` (LRN-3 backend increment)

**Status:** Accepted (implemented — LRN-3 read-model, 2026-07-29) · LRN-3 remains In Progress

**Context.**
LRN-3 makes the improvement trend visible so stakeholders trust autonomous operation — a dashboard view over LRN-2's metrics. LRN-2's `LearningMetrics` live in `ai-qa-os-learning`, and the **dashboard module does not depend on `learning`**, so the read-model that composes them belongs in `learning`; in a no-frontend environment the validatable substance is that composition + a health signal, not the UI.

**Decision.**
- **Read-model in `learning` (§0.4-A)** — a pure `LearningDashboardAssembler.assemble(LearningMetrics) → LearningDashboardView` (score, success rate, confidence-history series, trend, sample count) that derives a **`LearningHealth`** signal (`AT_RISK` when the trend is `REGRESSING` **or** the learning score `< 0.5`, else `HEALTHY`) plus a human headline. The HEAL-3/GOV-1 assembler pattern.
- **UI + controller deferred** — the dashboard UI is frontend; the dashboard-module controller/endpoint needs a `dashboard → learning` dependency (FI-LRN3-A); a persisted metric time-series is FI-LRN3-B. **LRN-3 stays In Progress.**

**Consequences.**
- *Positive:* learning health is now a programmatic, governance-useful signal (a stop-loss hook — an `AT_RISK` learning loop is visible), unit-proven (6 tests); additive no-arg `@Component` assembler, no blast radius.
- *Negative / trade-off:* no UI rendered, no endpoint, no historical time-series — LRN-3 is not Completed.
- *Imposed rule:* the learning dashboard consumes a composed read-model with an explicit health signal, not raw metrics.

**Related:** LRN-3; LRN-2/ADR-034 (`LearningMetrics` consumed), LRN-4 (the `REGRESSING`/at-risk signal can feed a stop-loss), HEAL-3/ADR-047 & GOV-1/ADR-026 (assembler pattern), OBS-3 (dashboards suite); FI-LRN3-A/B.

---

## ADR-051 — Prompt-quality read-model over the PE-2 leaderboard (PE-3 backend increment)

**Status:** Accepted (implemented — PE-3 read-model, 2026-07-29) · PE-3 remains In Progress

**Context.**
PE-3 lets prompt engineers see scores, standings, and leaderboard at a glance. PE-2's `LeaderboardEntry` rows live in `ai-qa-os-eval`, and the **dashboard module does not depend on `eval`**, so the read-model composing them belongs in `eval`; the validatable substance in a no-frontend environment is the composition, not the UI. Completes the eval spine MOD-3 → AI-3 → PE-1 → PE-2 → PE-3.

**Decision.**
- **Read-model in `eval` (§0.4-A)** — a pure `PromptQualityAssembler.summarize(List<LeaderboardEntry>) → PromptQualitySummary` (total versions, best/worst version + score, average, spread, ranked standings). The HEAL-3/LRN-3 assembler pattern.
- **UI, controller, and regressions deferred** — the dashboard UI is frontend; the controller/endpoint needs a `dashboard → eval` dependency (FI-PE3-A); regression detection vs the AI-3 committed baseline needs the baseline source (FI-PE3-B); per-execution history over `PromptExecutionEntity` is FI-PE3-C. **PE-3 stays In Progress.**

**Consequences.**
- *Positive:* prompt-quality standings + score stats are now a programmatic read-model, unit-proven (4 tests); additive no-arg `@Component` assembler, no blast radius; completes the eval spine's read side.
- *Negative / trade-off:* no UI/endpoint, and "regressions" (named in the roadmap why) waits on the AI-3 baseline (FI-PE3-B) — PE-3 is not Completed.
- *Imposed rule:* the prompt-quality dashboard consumes a composed read-model over the leaderboard, not raw entries.

**Related:** PE-3; PE-2/ADR-027 (`LeaderboardEntry`/`PromptLeaderboard` consumed), PE-1 (`PromptScore`), AI-3 (regression baseline — deferred source), HEAL-3/LRN-3 (assembler pattern), DX-4 (prompt playground — complements), OBS-3; FI-PE3-A/B/C.

---

## ADR-052 — RBAC admin read-model over the security entities (ENT-4 backend increment)

**Status:** Accepted (implemented — ENT-4 read-model, 2026-07-29) · ENT-4 remains In Progress

**Context.**
The RBAC entities (users, roles, permissions, sessions) exist in `ai-qa-os-security` but had no admin surface — administration would be raw SQL. ENT-4 activates them. The dashboard module already depends on `security`, so it can read them directly; in a no-frontend environment the validatable substance is the read-model, not the UI.

**Decision.**
- **Read-model in `dashboard` (§0.4-A)** — a pure `RbacAdminAssembler.summarize(users, roles, permissions) → RbacAdminSummary` (user/role/permission counts, security posture — disabled/locked/MFA-enabled — and safe per-user `AdminUserView`s). **Never exposes secrets** (no password hash / MFA secret). The GOV-1/HEAL-3 assembler pattern.
- **Write ops + UI deferred** — mutating admin operations (create/disable user, assign role) are the highest-risk part and belong with the UI + careful authz (FI-ENT4-A); the controller/endpoint (FI-ENT4-B) and user↔role mapping (no direct join accessor on `UserEntity` — FI-ENT4-C) are deferred. **ENT-4 stays In Progress.**

**Consequences.**
- *Positive:* the RBAC state (identity + security posture) is now a programmatic, secret-free read-model, unit-proven (3 tests); additive no-arg `@Component` (dashboard context green); no new persistence — exposes what exists.
- *Negative / trade-off:* read-only — no user/role administration or UI yet (FI-ENT4-A/B); user↔role mapping absent (FI-ENT4-C) — ENT-4 is not Completed.
- *Imposed rule:* the admin surface exposes RBAC state via a secret-free read-model; mutating auth data is a deliberate, authz-guarded follow-up.

**Related:** ENT-4; `ai-qa-os-security` RBAC entities (`UserEntity`/`RoleEntity`/`PermissionEntity`), GOV-1/HEAL-3 (assembler pattern), SEC-1 (auth), ENT-1 (tenancy — users carry `tenantId`); FI-ENT4-A/B/C.

---

## ADR-053 — Local infrastructure stack: a `compose` Spring profile over provisioned containers, with real service bindings deferred to their consumers

**Status:** Accepted (implemented — Phase 1, Checkpoints 1–4, 2026-07-30)

**Context.**
A large block of remaining work is *infrastructure-gated* — ENT-1 enforcement (FI-ENT1-C/D/E), SCALE-1/SCALE-2, ENT-5, and the dashboard write paths cannot be validated without a real database, broker, and object store. The `deployment/docker` stack had only Postgres/Redis/Qdrant/otel, carried a DB-name mismatch, and there was no local Spring profile pointing the apps at the containers. The naïve reading of the Phase 1 plan was "add Kafka + MinIO + health indicators for all of it." Grounding during implementation showed that most of the target Java code is **stub or absent**, which would make those health indicators theatre.

**Decision.**
- **Provision the full local stack** in `deployment/docker/docker-compose.yml`: Postgres (DB reconciled to `ai_qa_os_dashboard`), Redis, Qdrant, **Kafka** (`apache/kafka` KRaft — no Zookeeper; Redpanda a drop-in alt), **MinIO** + a one-shot bucket initializer (`aiqaos-artifacts`/`aiqaos-backups`), healthchecks on each; the containerised `gateway` sits behind `--profile apps`, inspection UIs behind `--profile tools`. Dev-default creds are overridable via `.env` (SEC-2).
- **A `compose` Spring profile lives in each app's own resources** (`ai-qa-os-gateway`, `ai-qa-os-dashboard`), *not* in `ai-qa-os-config` — neither app depends on the config module and there is no `spring.config.import`, so config-module YAML would never load. Gateway is the sole Flyway migration owner (ADR-024); dashboard runs with Flyway off.
- **Wire only the real binding — Redis short-term memory.** `RedisMemoryStore` + `spring-boot-starter-data-redis` already exist on the gateway classpath (via `brain → memory`), so the profile sets `aiqaos.memory.shortterm.provider=redis` and Spring Boot's auto `RedisHealthIndicator` surfaces it — no custom code.
- **Defer the Qdrant/Kafka/MinIO Java bindings to their consuming features.** `QdrantStoreClient` is a no-op stub (vector provider stays `in-memory`); there is zero Kafka usage anywhere (no producer to health-check); the `ObjectStorageClient` seam is real but its binding is unvalidatable in this environment and the interface javadoc already defers it — the MinIO client is deferred to **ENT-5** activation. Provisioning the *containers* now makes each later binding a thin, validatable addition.
- **`verify-infra.sh` (T10)** — a one-shot health verifier for the whole stack (required: postgres/redis/kafka/minio/qdrant; informational: Flyway version, MinIO buckets).

**Consequences.**
- *Positive:* every infra-gated item can now be validated against real infrastructure with one `docker compose up`; the DB-name mismatch is fixed; Flyway single-ownership is enforced by profile; the stack is honest — no scaffolding stands in for services nothing consumes.
- *Negative / trade-off:* Qdrant/Kafka/MinIO containers sit idle until their features land; runtime infra verification is user-run (the build sandbox has no Docker), so live health is confirmed outside CI for now.
- *Imposed rule:* **a real service binding (Qdrant client, Kafka producer, MinIO client) is added only when a consuming feature exists.** Containers may be provisioned ahead of bindings; bindings must not be built ahead of consumers. This mirrors the seam-discipline rule (ADR-010/ADR-015): a contract/binding is justified by a real implementer, not by anticipated need.

**Related:** Phase 1 Infrastructure Technical Design; ADR-024 (single migration owner + per-service pools); ADR-003 (Memory as a tiered store); SCALE-3 (Qdrant vector binding — deferred consumer), SCALE-2 (events/Kafka — deferred consumer), ENT-5 (backups/MinIO — deferred consumer); ENT-1 (tenancy enforcement — unblocked by real DB); SEC-2 (env-injected creds); ADR-010/ADR-015 (seam discipline).

---

## ADR-054 — Row-level persistence tenancy via Hibernate `@TenantId` discriminator (FI-ENT1-C pilot)

**Status:** Accepted (implemented — FI-ENT1-C pilot, 2026-07-30) · **ENT-1 remains In Progress**

**Context.**
The ENT-1 foundation (ADR-041) and the gateway filter (ADR-043) bind a `TenantContext` per request, but **nothing isolates data** — 50 `@Entity` classes share one Postgres schema with no tenant boundary. Phase 1 (ADR-053) provisioned real Postgres so isolation can finally be validated. A data-isolation boundary must be **secure-by-default**: a developer forgetting to add a tenant clause must not leak cross-tenant rows. Spring Boot 3.3 → Hibernate 6.5 offers native discriminator multi-tenancy.

**Decision.**
- **Mechanism:** Hibernate native **discriminator multi-tenancy** (§0.4-A of the FI-ENT1-C design). A `@TenantId String tenantId` column on each tenant-owned entity; a `CurrentTenantIdentifierResolver<String>` (`TenantIdentifierResolver`) bridging `TenantContextHolder` — **unbound ⇒ `SYSTEM_TENANT`** (never null, never an arbitrary tenant) — registered with Hibernate by a `HibernatePropertiesCustomizer` (`TenantHibernateCustomizer`). Hibernate then **auto-appends `WHERE tenant_id = ?` on read and stamps `tenant_id` on write** — application code never sets or queries it.
- **Pilot aggregate (this slice, 7 entities):** `ModuleEntity`, `TestCaseEntity` (core); `ExecutionEntity`/`ExecutionStepEntity`/`ExecutionArtifactEntity` (execution); `WorkflowExecutionEntity`, `HumanReviewEntity` (orchestration) — each `implements Tenanted` + `@TenantId`. Migration **V17** (gateway-owned, ADR-024) adds `tenant_id VARCHAR(64) NOT NULL DEFAULT '__system__'` + an index per table, **backfilling legacy rows to the system tenant** so current no-`X-Tenant-ID` flows keep working.
- **Type:** String discriminator, matching `TenantContext` (`SYSTEM_TENANT = "__system__"`). `UserEntity`'s pre-existing `UUID tenant_id` is a separate RBAC dimension — reconciled in **FI-ENT1-D**, not here.

**Consequences.**
- *Positive:* isolation is enforced by the ORM, not by developer memory — the property that matters for a security boundary; least code; the pilot proves the pattern end-to-end; resolver unit-proven (5 tests); full reactor green (22 modules).
- *Negative / trade-off:* only the 7 pilot entities are isolated (reporting/intelligence/brain/eval/agents follow in later slices, same mechanism); **native SQL bypasses the discriminator** — tenant-owned native queries must scope `tenant_id` by hand (rule below); **`@DataJpaTest` slices** don't scan `@Component`s, so they must `@Import(TenantHibernateCustomizer.class)` (done for `WorkflowExecutionRepositoryTest`); **true cross-tenant isolation is user-run** (needs real Postgres with two tenants' data). ENT-1 stays In Progress.
- *Imposed rule:* a tenant-owned entity **declares `@TenantId` + implements `Tenanted`**; the discriminator is a **String** matching `TenantContext`; **native queries on tenant-owned tables must scope `tenant_id` explicitly** (Hibernate cannot rewrite them); slice tests that touch a tenant-owned entity import the tenancy customizer.

**Related:** ENT-1; FI-ENT1-C; ADR-041 (tenant-context contract), ADR-043 (gateway tenant filter), ADR-053 (infra — real DB), ADR-024 (Flyway single owner); FI-ENT1-D (tenant-scoped RBAC — reconciles `UserEntity`'s UUID), FI-ENT1-E (tenant-scoped memory/cost/artifact).

---

## ADR-055 — Tenant-scoped RBAC: users tenant-owned, roles a global catalog, JWT-authoritative tenant (FI-ENT1-D)

**Status:** Accepted (implemented — FI-ENT1-D, 2026-07-30) · **ENT-1 remains In Progress**

**Context.**
ADR-054 isolated the test-management aggregate, but the identity layer lagged: `UserEntity.tenantId` was a **`UUID`** (out of sync with the String `TenantContext`/`@TenantId` contract), `username`/`email` were **globally unique**, and **nothing made authentication tenant-aware** — the request tenant came from the **spoofable `X-Tenant-ID` header** (ADR-043), and the user was loaded globally by id. A token minted for tenant A could act against any tenant. A data-isolation boundary must derive the authenticated tenant from something **trusted**.

**Decision.**
- **Users are tenant-owned; roles/permissions are a global catalog (§0.4-A).** `UserEntity.tenantId` **UUID → String**, `@TenantId` + `implements Tenanted` (reuses the ADR-054 resolver — no new wiring); `username`/`email` unique **per tenant** (`ux_users_tenant_username`/`_email`). `RoleEntity`/`PermissionEntity`/`RolePermissionEntity` stay global (platform-defined roles assigned to tenant users). Migration **V18** reconciles the column type (UUID→VARCHAR, NULL→`__system__`) and swaps global-unique → per-tenant-unique.
- **The JWT is the authoritative tenant for authenticated requests.** `JwtAuthenticationFilter` binds the `TenantContext` from the **signed token's `tenantId` claim** *before* loading the user (so `findById` is `@TenantId`-filtered) and for the whole downstream leg, restoring the previous context after. The gateway `TenantContextFilter` **yields to an already-bound tenant** — so tenant resolution is correct regardless of filter ordering, and the untrusted header is ignored for authenticated calls. Login (unauthenticated) still resolves the tenant from `X-Tenant-ID` (the login request names its tenant); the minted token carries it.

**Consequences.**
- *Positive:* a token for tenant A **cannot resolve a user in tenant B** (→ 401) — proven by unit test; identity is tenant-isolated; the authenticated tenant is trusted (signed token), not spoofable; the tenant id type is reconciled to the `core` String contract. Full reactor green (22 modules).
- *Negative / trade-off:* user **satellites** (`UserSessionEntity`, `ApiKeyEntity`, `PasswordHistoryEntity`) and `SecurityAuditEntity` attribution are **tenant-scoped / attributed as of FI-ENT1-D slice 2 (2026-07-30)** via `@TenantId` and `V21` migration; **no per-tenant custom roles** (global catalog); the ENT-4 admin read-model returns only the **current tenant's** users (correct for tenancy); **true multi-tenant login E2E is user-run** (needs real Postgres with users in two tenants). ENT-1 stays In Progress.
- *Imposed rule:* **an authenticated request derives its tenant from the signed JWT, never from the `X-Tenant-ID` header**; identity (`username`/`email`) is unique **per tenant**; the header is authoritative only on the unauthenticated login path; any filter binding tenant yields to an already-bound (trusted) tenant.

**Related:** ENT-1; FI-ENT1-D; ADR-041 (tenant-context contract), ADR-043 (gateway tenant filter — now yields to the token tenant), ADR-054 (persistence tenancy mechanism, reused), SEC-1 (auth), SEC-2 (secrets); FI-ENT1-E (tenant-scoped memory/cost/artifact), FI-ENT1-D slice 2 (user satellites + audit attribution — completed 2026-07-30).

---

## ADR-056 — Tenant-scoped memory, cost & artifact: `@TenantId` on the durable rows + tenant key-prefix for blobs (FI-ENT1-E)

**Status:** Accepted (implemented — FI-ENT1-E, 2026-07-30) · **ENT-1 remains In Progress**

**Context.**
FI-ENT1-C isolated the test-management aggregate and FI-ENT1-D the user identity. The last enforcement dimension the roadmap names for ENT-1 is **memory retrieval + cost isolation + artifact scoping**. Today: `MemoryNodeEntity`/`ConversationHistoryEntity`/`LLMCostEntity` persist with **no tenant column**; `MemoryMetadata` carried an unused `UUID tenantId`; and while FI-ENT1-C scoped the artifact JPA *row*, the **blob bytes remained cross-tenant addressable** — `ObjectStorageArtifactStore` keyed on a static prefix only.

**Decision.**
- **Persistence (memory + cost) — reuse ADR-054.** `@TenantId` + `implements Tenanted` on `MemoryNodeEntity`, `ConversationHistoryEntity`, `LLMCostEntity`; migration **V19** adds `tenant_id VARCHAR(64) NOT NULL DEFAULT '__system__'` + index on the three tables (backfilling legacy rows to system). Hibernate auto-stamps/filters — no query changes. `MemoryMetadata.tenantId` reconciled **UUID → String** (contract consistency; zero external callers).
- **Artifact blobs — tenant key-prefix (§0.4-A).** `ObjectStorageArtifactStore.fullKey = staticPrefix + <tenant> + "/" + key`, tenant from `TenantContextHolder` (unbound ⇒ `__system__`). One shared bucket, isolation by key namespace — storage-agnostic (in-memory now, MinIO later). `store/resolve/exists/delete/list` all route through it, so a tenant cannot read another's bytes by key.
- **Test wiring:** adding `@TenantId` to `LLMCostEntity` puts a discriminator entity into the **observability** persistence unit, so its `@DataJpaTest`s need the resolver — supplied once via `@Import(TenantHibernateCustomizer.class)` on that module's `TestApplication` (covers all its slice tests).

**Consequences.**
- *Negative / trade-off:* vector-store **search** filtering (in-memory client filtered as of FI-ENT1-E slice 2; real Qdrant client is a stub — ADR-053), short-term **Caffeine/Redis** key namespacing (`<tenant>:<key>`), and `LocalArtifactStore` path scoping (`<baseDir>/<tenant>/<key>`) are **completed in FI-ENT1-E slice 2 (2026-07-30)**; legacy object-store blobs are addressed under `__system__/`; **cross-tenant invisibility over a real DB is user-run**. ENT-1 non-stub scoping is complete.
- *Imposed rule:* durable tenant-owned rows declare `@TenantId` + `Tenanted`; shared object-storage keys and local filesystem artifact paths are tenant-namespaced; **adding `@TenantId` to an entity obliges that module's persistence-unit tests to supply the resolver** (module `TestApplication` `@Import`, or per-test `@Import`).

**Related:** ENT-1; FI-ENT1-E; ADR-054 (persistence tenancy mechanism, reused), ADR-055 (RBAC), ADR-053 (deferred vector/object-store bindings), ADR-024 (Flyway single owner), ENT-5 (object-storage artifact store); FI-ENT1-E slice 2 (vector-search / short-term / Local-artifact scoping — completed 2026-07-30).

---

## ADR-057 — Tenant-scope the remaining operational tables; catalogs global, telemetry system-scoped (FI-ENT1-C extension)

**Status:** Accepted (implemented — FI-ENT1-C extension, 2026-07-30) · **ENT-1 remains In Progress**

**Context.**
FI-ENT1-C/D/E tenant-isolated the test-management aggregate, user identity, and memory/cost/artifact — 11 entities. **~33 entities remained unclassified.** Extending `@TenantId` is mechanical (ADR-054), so the only real decision is **which of the remaining entities are tenant-owned** vs platform-global vs system telemetry — a boundary that must be set deliberately, since over-scoping (e.g. a discriminator on observability) would hide cross-tenant rows from platform operators.

**Decision — conservative boundary (§0.4-A).**
- **Tenant-owned → `@TenantId` (16, this slice):** the data a tenant *produces* — `ExecutionAuditEntity`; `WorkflowEntity`/`WorkflowStepEntity`; `ReportEntity`/`ReportArtifactEntity`/`FailureAnalysisEntity`/`TrendEntity`; `PromptExecutionEntity`; `AgentExecutionEntity`; `AgentMessageEntity`/`AgentRuntimeEntity`/`AgentTaskEntity`; `DecisionEntity`/`ReasoningTraceEntity`/`LearningEntity`; `EvaluationResultEntity`. Migration **V20** adds `tenant_id` + index (backfill `__system__`).
- **Platform-global catalog → stay global:** prompt catalog (`PromptTemplateEntity`/`PromptVersionEntity`/`VersionPinEntity`), agent roster (`AgentEntity`), RBAC catalog (`RoleEntity`/`PermissionEntity`/`RolePermissionEntity`, per ADR-055).
- **System telemetry / audit → stay system-scoped:** all observability entities except `LLMCostEntity` (billing, scoped in FI-ENT1-E), and `SecurityAuditEntity` — operator visibility must span tenants.

**Consequences.**
- *Positive:* nearly all durable tenant-owned data is now ORM-isolated; platform operators keep cross-tenant telemetry/audit; platform catalogs stay a single source of truth; a pure mechanical reuse of ADR-054 (full reactor green, 22 modules).
- *Negative / trade-off:* RBAC **satellites** (`UserSessionEntity`/`ApiKeyEntity`/`PasswordHistoryEntity`) remain for FI-ENT1-D slice 2; the classification of a few borderline entities (`LearningEntity`, `EvaluationResultEntity`, `ExecutionAuditEntity`) is a judgment call revisitable later; cross-tenant invisibility over a real DB is user-run.
- *Imposed rule:* **a tenant owns the data it produces; the platform owns definitions (catalogs) and operator telemetry/audit.** New tenant-owned entities take `@TenantId`; new catalog/telemetry entities stay global/system. Boundary = conservative (Option A).

**Related:** ENT-1; FI-ENT1-C extension; ADR-054 (mechanism, reused), ADR-055 (roles-global precedent), ADR-056 (memory/cost/artifact), ADR-024 (Flyway owner); FI-ENT1-D slice 2 (RBAC satellites).

---

## ADR-058 — Credential entities (session, API key) are tenant-attributed, not `@TenantId`-discriminated (FI-ENT1-D slice 2)

**Status:** Accepted (implemented — FI-ENT1-D slice 2, 2026-07-30) · **ENT-1 remains In Progress**

**Context.**
The RBAC satellites (`UserSessionEntity`, `ApiKeyEntity`, `PasswordHistoryEntity`) were given `@TenantId` (+ V21 columns). But **sessions and API keys are credentials looked up *before* a tenant is bound** — a token refresh with an expired access token binds no tenant, and an API key *is* what establishes the tenant. `@TenantId`'s auto-filter would scope those lookups by the current/`__system__` tenant and **fail to find the record**, silently breaking refresh and API-key auth. `PasswordHistoryEntity` has no such issue — it is only read inside authenticated, tenant-bound requests.

**Decision.**
- **`UserSessionEntity` + `ApiKeyEntity` → attribution-only:** keep the `tenant_id` column + `Tenanted` marker, **drop `@TenantId`**. Look them up **tenant-agnostically by their unguessable secret** (refresh token / key hash), then **bind the tenant from the resolved record** — mirrors ADR-055's token-authoritative principle. Login now sets `session.tenantId` explicitly (Hibernate no longer stamps it); `AuthenticationService.refresh` binds the session's tenant before the `@TenantId`-filtered user load.
- **`PasswordHistoryEntity` keeps `@TenantId`** (tenant-bound access only).
- **`SecurityAuditEntity`** stays a plain `tenant_id` attribution column (system-scoped, operator cross-tenant visibility — ADR-057); `AuditLogger` persistence remains a deferred stub.

**Consequences.**
- *Positive:* refresh/logout work with no pre-bound tenant; the unguessable secret is the isolation, with `tenant_id` as attribution + defence-in-depth; consistent with ADR-055. Refresh-binds-tenant unit-proven; full reactor green (22 modules).
- *Negative / trade-off:* credential entities rely on secret unguessability + an explicit attribution set at creation (login must set the session tenant); real API-key auth is still stubbed (`ApiKeyProvider`) — the rule applies when it is built.
- *Imposed rule:* **an entity looked up as a credential *before* tenant binding is tenant-attributed (`tenant_id` column), never `@TenantId`-discriminated; the auth flow binds the tenant from the resolved credential.** `@TenantId` is only for entities accessed inside an already-tenant-bound context.

**Related:** ENT-1; FI-ENT1-D slice 2; ADR-055 (credential/token resolves tenant), ADR-054 (`@TenantId` mechanism), ADR-057 (audit system-scoped), ADR-024 (Flyway owner — V21); `AuthenticationService` refresh/login.

---

## ADR-059 — Runtime tenant-scoping of short-term memory, vector search, and local artifacts (FI-ENT1-E slice 2)

**Status:** Accepted (implemented — FI-ENT1-E slice 2, 2026-07-30) · **completes ENT-1 enforcement**

**Context.**
ADR-056 (FI-ENT1-E slice 1) tenant-scoped the *durable* surfaces — memory/cost rows (`@TenantId`) and object-store blob keys. The remaining ENT-1 surfaces are **non-JPA runtime state**, which `@TenantId` cannot cover: short-term caches (`CaffeineMemoryStore`/`RedisMemoryStore` — key/value, not entities), the in-memory vector index (`InMemoryVectorStoreClient`), and dev-disk artifacts (`LocalArtifactStore`). Each must be explicitly tenant-namespaced or tenant-filtered.

**Decision.**
- **Short-term memory:** `CaffeineMemoryStore` + `RedisMemoryStore` namespace every cache key as `<tenant>:<key>`, tenant from `TenantContextHolder` (unbound ⇒ `__system__`).
- **Vector search:** `InMemoryVectorStoreClient.search` filters matches by the current tenant (falling back to `filters.getTenantId()`), so a query for tenant A never returns tenant B's embeddings.
- **Local artifacts:** `LocalArtifactStore` namespaces disk paths under `<baseDir>/<tenant>/<key>` — the on-disk mirror of `ObjectStorageArtifactStore`'s key-prefix (ADR-056).

**Consequences.**
- *Positive:* the non-JPA runtime surfaces (short-term memory, in-memory vector index, local files) are now tenant-isolated, so **ENT-1 enforcement is complete across every surface** — durable and in-flight; storage-agnostic and reused from `TenantContextHolder`.
- *Negative / trade-off:* the **real Qdrant client remains a stub** (ADR-053) — this filter applies to the in-memory dev store; when Qdrant lands, its `search` must apply the same tenant filter **server-side**. Legacy local artifacts are re-addressed under `<tenant>/`.
- *Imposed rule:* **non-JPA tenant-owned runtime state (caches, in-memory indexes, local files) is explicitly tenant-namespaced/filtered from `TenantContextHolder`**; any real vector/cache binding must carry the same scoping.

**Related:** ENT-1; FI-ENT1-E; ADR-056 (durable persistence + object-store key-prefix), ADR-053 (deferred Qdrant binding), ADR-054 (`@TenantId` mechanism).

---

## ADR-060 — In-process `EventBus` coordination seam over `core` `BaseEvent`; distributed binding deferred (SCALE-2)

**Status:** Accepted (implemented — SCALE-2 seam, 2026-07-30) · **SCALE-2 In Progress**

**Context.**
`core.event` already held a `BaseEvent` hierarchy (9 typed events), but there was **no unifying coordination seam** — event publishing was fragmented across a Spring-`ApplicationEvent`-based `PlatformEventBus` (in the high-level `integration` module) and per-module publishers (gateway/observability/orchestration/integration). SCALE-2 needs a *swappable* coordination bus: in-process now, distributed (Kafka) later — the same shape as SCALE-1's execution-queue seam.

**Decision.**
- **`core.event.EventBus`** — `publish(BaseEvent)` + `subscribe(Class<T>, Consumer<T>)`. A Spring-free contract in `core`, so any module already depending on `core` can publish/subscribe without an edge to a higher module.
- **`InProcessEventBus`** — synchronous default (`@Component`): a `ConcurrentHashMap<Class, handlers>` registry; `publish` walks the event's class hierarchy (a `Consumer<BaseEvent>` sees all, a `Consumer<WorkflowEvent>` sees only workflow events); a throwing handler is isolated (logged) so it can't stop the others. Deterministic, unit-tested, no infrastructure.
- **Distributed (Kafka) `EventBus` is deferred** — a drop-in of the same interface, built when a real cross-service consumer exists (container provisioned, ADR-053; no scaffolding without a consumer).
- **Consolidating** `PlatformEventBus` + the per-module publishers onto the seam is an additive follow-up (FI-SCALE2-A) — today's publishers keep working.

**Consequences.**
- *Positive:* one `core` coordination seam over the existing `BaseEvent`s; the Kafka binding is a clean swap; unit-proven (3 tests); full reactor green (22 modules). Mirrors SCALE-1 exactly.
- *Negative / trade-off:* synchronous, in-JVM only for now (no ordering/retry/async); the fragmented publishers are not yet migrated (working, so left additive); real distributed coordination waits on the deferred Kafka binding.
- *Imposed rule:* cross-module coordination flows through the `core` `EventBus` over `BaseEvent`; a distributed binding implements the same interface; no distributed scaffolding without a real consumer.

**Related:** SCALE-2; SCALE-1 (queue-seam pattern), ADR-053 (Kafka container ready, binding deferred), MOD-2/ENT-2 (event→notification routing), `core.event.BaseEvent`; FI-SCALE2-A (publisher consolidation).

---

## ADR-061 — Bridge the `core` `EventBus` seam to Spring events during publisher migration (FI-SCALE2-A)

**Status:** Accepted (implemented — FI-SCALE2-A, 2026-07-30) · **SCALE-2 In Progress**

**Context.**
SCALE-2 added the `core` `EventBus` seam (ADR-060), but the five existing publishers (`PlatformEventBus`, `Gateway`/`Workflow`/`Observability`/`Integration` publishers) are built on **Spring's `ApplicationEvent` model**, with working `@EventListener` consumers. A full one-shot migration to `BaseEvent`/`EventBus.subscribe` would touch five modules and risk those consumers.

**Decision — bridge, don't rip out (§fork-A).**
- **`PlatformEventBus` becomes the unified entry point:** `publish(BaseEvent)` routes to the canonical `core` `EventBus` seam; `publishEvent(ApplicationEvent)` stays for the legacy path.
- **`SpringEventBridge`** subscribes to the `core` `EventBus` (`BaseEvent`) and forwards each event to Spring's `ApplicationEventPublisher`, so **a `BaseEvent` published on the seam still reaches existing `@EventListener` consumers**.
- **`WorkflowEventPublisher` migrated as the representative proof** — `BaseEvent`s go through the seam (and still reach Spring consumers via the bridge); non-core events fall back to the Spring path. Converted `@Autowired` field → constructor injection.
- The remaining publishers migrate **opportunistically**; nothing is force-rewritten.

**Consequences.**
- *Positive:* publishers can adopt the seam without breaking consumers; the seam is the canonical path while Spring remains legacy-inbound; a unit test proves a seam event reaches both a core subscriber and a Spring consumer; full reactor green (22 modules).
- *Negative / trade-off:* two event models coexist during migration (transient); the bridge runs only where `integration` is on the classpath (not yet a direct app dependency — broader wiring is a follow-up) — the `core` seam itself works everywhere regardless.
- *Imposed rule:* new coordination publishes `BaseEvent`s via the seam; `SpringEventBridge` keeps legacy `@EventListener` consumers working; the raw `ApplicationEvent` path is legacy-only, retired as publishers migrate.

**Related:** SCALE-2; ADR-060 (the `EventBus` seam); FI-SCALE2-A; `PlatformEventBus`/`SpringEventBridge`/`WorkflowEventPublisher`.

---

## ADR-062 — Serve dashboard read-models by aggregating persisted data; PE-3 from `eval_results`, LRN-3 deferred (no faithful source)

**Status:** Accepted (implemented — PE-3, 2026-07-30; LRN-3 deferred) · **PE-3 & LRN-3 remain In Progress**

**Context.**
ENT-4/HEAL-3 read-models are a simple `repo.findAll() → assembler`. LRN-3 and PE-3 are not: their assemblers consume **computed** inputs — `LearningMetrics` (from transient `LearningObservation`s fed by the live loop) and a leaderboard (from **running** benchmarks). Neither has a directly-queryable source. The chosen strategy (Option A) is to **aggregate data that is already persisted** into the assembler inputs.

**Decision.**
- **PE-3 — done.** `dashboard → eval` dependency added; `PromptQualityService` aggregates `EvaluationResultRepository` (`eval_results`): mean `score` per `promptVersion` → ranked `LeaderboardEntry`s → `PromptQualityAssembler`. `GET /api/dashboard/prompt-quality`. Read-only, **no benchmark re-run**.
- **LRN-3 — deferred (honest data gap).** Aggregating `brain_learning` is **not faithful**: `LearningEntity` carries only `pattern`/`previousDecision`/`result`/`improvement` strings — it lacks the **`success` (boolean)** and **`confidence` (double)** that `LearningObservation` requires. Deriving them would *fabricate* the confidence signal, producing a misleading dashboard. LRN-3 is therefore deferred to **Option B** — the learning loop persisting a faithful metrics/observation snapshot — not built now.

**Consequences.**
- *Positive:* PE-3 is served from real persisted data with no loop change; the aggregation is pure and unit-proven; the dashboard shows a real leaderboard. The LRN-3 gap is surfaced honestly instead of faked.
- *Negative / trade-off:* LRN-3 stays unexposed until its producer persists a snapshot (Option B); PE-3 aggregation faithfulness vs live eval output is confirmed only against a real DB (user-run); the dashboard now depends on `eval`.
- *Imposed rule:* **a dashboard read-model is served by aggregating persisted data; if the persisted data lacks the signal the read-model needs, expose it only after its producer persists a faithful snapshot — never fabricate the missing signal.**

**Related:** PE-3, LRN-3; ADR-052 (assembler pattern), ENT-4/HEAL-3 (repo-query read-models); `EvaluationResultEntity` (`eval_results`), `LearningEntity` (`brain_learning`).

---

## ADR-063 — LRN-3 dashboard deferred: its observation pipeline is unbuilt; no empty-dashboard scaffolding

**Status:** Accepted (decision, 2026-07-30) · **LRN-3 remains In Progress**

**Context.**
Designing LRN-3's data source (Option B, deferred by ADR-062) revealed the LRN-2 metrics pipeline is **entirely unwired**: `LearningMetricsCalculator.compute` has **no callers**, `LearningObservation` has **no producers**, and no persisted table carries a faithful **`success` + `confidence`** per run (`brain_learning` has neither numeric; `brain_decisions` has confidence but its `decision` string is not run-success). The read-model (`LearningDashboardAssembler`) and calculator are built and waiting. So "Option B" is really **build the observation pipeline** — a producer that captures success (run pass/fail) + confidence (AI-1 gate) at run completion, persists it, and a read path.

**Decision.**
**Defer LRN-3.** Do **not** build a persistence + recorder half-pipeline with **no producer** — it would show an empty dashboard forever (scaffolding without a producer, contrary to the seam-discipline rule). Keep LRN-3 In Progress; the read-model, assembler, and the **full Option B design** (`AI-QA-OS-LRN3-OptionB-Technical-Design.md`) are on record. Build the pipeline (entity + repo + `V22` + `LearningObservationRecorder` + the **orchestration run-completion hook** + read path + UI) when the learning loop is genuinely integrated or LRN-3 is prioritised.

**Consequences.**
- *Positive:* honest — no empty-dashboard scaffolding; the read-model/assembler + a concrete build plan are ready; upholds and extends ADR-062's rule.
- *Negative / trade-off:* LRN-3 is the one of four dashboards (vs ENT-4/HEAL-3/PE-3) left unshipped — because it is the only one with no faithful persisted data source.
- *Imposed rule:* **a dashboard read-model ships only when a faithful producer/source exists; when it does not, defer with the design on record rather than scaffold an always-empty pipeline.**

**Related:** LRN-3; ADR-062 (PE-3 shipped, LRN-3 aggregation deferred), ADR-052 (assembler pattern); LRN-2 (`LearningMetricsCalculator`/`LearningObservation`); LRN-3 Option B design doc.

---

## ADR-064 — Distributed `EventBus` over Kafka: `KafkaEventBus` behind an optional dependency (SCALE-2 Kafka binding)

**Status:** Accepted (implemented — SCALE-2 Kafka binding, 2026-07-31) · **SCALE-2 In Progress** (live cross-instance E2E user-run)

**Context.**
SCALE-2's `EventBus` seam (ADR-060) + Spring bridge (ADR-061) run in-process. The **distributed** binding was deferred (ADR-053) until needed. Now: make the *same* bus work **cross-instance** over the provisioned Kafka — without forcing Kafka weight on modules that don't use it, and keeping in-process the zero-config default.

**Decision (Option A).**
- **`KafkaEventBus` in `core.event`**, beside `InProcessEventBus`; **`spring-kafka` is an *optional* dependency of `core`** (not transitive). The bean is **`@ConditionalOnClass(KafkaTemplate)` + `@ConditionalOnProperty(aiqaos.events.transport=kafka)`** — evaluated via bytecode, so the class never loads when Kafka is absent or unselected.
- **`InProcessEventBus` guarded** `@ConditionalOnProperty(...=in-process, matchIfMissing=true)` → the default, so exactly one `EventBus` bean exists.
- **Shared `EventDispatch`** (extracted) does the local type-hierarchy dispatch for both buses.
- **Wire format:** an `EventEnvelope {type (FQN), payload}` JSON so the consumer reconstructs the exact `BaseEvent` subtype (non-invasive — no polymorphic type info on `BaseEvent`). **Per-instance consumer group** → broadcast (every instance receives every event).
- **Gateway opts in:** `spring-kafka` dependency + `compose`-profile `aiqaos.events.transport` (env-flippable, **default in-process** so the common dev flow needs no Kafka).

**Consequences.**
- *Positive:* the same seam works cross-JVM when enabled; **zero Kafka weight by default** (optional dep + conditional beans); serialize/receive/dispatch + type reconstruction unit-proven; full reactor green with the default unchanged.
- *Negative / trade-off:* **live cross-instance delivery is user-run** (needs Kafka up + two instances); single topic with a per-type key (per-type topics a later optimisation); delivery/ordering guarantees are Kafka's defaults (no extra retry/idempotency layer yet).
- *Imposed rule:* a distributed binding lives behind an **optional dependency + conditional beans**; the seam is the abstraction and the transport is **config-selected**, defaulting to in-process.

**Related:** SCALE-2; ADR-060 (the `EventBus` seam), ADR-061 (Spring bridge), ADR-053 (Kafka container provisioned); FI-SCALE2-A (remaining publisher migration, independent).

---

## ADR-065 — Distributed `ExecutionJobQueue` over Redis Streams: `RedisStreamExecutionJobQueue` behind an optional dependency (SCALE-1)

**Status:** Accepted (implemented — SCALE-1 distributed queue, 2026-07-31) · **SCALE-1 In Progress** (live cross-instance E2E user-run)

**Context.**
SCALE-1's `ExecutionJobQueue` seam + `InProcessExecutionJobQueue` (single-JVM worker pool) exist; the remaining piece is the **distributed** binding so jobs run on a worker pool **spanning nodes**. Crucially, a job queue is the **opposite of the SCALE-2 event bus**: it needs **competing consumers** (exactly one worker per job) and the result must return to the *submitter* — so it cannot reuse the broadcast `EventBus`; it needs a real work queue. Redis is provisioned; the seam's javadoc names "Redis-Streams" as the target.

**Decision (Option A — Redis Streams).**
- **`RedisStreamExecutionJobQueue`** (`execution.queue`): `submit` → JSON job (`@JsonCreator` on the immutable `ExecutionJob`/`ExecutionJobResult`) → `XADD execution:jobs`; a per-instance worker consumes via the shared consumer group `execution-workers` (competing consumers), runs it through the shared **`ExecutionJobRunner`**, `SET`s `execution:result:<jobId>` (TTL), and `XACK`s; `awaitResult` polls the result key. **At-least-once** — unacked messages stay pending; each worker recovers its **own** pending on startup (crash-restart). Cross-instance `XAUTOCLAIM` reclaim of a permanently-dead worker is a **documented follow-up**.
- **Optional dependency + conditional selection** (the ADR-064 pattern): `spring-data-redis` `<optional>` in `execution`; `@ConditionalOnClass(StringRedisTemplate)` + `@ConditionalOnExpression(enabled AND provider=redis)`. `InProcessExecutionJobQueue` guarded `@ConditionalOnExpression(enabled AND provider!=redis)` = the default. Queueing is **off entirely by default** → `ExecutionStep` runs inline (non-breaking).
- **Gateway opts in** via env-flippable `AIQAOS_EXECUTION_QUEUE_ENABLED`/`_PROVIDER` (Redis already on its classpath via `brain→memory`).

**Consequences.**
- *Positive:* execution decoupled across instances when enabled; **zero Redis weight by default**; the shared runner + JSON round-trip are unit-proven; full reactor green with the default path unchanged. `ExecutionJob`/`Result` are now cleanly serialisable (`@JsonCreator`).
- *Negative / trade-off:* **live cross-instance distribution is user-run** (needs Redis + ≥2 instances — and a full job also needs Playwright, so doubly environment-bound); cross-instance `XAUTOCLAIM` reclaim, per-type partitioning, and backpressure are follow-ups.
- *Imposed rule:* a distributed **job queue** uses competing consumers over Redis Streams with `XACK` durability (not the broadcast event bus); the seam is the abstraction, the provider is config-selected, default in-process.

**Related:** SCALE-1; `ExecutionJobQueue`/`InProcessExecutionJobQueue` seam, ADR-064 (optional-dep + conditional pattern), ADR-053 (Redis provisioned), SCALE-2 (contrast: broadcast vs competing-consumers).

---

## ADR-066 — User↔role mapping via `@ElementCollection` + role-derived authorities (FI-ENT4-C)

**Status:** Accepted (implemented — FI-ENT4-C, 2026-07-31) · **ENT-4 remains In Progress**

**Context.**
Designing ENT-4's admin **write-ops** (FI-ENT4-A) surfaced that they **can't be built safely**: `UserEntity` had **no roles**, `JwtAuthenticationFilter` granted a hardcoded `ROLE_USER` (a "no user→role model yet" note), and there was **no user↔role mapping**. So a write endpoint would be an ungated hole or gated on an authority nobody holds, and the UI's client-chosen role was unvalidated server-side (a known tech-debt item). The missing foundation is FI-ENT4-C.

**Decision (Option A).**
- **`UserEntity.roles`** — an `@ElementCollection<String>` (`security_user_roles(user_id, role_name)`, EAGER), referencing the global role catalog **by name** (ADR-055); mirrors the existing `backupCodes` collection. Migration **V22** (gateway-owned, ADR-024) with an FK to `security_users(id)` `ON DELETE CASCADE`.
- **`AuthorityMapper`** — derives `GrantedAuthority`s: baseline `ROLE_USER` + `ROLE_<NAME>` per role (case-normalised, `ROLE_`-prefix-aware, deduped). `JwtAuthenticationFilter` uses it (the user is already loaded per request), so an ADMIN-role user carries `ROLE_ADMIN`.
- **Bootstrap admin** seeded `roles=["ADMIN"]` so there is a real admin.

**Consequences.**
- *Positive:* `hasRole('ADMIN')` / URL role rules now work; the UI `RoleGuard` role is validated server-side (closes the client-chosen-role tech-debt); **unblocks FI-ENT4-A write-ops**. `AuthorityMapper` unit-proven (4 tests); full reactor green; the FI-ENT1-D tenant filter test stays green.
- *Negative / trade-off:* role names aren't FK-enforced against `RoleEntity` (acceptable — small admin-curated catalog); per-permission (fine-grained) authorities from `RolePermissionEntity` are deferred; end-to-end `hasRole('ADMIN')` over a real DB with a seeded admin is user-run.
- *Imposed rule:* authenticated principals' authorities **derive from persisted roles** (`ROLE_<name>`), never a hardcoded baseline; roles are name-based against the global catalog.

**Related:** ENT-4; FI-ENT4-C; **FI-ENT4-A (now unblocked)**; ADR-055 (roles global catalog), ADR-052 (RBAC read-model), SEC-1 (auth), FI-ENT1-D (`JwtAuthenticationFilter` tenant binding — unchanged).

---

## ADR-067 — Admin write-ops API at `/api/admin/**` on the enforced chain, ADMIN-gated (FI-ENT4-A)

**Status:** Accepted (implemented — FI-ENT4-A, 2026-07-31) · **ENT-4 remains In Progress** (write-ops now landed; audit + unlock/reset still open)

**Context.**
FI-ENT4-C (ADR-066) gave a real `ROLE_ADMIN`, unblocking ENT-4's mutating half (create / disable / assign-roles). The obstacle is *where* it can be enforced: `DashboardSecurityConfig` (`@Order(1)`) `permitAll()`s `/api/dashboard/**` **with no JWT filter**, so any write placed under that prefix is unauthenticated. The secret-free read-model living there (FI-ENT4-B) is acceptable; user-management writes are not. Paths **outside** that matcher fall onto `SecurityConfig.enforcedFilterChain`, where the JWT filter runs and `@EnableMethodSecurity` (global) applies.

**Decision (Option A).**
- **`AdminUserController`** (`com.aiqaos.security.admin`) at **`/api/admin/users`** — `POST /` (create), `PATCH /{id}/enabled`, `PUT /{id}/roles`. Class-level **`@PreAuthorize("hasRole('ADMIN')")`**. Because `/api/admin/**` is deliberately **not** in the dashboard matcher, requests land on the enforced JWT chain on both apps; the method-security check then **fails closed** — an unauthenticated or non-admin caller is rejected regardless of which filter chain matched. **No change to the security chains.**
- **`AdminUserService`** — tenant is never client-supplied: `@TenantId` stamps new users with the caller's JWT-bound tenant and filters reads (ENT-1). BCrypt hashing (as bootstrap); role names validated against the **global catalog** (`RoleRepository`, case-insensitive → canonical) so an admin cannot invent authorities; business guards authz can't express — **no self-lockout** (disabling your own account) and **no self-demotion** (removing your own `ADMIN`) → 400.
- **Read-model `AdminUserView`** gains a secret-free **`id`** (so the UI can target `/{id}`) and **`roles`** (to render/prefill the editor).
- **React `AdminPage`** gains a create form + per-row enable/disable + role editor; `apiClient` already injects the bearer token, so writes carry auth automatically.

**Consequences.**
- *Positive:* ADMIN-gated, tenant-scoped user management; fails closed even if a chain is later misconfigured; zero change to the (security-critical) filter chains. Service guards unit-proven (`AdminUserServiceTest` 8/8); full reactor green (22 modules); UI type-checks/builds.
- *Negative / trade-off:* read stays open at `/api/dashboard/admin/rbac` while writes are at `/api/admin/users` — two base paths (acceptable; read is secret-free). Role names still un-FK'd at the DB (the write path validates against the catalog instead). Audit-log emission and unlock/password-reset are deferred (FI-ENT4-D/E). Live ADMIN-gated E2E over a real DB is user-run.
- *Imposed rule:* user-management **mutations require `ROLE_ADMIN` and live on the JWT-enforced chain**, never the permissive dashboard chain; an admin can neither self-lockout nor self-demote.

**Related:** ENT-4; FI-ENT4-A; FI-ENT4-C/ADR-066 (the unblocker); FI-ENT4-B/ADR-052 (read-model); ADR-055/058 (JWT-authoritative tenancy); SEC-1 (auth); `DashboardSecurityConfig` (the permissive chain this routes around).

---

## ADR-068 — Real object-storage binding over S3: `S3ObjectStorageClient` behind an optional dependency (ENT-5)

**Status:** Accepted (implemented — ENT-5 S3 binding, 2026-07-31) · **ENT-5 remains In Progress**

**Context.**
docker-compose provisions **MinIO** (`aiqaos-artifacts` bucket), but per ADR-053 the Java binding was deferred to its consumer — **ENT-5**. The `ObjectStorageClient` seam (sat on by `ObjectStorageArtifactStore`, tenant-key-prefixed per ADR-056) had only the **in-memory reference** client — no durable store, so SCALE-1's cross-host artifact reachability was unblocked in *code* but not in *fact*. ENT-5's deferred "real S3/GCS adapter" is this binding.

**Decision.**
Same shape as the SCALE-1 (Redis) and SCALE-2 (Kafka) bindings — **optional dependency + conditional bean, default unchanged**:
- **AWS SDK for Java v2** (`software.amazon.awssdk:s3` + `url-connection-client`, both `<optional>`; BOM imported in the parent for version consistency). The portable **S3 API** — one binding serves **MinIO today and real AWS S3 / any S3-compatible store (GCS interop, Ceph, R2) later** with config only (chosen over the MinIO-native SDK, which would pin the durable path to one vendor).
- **`S3ObjectStorageClient implements ObjectStorageClient`** maps the seam onto S3 (put/get/exists/list-paginated/delete/lastModified), preserving the contract (missing key → `NoSuchElementException`). **`S3StorageConfiguration`** builds the `S3Client` (static creds from `S3StorageProperties`, `endpointOverride` for MinIO, path-style, `UrlConnectionHttpClient`). Guard: `@ConditionalOnClass(S3Client)` + `@ConditionalOnExpression("store==object and provider==s3")`.
- **`InMemoryObjectStorageClient`** flipped from `@ConditionalOnMissingBean` to the **inverse expression** (`provider != s3`), so exactly one client bean is active regardless of component-scan order (the SCALE-1 lesson — `@ConditionalOnMissingBean` on a scanned `@Component` is order-fragile).
- **SEC-2:** access/secret keys are env/`.env`-injected (MinIO dev defaults), never committed. Gateway `compose` profile documents the opt-in; default `store=local` (single-host) is unchanged.

**Consequences.**
- *Positive:* durable, **portable** object storage; removes ENT-5's "no real store" gap and completes SCALE-1's artifact reachability at the storage layer; optional dep = zero AWS weight for modules that don't opt in. `S3ObjectStorageClientTest` 6/6 (fake `S3Client` proxy — seam↔S3 mapping incl. `NoSuchKey`→`NoSuchElement`); full reactor green (22 modules); default store unchanged.
- *Negative / trade-off:* the live MinIO/S3 round-trip is **user-run** (needs the container + creds). ENT-5's **execution-worker upload** on the hot path and **running backup CronJobs** are still deferred (FI-ENT5-A/B) — so ENT-5 stays In Progress.
- *Imposed rule:* object-storage bindings sit behind the `ObjectStorageClient` seam via the optional-dep + conditional pattern; the seam stays **S3-compatible-portable**, not vendor-pinned; exactly one client bean via mutually-exclusive expression guards.

**Related:** ENT-5; SCALE-1 (the `ArtifactStore`/`ObjectStorageArtifactStore` seam this makes durable); ADR-053 (MinIO provisioned, binding-deferred-to-consumer); ADR-064/ADR-065 (the same optional-dep + conditional binding pattern); ADR-056 (tenant key-prefixing, unchanged); SEC-2 (credentials from env/secret).

---

## ADR-069 — Prompt regression detection via temporal within-version score decline (FI-PE3-B)

**Status:** Accepted (implemented — FI-PE3-B, 2026-08-01) · **PE-3 remains In Progress** (FI-PE3-C per-execution history still open)

**Context.**
PE-3's read-model shows a prompt-version leaderboard (mean score per version) but **no regression signal** — a prompt engineer can't see which versions got *worse*. The existing CI-time `PromptRegressionHarness` compares a **fresh run** against a per-**case** file `Baseline` (`golden/<suite>.baseline.json`); it is not a read-model, and its per-case baseline is a **different granularity** than the dashboard's per-version means — joining them would fabricate a comparison (ADR-063). But the persisted `eval_results` carry **`score` + `createdTime`**, which supports a faithful temporal signal with no new data source.

**Decision (Option A — temporal within-version).**
- **`PromptRegressionAnalyzer`** (pure, `ai-qa-os-eval`): per version, take its scores **ordered by `createdTime`**, split at the midpoint into an **earlier** and a **recent** window, and flag it iff `recentMean < earlierMean − tolerance` (compared with a floating-point epsilon so a drop of *exactly* tolerance is not a regression). A version with fewer than `minSamples` results is **skipped — never flagged against a fabricated baseline** (ADR-063). Signals sorted worst-first. DTOs: `PromptRegressionSignal` (versionId, baseline/current/delta, sampleCount), `PromptRegressionReport` (tolerance, count, signals).
- **Dashboard:** `PromptQualityService.getRegressions()` groups `eval_results` by version, sorts each by `createdTime` (nulls last, so undated rows never masquerade as "recent"), delegates to the analyzer; tolerance/minSamples from config (`aiqaos.eval.regression.tolerance:0.05`, `min-samples:4`). New endpoint `GET /api/dashboard/prompt-quality/regressions`. React `PromptQualityPage` gains a regressions panel.
- **Rejected — champion-relative** (flag versions below the best by a margin): that measures *gap-to-best*, not *regression* — a brand-new weak version trips it while a genuinely-declining champion never does; it also duplicates the leaderboard spread.

**Consequences.**
- *Positive:* a genuine, faithful *regression* signal (temporal decline) with **no benchmark re-run and no new data source**; the existing `PromptQualitySummary`/`PromptQualityAssembler` and their tests are **untouched** (capability added alongside). `PromptRegressionAnalyzerTest` 7/7, `PromptQualityServiceTest` 3/3, full reactor green (22 modules); UI lint/build clean.
- *Negative / trade-off:* needs ≥ `minSamples` history per version to judge (else skipped — honest, not fabricated); a version whose results are all one batch splits by case-order rather than run-order; the live page over a real multi-run `eval_results` is user-run. FI-PE3-C (per-execution history) and regression **alerting** (an ENT-2 `NotificationEvent` hook, FI-PE3-D) are deferred.
- *Imposed rule:* on the dashboard, "regression" means **temporal within-version decline** derived from persisted `eval_results`; insufficient-sample versions are skipped, never flagged against a fabricated baseline (ADR-063).

**Related:** PE-3; FI-PE3-B; FI-PE3-A/ADR-062 (the leaderboard read-model this extends); ADR-063 (never fabricate a missing signal); `PromptRegressionHarness` (the CI-time, per-case regression gate — deliberately distinct); ENT-2 (future FI-PE3-D alerting path).

---

## ADR-070 — HEAL-3 locator-drift ranking (FI-HEAL3-B) deferred: no faithful, enumerable drift source

**Status:** Accepted (deferral, 2026-08-01) · **HEAL-3 remains In Progress** — this records why its remaining locator FIs are blocked, mirroring the LRN-3 honesty call (ADR-062/063).

**Context.** FI-HEAL3-B asks for a "most-drifting-locators" ranking. Investigating the data path found **no faithful source**:
1. The drift signal — how often a broken locator has re-drifted — lives only as `HealedLocatorRecord.reuseCount` inside **`HealingMemory`**, which stores records in the `memory` **`MemoryStore`**. That interface is `put`/`get`/`remove`/`clear` — **no enumeration/scan** — so the set of healed locators **cannot be listed** to rank them.
2. **`HealingMemory` has no production callers at all** — `remember(...)`/`recall(...)` are invoked nowhere; it is not injected into `LocatorHealingService`/`LocatorHealCoordinator` or the engine. So even the non-enumerable store holds **no data** — an unwired, producerless component (the LRN-2/LRN-3 situation, ADR-063).
3. The **enumerable** persisted path — `HealingMetricEntity` (observability, JPA repo, produced by `SelfHealingEngineImpl`) — records **execution-level** healing (failure category, strategy, retry, recovery status) and carries **no locator identity**, so it cannot be grouped by locator.

**Decision.** **Do not build FI-HEAL3-B now, and do not fabricate a source for it.** Ranking would require either enumerating a store that cannot be enumerated, or inventing locator identity the persisted metric does not carry — both violate ADR-063 (never fabricate a missing signal). Consistent with the LRN-3 B-defer: don't ship a producerless half-pipeline that shows an empty panel forever.

**What would unblock it (the real FI-HEAL3-A).** A **persisted, enumerable healed-locator store** (a JPA `HealedLocatorEntity`: brokenLocator, healedLocator, strategy, confidence, driftCount, tenantId, healedAt + repository) **fed by wiring `HealingMemory.remember(...)` into the locator-healing flow** (`LocatorHealingService`/`LocatorHealCoordinator`, which today don't call it). Once locator heals are actually produced and enumerably persisted, FI-HEAL3-B is a trivial read-model (group by locator, order by driftCount). The foundation is the wiring, not the ranking.

**Consequences.** HEAL-3's produced/enumerable read-model (the `HealingAnalyticsSummary` over `HealingMetricEntity`, already shipped) is unaffected and remains the faithful HEAL-3 surface. The locator-history half (FI-HEAL3-A/B) waits for the locator-healing producer to be wired — tracked, not silently dropped.

**Related:** HEAL-3; FI-HEAL3-A/B; HEAL-4 (`HealingMemory`, the unwired producer); ADR-063 (LRN-3 — same "no faithful/producer source" deferral); ADR-047 (the HEAL-3 analytics read-model that IS faithful).

---

## ADR-071 — Execution-worker artifact upload into `ArtifactStore` via a deterministic key (FI-ENT5-A)

**Status:** Accepted (implemented — FI-ENT5-A, 2026-08-01) · **ENT-5 remains In Progress** (backup CronJobs + dashboard resolve-from-store still open)

**Context.** ADR-068 gave a durable `ObjectStorageArtifactStore` (S3/MinIO), but **nothing pushed execution artifacts into it** — `ExecutionStep` (orchestration) persists `ExecutionArtifactEntity` rows holding **host-local file paths** (screenshot/video/trace/log/report), so a cross-host worker's artifacts stay unreachable elsewhere (the ENT-5/SCALE-1 gap). This slice connects that real, wired producer to the durable store. (Unlike the HEAL-3/PE-3-C read-models, this producer *is* wired — `ExecutionStep` genuinely writes these rows from `execResult.getArtifacts()`.)

**Decision (Option A — deterministic, reconstructable key; no schema change).**
- **`ArtifactUploader`** (`ai-qa-os-execution`, `@ConditionalOnProperty(aiqaos.artifacts.upload.enabled=true)`): reads each non-null, existing artifact file and `artifactStore.store(keyFor(...), bytes)`. **Best-effort** — a missing file or store failure is logged and skipped; artifact upload must never fail an execution. Injects `ArtifactStore` directly (exactly one bean: `LocalArtifactStore` default XOR `ObjectStorageArtifactStore`).
- **Key** = `executions/<executionId>/run-<runNumber>/<browser>/<testCaseId>/<type>` — a pure function of fields `ExecutionArtifactEntity` already carries, so the durable copy is addressable **without persisting the key** (reconstruct to resolve). `ArtifactUploader.keyFor(...)` is the single source of truth. **No new columns, no migration.**
- **Wiring:** `ExecutionStep` injects `ObjectProvider`-style `@Autowired(required=false) ArtifactUploader` (present only when enabled) and calls it immediately after `artifactRepo.save(art)`, inside a log-only try/catch. Opt-in (`aiqaos.artifacts.upload.enabled`, default **false**) → non-breaking; the tenant key-prefix (ADR-056) applies from the bound tenant automatically.
- **Rejected — explicit key columns + migration (V23):** records the key redundantly (it's derivable) for a wider entity + migration; Option A gets the link for free.

**Consequences.**
- *Positive:* execution artifacts become durable + **cross-host reachable** with the S3 binding — closes SCALE-1's artifact gap at the producer; zero schema change; default path unchanged. `ArtifactUploaderTest` 3/3 (fake store + `@TempDir`: keyed upload, null/missing skipped, throwing-store swallowed); full reactor green (22 modules).
- *Negative / trade-off:* the dashboard artifact-**serving** path still reads local files — resolving served artifacts from `ArtifactStore` (reconstruct `keyFor`) is a follow-on (FI-ENT5-C). Live browser execution → real bucket round-trip is user-run. Backup CronJobs (FI-ENT5-B) still open → ENT-5 stays In Progress.
- *Imposed rule:* durable artifact keys are **derived deterministically** from stable execution fields (`keyFor`), not persisted; artifact upload is always **best-effort** and never fails an execution.

**Related:** ENT-5; FI-ENT5-A; ADR-068 (the durable store this feeds); SCALE-1 (the cross-host artifact gap); ADR-056 (tenant key-prefix, applied automatically); `ExecutionStep` / `ExecutionArtifactEntity` (the wired producer).

---

## ADR-072 — HEAL-3 persisted locator store (FI-HEAL3-A) deferred: the locator-healing subsystem is unwired end-to-end

**Status:** Accepted (deferral, 2026-08-01) · **HEAL-3 remains In Progress** — records why FI-HEAL3-A (the FI-HEAL3-B unblocker in ADR-070) is itself blocked deeper than expected.

**Context.** ADR-070 identified FI-HEAL3-A — a persisted, enumerable healed-locator store fed by wiring `HealingMemory` — as the unblocker for the drift ranking. Re-grounding to build it found the blocker is **not** the persistence layer but the **entire locator-healing chain**, missing at every link:
1. **The locator healers are unwired.** `LocatorHealingService.heal(...)` and `LocatorHealCoordinator.heal(...)` (HEAL-1/HEAL-2) have **zero production callers**. The self-healing pipeline's only `.heal(...)` (`SelfHealingStep`) invokes `SelfHealingEngine` — the execution-**retry** path (retry/strategy level), which never touches individual locators.
2. **No failure → broken-locator source exists.** `LocatorHealingRequest` needs a structured `brokenLocator` + element attributes; nothing produces one. `ExecutionResult` carries an `errorMessage` string; the `SelfHealingAgent` decision JSON is `retryRequired`/`healingAction`/`reason` — no locator identity.
3. `HealingMemory.remember(...)` is therefore never called; no locator heals are ever produced.

**Decision.** **Defer FI-HEAL3-A; do not build the store now.** Persisting `HealingMemory` heals would feed off a subsystem that is never invoked and has no faithful input — a producerless half-pipeline (ADR-063). Crucially, the missing first link — deriving a **structured broken-locator from unstructured Playwright error text** — is exactly the kind of invented signal ADR-063 forbids; it must come from a genuine structured source, not text-scraping.

**What FI-HEAL3-A genuinely requires (the real, larger feature).** (a) Capture a structured broken-locator at failure time from a real source (e.g. the execution engine emitting the failing selector as structured data, not parsed from a message); (b) invoke `LocatorHealCoordinator` in the self-healing loop for locator failures; (c) `HealingMemory.remember(...)` + persist an enumerable `HealedLocatorEntity`; (d) the FI-HEAL3-B ranking read-model. This is a multi-part integration touching the core self-healing path — beyond a focused FI, and gated on (a) having a faithful, non-fabricated source.

**Consequences.** HEAL-3's produced/enumerable analytics read-model (`HealingAnalyticsSummary` over `HealingMetricEntity`, ADR-047) remains the faithful HEAL-3 surface. The locator-history half (FI-HEAL3-A/B) waits for the locator-healing subsystem to be wired end-to-end with a genuine broken-locator source — tracked, not silently dropped. Consistent with the LRN-3 (ADR-063) and FI-HEAL3-B (ADR-070) honesty calls: the codebase has several read-models/subsystems built ahead of their producers.

**Related:** HEAL-3; FI-HEAL3-A/B; ADR-070 (FI-HEAL3-B block — this is its unblocker, itself blocked); ADR-063 (never fabricate a missing signal); HEAL-1/HEAL-2 (`LocatorHealingService`/`LocatorHealCoordinator`/`HealingMemory` — the unwired subsystem); ADR-047 (the faithful HEAL-3 analytics surface).

---

## ADR-073 — Serve execution artifacts from `ArtifactStore` via an additive key endpoint (FI-ENT5-C)

**Status:** Accepted (implemented — FI-ENT5-C, 2026-08-01) · **ENT-5 remains In Progress** (backup CronJobs + multi-tenant serve-binding still open)

**Context.** FI-ENT5-A (ADR-071) uploads execution artifacts into the durable `ArtifactStore` under `keyFor(...)`. But the dashboard's `ArtifactController` serves only **local files** (`/api/artifacts/**` → `FileSystemResource` from `resolvedBaseDir`), and its metadata `toArtifactUrl(...)` **returns null when the local file is absent** — so on a different host / after local cleanup the durable copy exists but is unreachable. This slice is the read side of the FI-ENT5-A loop.

**Decision (Option A — additive key endpoint + opt-in durable URLs).**
- **`GET /api/artifacts/store/**`** (`serveFromStore`) resolves bytes from `ArtifactStore` by key: rejects `..` (400, defence-in-depth over the store's own guard), `getIfAvailable()` a null store → 404, `resolve(key)` in try/catch (throws on absent key) → 404, content-type from the key's trailing `/<type>` segment, streamed as `ByteArrayResource` with the **same SEC-4 headers** as file serving (`nosniff`, `default-src 'none'; sandbox`, `attachment` for HTML).
- **Opt-in URL emission:** when `aiqaos.artifacts.upload.enabled=true` (the same flag that governs upload), the metadata endpoints emit `/api/artifacts/store/<keyFor(...)>` URLs for produced artifacts (path column non-null) via `ArtifactUploader.keyFor` — which resolve regardless of local presence. When off, the local-file URLs are used **unchanged**. The React UI is agnostic (renders whatever URL the DTO carries) → **no UI change**.
- Dashboard gains an explicit `ai-qa-os-execution` dependency (previously only transitive via orchestration) since it now directly uses `ArtifactStore`/`keyFor`.
- **Rejected — fall back inside `/api/artifacts/**`:** that route's path is the local relative path, not `keyFor`, so it can't locate FI-ENT5-A's uploads without re-keying, and it entangles the SEC-4 filesystem guards with object-key resolution.

**Consequences.**
- *Positive:* durable artifacts are now servable → reachable cross-host / after local cleanup; additive + opt-in means **zero regression** to the working local path (default unchanged). `ArtifactStoreServingTest` 5/5 (fake store + `MockHttpServletRequest`: keyed serve + type→content-type, `..`→400, absent→404, no-store→404); full reactor green (22 modules), dashboard `@SpringBootTest` boots with the new dep.
- *Negative / trade-off:* **tenant scope** — `ArtifactStore` applies a tenant key-prefix (ADR-056) from the request's bound tenant; the open dashboard serve path has no tenant bound (→ system tenant), so durable serving is correct for **single-tenant/system** deployments and **safely 404s** (never leaks — different tenant prefixes) under multi-tenant. Multi-tenant serve-binding (tenant carried in the URL + bound at serve) is a follow-on (**FI-ENT5-E**). Backup CronJobs (FI-ENT5-B) still open → ENT-5 stays In Progress. Live cross-host round-trip is user-run.
- *Imposed rule:* durable serving reuses FI-ENT5-A's `keyFor` scheme on a dedicated route; it never mixes object-key resolution into the filesystem path guards, and it carries the same SEC-4 hardening.

**Related:** ENT-5; FI-ENT5-C; ADR-071 (FI-ENT5-A upload — the write side); ADR-068 (the durable store); ADR-056 (tenant key-prefix — the multi-tenant caveat); SEC-4 (artifact-serving hardening).

---

## ADR-074 — Runnable backup CronJobs to object storage (FI-ENT5-B)

**Status:** Accepted (implemented — FI-ENT5-B, 2026-08-02) · **completes ENT-5's original three ceilings** (real store ADR-068, worker upload ADR-071, running CronJobs — this) · ENT-5 eligible for Completed on live validation

**Context.** ENT-5's DR piece existed only as **placeholder templates** — `postgres-backup.yaml` `echo`'d "Upload … using …" without uploading, `artifacts-backup.yaml` synced to `s3://REPLACE-ME`, and the DB password was **plaintext** in the manifest. The tracker's third ENT-5 ceiling ("running CronJobs") was unmet.

**Decision.** Complete the three CronJobs into **runnable, credential-safe** manifests (`deployment/kubernetes/backup/`):
- **postgres** — `pg_dump -Fc` in an initContainer → shared `emptyDir` → `aws-cli` upload to `$BACKUP_BUCKET/postgres/`.
- **qdrant** — initContainer creates a full-storage snapshot via the Qdrant API + downloads it → `aws-cli` upload.
- **artifacts** — `aws s3 sync` the local PVC tree (legacy `store=local` deployments; with FI-ENT5-A the durable copy already exists in object storage).
- **aws-cli with `--endpoint-url` toggled by `S3_ENDPOINT`** → one manifest set works against **MinIO in-cluster and real AWS S3** (empty endpoint).
- **SEC-2:** all secrets (`AWS_*`, `S3_ENDPOINT`, `BACKUP_BUCKET`, `POSTGRES_PASSWORD`) come from an out-of-band `aiqaos-backup-secret` (`envFrom`/`secretKeyRef`) — the plaintext password is gone; `backup-secret.yaml` documents the shape only.
- **`kustomization.yaml`** bundles them; a **README** covers create-secret → adjust → apply → smoke-test → restore → retention (bucket lifecycle rule; in-app `ArtifactRetentionService` for live artifacts).

**Consequences.**
- *Positive:* ENT-5's backup/DR deliverable is real and applies with `kubectl apply -k`; no plaintext creds; portable MinIO↔S3. All three original ENT-5 ceilings are now code-complete (durable store + upload + backup). YAML syntax-validated.
- *Negative / trade-off:* **not applied to a live cluster here** (none in the build env) — semantic k8s validity (field correctness, image behaviour, the jq-less snapshot-name `sed`) is user-verified via `kubectl apply --dry-run=server` + one manual job (README step 3). Backup *retention* is a bucket lifecycle rule / follow-on scheduled trigger (FI-ENT5-F). ENT-5 stays In Progress pending the live validation (runbook §5).
- *Imposed rule:* backup manifests carry **no plaintext credentials** — everything sensitive is injected from a Secret created out-of-band; the same manifests target MinIO and real S3 via `S3_ENDPOINT`.

**Related:** ENT-5; FI-ENT5-B; ADR-068 (durable store) + ADR-071 (upload) — the other two ceilings; ADR-053 (MinIO/Postgres/Qdrant provisioned); SEC-2 (secrets from env/secret store, never committed); [Live E2E Runbook §5](./AI-QA-OS-Live-E2E-Validation-Runbook.md).

---

## ADR-075 — Per-workflow token/context budgeting mirroring the ENT-3 cost soft-cap (AI-6)

**Status:** Accepted (implemented — AI-6, 2026-08-02, **un-deferred at user request**) · AI-6 → Completed

**Context.** AI-6 ("context-window & cost budgeting per workflow") was Deferred. Its **cost** half was already delivered by ENT-3 (ADR-025: `CostBudgetEnforcer`/`SpendLedger`/`CostBudgetProperties`, soft-cap per global/workflow/agent, enforced in `LLMProviderManager.generate`). The novel half is **token/context budgeting** — and the faithful source already exists: `LLMResponse.getUsage().getInputTokens()/getOutputTokens()` are the **actual** per-call token counts, already seen by `CostTracker` keyed by workflow (`correlationId`) and agent.

**Decision (Option A — cumulative token budget, real counts only).** Mirror ENT-3 one-to-one for **tokens**:
- **`TokenLedger`** (counterpart to `SpendLedger`) accumulates **actual** `input+output` tokens per workflow / agent / global, daily rollover.
- **`TokenBudgetProperties`** (`aiqaos.context.budget.*`, disabled by default) — `perWorkflowTokens` / `perAgentTokens` / `globalDailyTokens` (`Long`, null = unlimited), `enabled`, `mode` (enforce/warn).
- **`TokenBudgetEnforcer.check(LLMRequest)`** → `BudgetVerdict` (reused; scopes `…-tokens`) — soft cap: block once **recorded actual** tokens ≥ limit.
- **`CostTracker`** feeds the `TokenLedger` with the real usage (optional field, mirrors the `SpendLedger` feed); **`LLMProviderManager.generate`** adds a parallel pre-flight block after the cost check, throwing **`TokenBudgetExceededException`** (enforce) or logging (warn).
- **Rejected — a per-request context-window pre-guard via an estimated input-token count** (`≈prompt.length()/4`): a genuine but *fuzzy* pre-call estimate; logged as FI-AI6-A rather than mixed into the faithful, real-count core.

**Consequences.**
- *Positive:* per-workflow (and agent/global) **token/context budgeting** enforced on **real counts** — no estimation, no fabrication (ADR-063); identical faithfulness + soft-cap contract to ENT-3. `TokenLedgerTest` 3/3 + `TokenBudgetEnforcerTest` 5/5; full reactor green (22 modules); default (`aiqaos.context.budget.enabled=false`) leaves `generate` unchanged (non-breaking).
- *Negative / trade-off:* the ledger is in-memory (cross-restart seeding shared with ENT-3's FI-ENT3-A); per-tenant token budgets deferred (ENT-1 scope); the per-request context pre-guard is FI-AI6-A. Live budget-tripping across a real multi-call workflow is user-run (the ledger+enforcer logic is deterministically unit-proven).
- *Imposed rule:* token/context budgeting enforces on **recorded actual token counts** (`LLMResponse.usage`), never a pre-call estimate, keeping it as faithful as the cost soft-cap.

**Related:** AI-6; ENT-3/ADR-025 (the cost soft-cap this mirrors); `CostTracker` (the shared real-usage feed); ADR-063 (never fabricate — hence real counts, not estimates); FI-AI6-A (estimated per-request context guard, deferred).

---

## ADR-076 — Artifact content signing (HMAC-SHA256) for tamper-evidence (SEC-6)

**Status:** Accepted (implemented — SEC-6 signed-artifacts half, 2026-08-02, **un-deferred at user request**) · **SEC-6 → In Progress** (mTLS half = FI-SEC6-B, infra)

**Context.** SEC-6 ("Signed artifacts & mTLS between services", regulated-deployment posture) has two deliverables: **signed artifacts** (app code) and **mTLS** (infra/config). Nothing signed artifacts; for a regulated posture, an artifact's integrity/provenance should be provable and tampering in the object store detectable.

**Decision (Option A — content signing, over signed-URLs).** The buildable half:
- **`ArtifactSigner`** (`ai-qa-os-execution`) — `sign(bytes)` = **HMAC-SHA256** hex (key from `aiqaos.artifacts.signing.secret`, SEC-2, env-injected); `verify(bytes, sig)` = constant-time compare (`MessageDigest.isEqual`). `isSigningEnabled()` = enabled **and** non-blank secret (blank → **fail closed**, treated as unsigned). Opt-in `aiqaos.artifacts.signing.enabled` (default false).
- **`ArtifactUploader`** (FI-ENT5-A) — on upload, if signing is on, also stores a **`<key>.sig`** sidecar (`store(key + ".sig", sign(bytes))`); best-effort, never fails the upload.
- **`ArtifactController.serveFromStore`** (FI-ENT5-C) — verifies the served bytes against the `<key>.sig` sidecar and sets an **`X-Artifact-Integrity`** response header: `verified` / `MISMATCH` (logged as a tamper signal) / `unverified` (no sidecar) / `unsigned` (signing off). A mismatch is **detected and surfaced, not denied** — serving returns the stored bytes; the header carries the verdict. Never verifies a `.sig` object against its own sidecar.
- **Rejected — signed URLs (access control):** a different concern from "are the artifacts trustworthy"; kept as separate FI-ENT5-D/E.

**Consequences.**
- *Positive:* real HMAC over real bytes → tamper-evidence/provenance for a regulated deployment; SEC-2 key handling; constant-time verify; default-off keeps every path byte-for-byte unchanged. `ArtifactSignerTest` 7/7 (round-trip, tamper, key-isolation, fail-closed, deterministic), `ArtifactUploaderTest` +signed-sidecar, `ArtifactStoreServingTest` +integrity-header (verified/MISMATCH/unsigned); full reactor green (22 modules).
- *Negative / trade-off:* the **mTLS** half (`FI-SEC6-B`) is infra — Spring SSL bundles / service-mesh config + cert Secrets + manifests, authored-and-cluster-validated like the backup CronJobs — so **SEC-6 stays In Progress**. Sidecar resolution shares FI-ENT5-C's tenant caveat (correct for single-tenant/system). Detection-not-denial is deliberate (an operator decides on a mismatch); a deny-on-mismatch mode is a possible FI.
- *Imposed rule:* artifact signing is **HMAC-SHA256 over the bytes** with an env/secret key (SEC-2), verified constant-time; a signature **mismatch is surfaced, never silently dropped**.

**Related:** SEC-6; ADR-071 (FI-ENT5-A upload — where sidecars are written) + ADR-073 (FI-ENT5-C serve — where they're verified); SEC-2 (signing key from env/secret); FI-SEC6-B (mTLS, infra half); FI-ENT5-D/E (signed URLs — the rejected reading, kept separate).

---

## ADR-077 — Remove the orphan `ai-qa-os-data` module (MOD-5: fold in)

**Status:** Accepted (implemented — MOD-5, 2026-08-02, **un-deferred at user request**) · MOD-5 → Completed · **reactor 22 → 21 modules**

**Context.** MOD-5 asked to "make `ai-qa-os-data` real, or fold it in." Grounding found it was **dead scaffolding**: 5 **completely empty** stub classes (`DatabaseConnectionManager`, `KnowledgeDatabase`, `ProjectDatabase`, `TransactionService`, `VectorDatabaseClient` — all `{}`), a `pom.xml` with **no dependencies**, and **zero dependents** (only the parent reactor referenced it). Every concern its names imply is already real elsewhere — vectors in `ai-qa-os-memory` (`VectorStoreClient`), project/knowledge data in the JPA repositories, connections/transactions in Spring (datasource + Flyway per ADR-024, `@Transactional`).

**Decision — fold in / remove (Option A).** Delete the module and drop it from the parent `<modules>`. **Zero code is lost** (all five classes are empty), nothing imports it, and no entity/migration/bean lived in it — so there is nothing to migrate. The reactor drops from 22 to **21 modules**. "Make it real" (Option B) was rejected: it would **duplicate** `memory`, the JPA repositories, and Spring, and cut across established boundaries — evolving the architecture to add redundancy, the opposite of the module-boundary discipline.

**Consequences.**
- *Positive:* removes a misleading empty "data" module (a named data layer that didn't exist); full reactor `mvn clean test` **BUILD SUCCESS with 21 modules** proves nothing depended on it; no functional change.
- *Neutral:* **going-forward reactor builds are "21 modules"** — historical ADR/tracker notes that say "22 modules" were accurate at the time and stay as written.
- *Imposed rule:* a shared data-access need belongs in the module that **owns** that data, not a cross-cutting "data" module — the boundary this decision reinforces.

**Related:** MOD-5; MOD-1/ADR-042 (`ai-qa-os-tenant` — a module that *does* own a concern, the contrast); ADR-024 (gateway-owned schema); `ai-qa-os-memory` (owns vector storage).

---

## Document Completion Status

**Status:** Active — new ADRs appended as decisions are made (per MNT-5)
**Version:** 1.0.0
**Convention:** ADRs are immutable once Accepted; a changed decision is recorded as a new ADR that *Supersedes* the old one, never by editing history
**Related documents:** [`AI-QA-OS-Documentation.md`](./AI-QA-OS-Documentation.md) · [`AI-QA-OS-Improvement-Roadmap.md`](./AI-QA-OS-Improvement-Roadmap.md) · [`AI-QA-OS-Implementation-Tracker.md`](./AI-QA-OS-Implementation-Tracker.md)
