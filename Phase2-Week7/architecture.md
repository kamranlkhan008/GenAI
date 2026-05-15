# Resume Intelligence Platform – System Design (RAG + Node.js)

## 1. Solution Overview

### Objective

Design a production-ready resume discovery platform that prioritizes relevance and matching accuracy over ultra-low latency. The platform should support semantic resume retrieval, keyword-based filtering, LLM-driven ranking, and optional candidate summaries.

The application will initially run as a single Node.js + Express deployment backed by MongoDB Atlas.

Target response time is approximately 3–5 seconds for complete search execution.

---

## 2. Core Technology Choices

### Backend Stack

* Node.js
* Express.js
* TypeScript
* MongoDB Atlas

### AI / LLM Stack

#### Embeddings

* Provider: Mistral
* Model: `mistral-embed`
* Embedding size: 1024 dimensions
* Query embeddings generated dynamically at runtime

#### LLM Provider

* Provider: Groq
* Model: `meta-llama/llama-4-scout-17b-16e-instruct`

All provider keys, model names, and runtime settings must be configurable through environment variables.

---

# 3. Design Priorities

### Performance Philosophy

The platform is not optimized for massive concurrency initially.

Main focus:

* Better ranking quality
* More accurate candidate selection
* Reliable semantic understanding
* Explainable search flow

Acceptable characteristics:

* Moderate throughput
* Vertical scaling
* Monolithic deployment
* Synchronous processing pipeline

---

# 4. Search Strategy

The platform combines:

1. Traditional BM25 keyword search
2. Vector similarity search
3. LLM-based re-ranking

The final ranking decision is controlled entirely by the LLM layer.

---

# 5. Retrieval Pipeline

## End-to-End Flow

1. Validate request
2. Generate embedding for incoming query
3. Execute BM25 search
4. Execute vector similarity search
5. Merge and deduplicate candidates
6. Re-rank top candidates using LLM
7. Optionally generate summaries
8. Return ordered response

Both BM25 and vector retrieval should run independently.

No direct score blending is required.

---

# 6. High-Level Architecture

## API Layer

Responsible for:

* Route handling
* Validation
* Versioning
* Request size limits
* Error responses

Example:

* `/v1/search`
* `/v1/health`

---

## Service Layer

### SearchService

Coordinates all retrieval operations.

Responsibilities:

* BM25 execution
* Vector search execution
* Hybrid orchestration
* Candidate merging
* LLM re-ranking
* Summarization orchestration

### EmbeddingService

Encapsulates embedding generation.

Responsibilities:

* Calling Mistral embedding APIs
* Handling configurable embedding models
* Query embedding generation

### LLMService

Handles all LLM interactions.

Responsibilities:

* Candidate re-ranking
* Candidate fit summaries
* Resume metadata extraction

### LoggingService

Handles:

* Structured logging
* Timing metrics
* Request tracing
* Error metadata

### ResumeRepository

MongoDB access abstraction.

Responsibilities:

* CRUD operations
* BM25 queries
* Vector queries
* Search pipelines

---

# 7. Suggested Project Structure

```text
src/
 ├── app.ts
 ├── server.ts
 ├── config/
 ├── routes/
 ├── services/
 ├── repositories/
 ├── middleware/
 ├── utils/
 ├── types/
 └── constants/
```

### Important Services

```text
services/
 ├── SearchService.ts
 ├── EmbeddingService.ts
 ├── LLMService.ts
```

### Repository Layer

```text
repositories/
 └── ResumeRepository.ts
```

### Middleware

```text
middleware/
 ├── requestLogger.ts
 ├── errorHandler.ts
 ├── requestId.ts
 └── payloadLimiter.ts
```

---

# 8. MongoDB Model

## Collection

`resumes`

## Example Shape

```json
{
  "_id": "ObjectId",
  "name": "ASHWIN P",
  "email": "ashwinp@gmail.com",
  "location": "Chennai, India",
  "company": "TCS",
  "role": "QA Engineer",
  "education": "B.E COMPUTER SCIENCE ENGINEERING",
  "totalExperience": 1.3,
  "relevantExperience": 1.3,
  "skills": [
    "Selenium",
    "TestNG",
    "Java",
    "SQL",
    "Jenkins"
  ],
  "rawText": "Full parsed resume text",
  "embedding": []
}
```

---

# 9. Search Capabilities

## BM25 Search

Search fields:

* Resume text
* Skills
* Job titles
* Experience summary

Implemented through MongoDB Atlas Search.

---

## Vector Search

Uses:

* MongoDB Atlas Vector Search
* Cosine similarity
* ANN retrieval

Optional exact re-scoring can be applied on the top candidate set.

---

## Hybrid Retrieval

Both search methods execute in parallel.

The system returns:

* BM25 candidates
* Vector candidates

This stage intentionally avoids score normalization or weighted merging.

