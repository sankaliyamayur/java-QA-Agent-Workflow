# SEC-6 — Signed artifacts & mTLS between services · Technical Design

**Item:** SEC-6 (un-deferred at user request 2026-08-02) — regulated-deployment posture.
**Status:** design (awaiting decision + implement approval).

---

## 0. Grounding + scope

SEC-6 has **two deliverables**:
1. **Signed artifacts** — the buildable, unit-testable **app-code** half (this slice).
2. **mTLS between services** — mutual-TLS for inter-service calls. This is **infrastructure/config** (certificates, a service mesh or Spring SSL bundles + k8s Secrets), **not application logic** — like the backup CronJobs, it's authored-but-cluster-validated IaC, not something with a Java unit test. Handled as **FI-SEC6-B** (a config/manifest deliverable) so it doesn't hold up the testable half.

Nothing signs artifacts today; SEC-2 already externalises secrets (so a signing key comes from env/secret, never committed).

### 0.1 The fork lives here — what "signed artifacts" means
Two legitimate, materially different readings:

- **Option A (recommended) — content signing (integrity / provenance).**
  When an artifact is stored, compute an **HMAC-SHA256 signature over its bytes** (key from `aiqaos.artifacts.signing.secret`, SEC-2) and persist it as a sidecar (`<key>.sig`). On retrieval the signature is **re-verified** (tamper-evidence) and exposed. This is what "signed artifacts" means for a **regulated-deployment posture** — an artifact's integrity/provenance can be proven, and tampering in the object store is detectable. Fits SEC-6's *Infrastructure / execution* framing and pairs naturally with mTLS (both are platform-trust, not user-access).

- **Option B — signed URLs (time-limited access).**
  HMAC-signed, expiring URLs (`…?tenant=…&exp=…&sig=…`) for the durable serve endpoint (FI-ENT5-C). Also closes FI-ENT5-D (signed/expiring) and FI-ENT5-E (multi-tenant serve — the signature authorises a specific tenant+key, so the browser loads it without a bearer token and cross-tenant enumeration is impossible). This is **access control**, a different concern from "are the artifacts trustworthy."

**Recommendation: Option A.** SEC-6's own framing — "*signed artifacts* & mTLS between services" for a "*regulated-deployment posture*", in the *Infrastructure/execution* lane — reads as **artifact integrity/provenance**, not access-control URLs. (Signed URLs remain a good, separate FI-ENT5-D/E.) → **ADR-076.**

---

## 1. Technical Design (Option A — content signing) · `ai-qa-os-execution`, `com.aiqaos.execution.artifact`

### 1.1 New
- **`ArtifactSignatureProperties`** (`@ConfigurationProperties("aiqaos.artifacts.signing")`): `enabled` (default **false**), `secret` (SEC-2, env-injected). Non-breaking by default.
- **`ArtifactSigner`** (`@Component`): `String sign(byte[] content)` → HMAC-SHA256 hex; `boolean verify(byte[] content, String signature)` → constant-time compare (`MessageDigest.isEqual`). Uses `javax.crypto.Mac` "HmacSHA256". Blank/absent secret when enabled → fail closed (log + treat as unsigned).

### 1.2 Integration (opt-in, best-effort — mirrors FI-ENT5-A discipline)
- **`ArtifactUploader`** (FI-ENT5-A): after storing an artifact, if signing is enabled, `artifactStore.store(key + ".sig", signer.sign(bytes).getBytes())` — a sidecar signature per durable artifact. Signing failure never fails the upload.
- **`ArtifactController.serveFromStore`** (FI-ENT5-C): when signing is enabled, load the `<key>.sig` sidecar, **verify** the served bytes against it, and set an **`X-Artifact-Signature`** (+ `X-Artifact-Integrity: verified|unverified`) response header so a consumer can check provenance. A verification **mismatch** is logged (tamper signal) and surfaced via the header; serving still returns the bytes (detection, not denial — the artifact is what's in the store).

### 1.3 Faithfulness / security
Real HMAC over real bytes — no fabrication. The signing key is env/secret-injected (SEC-2), never committed. Constant-time comparison avoids timing oracles. Default-off keeps every existing path byte-for-byte unchanged.

## 2. Testing (Mockito-free)
- **`ArtifactSignerTest`** — sign→verify round-trip; tampered bytes → verify false; different secret → verify false; blank secret → fail-closed; signature is stable/deterministic for the same (bytes, secret).
- **`ArtifactUploaderTest`** (extend) — with signing enabled + a fake store, a `<key>.sig` sidecar is written next to each artifact and verifies against the stored bytes.
- Full reactor `mvn clean test` green (22 modules); default (`signing.enabled=false`) unchanged.

## 3. What can't be validated here (user-run)
End-to-end tamper-evidence against a live object store; the mTLS half (FI-SEC6-B) on a real cluster.

## 4. Implementation plan
1. `ArtifactSignatureProperties`, `ArtifactSigner` + `ArtifactSignerTest`.
2. Sidecar write in `ArtifactUploader` (+ test); verify + headers in `ArtifactController.serveFromStore`.
3. Full reactor verify.
4. Docs: ADR-076, tracker SEC-6 note (signed-artifacts done; mTLS = FI-SEC6-B → SEC-6 stays In Progress), this doc's Implementation Outcome.

## 5. Follow-on
- **FI-SEC6-B — mTLS between services** (the infra half): Spring SSL bundles / service-mesh config + cert Secrets + k8s manifests (authored IaC, cluster-validated), like the backup CronJobs.
- FI-ENT5-D/E — signed/expiring + multi-tenant serve URLs (Option B), if wanted separately.

---

## Implementation Outcome

**Delivered 2026-08-02 (Option A / ADR-076). Full reactor green — 22 modules, 0 failures. SEC-6 → In Progress (signed-artifacts half done; mTLS = FI-SEC6-B).**

Shipped as designed:
- **`ArtifactSignatureProperties`** (`aiqaos.artifacts.signing`) + **`ArtifactSigner`** (`ai-qa-os-execution`) — HMAC-SHA256 sign/verify; `isSigningEnabled()` = enabled ∧ non-blank secret (fail-closed); constant-time compare.
- **`ArtifactUploader`** — stores a `<key>.sig` sidecar per artifact when signing is on (best-effort).
- **`ArtifactController.serveFromStore`** — verifies bytes vs the sidecar, sets `X-Artifact-Integrity` (`verified`/`MISMATCH`/`unverified`/`unsigned`); mismatch logged, still served.

**Tests:** `ArtifactSignerTest` 7/7 (round-trip, tampered-content, different-secret, deterministic, blank-secret fail-closed, disabled, null-sig); `ArtifactUploaderTest` 4/4 (added: signed sidecar written + verifies); `ArtifactStoreServingTest` 7/7 (added: verified + MISMATCH header, unsigned header). Full reactor green (22 modules); default (`signing.enabled=false`) leaves every path unchanged (integrity header = `unsigned`).

**Deviations:** none. Detection-not-denial on mismatch is intentional (an operator decides); a deny-on-mismatch mode is a possible FI.

**User-run (not validatable in sandbox):** end-to-end tamper-evidence against a live object store; the mTLS half on a real cluster.

**Follow-on:** **FI-SEC6-B — mTLS between services** (the infra half: Spring SSL bundles / service-mesh config + cert Secrets + manifests). FI-ENT5-D/E — signed/expiring + multi-tenant serve URLs (the rejected reading), if wanted separately.
