# ENT-5 — Real object-storage binding (S3 / MinIO) · Technical Design

**Item:** ENT-5 (Backup, DR & data lifecycle) → the deferred **real S3/GCS adapter**.
**Status:** design (awaiting decision + implement approval).
**Consumes:** the MinIO container already provisioned by docker-compose (ADR-053) — ENT-5 is its designated Java consumer.

---

## 0. Grounding + scope

### 0.1 What exists
- **Seam:** `ObjectStorageClient` (put/get/exists/list/delete/lastModified) in `ai-qa-os-execution`, sat on by `ObjectStorageArtifactStore` (an `ArtifactStore`, tenant-key-prefixed per ADR-056). Default binding is `InMemoryObjectStorageClient` — `@ConditionalOnProperty(aiqaos.artifacts.store=object)` + `@ConditionalOnMissingBean`. Unit-proven; **not durable**.
- **Selection:** `aiqaos.artifacts.store` is unset by default → `LocalArtifactStore` (single-host). `=object` switches to the object-storage store; the client is then whichever `ObjectStorageClient` bean is active.
- **Infra ready, binding absent:** docker-compose provisions **MinIO** (`aiqaos-artifacts` bucket, `minioadmin`/`minioadmin` dev creds overridable via `.env`, S3 API on `:9000`). Per ADR-053 the Java binding was deferred to its consumer — that consumer is ENT-5.
- **Established pattern:** SCALE-1 (Redis) and SCALE-2 (Kafka) both added a real binding as an **optional dependency + conditional bean**, default unchanged. This slice is the same shape for object storage.

### 0.2 What this slice delivers
A **durable `ObjectStorageClient`** backed by the S3 API (works against MinIO now, and real AWS S3 / any S3-compatible store later), behind an optional dependency so nothing that doesn't opt in carries the weight. This removes ENT-5's "no real object store" gap and completes SCALE-1's cross-host artifact reachability at the storage layer.

