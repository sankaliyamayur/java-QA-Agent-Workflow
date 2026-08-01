# FI-ENT5-A — Execution-worker artifact upload through `ArtifactStore` · Technical Design

**Item:** ENT-5 (Backup, DR & data lifecycle) → sub-item **FI-ENT5-A** — worker artifact upload.
**Status:** design (awaiting decision + implement approval).
**Builds on:** ADR-068 (durable `ObjectStorageArtifactStore` over S3/MinIO) — this is the producer that feeds it.

---

## 0. Grounding + scope

### 0.1 What exists (the real producer)
- Executions produce artifact **files** (screenshot / video / trace / log) on local disk (`PlaywrightExecutionEngine`, output under `./playwright-output`).
- **`ExecutionStep`** (orchestration) is the faithful, wired producer: it reads `execResult.getArtifacts()` (typed `type:testCaseId=path` strings), builds an **`ExecutionArtifactEntity`** per test case (screenshot/video/trace/log/report **paths** + `executionId`, `testCaseId`, `browser`, `runNumber`, `@TenantId`), and `artifactRepo.save(...)`s it (persisted, enumerable).
- **`ArtifactStore`** (execution) is the durable seam: `LocalArtifactStore` (default) and `ObjectStorageArtifactStore` (S3/MinIO, ADR-068), tenant-key-prefixed. Today **nothing pushes execution artifacts into it** — the paths stay host-local, so a cross-host worker's artifacts aren't reachable elsewhere (the gap ENT-5/SCALE-1 flagged).

### 0.2 What this slice delivers
When `ExecutionStep` persists an `ExecutionArtifactEntity`, also **read each referenced local file and `store()` its bytes into `ArtifactStore`** under a deterministic key. With the S3 binding active the bytes land in shared object storage — **cross-host reachable**, closing SCALE-1's artifact gap at the producer. **Opt-in** (`aiqaos.artifacts.upload.enabled`, default **false**) → fully non-breaking.

### 0.3 Deferred (out of scope)
- Rewriting the dashboard artifact-**serving** path to resolve from `ArtifactStore` (it reads local files today) — a separate slice; this delivers the durable *copy*.
- Backup CronJobs (FI-ENT5-B).
- Live browser execution + real object-store round-trip — **user-run** (no browser/bucket in sandbox).

---

## 0.4 / Decision for approval — how the durable copy is addressed

Both options upload the bytes via `ArtifactStore` (best-effort, never failing the execution). The fork is **how the object key is recorded** so the durable copy can be found later.

- **Option A (recommended) — deterministic, reconstructable key; no schema change.**
  Key = `executions/<executionId>/run-<runNumber>/<browser>/<testCaseId>/<type>` — a pure function of fields the `ExecutionArtifactEntity` already carries. `ArtifactUploader.keyFor(...)` is the single source of truth, used to store now and to reconstruct-and-resolve later. **No new columns, no migration.**

- **Option B — explicit object-key columns on `ExecutionArtifactEntity` (+ migration).**
  Add `screenshot_key`/`video_key`/… columns and store the returned keys. More explicit/linkable, but a schema change + migration (V23) + wider entity for a link that Option A derives for free.

**Recommendation: Option A** — the key is fully determined by existing stable fields, so recording it is redundant; keep the entity and schema untouched and non-breaking. → **ADR-071.**

---

## 1. Technical Design (Option A)

