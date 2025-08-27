---
layout: single
title: "Snowflake Cortex + Semantic Search — HRMS RAG Pipeline (and Bedrock Comparison)"
subtitle: "Enterprise search over HRMS data using Cortex Search & AI SQL. Includes a free, open‑source practice track and datasets."
categories: [snowflake, rag, vector-search, hr, aws]
tags: [cortex, ai_sql, vector, embeddings, bedrock, knowledge-bases, hrms, pgvector, qdrant, ollama]
permalink: /snowflake/hrms-semantic-search-cortex/
header:
  overlay_color: "#0B7285"  # teal
  overlay_filter: 0
excerpt: >
  Build an enterprise semantic search for HRMS (policies, benefits, PTO, payroll) with Snowflake Cortex Search + AI SQL.
  Step-by-step SQL, sample HR datasets, and a comparable Bedrock setup. Includes a free/open‑source practice path
  (Postgres + pgvector + Ollama) so anyone can reproduce without a license.
---

> TL;DR — Use **Cortex Search** for low‑latency hybrid search (vector + keyword) on Snowflake.
> For full control, store **VECTOR** embeddings and query with similarity functions. Compare against **Amazon Bedrock
> Knowledge Bases**. A free practice track with **pgvector + Ollama** is included.

---

## 0) What you’ll build

**Goal:** “Google‑like” search across HRMS knowledge (policies, FAQs, payroll, leave rules, HR helpdesk tickets),
with grounded answers and filters (region, department, policy type).

**Two paths** (pick one):

1) **Snowflake‑native (fastest): Cortex Search Service** — One SQL command to build a hybrid index; query via REST/Python/SQL preview.  
2) **DIY inside Snowflake:** store a **VECTOR** column using **AI_EMBED / EMBED_TEXT_1024**, then query with **COSINE** similarity.

> Cortex Search is hybrid (vector + keyword) and handles embeddings, refresh, and reranking for you. See: Snowflake docs.

---

## 1) Sample HR datasets (free)

Use one of these public datasets to practice (no PII):

- **IBM HR Analytics — Employee Attrition & Performance** (tabular HR records; ~1.5k rows). Great for filters like department/role/tenure.  
  Source: Kaggle (search “IBM HR Analytics Employee Attrition & Performance”).

- **Human Resources Data Set** (multiple linked HR sheets: salary grid, recruiting cost, staff performance).  
  Source: Kaggle (search “Human Resources Data Set Huebner”).

- **US GSA: Human Capital Golden Data Test Set** (synthetic HR profiles & scenarios; CC0).  
  Source: data.gov (search “Human Capital Golden Data Test Set”).

- **SAP hr-request-data-set** (synthetic HR text requests—ideal for search & RAG).  
  Source: GitHub (search “SAP hr-request-data-set”).

> Tip: Mix **structured attributes** (department, location, grade) with **unstructured text** (policy docs, HR tickets).

**How to load quickly**  
In Snowsight: *Data » Databases » + » Load* and upload CSV/JSON; or create a STAGE and `COPY INTO` from cloud storage.

---

## 2) Cortex Search — quickest enterprise search

**Why:** Single statement builds an index that fuses **semantic vector** and **keyword** retrieval with **semantic reranking**. Snowflake handles embeddings and refresh.  
(Official docs summarize this and give an example.)

### 2.1 Create DB/WH and demo table
```sql
-- one-time setup
CREATE DATABASE IF NOT EXISTS HRMS_DB;
CREATE OR REPLACE WAREHOUSE HRMS_WH WITH WAREHOUSE_SIZE='X-SMALL';
USE DATABASE HRMS_DB;
USE WAREHOUSE HRMS_WH;
CREATE OR REPLACE SCHEMA SEARCH;

-- unstructured HR corpus: text + facets
CREATE OR REPLACE TABLE SEARCH.HR_KB (
  id STRING,
  text VARCHAR,
  country STRING,
  department STRING,
  policy_type STRING,
  updated_at TIMESTAMP_NTZ
);

-- example rows (replace with your upload)
INSERT INTO SEARCH.HR_KB (id, text, country, department, policy_type, updated_at) VALUES
('kb-001', 'How to request PTO in India. Submit leave in HRMS > Leave > New. Manager approval required.', 'IN', 'All', 'PTO', CURRENT_TIMESTAMP()),
('kb-002', 'Salary revisions occur annually in April. Performance bands A/B/C map to 12/8/5% hikes.', 'IN', 'Compensation', 'Compensation', CURRENT_TIMESTAMP()),
('kb-003', 'US parental leave: 12 weeks paid. Contact benefits@company.com', 'US', 'Benefits', 'Parental', CURRENT_TIMESTAMP());
```

