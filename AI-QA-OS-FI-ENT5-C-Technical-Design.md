# FI-ENT5-C — Serve execution artifacts from `ArtifactStore` (durable / cross-host) · Technical Design

**Item:** ENT-5 (Backup, DR & data lifecycle) → sub-item **FI-ENT5-C** — durable artifact serving.
**Status:** design (awaiting decision + implement approval).
**Completes:** FI-ENT5-A (ADR-071 upload) → this is the read side of the same loop.

---

## 0. Grounding + scope

### 0.1 What exists
- **`ArtifactController`** (dashboard) serves artifacts two ways: metadata (`/api/dashboard/artifacts/{testCaseId}` → `ArtifactDTO` with per-type URLs, from `execution_artifacts` via `JdbcTemplate`) and raw bytes (`/api/artifacts/**` → streams a **local file** from `resolvedBaseDir` as `FileSystemResource`, with SEC-4 path-traversal/symlink guards + hardened headers).
- `toArtifactUrl(absolutePath)` builds each URL by relativising the **local** path — and **returns null if the local file doesn't exist**. So on a different host (or after local cleanup) the URLs go null and the UI shows nothing, even though FI-ENT5-A has uploaded a durable copy.
- **FI-ENT5-A (ADR-071):** `ArtifactUploader` stored each file's bytes in `ArtifactStore` under `keyFor(...) = executions/<executionId>/run-<n>/<browser>/<testCaseId>/<type>`.

### 0.2 What this slice delivers
Serve the **durable** copy from `ArtifactStore` so artifacts are reachable **cross-host / after local cleanup**. When durable artifacts are enabled, the metadata endpoints emit keyed store-URLs (which always resolve), and a new endpoint streams bytes from `ArtifactStore` by key. **Opt-in + additive** — default local-file serving is unchanged.

### 0.3 Deferred / out of scope
- Signed/expiring URLs (would be a SEC-6-adjacent FI).
- Migrating the `JdbcTemplate` metadata queries to repositories.
- Live cross-host round-trip validation — **user-run** (needs a running dashboard + object store).

---

## 0.4 / Decision for approval — how durable serving attaches

- **Option A (recommended) — additive key-based endpoint + opt-in durable URLs.**
  New `GET /api/artifacts/store/**` resolves bytes from `ArtifactStore` by key. When `aiqaos.artifacts.upload.enabled=true`, the metadata endpoints emit `/api/artifacts/store/<keyFor(...)>` URLs (which resolve regardless of local presence); when off, the current local-file URLs are used unchanged. **Zero regression** to the working local path; local and durable serving are cleanly separated. → **ADR-073.**

- **Option B — fall back inside the existing `/api/artifacts/**`.**
  Try the local file; if absent, resolve from `ArtifactStore`. But that route's path is the **local relative path**, not `keyFor` — it can't locate FI-ENT5-A's uploads without re-keying, and it entangles the SEC-4 filesystem guards with object-key resolution.

**Recommendation: Option A** — the durable key scheme (`keyFor`) already exists from FI-ENT5-A; serve it directly on its own route, leave the battle-tested local path untouched, and switch URLs by the same opt-in flag that governs upload. → **ADR-073.**

---

## 1. Technical Design (Option A)

### 1.1 Dependency
Add an explicit `ai-qa-os-execution` dependency to `ai-qa-os-dashboard` (today only transitive via orchestration) — the dashboard now directly uses `ArtifactStore` and `ArtifactUploader.keyFor`.

### 1.2 New serve-by-key endpoint (in `ArtifactController`)
`GET /api/artifacts/store/**` → `serveFromStore(HttpServletRequest)`:
- Extract the key after `/api/artifacts/store/`; reject any key containing `..` (400) — defence-in-depth on top of `ArtifactStore`'s own key guard.
- Resolve bytes via `ObjectProvider<ArtifactStore>` (`getIfAvailable()`); a missing store or a resolve failure (key absent) → **404** (try/catch — `resolve` may throw `NoSuchElementException`).
- Content-type from the key's trailing **type** segment (`screenshot`→`image/png`, `video`→`video/webm`, `trace`→`application/zip`, `report`→`text/html`, `log`→`text/plain`, else `application/octet-stream`).
- Stream as `ByteArrayResource` with the **same SEC-4 headers** as file serving: `X-Content-Type-Options: nosniff`, `Content-Security-Policy: default-src 'none'; sandbox`, and `Content-Disposition: attachment` for HTML (inline otherwise).