---

# 10. LLM Re-Ranking

The LLM acts as the final ranking authority.

## Re-ranking Inputs

* User query
* Candidate snippets
* Candidate metadata

## Re-ranking Outputs

* Sorted candidates
* Relevance ordering
* Optional fit reasoning

Top 8–10 candidates should be re-ranked by default.

This value should remain configurable.

---

# 11. Candidate Summaries

The platform may optionally generate recruiter-friendly summaries.

### Summary Modes

* Short
* Detailed

### Supported Controls

* Max token count
* Summary style
* Candidate subset

---

# 12. API Endpoints

## Health APIs

### `GET /v1/health`

Returns:

* Service status
* App version
* Uptime
* Configuration validation

---

### `GET /v1/health/db`

Checks:

* MongoDB connectivity
* Ping latency

---

## Embedding API

### `POST /v1/embeddings`

Example:

```json
{
  "model": "mistral-embed",
  "input": "backend engineer with nodejs experience"
}
```

Response:

```json
{
  "model": "mistral-embed",
  "embedding": []
}
```

---

## BM25 Search API

### `POST /v1/search/bm25`

Example:

```json
{
  "query": "senior backend engineer",
  "topK": 20,
  "filters": {
    "minYearsExperience": 5
  }
}
```

---

## Vector Search API

### `POST /v1/search/vector`

Processing:

1. Generate embedding
2. Run vector search
3. Return top matching resumes

---

## Hybrid Search API

### `POST /v1/search/hybrid`

Returns:

* BM25 ranking
* Vector ranking

Primarily useful for debugging and evaluation.

---

## Re-Ranking API

### `POST /v1/search/rerank`

Example:

```json
{
  "query": "senior node.js engineer",
  "candidates": [],
  "topK": 10
}
```

The endpoint returns candidates reordered by LLM relevance.

---

## Summarization API

### `POST /v1/search/summarize`

Example:

```json
{
  "query": "job description",
  "candidate": {},
  "style": "short",
  "maxTokens": 300
}
```

---

## Full Search API

### `POST /v1/search`

Executes the entire search pipeline synchronously.

Pipeline:

1. Validation
2. Query embedding generation
3. BM25 search
4. Vector retrieval
5. Candidate merge
6. Deduplication
7. LLM re-ranking
8. Optional summaries
9. Final response generation

---

# 13. Fault Tolerance Strategy

The platform should degrade gracefully whenever one retrieval component fails.

## Failure Handling

### If Vector Search Fails

Fallback:

* Return BM25-only results
* Include fallback metadata in response

### If BM25 Fails

Fallback:

* Return vector-only results

### If Re-ranking Fails

Fallback:

* Use merged retrieval ordering
* Prefer BM25 order first

### If Summarization Fails

Fallback:

* Return candidate list without summaries

---

# 14. Logging Requirements

Every request should produce structured JSON logs.

## Required Metadata

```json
{
  "requestId": "uuid",
  "endpoint": "/v1/search",
  "method": "POST",
  "statusCode": 200,
  "durationMs": 1234,
  "componentTimings": {
    "embeddingMs": 200,
    "bm25Ms": 300,
    "vectorMs": 250,
    "rerankMs": 400,
    "summarizeMs": 0
  }
}
```

Additional requirements:

* Request tracing
* Centralized error logging
* Payload size enforcement
* HTTP 413 support for oversized requests

---

# 15. Implementation Roadmap

## Phase 1

* Project initialization
* Express setup
* MongoDB connection
* Health APIs

---

## Phase 2

* EmbeddingService
* `/v1/embeddings`

---

## Phase 3

* BM25 Atlas Search setup
* `/v1/search/bm25`

---

## Phase 4

* Vector index configuration
* Vector retrieval implementation
* `/v1/search/vector`

---

## Phase 5

* Hybrid retrieval orchestration
* `/v1/search/hybrid`

---

## Phase 6

* LLM re-ranking implementation
* `/v1/search/rerank`

---

## Phase 7

* Candidate summaries
* `/v1/search/summarize`

---

## Phase 8

* Full synchronous search pipeline
* Retry and fallback logic
* Logging improvements

---

## Phase 9

Testing coverage:

* Unit tests
* Integration tests
* Endpoint validation
* Failure scenario testing

---

# 16. Additional Future Improvements

Potential enhancements:

* Containerization
* Distributed retrieval workers
* Resume ingestion pipeline
* Cached embeddings
* Async summarization queues
* Redis caching
* Evaluation datasets
* Search analytics dashboard
* Prompt versioning
* Multi-tenant support

---

# 17. Final Notes

The architecture intentionally favors:

* Clean separation of concerns
* Maintainability
* Incremental implementation
* High-quality ranking
* Strong debugging visibility

The first implementation should remain simple, synchronous, and monolithic while leaving room for future scalability improvements.