### 2.2 Enable change tracking (required for search services)
```sql
ALTER TABLE SEARCH.HR_KB SET CHANGE_TRACKING = TRUE;
```

### 2.3 Create the Cortex Search Service
```sql
-- Pick an embedding model; Snowflake hosts Arctic Embed variants.
-- Target lag defines refresh cadence from base tables.
CREATE OR REPLACE CORTEX SEARCH SERVICE HR_KB_SEARCH
  ON text
  ATTRIBUTES country, department, policy_type, updated_at
  WAREHOUSE = HRMS_WH
  TARGET_LAG = '1 day'
  EMBEDDING_MODEL = 'snowflake-arctic-embed-l-v2.0'  -- multilingual option available
AS (
  SELECT id, text, country, department, policy_type, updated_at
  FROM SEARCH.HR_KB
);
```

### 2.4 Query it (3 ways)

**(a) SQL — preview only (debugging):**
```sql
SELECT PARSE_JSON(
  SNOWFLAKE.CORTEX.SEARCH_PREVIEW(
    'HR_KB_SEARCH',
    '{
      "query": "How do I apply for parental leave?",
      "columns": ["id","country","department","policy_type","updated_at"],
      "filter": { "@eq": { "country": "US" } },
      "limit": 5
    }'
  )
)['results'] AS results;
```

**(b) Python API:**
```python
from snowflake.core import Root
from snowflake.snowpark import Session

session = Session.builder.configs({...}).create()
root = Root(session)
svc = root.databases["HRMS_DB"].schemas["SEARCH"].cortex_search_services["HR_KB_SEARCH"]
resp = svc.search(query="PTO carry forward India", columns=["id","country","policy_type"], limit=5)
print(resp.to_json())
```

**(c) REST (for apps):**
```bash
curl -s -H "Authorization: Bearer $PAT"   -H "Content-Type: application/json"   https://<ACCOUNT_URL>/api/v2/databases/HRMS_DB/schemas/SEARCH/cortex-search-services/HR_KB_SEARCH:query   -d '{"query":"PTO India carry forward","columns":["id","country","policy_type"],"limit":5}'
```

> Notes: You can **filter** on `ATTRIBUTES` (e.g., `country="IN"`), and customize ranking via numeric boosts and time decays.

---

## 3) DIY inside Snowflake (VECTOR + AI_EMBED + similarity)

If you prefer full control (or just to learn the internals), store embeddings and query with similarity metrics.

### 3.1 Table with a VECTOR column
```sql
CREATE OR REPLACE TABLE SEARCH.HR_KB_VECTORS (
  id STRING,
  text VARCHAR,
  country STRING,
  department STRING,
  policy_type STRING,
  emb VECTOR(FLOAT, 1024)  -- 1024‑dimensional
);
```

### 3.2 Populate embeddings
```sql
-- AI_EMBED is the newest API; EMBED_TEXT_1024 also works.
INSERT INTO SEARCH.HR_KB_VECTORS
SELECT
  id, text, country, department, policy_type,
  SNOWFLAKE.CORTEX.EMBED_TEXT_1024('snowflake-arctic-embed-l-v2.0', text) AS emb
FROM SEARCH.HR_KB;
```

### 3.3 Query with cosine similarity
```sql
WITH q AS (
  SELECT SNOWFLAKE.CORTEX.EMBED_TEXT_1024('snowflake-arctic-embed-l-v2.0', 'US paternity leave policy') AS qvec
)
SELECT id, country, department, policy_type,
       VECTOR_COSINE_SIMILARITY(emb, (SELECT qvec FROM q)) AS score
FROM SEARCH.HR_KB_VECTORS
ORDER BY score DESC
LIMIT 5;
```

> Advanced: You can store multiple embedding models, chunking strategies, and apply **reranking** with `AI_COMPLETE` on the top‑k passages.

---

## 4) Generate grounded answers with AI SQL

Use **AI_COMPLETE** (the updated `COMPLETE`) to compose an answer that cites the retrieved snippets.

```sql
WITH topk AS (
  SELECT ARRAY_AGG(text) AS ctx
  FROM (
    WITH q AS (
      SELECT SNOWFLAKE.CORTEX.EMBED_TEXT_1024('snowflake-arctic-embed-l-v2.0', 'How to request PTO in India?') AS qvec
    )
    SELECT text
    FROM SEARCH.HR_KB_VECTORS
    ORDER BY VECTOR_COSINE_SIMILARITY(emb, (SELECT qvec FROM q)) DESC
    LIMIT 5
  )
)
SELECT AI_COMPLETE(
  'snowflake-arctic-instruct',  -- pick a supported chat model
  OBJECT_CONSTRUCT(
    'prompt', 'Answer using ONLY the provided HR policy snippets. If unknown, say so.',
    'context', (SELECT ctx FROM topk)
  ),
  {'response_format': OBJECT_CONSTRUCT('type','json')}
) AS answer_json;
```