### 1.3 Durable URL emission (metadata)
`@Value("${aiqaos.artifacts.upload.enabled:false}") boolean durableArtifacts`. A pure helper `durableUrl(executionId, runNumber, browser, testCaseId, type)` → `dashboardBaseUrl + "/api/artifacts/store/" + ArtifactUploader.keyFor(new ArtifactUploadRequest(...), type)`. In `mapRowToDto`/`buildHistory`: when `durableArtifacts` is on, emit `durableUrl(...)` for screenshot/video/trace/report (and resolve the console log via the store); when off, keep `toArtifactUrl(localPath)` unchanged. The React UI is agnostic — it renders whatever URL the DTO carries, so **no UI change**.

## 2. Testing (honest)
- **`ArtifactStoreServingTest`** (dashboard, unit): construct `ArtifactController` with a fake `ArtifactStore` (in-memory) + `MockHttpServletRequest`. Assert: a stored key streams its bytes with the type-derived content-type; a `..` key → 400; an absent key / no-store → 404; `durableUrl(...)`/`contentTypeForKey(...)` pure-helper outputs. (`keyFor` round-trips with FI-ENT5-A.)
- Full reactor `mvn clean test` green (22 modules). Default (`upload.enabled=false`) leaves the metadata/serving behaviour byte-for-byte unchanged, so existing dashboard `@SpringBootTest`s don't shift.

## 3. What can't be validated here (user-run)
The live cross-host flow: execute on host A (`upload.enabled=true`, `store=object`/`s3`), fetch the artifact from the dashboard on host B via the store-URL.

## 4. Implementation plan
1. `ai-qa-os-dashboard` pom: explicit `ai-qa-os-execution` dep.
2. `ArtifactController`: `serveFromStore` endpoint + `contentTypeForKey` + `durableUrl` helpers; opt-in `durableArtifacts` URL emission in `mapRowToDto`/`buildHistory`; inject `ObjectProvider<ArtifactStore>`.
3. `ArtifactStoreServingTest`.
4. Full reactor verify.
5. Docs: ADR-073, tracker ENT-5 note, this doc's Implementation Outcome.

## 5. Follow-on (outside this slice)
- FI-ENT5-B: activate backup CronJob templates against MinIO/S3.
- FI-ENT5-D: signed/expiring artifact URLs (SEC-6-adjacent).

---

## Implementation Outcome

**Delivered 2026-08-01 (Option A / ADR-073). Full reactor green — 22 modules, 0 failures. ENT-5 stays In Progress (FI-ENT5-B/E remain).**

Shipped as designed:
- **`ai-qa-os-dashboard` pom** — explicit `ai-qa-os-execution` dependency (was transitive via orchestration).
- **`ArtifactController.serveFromStore`** (`GET /api/artifacts/store/**`) — key after `/store/`; `..`→400; `ObjectProvider<ArtifactStore>.getIfAvailable()` null→404; `resolve(key)` in try/catch (throws on absent)→404; content-type from the key's trailing type via `contentTypeForKey`; `ByteArrayResource` + SEC-4 headers (nosniff, `default-src 'none'; sandbox`, attachment-for-HTML).
- **Opt-in durable URLs** — `@Value durableArtifacts` (`aiqaos.artifacts.upload.enabled`); `artifactUrl(row,type,localPath)` emits `durableUrl(...)` (= `dashboardBaseUrl + "/api/artifacts/store/" + ArtifactUploader.keyFor(...)`) when on and the artifact was produced, else `toArtifactUrl(localPath)` unchanged. Wired into `mapRowToDto` + `buildHistory` (added `ea.test_case_id` to the history query). No UI change (the DTO carries the URL).

**Tests:** `ArtifactStoreServingTest` 5/5 — fake `ArtifactStore` + `MockHttpServletRequest`: stored key streams bytes with type-derived content-type; `..`→400; absent→404; no-store→404; `contentTypeForKey` mapping. Full reactor green (22 modules); default (`upload.enabled=false`) leaves metadata/serving byte-for-byte unchanged, so existing dashboard `@SpringBootTest`s don't shift.

**Deviation from the design (tenant scope, made explicit):** the design didn't address the `ArtifactStore` tenant key-prefix (ADR-056). As built, durable serving resolves under the request's **bound tenant** — system on the open dashboard — so it is correct for single-tenant/system deployments and **safely 404s (never leaks)** under multi-tenant (distinct tenant prefixes). Carrying the tenant in the store-URL and binding it at serve time is deferred as **FI-ENT5-E** (avoids exposing tenant IDs in URLs and the extra binding for now).

**User-run (not validatable in sandbox):** execute on host A (`upload.enabled=true` + `store=object`/`s3`), fetch the artifact from the dashboard on host B via the store-URL.

**Follow-on:** FI-ENT5-E (multi-tenant durable serve-binding), FI-ENT5-B (backup CronJobs), FI-ENT5-D (signed/expiring URLs).
