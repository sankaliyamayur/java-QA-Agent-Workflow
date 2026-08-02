# AI-5 — Complete/remove Claude; wire local model · Technical Design

**Item:** AI-5 (un-deferred at user request 2026-08-02) — resolve the two stub LLM providers.
**Status:** design (awaiting decision + implement approval).

---

## 0. Grounding

The provider layer has a clean contract (`LLMProvider`: `generate` / `getProviderName` / `isAvailable` / `supports`) and a proven, fully-wired reference: **`OpenAIProvider`** — Spring `RestClient` to the vendor API, `ApiKeyPool` reading `OPENAI_API_KEY` from `SecretManager`, JSON request/response mapping into `LLMResponse` + `TokenUsage`, and key-rotation on 401/403/429. `GeminiProvider` is likewise wired; `SimulatorProvider` is the deterministic local fallback.

**The two unresolved stubs:**
- **`ClaudeProvider`** — `generate()`/`stream()` throw "not wired to a real API yet"; `isAvailable()` returns **false** (so `ModelRouter` never picks it). Honest stub.
- **`OllamaProvider`** — identical honest stub. This is the intended **local model** provider (Ollama runs models locally, no cloud key).

AI-5 = "**complete/remove Claude; wire local model**" — wire Ollama (the directive) and decide Claude's fate (the fork).

---

## 0.1 / Decision for approval — Claude

- **Option A (recommended) — complete Claude + wire Ollama.**
  Both stubs become real providers, each mirroring `OpenAIProvider`:
  - **Ollama** → `RestClient` to a local Ollama server (`/api/generate`); no API key (local); config-gated.
  - **Claude** → `RestClient` to the Anthropic **Messages API** (`/v1/messages`, `x-api-key` + `anthropic-version` headers), `ApiKeyPool` on `ANTHROPIC_API_KEY`.
  The platform is Claude-centric (it's built with Claude), so **completing** Claude fits its intent, and Ollama gives a **cloud-free** path. → **ADR-078.**

- **Option B — wire Ollama + remove Claude.**
  If Claude isn't a target, delete the stub (like MOD-5's empty-module removal) and only wire the local model. Simpler, but drops a first-class provider the platform otherwise assumes.

**Recommendation: Option A.** Keep + complete Claude (aligned with the platform), and wire the local model. Removing a provider the platform is named around is the weaker choice.

---

## 1. Technical Design (Option A) — `ai-qa-os-ai-provider`

### 1.1 `OllamaProvider` (rewrite from stub)
- `RestClient` to `${aiqaos.provider.ollama.base-url:http://localhost:11434}/api/generate`.
- Request JSON: `{model, prompt, system?, stream:false, options:{temperature, num_predict:maxTokens}}` (model from `aiqaos.provider.ollama.model`, default e.g. `llama3`).
- Response: `{response, model, prompt_eval_count, eval_count}` → `LLMResponse(response, model, new TokenUsage(prompt_eval_count, eval_count), durationMs)` — Ollama returns **real token counts**, so cost/AI-6 budgeting stays faithful.
- `isAvailable()` = `aiqaos.provider.ollama.enabled` (default **false**) — opt-in, so a missing local server never makes the router pick it.
- **Test seam:** package-private ctor taking a `RestClient` so `MockRestServiceServer` can drive it.

### 1.2 `ClaudeProvider` (rewrite from stub)
- `RestClient` to `https://api.anthropic.com/v1/messages`; headers `x-api-key: <key>`, `anthropic-version: 2023-06-01`, `content-type: application/json`.
- Request: `{model, max_tokens, system?, messages:[{role:"user", content:prompt}]}` (model from `aiqaos.provider.claude.model`, default `claude-3-5-sonnet-latest`).
- Response: `{content:[{type:"text", text}], model, usage:{input_tokens, output_tokens}}` → `LLMResponse`.
- `ApiKeyPool` on `ANTHROPIC_API_KEY` (mirrors OpenAI); key-rotation on 401/403/429; `isAvailable()` = `keyPool.hasKeys()`.
- Same `RestClient` test seam.

