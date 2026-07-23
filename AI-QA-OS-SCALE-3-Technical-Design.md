# AI-QA-OS — Technical Design: SCALE-3

**Feature ID:** SCALE-3  
**Title:** Standardise on One Vector Store Provider  
**Category:** Category H — Scalability  
**Status:** Completed  
**Owner:** Memory  
**Target Phase:** Continuous  
**Target Version:** v1.5  
**Architectural Impact:** Reduces maintenance surface by standardising on Qdrant for production and InMemory for dev/test fallback, isolating experimental vector store stubs.  

---

## 1. Problem Statement & Context

`ai-qa-os-memory` shipped with five `VectorStoreClient` implementations (`QdrantStoreClient`, `InMemoryVectorStoreClient`, `ChromaStoreClient`, `MilvusStoreClient`, `PgVectorStoreClient`). Maintaining five vector clients creates unnecessary surface area and cognitive load. 

Deployment infrastructure natively provisions Qdrant (`qdrant:6333`).

---

## 2. Design Details

### 2.1 Provider Hierarchy
- **Qdrant (`qdrant`)**: Standardised as the single supported **production** vector store provider. Selected via `aiqaos.memory.vector.provider=qdrant`.
- **In-Memory (`in-memory`)**: Standardised as the supported **dev/test** fallback provider. Selected via `aiqaos.memory.vector.provider=in-memory` or when the property is absent (`matchIfMissing = true`).
- **Experimental Stubs (`chroma`, `milvus`, `pgvector`)**: Marked as `@Deprecated` and gated behind an explicit experimental property (`aiqaos.memory.vector.experimental.enabled=true`).

### 2.2 Configuration Properties
| Property | Allowed Values | Default | Description |
|---|---|---|---|
| `aiqaos.memory.vector.provider` | `qdrant`, `in-memory` | `in-memory` | Selects active vector store provider |
| `aiqaos.memory.vector.experimental.enabled` | `true`, `false` | `false` | Enables deprecated experimental stubs |

---

## 3. Implementation Summary

1. **`InMemoryVectorStoreClient.java`**: Added `havingValue = "in-memory"` to `@ConditionalOnProperty`.
2. **`ChromaStoreClient.java`, `MilvusStoreClient.java`, `PgVectorStoreClient.java`**: Annotated with `@Deprecated` and updated `@ConditionalOnProperty` to require `aiqaos.memory.vector.experimental.enabled=true`.
3. **`MemoryConfig.java`**: Updated Javadoc documenting vector store provider hierarchy.
4. **`InMemoryVectorStoreClientTest.java`**: Added unit tests covering vector save, cosine similarity search, update, delete, and collection management operations.
5. **ADR-020**: Recorded decision in `AI-QA-OS-Architecture-Decisions.md`.

---

## 4. Verification

- `mvn test -pl ai-qa-os-memory` verified 4 unit tests passing cleanly.
- `mvn clean test` reactor build verified across all 22 modules.