### 1.1 New — `ArtifactUploader` (`ai-qa-os-execution`, `com.aiqaos.execution.artifact`)
`@Component @ConditionalOnProperty(aiqaos.artifacts.upload.enabled=true)`. Depends on `ObjectProvider<ArtifactStore>` (the store is itself conditional; no-op if absent).
- `List<String> upload(ArtifactUploadRequest req)` — for each non-null, existing file path in the request, read bytes and `artifactStore.store(keyFor(req, type), bytes)`; collect the stored keys. **Best-effort:** a missing file or an `IOException`/store failure is **logged and skipped**, never propagated (artifacts must never fail an execution — mirrors `BootstrapAdminInitializer`'s discipline).
- `static String keyFor(...)` — the deterministic key scheme (single source of truth; also usable by a future resolve path).
- `ArtifactUploadRequest` — `executionId`, `testCaseId`, `browser`, `runNumber`, and the typed paths (screenshot/video/trace/log/report).

### 1.2 Wiring — `ExecutionStep` (orchestration)
Inject `ObjectProvider<ArtifactUploader>` (present only when upload is enabled). Immediately after `artifactRepo.save(art)`, if an uploader is present, build an `ArtifactUploadRequest` from the just-persisted entity and call `upload(...)` inside a try/catch that only logs — the persistence and pipeline are never affected. Tenant context is already bound on this thread (ENT-1), so `ObjectStorageArtifactStore`'s tenant key-prefix applies automatically.

### 1.3 Config
`aiqaos.artifacts.upload.enabled` (default `false`). Gateway `compose` profile documents it alongside the ADR-068 object-store opt-in (enable together: `store=object` + `object.provider=s3` + `upload.enabled=true`).

## 2. Testing (honest, sandbox-faithful)
- **`ArtifactUploaderTest`** (unit): a **fake `ArtifactStore`** (in-memory map) + a JUnit `@TempDir` with real files. Asserts: each existing file's bytes stored under the expected `keyFor(...)` key; null paths skipped; a non-existent path skipped without throwing; a throwing store is swallowed (best-effort) and other files still upload. `keyFor` scheme asserted directly.
- No browser/bucket needed — the uploader's contract (files → keyed bytes in `ArtifactStore`) is fully exercised over a temp dir + fake store.
- Full reactor `mvn clean test` green (22 modules); default (`upload.enabled=false`) means `ExecutionStep` behaviour is unchanged, so existing orchestration tests don't shift.

## 3. What can't be validated here (user-run)
A live Playwright execution producing real artifacts, uploaded to the running MinIO/S3 bucket (`upload.enabled=true` + `store=object`/`provider=s3`), fetched from another instance.

## 4. Implementation plan
1. `ArtifactUploadRequest`, `ArtifactUploader` (+ `keyFor`) in `ai-qa-os-execution`.
2. `ArtifactUploaderTest` (fake store + `@TempDir`).
3. Wire `ObjectProvider<ArtifactUploader>` into `ExecutionStep` (best-effort, post-save).
4. Config + gateway `compose` doc.
5. Full reactor verify.
6. Docs: ADR-071, tracker ENT-5 note, this doc's Implementation Outcome.

## 5. Follow-on (outside this slice)
- FI-ENT5-C: resolve the dashboard artifact-serving path from `ArtifactStore` (reconstruct `keyFor(...)`), so served artifacts come from durable storage.
- FI-ENT5-B: activate backup CronJob templates against MinIO/S3.

---

## Implementation Outcome

**Delivered 2026-08-01 (Option A / ADR-071). Full reactor green — 22 modules, 0 failures. ENT-5 stays In Progress (FI-ENT5-B/C remain).**

Shipped as designed:
- **`ArtifactUploadRequest`** (record) + **`ArtifactUploader`** (`ai-qa-os-execution`, `@ConditionalOnProperty(aiqaos.artifacts.upload.enabled=true)`) — reads each non-null existing file and `ArtifactStore.store(keyFor(...), bytes)`; best-effort (missing file / store failure logged and skipped). Injects `ArtifactStore` directly (exactly one bean by construction). `keyFor` = `executions/<executionId>/run-<n>/<browser>/<testCaseId>/<type>`, null-safe.
- **`ExecutionStep`** (orchestration) — `@Autowired(required=false) ArtifactUploader` (present only when enabled), called immediately after `artifactRepo.save(art)` inside a log-only try/catch. No schema change.
- **Gateway `application-compose.yml`** documents `AIQAOS_ARTIFACTS_UPLOAD_ENABLED` (default off) alongside the ADR-068 object-store opt-in.

**Tests:** `ArtifactUploaderTest` 3/3 — fake in-memory `ArtifactStore` + real `@TempDir`: existing files stored under the expected keys, null/missing paths skipped, a throwing store swallowed (best-effort). `keyFor` null-safety asserted. Full reactor green (22 modules); default (`upload.enabled=false`) leaves `ExecutionStep` behaviour unchanged, so no existing orchestration test shifted.

**Deviations:** none. (Injected `ArtifactStore` directly rather than via `ObjectProvider` — verified `LocalArtifactStore` and `ObjectStorageArtifactStore` are mutually exclusive conditionals, so exactly one bean always exists.)

**User-run (not validatable in sandbox):** a live Playwright execution producing real artifacts, uploaded to the running MinIO/S3 bucket (`upload.enabled=true` + `store=object`/`provider=s3`), fetched from another instance.

**Follow-on:** FI-ENT5-C (dashboard artifact-serving resolves from `ArtifactStore` via `keyFor`), FI-ENT5-B (backup CronJobs against MinIO/S3).