### 1.3 Config
`ProviderProperties`-style `@Value`s (no new global surface): `aiqaos.provider.ollama.{enabled,base-url,model}`, `aiqaos.provider.claude.model`. Both default to today's behaviour (Claude unavailable until a key is set; Ollama off until enabled) — non-breaking; `ModelRouter`/fallback pick them up only when genuinely available.

## 2. Testing (Mockito-free, `MockRestServiceServer`)
- **`OllamaProviderTest`** — bind `MockRestServiceServer` to the injected `RestClient`; assert the POST hits `/api/generate` with the mapped body, and a canned Ollama JSON response maps to `LLMResponse` (text + token counts); `isAvailable()` reflects the enabled flag.
- **`ClaudeProviderTest`** — assert the POST hits `/v1/messages` with the `x-api-key`/`anthropic-version` headers + mapped body, and a canned Anthropic response maps correctly; `isAvailable()` reflects a stubbed `SecretManager` key; a 401 marks the key exhausted (mirrors OpenAI). `SecretManager` is hand-stubbed.
- Full reactor `mvn clean test` green (21 modules); the existing `LLMProviderManager`/router tests are unaffected (both providers still default to unavailable/off).

## 3. What can't be validated here (user-run)
Real calls to a running **Ollama** server and to the **Anthropic API** with a real key. The request/response **mapping** is proven in-sandbox against the documented schemas via `MockRestServiceServer`; the live round-trip confirms the schema matches.

## 4. Implementation plan
1. `OllamaProvider` (+ config) + `OllamaProviderTest`.
2. `ClaudeProvider` (+ config, `ApiKeyPool` on `ANTHROPIC_API_KEY`) + `ClaudeProviderTest`.
3. Full reactor verify.
4. Docs: ADR-078, tracker AI-5 (Deferred → Completed) + count, this doc's Implementation Outcome.

## 5. Follow-on
- FI-AI5-A: Ollama/Claude **streaming** (`StreamingLLMProvider.stream`) beyond the current generate-then-emit; embeddings via a local model.

---

## Implementation Outcome

**Delivered 2026-08-02 (Option A / ADR-078). Full reactor green — 21 modules, 0 failures. AI-5 → Completed.**

Shipped as designed (`ai-qa-os-ai-provider`):
- **`OllamaProvider`** rewritten from stub → `RestClient` to `/api/generate`; real token counts from `prompt_eval_count`/`eval_count`; opt-in `aiqaos.provider.ollama.{enabled,base-url,model}`; `isAvailable()` = enabled flag.
- **`ClaudeProvider`** completed → `RestClient` to the Anthropic Messages API with `ApiKeyPool` on `ANTHROPIC_API_KEY`, key-rotation on 401/403/429, `aiqaos.provider.claude.model` (default `claude-3-5-sonnet-latest`); `isAvailable()` = key present.
- Both keep a package-private `RestClient` test seam; the public `@Component` constructor is `@Autowired`.

**Tests:** `OllamaProviderTest` 2/2 + `ClaudeProviderTest` 3/3 via `MockRestServiceServer` — assert the outbound request (URI, Anthropic headers, mapped JSON body incl. `max_tokens`) and the response mapping (text + token usage), plus `isAvailable()` and the no-key throw. Full reactor green (21 modules); default (no key / Ollama off) leaves `ModelRouter` behaviour unchanged.

**Deviations / bugs caught:**
1. `LLMRequest.getMaxTokens()` defaults to **2048** (not 0) — the test assumption was wrong; the code correctly carries the request's value. Test assertion corrected to 2048.
2. A `@Component` with **two constructors** failed context boot ("no default constructor") — fixed by `@Autowired` on the public constructor of both providers (caught by the integration `@SpringBootTest`, not the module's own unit tests).

**Decision confirmed:** Option A — completed Claude (not removed) + wired Ollama.

**User-run (not validatable in sandbox):** a real call to a running Ollama server (`aiqaos.provider.ollama.enabled=true`) and to the Anthropic API with a real `ANTHROPIC_API_KEY`.

**Follow-on:** FI-AI5-A — true streaming (`stream`) beyond generate-then-emit; embeddings via a local model.
