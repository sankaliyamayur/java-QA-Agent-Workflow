# PLG-4 — Marketplace architecture · Technical Design

**Item:** PLG-4 (un-deferred at user request 2026-08-02) — plugin marketplace.
**Status:** design (awaiting decision + implement approval).
**Roadmap framing:** "marketplace (**new service**) · **Vision · v3.0**" — depends on PLG-1 + ENT-1.

---

## 0. Grounding — honest scope

The plugin ecosystem is otherwise complete: **PLG-1** (SPI + `PluginRegistry` runtime), **PLG-2** (integration plugins), **PLG-3** (`ExtensionRegistry` SDK), **DX-5** (`PluginManifestLoader` — declarative `plugin.json`). The registries track **installed** plugins.

PLG-4 is tiered by the roadmap itself as a **new standalone service, Vision / v3.0** — i.e. a full marketplace (hosting, distribution, publishing pipeline, browse UI, ratings, possibly billing) is deliberately **out of scope** for a bounded, faithful slice, and would be a large greenfield service. Building "the marketplace service" here would be neither honest nor verifiable. What *is* faithful and buildable now: the **discovery/publish core** the marketplace is built on — a **plugin catalog** — plus recording the **architecture** for the full service (the item's name is literally "marketplace *architecture*").

A marketplace **catalog** is distinct from `PluginRegistry`: the registry holds **installed/running** plugins; the catalog holds **available/publishable** plugin *listings* one could discover and install. It's platform infrastructure of the same class as `ExtensionRegistry`/`PluginRegistry` (an API you publish into / query), not a data read-model.

---

## 0.1 / Decision for approval — how much of PLG-4 to build now

- **Option A (recommended) — catalog foundation + architecture ADR.**
  Build **`PluginCatalog`** (publish/discover plugin *listings*, governed) — the marketplace's discovery core, reusing DX-5's manifest concept + PLG-1's `SemanticVersion` compat — and record the **full marketplace-service architecture** in ADR-080 (how the standalone service, publish pipeline, install flow, tenancy, and artifact signing compose). Fully unit-testable. **PLG-4 → In Progress** (foundation + architecture done; the standalone v3.0 service is the honest remainder). → **ADR-080.**

- **Option B — architecture only (no code).**
  Just the ADR/design for the marketplace service, since it's a Vision/v3.0 "new service." Honest, but ships no verifiable code; PLG-4 stays In Progress on a design alone.

**Recommendation: Option A.** Ship a real, testable foundation *and* the architecture — not pure design — while being explicit that the standalone marketplace **service** is v3.0 and **not** built here (so PLG-4 is **In Progress**, not Completed).

---

## 1. Technical Design (Option A) — `ai-qa-os-integration`, `com.aiqaos.integration.plugin.marketplace`

### 1.1 New
- **`PluginListing`** (record) — a catalog entry: `id`, `name`, `version` (`SemanticVersion`), `sdkApiVersion` (`SemanticVersion`), `description`, `category` (String — marketplace categories evolve), `author`, `capabilities` (`Set<String>`). Static `from(PluginManifest manifest, String name, String description, String category, String author)` — so a published `plugin.json` (via DX-5) becomes a listing directly.
- **`PluginCatalogException`** — publish-rejection errors (author-facing).
- **`PluginCatalog`** (`@Component`) — the discovery core, mirroring `ExtensionRegistry`:
  - `publish(PluginListing)` — governance: non-blank id/name; SDK-version **compatible** with the runtime (`ExtensionSdkProperties.apiVersion`, reused); reject a **duplicate `id@version`** (republish of the *same* version); newer version of an existing id supersedes as the "latest".
  - `find(id)` → latest listing; `versions(id)` → all published versions; `search(query)` → by id/name/description substring; `byCategory(category)`; `byCapability(cap)`; `all()`.

### 1.2 Architecture (recorded in ADR-080, not built)
The full service: a **marketplace-service** deployable fronting `PluginCatalog`; a **publish** API (auth'd, SEC-6-signed plugin artifacts, SEC-2 secrets); an **install** flow (fetch → `PluginManifestLoader.load` → `PluginRegistry.register` → enable) reusing what exists; **tenant-scoped** catalogs (ENT-1); browse UI; ratings/versioning. This slice delivers the catalog seam it all sits on.

## 2. Testing (Mockito-free)
- **`PluginCatalogTest`** — publish + `find`/`search`/`byCategory`/`byCapability`; duplicate `id@version` rejected; blank id/name rejected; SDK-incompatible listing rejected; a newer version supersedes as `find` "latest" while `versions(id)` keeps both; `from(manifest, …)` maps correctly.
- Full reactor `mvn clean test` green (21 modules); purely additive.

## 3. What can't be validated here
The standalone marketplace **service** (hosting/distribution/publish pipeline/UI) — explicitly v3.0, not built. The catalog core is fully unit-verified.

## 4. Implementation plan
1. `PluginListing`, `PluginCatalogException`, `PluginCatalog` (`plugin.marketplace`).
2. `PluginCatalogTest`.
3. Full reactor verify.
4. Docs: ADR-080 (catalog + full-service architecture), tracker PLG-4 (Deferred → In Progress) + counts, this doc's Implementation Outcome.

## 5. Follow-on (the v3.0 service)
- FI-PLG4-A: marketplace-service module + publish/browse API (auth, SEC-6-signed artifacts).
- FI-PLG4-B: install flow (catalog → loader → registry) + tenant-scoped catalogs (ENT-1).
- FI-PLG4-C: browse UI, ratings, dependency resolution.

---

## Implementation Outcome

**Delivered 2026-08-02 (Option A / ADR-080). Full reactor green — 21 modules, 0 failures. PLG-4 → In Progress (foundation + architecture; standalone service is v3.0).**

Shipped as designed (`ai-qa-os-integration`, `com.aiqaos.integration.plugin.marketplace`):
- **`PluginListing`** (record) + `from(PluginManifest, name, description, category, author)` — the DX-5 link.
- **`PluginCatalog`** (`@Component`) — governed `publish` (SDK-compat via `ExtensionSdkProperties`, one publish per `id@version`, newer supersedes as latest) + `find`/`versions`/`search`/`byCategory`/`byCapability`/`all`.
- **`PluginCatalogException`**.

**Tests:** `PluginCatalogTest` 7/7 — publish+discover (find/search/category/capability), newer-version-supersedes while `versions` keeps both, duplicate `id@version` rejected, blank id/name rejected, SDK-incompatible rejected, `from` mapping, empty catalog. Full reactor green (21 modules); purely additive.

**Deviations:** none. Category is a free-form String (marketplace categories evolve) rather than an enum.

**Scope honesty:** the standalone marketplace **service** (hosting, distribution, publish pipeline, browse UI, ratings) is **not built** — it is Vision/v3.0. This slice is the discovery-core foundation + the recorded architecture, so **PLG-4 stays In Progress**.

**Follow-on (the v3.0 service):** FI-PLG4-A (marketplace-service module + publish/browse API over SEC-6-signed artifacts), FI-PLG4-B (install flow + tenant-scoped catalogs), FI-PLG4-C (browse UI, ratings, dependency resolution).