### 0.3 Deferred (explicitly out of scope)
- Execution-worker **upload** wiring on the hot path and running backup **CronJobs** (ENT-5's other deferred pieces) — separate slices; the `ArtifactStore` seam they'd use is unchanged.
- GCS-native adapter (GCS's S3-interop endpoint is reachable via this same S3 client; a native `google-cloud-storage` binding is a future FI if needed).
- Live MinIO round-trip validation (needs the container up + creds) — **user-run**.

---

## 0.4 / Decision for approval — which storage SDK backs the binding

- **Option A (recommended) — AWS SDK for Java v2 (`software.amazon.awssdk:s3`).**
  The portable S3 API. MinIO is S3-compatible (endpoint override + path-style access), so one binding serves **MinIO today and real AWS S3 (or any S3-compatible: GCS interop, Ceph, R2) later** with only config changes. Optional deps: `s3` + `url-connection-client` (a minimal JDK-based HTTP client — avoids Apache's bulk). → **ADR-068.**

- **Option B — MinIO Java SDK (`io.minio:minio`).**
  Slightly simpler MinIO-native API, but **MinIO-specific** — moving to AWS S3 / GCS in production means a second binding. Contradicts "provision-agnostic seam, swap the binding" and locks the durable path to one vendor.

**Recommendation: Option A** — the seam should be swappable across S3-compatible stores, not pinned to MinIO. The dev container is MinIO; production may not be.

---

## 1. Technical Design (Option A)

### 1.1 Dependencies
- Parent `pom.xml` `dependencyManagement`: import the **AWS SDK BOM** (`software.amazon.awssdk:bom:${awssdk.version}`, `2.28.0`) so module deps stay versionless/consistent.
- `ai-qa-os-execution/pom.xml`: add **`software.amazon.awssdk:s3`** and **`software.amazon.awssdk:url-connection-client`**, both `<optional>true</optional>` (not transitive — mirrors the Redis dep). No other module is affected.

### 1.2 Config — `S3StorageProperties` (`@ConfigurationProperties("aiqaos.artifacts.object.s3")`)
`bucket`, `endpoint` (optional — set for MinIO/S3-compatible, empty for real AWS), `region` (default `us-east-1`), `accessKey`, `secretKey`, `pathStyleAccess` (default `true` — MinIO requires path-style). **SEC-2:** `access-key`/`secret-key` are env/secret-injected, never committed (`.env` already carries `MINIO_ROOT_USER`/`MINIO_ROOT_PASSWORD`).

### 1.3 `S3StorageConfiguration` (`@Configuration`)
`@ConditionalOnClass(S3Client.class)` + `@ConditionalOnExpression("'${aiqaos.artifacts.store:}' == 'object' and '${aiqaos.artifacts.object.provider:in-memory}' == 's3'")`. Produces the `S3Client` `@Bean`: credentials from properties (`StaticCredentialsProvider`), `region`, `endpointOverride` when set, `serviceConfiguration(pathStyleAccessEnabled)`, `httpClient(UrlConnectionHttpClient.create())`.

### 1.4 `S3ObjectStorageClient implements ObjectStorageClient`
Same conditional as the config. Constructor takes `S3Client` + `S3StorageProperties` (bucket). Maps the seam onto S3:
- `put` → `putObject(bucket,key, RequestBody.fromBytes(content))`
- `get` → `getObjectAsBytes(...).asByteArray()`; wrap `NoSuchKeyException` → `NoSuchElementException` (matches the in-memory client's contract)
- `exists` → `headObject(...)` true / `NoSuchKeyException` → false
- `list(prefix)` → `listObjectsV2(bucket, prefix)` paginated → object keys
- `delete` → `deleteObject(...)` (no-op semantics — S3 delete is idempotent)
- `lastModified` → `headObject(...).lastModified()`; missing → `NoSuchElementException`

### 1.5 Selection wiring (robustness — the SCALE-1 lesson)
`InMemoryObjectStorageClient`'s guard changes from `@ConditionalOnMissingBean` to the **inverse expression** (`... provider != 's3'`), so exactly one `ObjectStorageClient` bean is active regardless of component-scan order (the `@ConditionalOnMissingBean`-with-`@Component` ordering is not deterministic). `ObjectStorageArtifactStore` is unchanged — it depends only on the seam.

## 2. Testing (honest)
- **`S3ObjectStorageClientTest`** (unit, Mockito-free): a hand-written **fake `S3Client`** (JDK proxy over the used methods) returning SDK response objects (`ResponseBytes`, `HeadObjectResponse`, `ListObjectsV2Response`) — proves the seam→S3 request/response/exception mapping (put/get round-trip, exists true/false via `NoSuchKeyException`, list-by-prefix, delete, lastModified, missing-key → `NoSuchElementException`). The optional `s3` dep is on the module's own test classpath (optional only blocks *downstream* transitivity), so the SDK types are available in-test.
- Full reactor `mvn clean test` stays green (22 modules); default artifact store (`local`) unchanged, so no existing test shifts.

## 3. What can't be validated here (user-run)
Live round-trip against the running MinIO container (`aiqaos.artifacts.store=object`, `...object.provider=s3`, endpoint `http://localhost:9000`, bucket `aiqaos-artifacts`) and against real AWS S3 with real creds.

## 4. Implementation plan
1. Parent pom: AWS SDK BOM + `awssdk.version` property.
2. execution pom: optional `s3` + `url-connection-client`.
3. `S3StorageProperties`, `S3StorageConfiguration` (S3Client bean).
4. `S3ObjectStorageClient`.
5. Flip `InMemoryObjectStorageClient` to the inverse-expression guard.
6. `S3ObjectStorageClientTest` (fake S3Client).
7. Compose config: document `object`/`s3` opt-in for the `compose` profile (default stays local/off).
8. Full reactor verify.
9. Docs: ADR-068, tracker ENT-5 note, this doc's Implementation Outcome.

## 5. Follow-on (outside this slice)
- FI-ENT5-A: execution-worker artifact upload on the hot path (worker writes results through `ArtifactStore`).
- FI-ENT5-B: activate the backup CronJob templates against MinIO/S3.

---

## Implementation Outcome

**Delivered 2026-07-31 (ADR-068). Full reactor green — 22 modules, 0 failures. ENT-5 stays In Progress (worker-upload + CronJobs remain).**

Shipped as designed:
- **Deps:** AWS SDK v2 BOM in parent `dependencyManagement` (`awssdk.version=2.28.0`); `software.amazon.awssdk:s3` + `url-connection-client` added `<optional>` to `ai-qa-os-execution` (not transitive — no other module carries AWS weight).
- **`S3StorageProperties`** (`aiqaos.artifacts.object.s3.*`) — bucket/endpoint/region/accessKey/secretKey/pathStyleAccess (SEC-2: creds env-injected).
- **`S3StorageConfiguration`** — builds the `S3Client` bean (static creds, `endpointOverride` when set, path-style, `UrlConnectionHttpClient`); `@ConditionalOnClass(S3Client)` + `@ConditionalOnExpression(store==object and provider==s3)`.
- **`S3ObjectStorageClient implements ObjectStorageClient`** — put/get/exists/list(paginated)/delete/lastModified over S3; `NoSuchKeyException` → `NoSuchElementException` (seam contract preserved).
- **`InMemoryObjectStorageClient`** flipped from `@ConditionalOnMissingBean` to the inverse expression (`provider != s3`) → exactly one client bean regardless of scan order. `ObjectStorageArtifactStore` unchanged.
- **Gateway `application-compose.yml`** documents the opt-in (`AIQAOS_ARTIFACTS_STORE=object` + `AIQAOS_ARTIFACTS_OBJECT_PROVIDER=s3`, MinIO endpoint/creds from `.env`); default `store=local` unchanged.

**Tests:** `S3ObjectStorageClientTest` 6/6 — Mockito-free, a JDK-proxy fake `S3Client` over an in-memory bucket proves put→get round-trip, missing-key → `NoSuchElementException`, exists true/false via `NoSuchKeyException`, list-by-prefix, delete, lastModified. Existing `ObjectStorageArtifactStoreTest` 4/4 still green with the conditional change. Full reactor green (22 modules); default artifact store unchanged so no existing test shifted.

**Deviations:** the S3 client builder type is `S3ClientBuilder` (not a nested `S3Client.Builder`) — used `var` (Java 21). Otherwise none.

**Decision confirmed:** Option A (AWS SDK v2 S3) — portable across S3-compatible stores rather than MinIO-pinned.

**User-run (not validatable in sandbox):** live round-trip against the running MinIO container (`store=object`, `provider=s3`, endpoint `http://localhost:9000`, bucket `aiqaos-artifacts`) and against real AWS S3 with real creds.

**Follow-on (ENT-5's remaining ceiling):** FI-ENT5-A (execution-worker artifact upload on the hot path), FI-ENT5-B (activate backup CronJob templates against MinIO/S3).
