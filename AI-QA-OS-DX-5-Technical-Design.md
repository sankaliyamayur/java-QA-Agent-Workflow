# DX-5 — Plugin SDK (developer half of Category M) · Technical Design

**Item:** DX-5 (un-deferred at user request 2026-08-02) — the author-facing plugin SDK.
**Status:** design (awaiting decision + implement approval).
**Depends on:** PLG-1 (plugin SPI + registry) — done.

---

## 0. Grounding — what the plugin ecosystem already has

The **runtime** half is built:
- **PLG-1:** `Plugin` SPI (lifecycle hooks), `PluginManifest` (id / version / sdkApiVersion / capabilities / requiredPermissions), `PluginContext` (granted permissions + config), `PluginRegistry.register(Plugin, PluginManifest)` (governance: unique id, SDK-version compatibility via `SemanticVersion`, permission checks), `PluginState`.
- **PLG-3:** the extension SDK — `ExtensionRegistry` / `ExtensionSdkProperties` for custom `Extension`s (agent/engine/reporter/browser) with SDK-version governance.
- Integration plugins (GitHub/Jira/Slack/… via `AbstractIntegrationPlugin`).

**The developer-facing gap:** a plugin author must **hand-construct `PluginManifest` in Java** (`new PluginManifest(...)`, only in `AbstractIntegrationPlugin`). There is **no declarative manifest** — no way to ship a `plugin.json` and have the SDK parse + validate it. Every mature plugin SDK has this (`package.json`, `plugin.yml`): the author declares metadata in a file, the runtime loads it. That declarative manifest + validator is the missing developer half.

---

## 0.1 / Decision for approval — DX-5 scope

- **Option A (recommended) — declarative manifest + loader/validator.**
  A `PluginManifestLoader` (in `plugin.sdk`) reads a `plugin.json` (string / stream / classpath) into a validated `PluginManifest`, with clear, actionable errors (`PluginManifestException`). This is the concrete missing ergonomic — authors declare a plugin in a file instead of writing constructor code — and it's fully unit-testable. → **ADR-079.**

- **Option B — a broader SDK bundle.**
  A + an `AbstractPlugin` base class + a reference/example plugin + author docs. More surface, but the base classes largely exist (`AbstractIntegrationPlugin`, the default-method `Plugin` SPI), and "example + docs" is open-ended. Better logged as follow-on FIs than bundled into one slice.

**Recommendation: Option A.** Ship the one concrete, testable developer ergonomic that's genuinely missing (the declarative manifest), and log the rest as FIs. DX-5 → Completed on A; the SDK's runtime pieces already exist.

---

## 1. Technical Design (Option A) — `ai-qa-os-integration`, `com.aiqaos.integration.plugin.sdk`

### 1.1 New
- **`PluginManifestException extends RuntimeException`** — a clear, author-facing error for any parse/validation failure.
- **`PluginManifestLoader`** (`@Component`, constructor takes `ObjectMapper`):
  - `PluginManifest load(String json)` — parse + validate.
  - `PluginManifest load(InputStream json)` — same, for a file/resource stream.
  - `PluginManifest loadFromClasspath(String resource)` — the common author path (`plugin.json` on the classpath); missing resource → `PluginManifestException`.

### 1.2 The `plugin.json` schema
```json
{
  "id": "com.acme.my-plugin",
  "version": "1.2.0",
  "sdkApiVersion": "1.0.0",
  "capabilities": ["report.export"],
  "requiredPermissions": ["network.http"]
}
```
Validation (all failures → `PluginManifestException` with a specific message):
- `id` — required, non-blank.
- `version` / `sdkApiVersion` — required, parseable via **PLG-1's `SemanticVersion.parse`** (reused, so the compatibility check downstream is consistent).
- `capabilities` / `requiredPermissions` — optional arrays → `Set<String>` (default empty), blanks skipped.
- Malformed JSON / unknown-but-ignored fields handled gracefully (fail only on the fields that matter).

The result is a `PluginManifest` ready for `PluginRegistry.register(plugin, manifest)` — closing the loop from declarative file → governed registration.

## 2. Testing (Mockito-free)
- **`PluginManifestLoaderTest`** — full manifest → all fields mapped (id, versions, capability/permission sets); minimal manifest (id + versions only) → empty sets; missing/blank `id` → exception; unparseable `version` → exception; malformed JSON → exception; `loadFromClasspath` of a bundled test `plugin.json` → success; missing classpath resource → exception.
- Full reactor `mvn clean test` green (21 modules); purely additive (new SDK class), so nothing existing shifts.

## 3. What can't be validated here
Nothing — this is self-contained, faithful, fully unit-verified logic.

## 4. Implementation plan
1. `PluginManifestException`, `PluginManifestLoader` (`plugin.sdk`).
2. `PluginManifestLoaderTest` (+ a `src/test/resources` sample `plugin.json`).
3. Full reactor verify.
4. Docs: ADR-079, tracker DX-5 (Deferred → Completed) + count, this doc's Implementation Outcome.

## 5. Follow-on
- FI-DX5-A: a reference/example plugin + author guide (docs).
- FI-DX5-B: an `AbstractPlugin` convenience base for the general SPI (beyond `AbstractIntegrationPlugin`); a `PluginSdk` facade that loads a classpath manifest and registers in one call.

---

## Implementation Outcome

**Delivered 2026-08-02 (Option A / ADR-079). Full reactor green — 21 modules, 0 failures. DX-5 → Completed.**

Shipped as designed (`ai-qa-os-integration`, `com.aiqaos.integration.plugin.sdk`):
- **`PluginManifestException`** — field-specific author-facing errors.
- **`PluginManifestLoader`** (`@Component`) — `load(String)` / `load(InputStream)` / `loadFromClasspath(String)` → validated `PluginManifest`; reuses `SemanticVersion.parse`; optional capability/permission arrays → `Set<String>`.
- A `src/test/resources/test-plugin.json` sample manifest.

**Tests:** `PluginManifestLoaderTest` 9/9 — full manifest maps all fields; minimal manifest → empty sets; missing/blank id, unparseable version, missing sdkApiVersion, malformed JSON, empty input all throw `PluginManifestException`; `loadFromClasspath` reads the bundled sample; missing resource throws. Full reactor green (21 modules); purely additive (new SDK class), nothing existing shifted.

**Deviations:** none.

**Decision confirmed:** Option A (declarative manifest + loader) — the concrete missing ergonomic; broader bundle deferred to FIs.

**User-run:** none required — self-contained, fully unit-verified.

**Follow-on:** FI-DX5-A (reference example plugin + author guide), FI-DX5-B (`AbstractPlugin` convenience base + a `PluginSdk` facade that loads a classpath manifest and registers in one call).
