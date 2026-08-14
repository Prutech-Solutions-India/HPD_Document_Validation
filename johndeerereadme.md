# Intelligent Telemetry & Knowledge Platform — Architecture

**Document type:** Solution Architecture (condensed)
**Audience:** Solution Architects / .NET Architects

---

## The Whole Flow in 5 Lines

1. **Data Lake → Event → Queue → Worker:** Whenever new data arrives, an event triggers scalable Worker Services to validate, clean, transform, and process it.
2. **Worker → SQL / Vector DB:** Structured data goes to SQL/Data Warehouse; unstructured data is chunked, embedded with metadata, and stored in the Vector DB.
3. **UI → API Gateway → Auth → AI Agent:** User questions reach the Agent/Orchestrator, which identifies the intent and required data source.
4. **Agent → SQL / RAG / ML:** Structured queries use controlled SQL tools, document questions use RAG + Vector DB, and predictions use the ML service; mixed queries can use multiple tools.
5. **Results → LLM → UI:** Retrieved results are combined and passed to the LLM to structure, summarize, and generate a grounded response for the user.

> **Two invariants:** the Agent decides _which tool_ to call but never _what data_ the user may see; and the LLM gets typed, authorized tools — never a database connection.

---

## Index

| #   | Section                                               | Stage          |
| --- | ----------------------------------------------------- | -------------- |
| 1   | [Ingestion Workflow](#1-ingestion-workflow)           | Ingestion time |
| 2   | [Worker Responsibilities](#2-worker-responsibilities) | Ingestion time |
| 3   | [Live Request Flow](#3-live-request-flow)             | Query time     |
| 4   | [RAG Retrieval](#4-rag-retrieval)                     | Query time     |

Sections 1–2 run **once per file, asynchronously**. Sections 3–4 run **once per question, synchronously**, and read only what 1–2 have already prepared.

---

## 1. Ingestion Workflow

What happens when new data arrives in the Data Lake. Work is triggered by an **event**, buffered by a **queue**, and executed by **workers** — never by polling the lake.

```mermaid
flowchart TB
    A["1 - New data lands in Data Lake"] --> B["2 - Emit NewDataAvailable event"]
    B --> C["3 - Publish to Event Bus / Message Broker"]
    C --> D["4 - Enqueue work item on processing queue"]
    D --> E["5 - Worker Service consumes message"]
    E --> F["6 - Validate<br/>schema, required fields, ranges, checksum"]
    F -- "invalid" --> QUAR["Quarantine zone + DataRejected event"]
    F -- "valid" --> G["7 - Clean & normalize<br/>units, encodings, nulls, dedupe"]
    G --> H["8 - Transform & enrich<br/>joins, lookups, region, business unit"]
    H --> I{"9 - Detect data type"}
    I -- "Structured" --> J["ETL / Dimensional modelling"]
    I -- "Unstructured" --> K["Document processing branch"]
    J --> L[("SQL Database /<br/>Data Warehouse")]
    K --> M[("Vector Database")]
    L --> N["Emit IngestionCompleted event"]
    M --> N
```

The original document always stays in the Data Lake. The SQL Warehouse and Vector DB are **derived indexes**, so a bad chunking or embedding decision is fixed by replaying — not by a data-loss incident.

---

## 2. Worker Responsibilities

What a single worker does with one message, including how it fails safely.

```mermaid
flowchart TB
    A["Consume queue message"] --> B["Read source data from lake<br/>using the URI in the message"]
    B --> C["Validate schema & business rules"]
    C --> D["Transform & normalize"]
    D --> E{"Data type?"}
    E -- "Structured" --> F["Map to warehouse model"]
    E -- "Unstructured" --> G["Extract text -> chunk"]
    G --> H["Generate embeddings<br/>batched calls"]
    H --> I["Generate & attach metadata"]
    F --> J["Persist structured data<br/>idempotent upsert"]
    I --> K["Persist vector data<br/>deterministic chunk key"]
    J --> L["Emit IngestionCompleted event"]
    K --> L
    L --> M["Emit logs, metrics, traces"]
    C -- "invalid / poison" --> DLQ[("Dead Letter Queue")]
    D -- "transient error" --> RTY["Retry with exponential backoff"]
    RTY -- "attempts exhausted" --> DLQ
```

Key points:

- **Idempotent writes.** Deterministic keys and upserts make at-least-once delivery harmless — duplicates are expected, not prevented.
- **Retry then DLQ.** Transient failures back off and retry; poison messages are quarantined so one bad file cannot block the queue.
- **Stateless and horizontally scaled**, with queue depth and backlog age as the autoscaling signal.

---

## 3. Live Request Flow

What happens per user question. Routing is decided **server-side by the Agent**, never in the browser.

```mermaid
flowchart TB
    U["User"] --> UI["Web UI"]
    UI --> GW["API Gateway<br/>TLS, rate limit, WAF, routing"]
    GW --> AU["Authentication<br/>validate JWT / OIDC token"]
    AU --> AZ["Authorization<br/>roles, tenant, allowed regions"]
    AZ --> API["Backend API<br/>builds trusted SecurityContext"]
    API --> CACHE{"Cache hit?"}
    CACHE -- "yes" --> RESP["Return cached response"]
    CACHE -- "no" --> AG["AI Agent / Orchestrator"]
    AG --> IR{"Intent Router<br/>classify & plan"}
    IR -- "structured" --> T1["QueryStructuredData"]
    IR -- "knowledge" --> T2["SearchKnowledgeBase"]
    IR -- "prediction" --> T3["PredictFailure"]
    IR -- "analytics" --> T4["RunAnalytics"]
    IR -- "mixed" --> T5["Plan: multiple tools"]
    T1 --> DS1[("SQL / Warehouse")]
    T2 --> DS2[("Vector DB")]
    T3 --> DS3["ML Inference Service"]
    T4 --> DS4[("Analytics / Aggregates")]
    T5 --> T1
    T5 --> T2
    DS1 --> AGG["Aggregate tool results<br/>+ provenance"]
    DS2 --> AGG
    DS3 --> AGG
    DS4 --> AGG
    AGG --> LLM["LLM<br/>structure, summarize, explain<br/>grounded in supplied context only"]
    LLM --> API2["Backend API<br/>validate, redact, attach citations"]
    API2 --> UI2["Web UI<br/>answer + sources + confidence"]
    RESP --> UI2
```

Exact metrics come from **SQL**; semantic content comes from **RAG**; predictions come from the **ML service**. The LLM contributes language, never a number of its own. Anything too large for a synchronous response becomes an async job — `202 Accepted`, queue, worker, notification.

---

## 4. RAG Retrieval

How a document question is answered. The mirror of ingestion: indexing ran once per document, retrieval runs once per question.

```mermaid
flowchart TB
    Q["User question"] --> AG["Agent"]
    AG --> RS["RAG Service"]
    RS --> QE["Embed the query<br/>same model & version as indexing"]
    QE --> PF["Pre-filter by metadata<br/>TenantId, AllowedRegions,<br/>DocumentType, Version, date"]
    PF --> VS["Vector similarity search<br/>+ optional keyword hybrid"]
    VS --> TK["Top-K candidate chunks"]
    TK --> RR["Re-rank & de-duplicate<br/>drop low-relevance chunks"]
    RR --> CTX["Assemble bounded context<br/>with citations"]
    CTX --> LLM["LLM"]
    LLM --> A["Grounded answer<br/>with source references"]
    RR -- "nothing relevant found" --> NF["Return 'no supporting<br/>information found'<br/>instead of guessing"]
```

Two details that are easy to get wrong:

- The query must be embedded with the **same model and version used for indexing**, so a model upgrade forces a re-index.
- **Metadata filtering is applied before ranking** and is derived from the server-side `SecurityContext` — never from the user's text or the Agent's output. Filtering afterwards would let unauthorised content reach the LLM.
