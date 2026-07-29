# AI-QA-OS — Technical Design: AI-4

**Feature ID:** AI-4  
**Title:** Semantic & Prompt Caching for AI Invocations  
**Category:** Category C — Missing AI Capabilities  
**Status:** In Progress  
**Owner:** AI-Provider / Memory  
**Target Phase:** Phase 2  
**Target Version:** v1.5  
**Architectural Impact:** Intercepts LLM calls in `LLMProviderManager` to serve cached responses when cosine similarity between query and stored prompt embeddings exceeds a configurable threshold (0.95), cutting latency and LLM token costs.  

---

## 1. Problem Statement & Context

Every pipeline step (e.g., test case generation, script generation, bug analysis, self-healing) invokes external LLMs via `LLMProviderManager.generate(LLMRequest)`. When identical or semantically equivalent prompts are executed across test runs, the platform currently incurs full latency (2-15 seconds) and token costs for duplicate LLM invocations.

With **SCALE-3** completed, `ai-qa-os-memory` provides a standardized vector store (`VectorStoreClient` with `Qdrant` in prod and `InMemory` in dev/test) capable of high-performance vector search and cosine similarity scoring.

---

## 2. Design Details

```
   LLMRequest
       │
       ▼
┌───────────────────────────────┐
│     LLMProviderManager        │
└──────────────┬────────────────┘
               │
       [Cache Enabled?]
        /             \
     Yes               No
      │                 │
      ▼                 │
┌──────────────────┐    │
│ PromptCache      │    │
│ (VectorStore)    │    │
└──────┬───────────┘    │
       │                │
  [Similarity >= 0.95?] │
    /        \          │
  Hit        Miss       │
   │           └────────┴─────────┐
   ▼                              ▼
Return Cached               Invoke LLM Provider
LLMResponse                 (Gemini/OpenAI/Claude)
(0ms, $0)                         │
                                  ▼
                            Save to Cache
                                  │
                                  ▼
                             Return Response
```

### 2.1 Component Architecture
1. **`PromptCacheManager.java`** (in `ai-qa-os-ai-provider` or `ai-qa-os-memory`):
   - Implements cache-aside semantics for `LLMRequest` / `LLMResponse`.
   - Computes embedding vector for input prompt (or hashes raw text for exact match fallback).
   - Searches collection `"prompt_cache"` via `VectorStoreClient.search(...)`.
   - Evaluates similarity score against `aiqaos.ai.cache.similarity-threshold` (default `0.95`).
   - On hit: deserializes cached response, marks `cached=true`, sets latency = 1ms.
   - On miss: delegates to LLM provider, saves new prompt vector + metadata (`responseJson`, `model`, `promptHash`) to vector store.

2. **Configuration (`PromptCacheProperties.java`)**:
   - `aiqaos.ai.cache.enabled`: `true` / `false` (default: `true`).
   - `aiqaos.ai.cache.similarity-threshold`: `0.95` (configurable `0.80`–`1.00`).
   - `aiqaos.ai.cache.ttl-seconds`: `86400` (default: 24h).

---

## 3. Proposed Changes

### `ai-qa-os-ai-provider`

#### [NEW] [PromptCacheProperties.java](file:///d:/QA%20AI%20Automation/AI-QA-OS%20Architecture/AI-QA-OS-Core/ai-qa-os-ai-provider/src/main/java/com/aiqaos/provider/cache/PromptCacheProperties.java)
- Spring `@ConfigurationProperties(prefix = "aiqaos.ai.cache")` for enabling cache and configuring threshold.

#### [NEW] [PromptCacheManager.java](file:///d:/QA%20AI%20Automation/AI-QA-OS%20Architecture/AI-QA-OS-Core/ai-qa-os-ai-provider/src/main/java/com/aiqaos/provider/cache/PromptCacheManager.java)
- Service managing cache lookup, vector matching, and saving responses.

#### [MODIFY] [LLMProviderManager.java](file:///d:/QA%20AI%20Automation/AI-QA-OS%20Architecture/AI-QA-OS-Core/ai-qa-os-ai-provider/src/main/java/com/aiqaos/provider/manager/LLMProviderManager.java)
- Inject `ObjectProvider<PromptCacheManager>`.
- Intercept `generate(LLMRequest)`: check cache before LLM call; save on miss.

#### [NEW] [PromptCacheManagerTest.java](file:///d:/QA%20AI%20Automation/AI-QA-OS%20Architecture/AI-QA-OS-Core/ai-qa-os-ai-provider/src/test/java/com/aiqaos/provider/cache/PromptCacheManagerTest.java)
- Unit test for cache hit, cache miss, threshold enforcement, and disabling flag.

---

## 4. Verification Plan

### Automated Tests
- Run `mvn test -pl ai-qa-os-ai-provider` to verify prompt cache unit tests pass.
- Run `mvn clean test` across reactor.

### Manual Verification
- Verify cache hit returns identical `LLMResponse` with near-zero latency.