---

## 5) Bedrock comparison (how to run a fair test)

If you want to compare **latency & cost** against **Amazon Bedrock Knowledge Bases**, run the same corpus and prompts.

**Bedrock setup outline:**  
- Create a Knowledge Base and choose a vector store (OpenSearch Serverless quick create, or your own Aurora PostgreSQL/pgvector, Redis Enterprise Cloud, Pinecone).  
- Choose an embedding model (e.g., Titan Embeddings).  
- Ingest the same documents and run the same top‑k and answer prompts via `RetrieveAndGenerate` or a simple RAG lambda.

**Measure:**  
- **Serving latency:** p50/p95 for 100/1,000 queries with warm caches.  
- **Ingestion time:** time to first query after corpus upload.  
- **Cost drivers:** Snowflake: warehouse refresh credits, per‑token embedding credits, serving GB‑mo for the search service. Bedrock: embedding calls, vector DB storage/IO, LLM inference.  
- **Quality:** manual relevance judgments on a 100‑question eval set (grade 0–3).

> Tip: keep **top‑k, chunk size, and models** as equal as possible. Run each stack in the same AWS region if you can.

---

## 6) Free & open‑source practice track (mirrors this tutorial)

If you don’t have Snowflake/AWS credits, reproduce the pipeline locally:

**Stack:** PostgreSQL + `pgvector`, Qdrant (alternative), and **Ollama** (Llama 3, Phi‑3, etc.).

**Docker Compose (quick start):**
```yaml
version: "3.8"
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: postgres
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]
  pgvector:
    image: ankane/pgvector
    depends_on: [db]
  qdrant:
    image: qdrant/qdrant:latest
    ports: ["6333:6333"]
  ollama:
    image: ollama/ollama:latest
    ports: ["11434:11434"]
    volumes: ["ollama:/root/.ollama"]
volumes: { pgdata: {}, ollama: {} }
```

**Practice steps:**
1. Install `pgvector` extension; create a table with `vector(768)`; use `sentence-transformers` (e.g., `all-MiniLM-L6-v2`) for free embeddings.  
2. Ingest the same HR corpus (CSV or the SAP hr‑request dataset).  
3. Search with cosine similarity; answer with `ollama run llama3:instruct` using the top‑k passages.  
4. Optional UI: build a small FastAPI/Streamlit chat that hits Postgres/Qdrant and Ollama.

---

## 7) Production checklist

- Sensitive data: ensure HR PII is excluded or masked; use row‑level security.  
- Refresh cadence: set `TARGET_LAG` appropriately (daily for policy docs, hourly for tickets).  
- Filters: country/department/policy type as attributes.  
- Observability: log query text (hashed), latency, top‑k sources, answer length, refusal rate.  
- Guardrails: prompt templates that forbid answering without citations; deflection to HR if confidence < threshold.  
- Cost control: smaller embedding models for frequent refresh; bigger LLM only for answer generation; cache embeddings; batch refresh windows.

---

## 8) Getting access (free paths)

- **Snowflake trial:** 30‑day trial with free credits. Create an account and select region/edition during signup.  
- **Datasets:** Kaggle “IBM HR Analytics Attrition”; “Human Resources Data Set”; GSA “Human Capital Golden Data Test Set”; GitHub “SAP hr‑request‑data‑set”.  
- **Open‑source:** Postgres + `pgvector`, Qdrant, or Weaviate; embeddings via `sentence-transformers`; LLM via `Ollama`.

---

## Appendix — handy references

- Cortex Search overview & example: build a service with `CREATE CORTEX SEARCH SERVICE` and query via REST/Python/`SEARCH_PREVIEW`.  
- Vector data type & similarity: `VECTOR(FLOAT, 1024)` with cosine/inner‑product/L1/L2 functions.  
- AI SQL functions: `AI_COMPLETE` (new), `EMBED_TEXT_1024` / `AI_EMBED` for embeddings.  
- Bedrock Knowledge Bases: supports OpenSearch Serverless, Pinecone, Aurora pgvector, Redis, etc.

```ascii
Architecture (Snowflake path)
User → Query → Cortex Search (hybrid retrieval) → top‑k text
                      ↘ attributes filter (country/department)
top‑k → AI_COMPLETE → grounded answer (+ citations)
```
