# Multi-Document Processing Pipeline — Detailed Implementation Plan

> **Elevator pitch:** An LLM-powered ETL pipeline that ingests mixed document types (invoices, contracts, emails, receipts), classifies each document, extracts structured data into Pydantic schemas with field validation, routes validated records to downstream systems, and escalates low-confidence or invalid extractions to a human review queue. This is the bridge project — it takes the data engineer's existing ETL/ELT muscle and replaces the brittle regex/OCR extraction layer with an LLM, while keeping validation, orchestration, and routing firmly in traditional data engineering territory.

---

## Table of Contents

1. [Project Goals & Non-Goals](#1-project-goals--non-goals)
2. [Architecture Overview](#2-architecture-overview)
3. [Tech Stack](#3-tech-stack)
4. [Database Schema](#4-database-schema)
5. [Project Structure](#5-project-structure)
6. [Implementation Phases](#6-implementation-phases)
7. [Component Specifications](#7-component-specifications)
8. [Pydantic Schemas (Per Document Type)](#8-pydantic-schemas-per-document-type)
9. [Validation Rules](#9-validation-rules)
10. [Human Review Queue Design](#10-human-review-queue-design)
11. [Orchestration DAG](#11-orchestration-dag)
12. [Security & Safety Considerations](#12-security--safety-considerations)
13. [API Specification](#13-api-specification)
14. [Testing Strategy](#14-testing-strategy)
15. [Deployment](#15-deployment)
16. [Roadmap & Milestones](#16-roadmap--milestones)
17. [Risk Register](#17-risk-register)
18. [Appendix A: Quick Start](#appendix-a-quick-start)
19. [Appendix B: Key Design Decisions](#appendix-b-key-design-decisions)
20. [Appendix C: Traditional ETL vs LLM ETL](#appendix-c-traditional-etl-vs-llm-etl)

---

## 1. Project Goals & Non-Goals

### Goals

| # | Goal | Success Metric |
|---|------|----------------|
| G1 | **Multi-format ingestion** — accept PDF, DOCX, TXT, EML, and image files (with OCR fallback) | All 5 formats parse to text; images via Tesseract OCR |
| G2 | **LLM classification** — classify each document as invoice / contract / email / receipt / other with confidence score | Classification accuracy >= 90% on a 50-doc golden test set |
| G3 | **Structured extraction** — per document type, extract fields into typed Pydantic v2 models with structured output | Extraction field accuracy >= 85% on golden test set |
| G4 | **Validation layer** — Pydantic validation + custom business rules catch extraction errors before they reach downstream | 100% of records pass all validation rules before routing; invalid records never reach the structured data store |
| G5 | **Error handling with retry** — low-confidence classifications and validation failures trigger retry with error feedback, then human review | Retry reduces human review queue by >= 40% vs no-retry baseline |
| G6 | **Routing** — validated records -> PostgreSQL (structured data) + document store (original files) + audit log | Every processed document has a row in `documents`, `extraction_records`, and `audit_logs` |
| G7 | **Orchestration** — Airflow DAG for batch processing with retries, SLA monitoring, and observability | DAG runs end-to-end on a scheduled batch; SLA miss alerts fire on timeout |
| G8 | **Human-in-the-loop** — Streamlit review queue for low-confidence extractions | Reviewer can view, edit, approve, or reject queued items; approved items route to downstream |
| G9 | **Production-deployable** — containerized, documented, testable end-to-end | `docker compose up` starts the full system (Airflow, Postgres, API, Streamlit); health checks pass |

### Non-Goals (explicitly out of scope)

- Real-time / streaming ingestion (v1 is batch-oriented via Airflow DAG)
- Fine-tuning classification or extraction models (v1 uses prompt engineering + structured output)
- Multi-language document support (v1 is English-only; German is a v2 goal)
- Document versioning / diffing (latest ingest overwrites previous)
- Web-based upload UI (v1 uses API + file watcher; Streamlit is review-only, not upload)
- OCR on handwritten documents (Tesseract handles printed text only)
- Cross-system deduplication across external sources (dedup is within-pipeline only via file hash)
- Automated human-review resolution (humans make the final call; no auto-approve)

---

## 2. Architecture Overview

```
         +-----------------------------------------------------------------+
         |                    INGESTION SOURCES                            |
         |  +------------+    +------------+    +----------------+         |
         |  | File Watch |    | REST API   |    | Airflow Batch  |         |
         |  | (drop dir) |    | POST/ingest|    | (S3/dir scan)  |         |
         |  +-----+------+    +-----+------+    +-------+--------+         |
         +--------+-----------------+-----------------+--------------------+
                  |                 |                  |
         +--------v-----------------v------------------v-------------------+
         |              INGESTION LAYER                                |
         |  +----------+  +-----------+  +----------------------+        |
         |  | Parser   |  | OCR       |  | Text Normalizer      |        |
         |  | PDF/DOCX |  | Tesseract |  | (whitespace, encode) |        |
         |  | /TXT/EML |  | (images)  |  |                      |        |
         |  +----+-----+  +-----+-----+  +----------+-----------+        |
         |       +--------------+------------------+                    |
         |                      | raw_text                             |
         +----------------------+---------------------------------------+
                                |
         +----------------------v---------------------------------------+
         |              CLASSIFICATION LAYER                           |
         |  +----------------------------------------------+           |
         |  | LLM Classifier                               |           |
         |  | Input: raw_text (truncated to context win)   |           |
         |  | Output: {doc_type, confidence} structured out|           |
         |  +----------------------+-----------------------+           |
         |                         |                                 |
         |          +--------------v--------------+                  |
         |          | Confidence Gate              |                  |
         |          | confidence >= THRESHOLD?     |                  |
         |          |   YES -> proceed to extract  |                  |
         |          |   NO  -> human review queue  |                  |
         |          +------+--------------+--------+                  |
         +-----------------+--------------+---------------------------+
                           |              |
         +-----------------v--------------+                           |
         |       EXTRACTION LAYER         |                           |
         |  +---------------------------+ |                           |
         |  | Schema Registry           | |                           |
         |  | Invoice/Contract/Email/   | |                           |
         |  | Receipt/Other schemas     | |                           |
         |  +-------------+-------------+ |                           |
         |  +-------------v-------------+ |                           |
         |  | LLM Structured Extract     | |                           |
         |  | (Pydantic response_format) | |                           |
         |  +-------------+-------------+ |                           |
         +----------------+--------------+                           |
                          |                                          |
         +----------------v--------------+                           |
         |       VALIDATION LAYER        |                           |
         |  +---------------------------+|                           |
         |  | Pydantic Validation       ||                           |
         |  | (type constraints, req    ||                           |
         |  |  fields, field validators)||                           |
         |  +-------------+-------------+|                           |
         |  +-------------v-------------+|                           |
         |  | Business Rule Validators  ||                           |
         |  | (invoice total = sum,     ||                           |
         |  |  date plausibility, tax   ||                           |
         |  |  rate ranges, etc.)       ||                           |
         |  +-------------+-------------+|                           |
         |          +-----v------+       |                           |
         |          | Valid?     |       |                           |
         |          | YES -> route|      |                           |
         |          | NO  -> retry|      |                           |
         |          +-----+------+       |                           |
         +----------------+--------------+                           |
                          |                                          |
         +----------------v--------------+                           |
         |    RETRY / ERROR HANDLING     |                           |
         |  +---------------------------+|                           |
         |  | Retry with Error Feedback ||                           |
         |  | (append validation errors ||                           |
         |  |  to prompt, re-extract)   ||                           |
         |  |  max_retries = 2          ||                           |
         |  +-------------+-------------+|                           |
         |          +-----v------+       |                           |
         |          | Still bad? |       |                           |
         |          | YES -> human|------+---------------------------+--+
         |          | NO  -> route|      |                              |
         |          +-----+------+       |                              |
         +----------------+--------------+                              |
                          |                                             |
         +----------------v--------------+                              |
         |       ROUTING LAYER           |                              |
         |  +----------+ +------------+  |                              |
         |  |PostgreSQL| | Document   |  |                              |
         |  |(structured| | Store      |  |                              |
         |  | data)    | | (original  |  |                              |
         |  |          | |  files)    |  |                              |
         |  +----+-----+ +------+-----+  |                              |
         |       +----+--------+         |                              |
         |       | Audit Log   |         |                              |
         |       | (every step)|         |                              |
         |       +-------------+         |                              |
         +----------------------+--------+                              |
                                |                                     |
         +----------------------v----------------------------------------+
         |           HUMAN REVIEW QUEUE                                  |
         |  +----------------------------------------+                  |
         |  | Streamlit Review UI                    |                  |
         |  | - View original doc + extracted data   |                  |
         |  | - Edit fields manually                 |                  |
         |  | - Approve -> route to downstream       |                  |
         |  | - Reject -> log + archive              |                  |
         |  +----------------------------------------+                  |
         +--------------------------------------------------------------+

         +--------------------------------------------------------------+
         |              ORCHESTRATION (Airflow)                         |
         |  +--------------------------------------------------------+  |
         |  | DAG: document_processing_pipeline                      |  |
         |  | scan -> ingest -> classify -> extract -> validate ->   |  |
         |  | retry_if_needed -> route -> audit                      |  |
         |  | Retries: per-task (Airflow) + per-doc (app-level)      |  |
         |  | SLA: batch must complete within 60 min                 |  |
         |  +--------------------------------------------------------+  |
         +--------------------------------------------------------------+
```

### End-to-End Document Flow

```
1. Document arrives (file watcher / API / Airflow batch scan)
2. Ingestion: parse to raw_text (PDF->text, DOCX->text, EML->text, image->OCR->text)
3. Store original file in document store (file_hash for dedup)
4. Classification: LLM classifies doc_type with confidence score
   - confidence >= 0.85 -> proceed to extraction
   - confidence <  0.85 -> enqueue to human review queue, stop
5. Extraction: LLM extracts structured data into doc_type-specific Pydantic schema
   (uses OpenAI structured output / response_format with Pydantic schema)
6. Validation:
   a. Pydantic validation (type constraints, required fields, field validators)
   b. Business rule validation (invoice total = sum of line items, date plausibility, etc.)
   - all pass -> proceed to routing
   - any fail -> retry extraction with error feedback appended to prompt (max 2 retries)
   - still failing after retries -> enqueue to human review queue, stop
7. Routing:
   a. Insert structured record into extraction_records table (typed JSONB per doc_type)
   b. Original file already in document store
   c. Audit log entry for the full pipeline run
8. Human Review (async, via Streamlit):
   - Reviewer sees: original document, extracted data, validation errors, confidence
   - Reviewer edits fields -> approves -> record routes to downstream
   - Reviewer rejects -> logged + archived
```

---

## 3. Tech Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **Language** | Python 3.11+ | Ecosystem maturity for AI/ML, Pydantic v2 native, type hints |
| **Web Framework** | FastAPI | Async, auto OpenAPI docs, Pydantic validation, file upload support |
| **LLM Framework** | Raw OpenAI SDK (structured output) | `client.beta.chat.completions.parse()` with Pydantic `response_format` — no LangChain abstraction needed for structured extraction; keeps the LLM call transparent and debuggable. LangChain is an alternative if chain composition is needed later. |
| **LLM** | `gpt-4o-mini` (OpenAI) | Fast, cheap ($0.15/1M input tokens), strong structured output adherence. Alternative: Llama 3.1 70B via Ollama for on-prem / GDPR-sensitive deployments. |
| **Schema Validation** | Pydantic v2 | Rust-backed validation, `model_validate`, `field_validator`, `model_dump_json`. The schema IS the contract between LLM output and downstream systems. |
| **Document Parsing** | `pypdf` (PDF), `python-docx` (DOCX), raw (TXT), `mailparser` (EML) | Covers all required formats; each parser returns plain text |
| **OCR** | Tesseract (`pytesseract` + `Pillow`) | Open-source, widely available, handles printed text in images. Optional — only triggered for image files. |
| **Orchestration** | Apache Airflow 2.9+ | The data engineer already knows Airflow. DAGs, retries, SLA monitoring, XCom for inter-task data, scheduling. Alternative: Dagster (asset-based model) — noted in design decisions. |
| **Database** | PostgreSQL 16 | Structured data store + audit log + review queue. JSONB columns hold per-doc-type extracted data. |
| **Review UI** | Streamlit 1.39+ | Rapid Python-native UI for the human review queue. No frontend build step. |
| **Containerization** | Docker + Docker Compose | Reproducible multi-service stack: Airflow, Postgres, API, Streamlit |
| **Testing** | pytest + pytest-asyncio + httpx | Async test support, FastAPI test client, fixture-based test data |
| **Linting** | ruff | Fast, replaces flake8 + isort + black |

### Why raw OpenAI structured output over LangChain?

For structured extraction, the OpenAI SDK's `response_format` with a Pydantic schema is a single, transparent call:

```python
completion = client.beta.chat.completions.parse(
    model="gpt-4o-mini",
    response_format=InvoiceSchema,  # Pydantic model passed directly
    messages=[{"role": "user", "content": prompt}],
)
invoice = completion.choices[0].message.parsed  # typed Pydantic object
```

LangChain's `with_structured_output()` wraps this same call but adds an abstraction layer that obscures the prompt-to-schema mapping and makes debugging harder. For a pipeline where the schema IS the contract, transparency matters more than framework convenience. If chain composition (multi-step reasoning, tool use) is needed in v2, LangChain can be introduced for those components without rewriting extraction.

### Why Airflow over Dagster?

The target audience already knows Airflow — DAGs, operators, XCom, retries, SLAs are familiar concepts. Airflow's task-based model maps cleanly to the pipeline stages (ingest -> classify -> extract -> validate -> route). Dagster's asset-based model is powerful but requires a mental model shift that adds learning overhead without proportional benefit for this project. The design decisions appendix notes how to adapt to Dagster if preferred.

---

## 4. Database Schema

### 4.1 Extensions

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
```

### 4.2 Tables

```sql
-- =====================================================
-- DOCUMENTS: metadata for every ingested document
-- =====================================================
CREATE TABLE documents (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    file_name       TEXT NOT NULL,
    file_path       TEXT NOT NULL,                 -- storage path (local / S3 key)
    file_type       TEXT NOT NULL,                 -- 'pdf', 'docx', 'txt', 'eml', 'image'
    file_hash       TEXT NOT NULL,                 -- SHA-256 for dedup
    file_size_bytes BIGINT NOT NULL,
    raw_text        TEXT,                          -- extracted text (NULL if OCR failed)
    ocr_used        BOOLEAN NOT NULL DEFAULT FALSE,
    source          TEXT NOT NULL DEFAULT 'api',   -- 'api', 'file_watcher', 'airflow_batch'
    status          TEXT NOT NULL DEFAULT 'ingested',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    UNIQUE(file_hash)
);

-- Status values: ingested -> classified -> extracted -> validated -> routed
--                | -> review_pending -> reviewed -> routed
--                | -> failed

CREATE INDEX idx_documents_status ON documents(status);
CREATE INDEX idx_documents_created ON documents(created_at DESC);
CREATE INDEX idx_documents_type ON documents(file_type);

-- =====================================================
-- CLASSIFICATIONS: LLM classification result per document
-- =====================================================
CREATE TABLE classifications (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    doc_id          UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    doc_type        TEXT NOT NULL,                 -- 'invoice','contract','email','receipt','other'
    confidence      NUMERIC(5,4) NOT NULL,         -- 0.0000 to 1.0000
    model           TEXT NOT NULL,                 -- which LLM model was used
    prompt_version  TEXT NOT NULL,                 -- prompt template version for reproducibility
    raw_response    JSONB,                         -- full LLM response for debugging
    retry_count     INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    UNIQUE(doc_id)
);

CREATE INDEX idx_classifications_doc ON classifications(doc_id);
CREATE INDEX idx_classifications_type ON classifications(doc_type);
CREATE INDEX idx_classifications_confidence ON classifications(confidence);

-- =====================================================
-- EXTRACTION_RECORDS: structured data extracted by LLM
-- =====================================================
CREATE TABLE extraction_records (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    doc_id          UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    doc_type        TEXT NOT NULL,                 -- denormalized from classification
    extracted_data  JSONB NOT NULL,                -- Pydantic model serialized to JSON
    schema_version  TEXT NOT NULL,                 -- Pydantic schema version
    model           TEXT NOT NULL,                 -- which LLM model was used
    prompt_version  TEXT NOT NULL,
    retry_count     INT NOT NULL DEFAULT 0,        -- how many retries were needed
    validation_errors JSONB,                       -- NULL if valid; array of errors if not
    is_valid        BOOLEAN NOT NULL DEFAULT FALSE,
    extracted_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    UNIQUE(doc_id)
);

CREATE INDEX idx_extraction_doc ON extraction_records(doc_id);
CREATE INDEX idx_extraction_type ON extraction_records(doc_type);
CREATE INDEX idx_extraction_valid ON extraction_records(is_valid);
CREATE INDEX idx_extraction_data ON extraction_records USING GIN(extracted_data);

-- =====================================================
-- REVIEW_QUEUE: items needing human review
-- =====================================================
CREATE TABLE review_queue (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    doc_id          UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    reason          TEXT NOT NULL,                 -- 'low_confidence'|'validation_failed'|'manual'
    reason_detail   TEXT,                          -- e.g., specific validation errors
    confidence      NUMERIC(5,4),                  -- NULL if reason is validation_failed
    priority        INT NOT NULL DEFAULT 5,        -- 1 (highest) to 10 (lowest)
    status          TEXT NOT NULL DEFAULT 'pending', -- 'pending'|'approved'|'rejected'|'escalated'
    assigned_to     TEXT,                          -- reviewer username (NULL = unassigned)
    reviewed_at     TIMESTAMPTZ,
    review_notes    TEXT,
    corrected_data  JSONB,                         -- reviewer's edited extraction (if approved)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    UNIQUE(doc_id) WHERE (status = 'pending')
);

CREATE INDEX idx_review_status ON review_queue(status);
CREATE INDEX idx_review_priority ON review_queue(priority, created_at);
CREATE INDEX idx_review_assigned ON review_queue(assigned_to) WHERE (status = 'pending');

-- =====================================================
-- AUDIT_LOGS: immutable record of every pipeline step
-- =====================================================
CREATE TABLE audit_logs (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    timestamp       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    doc_id          UUID REFERENCES documents(id) ON DELETE CASCADE,
    step            TEXT NOT NULL,                 -- 'ingest','classify','extract','validate','route','review'
    success         BOOLEAN NOT NULL DEFAULT TRUE,
    duration_ms     INT,                           -- step latency
    detail          JSONB,                         -- step-specific metadata (model, errors, etc.)
    error_message   TEXT,

    CONSTRAINT audit_no_update CHECK (true)
);

CREATE INDEX idx_audit_doc ON audit_logs(doc_id, timestamp DESC);
CREATE INDEX idx_audit_step ON audit_logs(step);
CREATE INDEX idx_audit_time ON audit_logs(timestamp DESC);

-- Trigger: prevent UPDATE and DELETE on audit_logs
CREATE OR REPLACE FUNCTION prevent_audit_modification()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'audit_logs is append-only; UPDATE and DELETE are forbidden';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER no_audit_update
    BEFORE UPDATE ON audit_logs
    FOR EACH ROW EXECUTE FUNCTION prevent_audit_modification();

CREATE TRIGGER no_audit_delete
    BEFORE DELETE ON audit_logs
    FOR EACH ROW EXECUTE FUNCTION prevent_audit_modification();

-- =====================================================
-- PIPELINE_RUNS: Airflow DAG run metadata (for observability)
-- =====================================================
CREATE TABLE pipeline_runs (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    dag_run_id      TEXT NOT NULL UNIQUE,          -- Airflow run ID
    batch_size      INT NOT NULL,                  -- number of documents in batch
    succeeded       INT NOT NULL DEFAULT 0,
    failed          INT NOT NULL DEFAULT 0,
    review_queued   INT NOT NULL DEFAULT 0,
    started_at      TIMESTAMPTZ NOT NULL,
    completed_at    TIMESTAMPTZ,
    sla_met         BOOLEAN,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_pipeline_runs_started ON pipeline_runs(started_at DESC);
```

### 4.3 Entity Relationships

```
documents (1) ---- (1) classifications      # one classification per document
documents (1) ---- (1) extraction_records    # one extraction per document
documents (1) ---- (0..1) review_queue       # zero or one review entry (when queued)
documents (1) ---- (N) audit_logs            # many audit entries (one per pipeline step)
pipeline_runs (1) ---- (N) documents         # documents processed in a batch run (via audit_logs)
```

---

## 5. Project Structure

```
doc-pipeline/
├── docker-compose.yml
├── .env.example
├── pyproject.toml
├── README.md
├── IMPLEMENTATION_PLAN.md          <- this file
├── agent.md                      # Herdr multi-agent orchestration guide
│
├── src/
│   ├── __init__.py
│   ├── main.py                     # FastAPI app factory + lifespan
│   ├── config.py                   # Pydantic Settings (env vars)
│   │
│   ├── ingest/
│   │   ├── __init__.py
│   │   ├── parser.py               # PDF/DOCX/TXT/EML text extraction
│   │   ├── ocr.py                  # Tesseract OCR for images
│   │   ├── normalizer.py           # text normalization (whitespace, encoding)
│   │   ├── file_watcher.py         # watchdog-based directory watcher
│   │   └── service.py              # Orchestrates: receive -> parse/OCR -> normalize -> store
│   │
│   ├── classify/
│   │   ├── __init__.py
│   │   ├── classifier.py           # LLM classification with structured output
│   │   ├── prompts.py              # Classification prompt templates (versioned)
│   │   └── service.py              # Orchestrates: classify -> confidence gate -> review queue
│   │
│   ├── extract/
│   │   ├── __init__.py
│   │   ├── extractor.py            # LLM structured extraction with Pydantic response_format
│   │   ├── prompts.py              # Extraction prompt templates per doc_type (versioned)
│   │   ├── retry.py                # Retry-with-error-feedback logic
│   │   └── service.py              # Orchestrates: extract -> validate -> retry -> review queue
│   │
│   ├── schemas/
│   │   ├── __init__.py
│   │   ├── base.py                 # BaseSchema with common fields
│   │   ├── invoice.py              # InvoiceSchema + LineItemSchema
│   │   ├── contract.py             # ContractSchema + PartySchema + ClauseSchema
│   │   ├── email.py                # EmailSchema
│   │   ├── receipt.py              # ReceiptSchema
│   │   ├── other.py                # OtherSchema (freeform key-value)
│   │   └── registry.py             # doc_type -> schema mapping
│   │
│   ├── validate/
│   │   ├── __init__.py
│   │   ├── pydantic_validator.py   # Pydantic v2 validation wrapper
│   │   ├── business_rules.py       # Custom business rule validators per doc_type
│   │   ├── rules_registry.py       # doc_type -> list[BusinessRule] mapping
│   │   └── service.py              # Orchestrates: pydantic validate -> business rules -> result
│   │
│   ├── route/
│   │   ├── __init__.py
│   │   ├── router.py               # Routes validated records to DB + document store + audit
│   │   ├── document_store.py       # File storage abstraction (local FS / S3)
│   │   └── service.py              # Orchestrates: store structured data -> store file -> audit
│   │
│   ├── review/
│   │   ├── __init__.py
│   │   ├── queue.py                # Review queue CRUD (enqueue, list, approve, reject)
│   │   └── api.py                  # FastAPI routes for review queue (used by Streamlit)
│   │
│   ├── audit/
│   │   ├── __init__.py
│   │   └── logger.py               # Append-only audit log writer
│   │
│   ├── db/
│   │   ├── __init__.py
│   │   ├── connection.py           # asyncpg pool
│   │   ├── schema.sql              # full DDL (tables, indexes, triggers)
│   │   └── migrations/
│   │       ├── 001_initial.sql
│   │       └── 002_audit_triggers.sql
│   │
│   ├── llm/
│   │   ├── __init__.py
│   │   ├── client.py               # OpenAI client wrapper (with retry, timeout, cost tracking)
│   │   └── cost_tracker.py         # Token usage + cost logging per call
│   │
│   └── api/
│       ├── __init__.py
│       ├── routes/
│       │   ├── __init__.py
│       │   ├── ingest.py           # POST /ingest (single file)
│       │   ├── batch.py            # POST /batch (directory path for batch processing)
│       │   ├── documents.py        # GET /documents, GET /documents/{id}
│       │   ├── review.py           # GET /review/queue, POST /review/{id}/approve, etc.
│       │   ├── stats.py            # GET /stats (pipeline metrics dashboard data)
│       │   └── health.py           # GET /health
│       └── dependencies.py         # FastAPI dependency injection
│
├── airflow/
│   ├── dags/
│   │   └── document_processing_dag.py  # Main DAG: scan -> process batch -> route
│   ├── plugins/
│   │   └── doc_pipeline_plugin.py      # Custom operators (if needed)
│   └── config/
│       └── airflow.cfg.template
│
├── streamlit/
│   ├── app.py                      # Streamlit review queue UI
│   ├── components/
│   │   ├── queue_view.py           # Pending review items list
│   │   ├── detail_view.py          # Single item: original doc + extracted data + edit form
│   │   └── stats_view.py           # Pipeline metrics dashboard
│   └── styles.css
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py                 # pytest fixtures: test DB, test client, sample docs
│   ├── fixtures/                   # Sample documents for testing
│   │   ├── invoices/
│   │   ├── contracts/
│   │   ├── emails/
│   │   ├── receipts/
│   │   └── images/
│   ├── test_ingest.py              # Parsing, OCR, normalization
│   ├── test_classify.py            # Classification accuracy, confidence gate
│   ├── test_extract.py             # Structured extraction per doc_type
│   ├── test_schemas.py             # Pydantic schema validation
│   ├── test_validate.py            # Business rules, validation failures
│   ├── test_retry.py               # Retry-with-error-feedback logic
│   ├── test_route.py               # Routing to DB + document store + audit
│   ├── test_review_queue.py        # Enqueue, approve, reject flows
│   ├── test_audit.py               # Audit log immutability
│   ├── test_api.py                 # API endpoints end-to-end
│   ├── test_dag.py                 # Airflow DAG structure and task dependencies
│   └── test_e2e.py                 # Full pipeline: ingest -> classify -> extract -> validate -> route
│
├── scripts/
│   ├── init_db.py                  # Run schema.sql against fresh database
│   ├── seed_test_data.py           # Load sample documents into DB
│   ├── generate_golden_set.py      # Create golden test set with expected extractions
│   └── run_pipeline.py             # CLI to run pipeline on a single file or directory
│
└── docker/
    ├── Dockerfile.api              # FastAPI app image
    ├── Dockerfile.streamlit        # Streamlit review UI image
    ├── Dockerfile.airflow          # Airflow image (webserver + scheduler + worker)
    └── postgres/
        └── init.sql                # Extensions on first boot
```

---

## 6. Implementation Phases

### Phase 1: Foundation & Ingestion (Days 1-2)

**Goal:** Docker Compose stack running (Postgres + API), document parsing for all formats, files stored with metadata.

| Step | Task | Deliverable |
|------|------|-------------|
| 1.1 | Write `docker-compose.yml` with Postgres, API, Airflow, Streamlit services | `docker compose up` starts all services |
| 1.2 | Write `docker/postgres/init.sql` (extensions) + `src/db/schema.sql` (full DDL) | DB initializes on first boot |
| 1.3 | Write `src/config.py` (Pydantic Settings) | Env-driven config |
| 1.4 | Write `src/db/connection.py` (asyncpg pool) | DB connection pool works |
| 1.5 | Write `src/ingest/parser.py` (PDF, DOCX, TXT, EML) | Text extraction from all 4 formats |
| 1.6 | Write `src/ingest/ocr.py` (Tesseract for images) | Image -> text via OCR |
| 1.7 | Write `src/ingest/normalizer.py` | Clean whitespace, fix encoding |
| 1.8 | Write `src/ingest/service.py` (orchestrator: receive -> parse -> store) | Full ingestion: file -> raw_text + metadata in DB |
| 1.9 | Write `src/api/routes/ingest.py` (`POST /ingest`) | API endpoint accepts files |
| 1.10 | Write `src/audit/logger.py` + wire into ingestion | Every ingest logged |
| 1.11 | Write `tests/test_ingest.py` with sample docs per format | Ingestion tests pass |

**Verification:** `docker compose up` -> `POST /ingest` with a PDF, DOCX, TXT, EML, and image -> check `documents` table has rows with `raw_text` populated. OCR flag set for images.

---

### Phase 2: Classification (Days 3-4)

**Goal:** LLM classifies each document with confidence score; low-confidence items enqueued for review.

| Step | Task | Deliverable |
|------|------|-------------|
| 2.1 | Write `src/llm/client.py` (OpenAI wrapper with retry, timeout, cost tracking) | LLM calls work, cost logged |
| 2.2 | Write `src/classify/prompts.py` (versioned classification prompt) | Prompt template with doc_type enum + confidence |
| 2.3 | Write `src/classify/classifier.py` (structured output classification) | LLM returns `{doc_type, confidence}` |
| 2.4 | Write `src/classify/service.py` (classify -> confidence gate -> review queue) | Confidence < threshold -> review_queue row |
| 2.5 | Write `src/review/queue.py` (enqueue, list, update status) | Review queue CRUD works |
| 2.6 | Write `src/api/routes/review.py` (GET /review/queue) | API returns pending review items |
| 2.7 | Write `tests/test_classify.py` (accuracy on golden set, confidence gate) | Classification tests pass |
| 2.8 | Write `tests/test_review_queue.py` (enqueue, list, status transitions) | Review queue tests pass |

**Verification:** Ingest 10 sample docs (2 per type) -> all classified -> check `classifications` table has doc_type + confidence. Manually set confidence threshold high -> verify items land in `review_queue`.

---

### Phase 3: Pydantic Schemas & Structured Extraction (Days 5-6)

**Goal:** Per doc_type, LLM extracts structured data into Pydantic models with structured output.

| Step | Task | Deliverable |
|------|------|-------------|
| 3.1 | Write `src/schemas/base.py` (BaseSchema with common fields) | Shared base model |
| 3.2 | Write `src/schemas/invoice.py` (InvoiceSchema + LineItemSchema) | Invoice Pydantic model with validators |
| 3.3 | Write `src/schemas/contract.py` (ContractSchema + PartySchema + ClauseSchema) | Contract Pydantic model |
| 3.4 | Write `src/schemas/email.py` (EmailSchema) | Email Pydantic model |
| 3.5 | Write `src/schemas/receipt.py` (ReceiptSchema) | Receipt Pydantic model |
| 3.6 | Write `src/schemas/other.py` (OtherSchema — freeform key-value) | Fallback for unclassified docs |
| 3.7 | Write `src/schemas/registry.py` (doc_type -> schema mapping) | Schema lookup by doc_type |
| 3.8 | Write `src/extract/prompts.py` (per-doc_type extraction prompts, versioned) | Prompt templates for each type |
| 3.9 | Write `src/extract/extractor.py` (LLM structured extraction with `response_format`) | LLM returns typed Pydantic object |
| 3.10 | Write `src/extract/service.py` (extract -> store in extraction_records) | Extraction results in DB |
| 3.11 | Write `tests/test_schemas.py` (Pydantic validation per schema) | Schema validation tests pass |
| 3.12 | Write `tests/test_extract.py` (extraction accuracy on golden set) | Extraction tests pass |

**Verification:** Ingest + classify a sample invoice -> extraction produces an `InvoiceSchema` object with vendor, invoice_number, line_items, total. Check `extraction_records.extracted_data` contains the serialized Pydantic model.

---

### Phase 4: Validation Layer & Retry Logic (Days 7-8)

**Goal:** Pydantic validation + business rules catch errors; retry with error feedback; failures -> human review.

| Step | Task | Deliverable |
|------|------|-------------|
| 4.1 | Write `src/validate/pydantic_validator.py` (Pydantic v2 validation wrapper) | Validates extracted JSON against schema |
| 4.2 | Write `src/validate/business_rules.py` (per-doc_type business rules) | All rules from validation table implemented |
| 4.3 | Write `src/validate/rules_registry.py` (doc_type -> list[BusinessRule]) | Rule lookup by doc_type |
| 4.4 | Write `src/validate/service.py` (pydantic validate -> business rules -> result) | Returns ValidationResult with errors |
| 4.5 | Write `src/extract/retry.py` (retry with error feedback appended to prompt) | Re-extraction with validation errors in prompt |
| 4.6 | Wire retry into `src/extract/service.py` (max 2 retries, then review queue) | Failed validations retry, then enqueue |
| 4.7 | Write `tests/test_validate.py` (each business rule, pass + fail cases) | Validation tests pass |
| 4.8 | Write `tests/test_retry.py` (retry reduces errors, max retries enforced) | Retry tests pass |

**Verification:** Feed a document where LLM extracts an invoice with `total != sum of line_items` -> validation fails -> retry with error feedback -> if fixed, routes; if not, lands in review queue with `reason='validation_failed'` and `reason_detail` listing the specific error.

---

### Phase 5: Routing & Audit (Day 9)

**Goal:** Validated records route to PostgreSQL + document store + audit log.

| Step | Task | Deliverable |
|------|------|-------------|
| 5.1 | Write `src/route/document_store.py` (local FS abstraction, S3-ready interface) | Files stored and retrievable |
| 5.2 | Write `src/route/router.py` (insert extraction_records, update document status, audit) | Full routing: DB + file + audit |
| 5.3 | Write `src/route/service.py` (orchestrator) | End-to-end routing works |
| 5.4 | Wire audit logging into all pipeline steps (ingest, classify, extract, validate, route) | Every step logged |
| 5.5 | Write `src/api/routes/documents.py` (GET /documents, GET /documents/{id}) | API returns document + extraction data |
| 5.6 | Write `src/api/routes/stats.py` (GET /stats — pipeline metrics) | API returns counts by status, type, etc. |
| 5.7 | Write `tests/test_route.py` (routing correctness, audit completeness) | Routing tests pass |
| 5.8 | Write `tests/test_audit.py` (immutability, completeness) | Audit tests pass |

**Verification:** Process a valid invoice end-to-end -> check `extraction_records.is_valid=true`, `documents.status='routed'`, `audit_logs` has entries for every step, original file in document store.

---

### Phase 6: Airflow DAG & Orchestration (Day 10)

**Goal:** Airflow DAG runs the full pipeline as a batch with retries and SLA monitoring.

| Step | Task | Deliverable |
|------|------|-------------|
| 6.1 | Write `airflow/dags/document_processing_dag.py` (DAG with task dependencies) | DAG visible in Airflow UI |
| 6.2 | Implement DAG tasks: scan -> ingest_batch -> classify_batch -> extract_batch -> validate_batch -> route_batch | Each task calls the corresponding service |
| 6.3 | Configure Airflow retries (task-level: 2 retries, exponential backoff) | Tasks retry on transient failures |
| 6.4 | Configure SLA (batch must complete within 60 min; SLA miss callback alerts) | SLA monitoring active |
| 6.5 | Write `scripts/run_pipeline.py` (CLI for single-file or directory processing) | Can run pipeline outside Airflow |
| 6.6 | Write `tests/test_dag.py` (DAG structure, task dependencies, no orphaned tasks) | DAG tests pass |
| 6.7 | Write `tests/test_e2e.py` (full pipeline on a batch of mixed docs) | E2E test passes |

**Verification:** Trigger DAG manually in Airflow UI with a directory of 10 mixed documents -> all process -> check `pipeline_runs` table has run metadata with succeeded/failed/review_queued counts. SLA miss callback fires if batch exceeds timeout.

---

### Phase 7: Streamlit Review UI & Hardening (Days 11-12, buffer)

**Goal:** Human review queue UI works; full system hardened and documented.

> **Note:** The core 10-day plan spans Phases 1-6. Phase 7 uses the buffer days (evenings/weekend) to complete the review UI and polish. If working full-time, this fits within the 2-week window.

| Step | Task | Deliverable |
|------|------|-------------|
| 7.1 | Write `streamlit/app.py` (main app with sidebar navigation) | Streamlit app starts |
| 7.2 | Write `streamlit/components/queue_view.py` (pending items, sortable by priority) | Reviewer sees queue |
| 7.3 | Write `streamlit/components/detail_view.py` (original doc + extracted data + edit form) | Reviewer can view + edit + approve/reject |
| 7.4 | Write `streamlit/components/stats_view.py` (pipeline metrics dashboard) | Metrics visible |
| 7.5 | Wire approve/reject to `src/review/api.py` -> routing | Approved items route to downstream |
| 7.6 | Add structured logging (structlog) | JSON logs for observability |
| 7.7 | Write `README.md` (setup, usage, architecture, metrics) | Someone else can run it |
| 7.8 | Add `.env.example` with all env vars | Configuration documented |
| 7.9 | Run full test suite end-to-end | All tests green |
| 7.10 | Manual smoke test: drop 5 docs in watched dir -> check full pipeline + review queue | Works without intervention |

**Verification:** Drop a low-quality image invoice in the watched directory -> it classifies with low confidence -> appears in Streamlit review queue -> reviewer edits fields -> approves -> record routes to `extraction_records` with `is_valid=true`.

---

## 7. Component Specifications

### 7.1 Ingestion Service (`src/ingest/service.py`)

```python
"""
Full ingestion pipeline: file -> text -> metadata -> database -> audit.

Handles all supported formats. Images go through OCR.
Returns the document ID and raw text for downstream processing.
"""

async def ingest_document(
    file_content: bytes,
    file_name: str,
    source: str = "api",
) -> IngestResult:
    # 1. Hash for dedup
    file_hash = hashlib.sha256(file_content).hexdigest()
    existing = await db.fetchrow(
        "SELECT id FROM documents WHERE file_hash = $1", file_hash
    )
    if existing:
        raise DuplicateDocumentError(existing["id"])

    # 2. Detect file type
    file_type = detect_file_type(file_name, file_content)

    # 3. Extract text based on type
    ocr_used = False
    if file_type == "image":
        raw_text = ocr_extract(file_content)
        ocr_used = True
    else:
        raw_text = parse_file(file_content, file_name, file_type)

    # 4. Normalize text
    raw_text = normalize_text(raw_text)

    # 5. Store original file in document store
    doc_id = uuid4()
    file_path = await document_store.save(file_content, doc_id, file_name)

    # 6. Insert document metadata
    await db.execute(
        """INSERT INTO documents
           (id, file_name, file_path, file_type, file_hash, file_size_bytes,
            raw_text, ocr_used, source, status)
           VALUES ($1,$2,$3,$4,$5,$6,$7,$8,$9,'ingested')""",
        doc_id, file_name, file_path, file_type, file_hash,
        len(file_content), raw_text, ocr_used, source
    )

    # 7. Audit log
    await audit_logger.log(
        doc_id=doc_id, step="ingest", success=True,
        detail={"file_type": file_type, "ocr_used": ocr_used,
                "text_length": len(raw_text)}
    )

    return IngestResult(doc_id=doc_id, raw_text=raw_text, file_type=file_type)
```

### 7.2 Classification Service (`src/classify/service.py`)

```python
"""
Classifies a document using LLM structured output.
Low-confidence classifications are enqueued for human review.
"""

CLASSIFICATION_THRESHOLD = 0.85

async def classify_document(doc_id: UUID, raw_text: str) -> ClassificationResult:
    # 1. Truncate text to fit context window (keep head + tail)
    truncated = truncate_for_context(raw_text, max_tokens=12000)

    # 2. Call LLM with structured output
    response = await llm_client.beta.chat.completions.parse(
        model=settings.LLM_MODEL,
        response_format=ClassificationResponse,  # Pydantic: {doc_type, confidence}
        messages=[
            {"role": "system", "content": CLASSIFICATION_PROMPT},
            {"role": "user", "content": truncated},
        ],
    )
    result = response.choices[0].message.parsed  # ClassificationResponse

    # 3. Store classification
    classification_id = uuid4()
    await db.execute(
        """INSERT INTO classifications
           (id, doc_id, doc_type, confidence, model, prompt_version, raw_response)
           VALUES ($1,$2,$3,$4,$5,$6,$7)""",
        classification_id, doc_id, result.doc_type, result.confidence,
        settings.LLM_MODEL, CLASSIFICATION_PROMPT_VERSION,
        json.dumps(response.model_dump())
    )

    # 4. Update document status
    await db.execute(
        "UPDATE documents SET status = 'classified', updated_at = NOW() WHERE id = $1",
        doc_id
    )

    # 5. Audit log
    await audit_logger.log(
        doc_id=doc_id, step="classify", success=True,
        detail={"doc_type": result.doc_type, "confidence": float(result.confidence)}
    )

    # 6. Confidence gate
    if result.confidence < CLASSIFICATION_THRESHOLD:
        await enqueue_for_review(
            doc_id=doc_id,
            reason="low_confidence",
            reason_detail=f"Confidence {result.confidence:.2f} < threshold {CLASSIFICATION_THRESHOLD}",
            confidence=result.confidence,
        )
        return ClassificationResult(
            doc_type=result.doc_type, confidence=result.confidence,
            enqueued_for_review=True
        )

    return ClassificationResult(
        doc_type=result.doc_type, confidence=result.confidence,
        enqueued_for_review=False
    )
```

### 7.3 Extraction Service with Retry (`src/extract/service.py`)

```python
"""
Extracts structured data from a document using LLM structured output.
Uses the Pydantic schema registered for the classified doc_type.
On validation failure, retries with error feedback appended to the prompt.
"""

MAX_RETRIES = 2

async def extract_document(
    doc_id: UUID, doc_type: str, raw_text: str
) -> ExtractionResult:
    schema = get_schema_for_doc_type(doc_type)   # from registry
    prompt_template = get_extraction_prompt(doc_type)  # versioned

    retry_count = 0
    error_feedback = None
    extracted = None
    validation_result = None

    while retry_count <= MAX_RETRIES:
        # 1. Build prompt (with error feedback on retry)
        prompt = build_extraction_prompt(prompt_template, raw_text, error_feedback)

        # 2. Call LLM with structured output (Pydantic response_format)
        extracted, cost = await llm_client.structured_complete(
            response_format=schema,
            system_prompt=prompt["system"],
            user_prompt=prompt["user"],
        )

        # 3. Validate (Pydantic + business rules)
        validation_result = await validate_extraction(doc_type, extracted)

        if validation_result.is_valid:
            # 4a. Store valid extraction
            await store_extraction(
                doc_id, doc_type, extracted, retry_count,
                validation_errors=None, is_valid=True
            )
            await db.execute(
                "UPDATE documents SET status = 'validated', "
                "updated_at = NOW() WHERE id = $1",
                doc_id
            )
            await audit_logger.log(
                doc_id=doc_id, step="extract", success=True,
                detail={"retry_count": retry_count, "doc_type": doc_type,
                        "cost": cost.model_dump()}
            )
            return ExtractionResult(
                doc_id=doc_id, extracted_data=extracted,
                is_valid=True, retry_count=retry_count
            )

        # 4b. Validation failed — prepare error feedback for retry
        error_feedback = format_validation_errors(validation_result.errors)
        retry_count += 1

    # 5. Exhausted retries — store invalid extraction + enqueue for review
    await store_extraction(
        doc_id, doc_type, extracted, retry_count,
        validation_errors=validation_result.errors, is_valid=False
    )
    await enqueue_for_review(
        doc_id=doc_id,
        reason="validation_failed",
        reason_detail=format_validation_errors(validation_result.errors),
    )
    await audit_logger.log(
        doc_id=doc_id, step="extract", success=False,
        detail={"retry_count": retry_count,
                "errors": validation_result.errors},
        error_message="Validation failed after max retries"
    )
    return ExtractionResult(
        doc_id=doc_id, extracted_data=extracted,
        is_valid=False, retry_count=retry_count,
        enqueued_for_review=True
    )


def build_extraction_prompt(
    template: str, raw_text: str, error_feedback: str | None
) -> dict:
    """
    On first attempt: standard extraction prompt.
    On retry: append validation errors so the LLM can self-correct.
    """
    system = template
    user = f"Document text:\n\n{raw_text}\n\nExtract all fields according to the schema."
    if error_feedback:
        user += (
            f"\n\n--- PREVIOUS ATTEMPT FAILED VALIDATION ---\n"
            f"The previous extraction had these validation errors:\n"
            f"{error_feedback}\n"
            f"Please correct these errors and re-extract."
        )
    return {"system": system, "user": user}
```

### 7.4 Validation Service (`src/validate/service.py`)

```python
"""
Two-layer validation:
  1. Pydantic validation — type constraints, required fields, field validators
  2. Business rule validation — custom rules (invoice total = sum of lines, etc.)

Returns a ValidationResult with all errors collected (not just the first).
"""

async def validate_extraction(
    doc_type: str, extracted: BaseModel
) -> ValidationResult:
    errors: list[str] = []

    # Layer 1: Pydantic validation
    # (The LLM returned a parsed Pydantic object, but re-validate to be safe
    # and to collect all errors at once rather than raising on the first)
    try:
        extracted.model_validate(extracted.model_dump())
    except ValidationError as e:
        errors.extend(format_pydantic_errors(e))

    # Layer 2: Business rules
    rules = get_business_rules(doc_type)  # from rules_registry
    for rule in rules:
        try:
            rule.check(extracted)
        except BusinessRuleError as e:
            errors.append(f"[{rule.name}] {e}")

    is_valid = len(errors) == 0
    return ValidationResult(is_valid=is_valid, errors=errors)
```

### 7.5 Routing Service (`src/route/service.py`)

```python
"""
Routes validated extraction records to:
  1. PostgreSQL (extraction_records table — structured data)
  2. Document store (original file — already stored at ingest)
  3. Audit log (every routing action logged)

Only called for records that passed validation.
"""

async def route_document(
    doc_id: UUID, extraction_result: ExtractionResult
) -> None:
    # 1. Update extraction_records is_valid flag
    await db.execute(
        "UPDATE extraction_records SET is_valid = TRUE WHERE doc_id = $1",
        doc_id
    )

    # 2. Update document status
    await db.execute(
        "UPDATE documents SET status = 'routed', updated_at = NOW() WHERE id = $1",
        doc_id
    )

    # 3. Audit log
    await audit_logger.log(
        doc_id=doc_id, step="route", success=True,
        detail={
            "doc_type": extraction_result.doc_type,
            "retry_count": extraction_result.retry_count,
        }
    )
```

### 7.6 Review Queue (`src/review/queue.py`)

```python
"""
Human review queue management.
Items are enqueued when:
  - Classification confidence < threshold (reason='low_confidence')
  - Extraction validation fails after max retries (reason='validation_failed')
  - Manual flag (reason='manual')

Approved items with corrected data route to downstream.
Rejected items are archived.
"""

async def enqueue_for_review(
    doc_id: UUID,
    reason: str,
    reason_detail: str | None = None,
    confidence: float | None = None,
    priority: int = 5,
) -> UUID:
    review_id = uuid4()
    await db.execute(
        """INSERT INTO review_queue
           (id, doc_id, reason, reason_detail, confidence, priority, status)
           VALUES ($1,$2,$3,$4,$5,$6,'pending')""",
        review_id, doc_id, reason, reason_detail, confidence, priority
    )
    await db.execute(
        "UPDATE documents SET status = 'review_pending', "
        "updated_at = NOW() WHERE id = $1",
        doc_id
    )
    await audit_logger.log(
        doc_id=doc_id, step="review", success=True,
        detail={"action": "enqueued", "reason": reason,
                "review_id": str(review_id)}
    )
    return review_id


async def approve_review(
    review_id: UUID,
    corrected_data: dict,
    reviewer: str,
    notes: str | None = None,
) -> None:
    """Reviewer approves with (possibly edited) corrected data -> routes downstream."""
    review = await db.fetchrow(
        "SELECT doc_id FROM review_queue WHERE id = $1", review_id
    )

    await db.execute(
        """UPDATE review_queue
           SET status = 'approved', assigned_to = $1, reviewed_at = NOW(),
               review_notes = $2, corrected_data = $3, updated_at = NOW()
           WHERE id = $4""",
        reviewer, notes, json.dumps(corrected_data), review_id
    )

    # Update extraction_records with corrected data
    await db.execute(
        """UPDATE extraction_records
           SET extracted_data = $1, is_valid = TRUE, validation_errors = NULL
           WHERE doc_id = $2""",
        json.dumps(corrected_data), review["doc_id"]
    )

    # Route to downstream
    await db.execute(
        "UPDATE documents SET status = 'routed', updated_at = NOW() WHERE id = $1",
        review["doc_id"]
    )
    await audit_logger.log(
        doc_id=review["doc_id"], step="review", success=True,
        detail={"action": "approved", "reviewer": reviewer,
                "review_id": str(review_id)}
    )


async def reject_review(
    review_id: UUID,
    reviewer: str,
    notes: str,
) -> None:
    """Reviewer rejects -> document archived, not routed."""
    review = await db.fetchrow(
        "SELECT doc_id FROM review_queue WHERE id = $1", review_id
    )

    await db.execute(
        """UPDATE review_queue
           SET status = 'rejected', assigned_to = $1, reviewed_at = NOW(),
               review_notes = $2, updated_at = NOW()
           WHERE id = $3""",
        reviewer, notes, review_id
    )
    await db.execute(
        "UPDATE documents SET status = 'failed', updated_at = NOW() WHERE id = $1",
        review["doc_id"]
    )
    await audit_logger.log(
        doc_id=review["doc_id"], step="review", success=False,
        detail={"action": "rejected", "reviewer": reviewer,
                "review_id": str(review_id)},
        error_message=notes
    )
```

### 7.7 LLM Client (`src/llm/client.py`)

```python
"""
OpenAI client wrapper with:
  - Retry with exponential backoff (for rate limits / transient errors)
  - Timeout enforcement
  - Cost tracking (token usage + estimated cost per call)
  - Model abstraction (can swap to Ollama for on-prem)
"""

from openai import OpenAI, RateLimitError, APITimeoutError, APIConnectionError
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type

class LLMClient:
    def __init__(self, api_key: str, model: str, timeout: int = 60):
        self.client = OpenAI(api_key=api_key, timeout=timeout)
        self.model = model
        self.cost_tracker = CostTracker()

    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=2, max=30),
        retry=retry_if_exception_type(
            (RateLimitError, APITimeoutError, APIConnectionError)
        ),
    )
    def structured_complete(
        self,
        response_format: type[BaseModel],
        system_prompt: str,
        user_prompt: str,
    ) -> tuple[BaseModel, CostRecord]:
        response = self.client.beta.chat.completions.parse(
            model=self.model,
            response_format=response_format,
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": user_prompt},
            ],
        )
        parsed = response.choices[0].message.parsed
        cost = self.cost_tracker.record(response.usage)
        return parsed, cost
```

### 7.8 Audit Logger (`src/audit/logger.py`)

```python
"""
Append-only audit logger. Every pipeline step is recorded.
The audit_logs table has triggers that prevent UPDATE and DELETE,
making it tamper-evident. Critical for compliance and debugging.
"""

async def log(
    doc_id: UUID | None,
    step: str,
    success: bool = True,
    duration_ms: int | None = None,
    detail: dict | None = None,
    error_message: str | None = None,
) -> None:
    await db.execute(
        """INSERT INTO audit_logs
           (doc_id, step, success, duration_ms, detail, error_message)
           VALUES ($1,$2,$3,$4,$5,$6)""",
        doc_id, step, success, duration_ms,
        json.dumps(detail) if detail else None,
        error_message
    )
```

---

## 8. Pydantic Schemas (Per Document Type)

### 8.1 Base Schema (`src/schemas/base.py`)

```python
from pydantic import BaseModel, Field
from datetime import datetime
from uuid import UUID

class BaseExtractionSchema(BaseModel):
    """Common fields for all extraction schemas."""
    doc_id: UUID | None = None           # set by pipeline, not the LLM
    extracted_at: datetime | None = None # set by pipeline, not the LLM

    model_config = {
        "extra": "forbid",               # reject unknown fields from LLM
        "str_strip_whitespace": True,    # clean whitespace from all strings
    }
```

### 8.2 Invoice Schema (`src/schemas/invoice.py`)

```python
from pydantic import BaseModel, Field, field_validator, model_validator
from datetime import date
from decimal import Decimal
from enum import Enum
from .base import BaseExtractionSchema

class Currency(str, Enum):
    USD = "USD"
    EUR = "EUR"
    GBP = "GBP"
    JPY = "JPY"
    CAD = "CAD"
    AUD = "AUD"
    CHF = "CHF"
    OTHER = "OTHER"

class LineItem(BaseModel):
    model_config = {"extra": "forbid", "str_strip_whitespace": True}

    description: str = Field(..., min_length=1, description="Item description")
    quantity: Decimal = Field(..., gt=0, description="Quantity (must be positive)")
    unit_price: Decimal = Field(..., ge=0, description="Unit price (non-negative)")
    line_total: Decimal = Field(..., ge=0, description="Line total = quantity * unit_price")

    @model_validator(mode="after")
    def check_line_total(self):
        expected = (self.quantity * self.unit_price).quantize(Decimal("0.01"))
        if abs(self.line_total - expected) > Decimal("0.02"):
            raise ValueError(
                f"line_total ({self.line_total}) != quantity * unit_price "
                f"({expected}) for '{self.description}'"
            )
        return self

class InvoiceSchema(BaseExtractionSchema):
    """Schema for invoice extraction. Enforced via LLM structured output + validation."""

    invoice_number: str = Field(..., min_length=1, description="Invoice identifier")
    invoice_date: date = Field(..., description="Invoice issue date")
    due_date: date | None = Field(None, description="Payment due date (if present)")

    vendor_name: str = Field(..., min_length=1, description="Vendor / supplier name")
    vendor_address: str | None = Field(None, description="Vendor address")
    vendor_tax_id: str | None = Field(None, description="Vendor tax ID / VAT number")

    customer_name: str | None = Field(None, description="Customer / buyer name")
    customer_address: str | None = Field(None, description="Customer address")

    line_items: list[LineItem] = Field(
        ..., min_length=1, description="At least one line item"
    )

    subtotal: Decimal = Field(..., ge=0, description="Sum of line item totals")
    tax_rate: Decimal = Field(
        ..., ge=0, le=1, description="Tax rate as decimal (0.19 = 19%)"
    )
    tax_amount: Decimal = Field(..., ge=0, description="Tax amount")
    total: Decimal = Field(..., ge=0, description="Invoice total = subtotal + tax_amount")

    currency: Currency = Field(..., description="Currency code")
    payment_terms: str | None = Field(
        None, description="Payment terms text (e.g., 'Net 30')"
    )

    @field_validator("invoice_date", "due_date")
    @classmethod
    def date_not_future(cls, v: date | None) -> date | None:
        if v and v > date.today():
            raise ValueError(
                f"Date {v} is in the future — invoices should not be post-dated"
            )
        return v

    @model_validator(mode="after")
    def check_subtotal_equals_line_items(self):
        expected_subtotal = sum(
            (item.line_total for item in self.line_items), Decimal("0")
        ).quantize(Decimal("0.01"))
        if abs(self.subtotal - expected_subtotal) > Decimal("0.02"):
            raise ValueError(
                f"subtotal ({self.subtotal}) != sum of line items "
                f"({expected_subtotal})"
            )
        return self

    @model_validator(mode="after")
    def check_tax_amount(self):
        expected_tax = (self.subtotal * self.tax_rate).quantize(Decimal("0.01"))
        if abs(self.tax_amount - expected_tax) > Decimal("0.02"):
            raise ValueError(
                f"tax_amount ({self.tax_amount}) != subtotal * tax_rate "
                f"({expected_tax})"
            )
        return self

    @model_validator(mode="after")
    def check_total(self):
        expected_total = (self.subtotal + self.tax_amount).quantize(Decimal("0.01"))
        if abs(self.total - expected_total) > Decimal("0.02"):
            raise ValueError(
                f"total ({self.total}) != subtotal + tax_amount ({expected_total})"
            )
        return self

    @model_validator(mode="after")
    def check_due_date_after_invoice_date(self):
        if (self.due_date and self.invoice_date
                and self.due_date < self.invoice_date):
            raise ValueError("due_date cannot be before invoice_date")
        return self
```

### 8.3 Contract Schema (`src/schemas/contract.py`)

```python
from pydantic import BaseModel, Field, field_validator, model_validator
from datetime import date
from decimal import Decimal
from enum import Enum
from .base import BaseExtractionSchema

class ContractType(str, Enum):
    SERVICE_AGREEMENT = "service_agreement"
    NDA = "nda"
    EMPLOYMENT = "employment"
    LEASE = "lease"
    PURCHASE_AGREEMENT = "purchase_agreement"
    PARTNERSHIP = "partnership"
    OTHER = "other"

class Party(BaseModel):
    model_config = {"extra": "forbid", "str_strip_whitespace": True}

    name: str = Field(..., min_length=1, description="Party name")
    role: str = Field(
        ..., description="Role: 'buyer', 'seller', 'client', 'contractor', etc."
    )
    address: str | None = None
    contact_email: str | None = None

class Clause(BaseModel):
    model_config = {"extra": "forbid", "str_strip_whitespace": True}

    clause_type: str = Field(
        ...,
        description="e.g., 'termination', 'payment', 'confidentiality', 'liability'"
    )
    summary: str = Field(..., min_length=10, description="Summary of the clause")
    page_reference: str | None = Field(
        None, description="Page or section reference if available"
    )

class ContractSchema(BaseExtractionSchema):
    """Schema for contract extraction."""

    contract_title: str = Field(
        ..., min_length=1, description="Title or subject of the contract"
    )
    contract_type: ContractType = Field(..., description="Type of contract")
    effective_date: date = Field(..., description="Date the contract takes effect")
    expiration_date: date | None = Field(
        None, description="Expiration date (if fixed-term)"
    )

    parties: list[Party] = Field(
        ..., min_length=2, description="At least 2 parties"
    )

    contract_value: Decimal | None = Field(
        None, ge=0, description="Total contract value (if specified)"
    )
    currency: str | None = Field(
        None, description="Currency code if value is specified"
    )

    key_clauses: list[Clause] = Field(
        ..., min_length=1, description="Key clauses extracted"
    )

    termination_notice_days: int | None = Field(
        None, ge=0, description="Notice period in days for termination"
    )
    governing_law: str | None = Field(
        None, description="Governing law jurisdiction"
    )

    @field_validator("effective_date")
    @classmethod
    def effective_date_plausibility(cls, v: date) -> date:
        # Contracts can be future-dated, but flag if > 1 year out
        one_year_out = date.today().replace(year=date.today().year + 1)
        if v > one_year_out:
            raise ValueError(
                f"effective_date {v} is more than 1 year in the future — verify"
            )
        return v

    @model_validator(mode="after")
    def check_expiration_after_effective(self):
        if self.expiration_date and self.effective_date:
            if self.expiration_date <= self.effective_date:
                raise ValueError("expiration_date must be after effective_date")
        return self
```

### 8.4 Email Schema (`src/schemas/email.py`)

```python
from pydantic import BaseModel, Field, field_validator, model_validator
from datetime import datetime
from enum import Enum
from .base import BaseExtractionSchema

class EmailPriority(str, Enum):
    HIGH = "high"
    NORMAL = "normal"
    LOW = "low"

class EmailSchema(BaseExtractionSchema):
    """Schema for email extraction from .eml files."""

    sender_name: str | None = Field(None, description="Sender display name")
    sender_email: str = Field(
        ..., min_length=3, description="Sender email address"
    )
    recipients: list[str] = Field(
        ..., min_length=1, description="Recipient email addresses"
    )
    cc_recipients: list[str] = Field(
        default_factory=list, description="CC recipients"
    )
    bcc_recipients: list[str] = Field(
        default_factory=list, description="BCC recipients"
    )

    subject: str = Field(..., min_length=1, description="Email subject line")
    body: str = Field(..., min_length=1, description="Email body text")
    sent_at: datetime = Field(..., description="Email sent timestamp")

    priority: EmailPriority = Field(
        EmailPriority.NORMAL, description="Email priority"
    )
    has_attachments: bool = Field(
        False, description="Whether the email has attachments"
    )
    attachment_names: list[str] = Field(
        default_factory=list, description="Attachment file names"
    )

    @field_validator("sender_email")
    @classmethod
    def validate_email_format(cls, v: str) -> str:
        if "@" not in v or "." not in v.split("@")[-1]:
            raise ValueError(
                f"sender_email '{v}' does not look like a valid email address"
            )
        return v.lower().strip()

    @field_validator("recipients", "cc_recipients", "bcc_recipients")
    @classmethod
    def validate_recipient_list(cls, v: list[str]) -> list[str]:
        for email in v:
            if "@" not in email:
                raise ValueError(f"Recipient '{email}' is not a valid email address")
        return [e.lower().strip() for e in v]

    @field_validator("sent_at")
    @classmethod
    def sent_at_not_future(cls, v: datetime) -> datetime:
        if v > datetime.now():
            raise ValueError(f"sent_at {v} is in the future")
        return v

    @model_validator(mode="after")
    def check_attachments_consistency(self):
        if self.has_attachments and not self.attachment_names:
            raise ValueError(
                "has_attachments is True but attachment_names is empty"
            )
        if not self.has_attachments and self.attachment_names:
            raise ValueError(
                "has_attachments is False but attachment_names is not empty"
            )
        return self
```

### 8.5 Receipt Schema (`src/schemas/receipt.py`)

```python
from pydantic import BaseModel, Field, field_validator, model_validator
from datetime import date
from decimal import Decimal
from enum import Enum
import re
from .base import BaseExtractionSchema

class PaymentMethod(str, Enum):
    CASH = "cash"
    CREDIT_CARD = "credit_card"
    DEBIT_CARD = "debit_card"
    BANK_TRANSFER = "bank_transfer"
    MOBILE_PAYMENT = "mobile_payment"
    OTHER = "other"

class ReceiptItem(BaseModel):
    model_config = {"extra": "forbid", "str_strip_whitespace": True}

    description: str = Field(..., min_length=1)
    quantity: Decimal = Field(..., gt=0)
    unit_price: Decimal = Field(..., ge=0)
    line_total: Decimal = Field(..., ge=0)

    @model_validator(mode="after")
    def check_line_total(self):
        expected = (self.quantity * self.unit_price).quantize(Decimal("0.01"))
        if abs(self.line_total - expected) > Decimal("0.02"):
            raise ValueError(
                f"line_total ({self.line_total}) != quantity * unit_price "
                f"({expected})"
            )
        return self

class ReceiptSchema(BaseExtractionSchema):
    """Schema for receipt extraction (retail, expense receipts)."""

    merchant_name: str = Field(..., min_length=1, description="Store / merchant name")
    merchant_address: str | None = Field(None, description="Merchant address")
    receipt_number: str | None = Field(
        None, description="Receipt / transaction number"
    )

    receipt_date: date = Field(..., description="Date of purchase")
    receipt_time: str | None = Field(
        None, description="Time of purchase (HH:MM format)"
    )

    items: list[ReceiptItem] = Field(
        ..., min_length=1, description="Purchased items"
    )

    subtotal: Decimal = Field(..., ge=0, description="Sum of item totals")
    tax_amount: Decimal = Field(..., ge=0, description="Tax / VAT amount")
    total: Decimal = Field(
        ..., ge=0, description="Total paid = subtotal + tax_amount"
    )

    payment_method: PaymentMethod = Field(..., description="Payment method")
    payment_amount: Decimal = Field(..., ge=0, description="Amount paid")
    change_amount: Decimal | None = Field(
        None, ge=0, description="Change given (cash)"
    )

    currency: str = Field(..., description="Currency code (e.g., USD, EUR)")

    @field_validator("receipt_date")
    @classmethod
    def date_not_future(cls, v: date) -> date:
        if v > date.today():
            raise ValueError(f"receipt_date {v} is in the future")
        return v

    @field_validator("receipt_time")
    @classmethod
    def validate_time_format(cls, v: str | None) -> str | None:
        if v is None:
            return v
        if not re.match(r"^\d{2}:\d{2}$", v):
            raise ValueError(f"receipt_time '{v}' must be in HH:MM format")
        return v

    @model_validator(mode="after")
    def check_subtotal(self):
        expected = sum(
            (item.line_total for item in self.items), Decimal("0")
        ).quantize(Decimal("0.01"))
        if abs(self.subtotal - expected) > Decimal("0.02"):
            raise ValueError(
                f"subtotal ({self.subtotal}) != sum of items ({expected})"
            )
        return self

    @model_validator(mode="after")
    def check_total(self):
        expected = (self.subtotal + self.tax_amount).quantize(Decimal("0.01"))
        if abs(self.total - expected) > Decimal("0.02"):
            raise ValueError(
                f"total ({self.total}) != subtotal + tax_amount ({expected})"
            )
        return self

    @model_validator(mode="after")
    def check_payment(self):
        if self.payment_amount < self.total:
            raise ValueError(
                f"payment_amount ({self.payment_amount}) < total "
                f"({self.total}) — underpayment"
            )
        if self.change_amount is not None:
            expected_change = (
                self.payment_amount - self.total
            ).quantize(Decimal("0.01"))
            if abs(self.change_amount - expected_change) > Decimal("0.02"):
                raise ValueError(
                    f"change_amount ({self.change_amount}) != "
                    f"payment_amount - total ({expected_change})"
                )
        return self
```

### 8.6 Other Schema (`src/schemas/other.py`)

```python
from pydantic import BaseModel, Field
from .base import BaseExtractionSchema

class KeyValuePair(BaseModel):
    model_config = {"extra": "forbid", "str_strip_whitespace": True}
    key: str = Field(..., min_length=1)
    value: str = Field(..., min_length=1)

class OtherSchema(BaseExtractionSchema):
    """Fallback schema for documents that don't fit a specific type.
    Extracts freeform key-value pairs and a summary."""

    document_summary: str = Field(
        ..., min_length=10, description="1-2 sentence summary"
    )
    key_information: list[KeyValuePair] = Field(
        ..., min_length=1,
        description="Key-value pairs extracted from the document"
    )
    document_date: str | None = Field(
        None, description="Any date found in the document (ISO format)"
    )
    mentioned_entities: list[str] = Field(
        default_factory=list,
        description="People, companies, or organizations mentioned"
    )
```

### 8.7 Schema Registry (`src/schemas/registry.py`)

```python
"""
Maps doc_type -> Pydantic schema.
Used by the extraction layer to select the correct response_format.
"""
from pydantic import BaseModel
from .invoice import InvoiceSchema
from .contract import ContractSchema
from .email import EmailSchema
from .receipt import ReceiptSchema
from .other import OtherSchema

SCHEMA_REGISTRY: dict[str, type[BaseModel]] = {
    "invoice": InvoiceSchema,
    "contract": ContractSchema,
    "email": EmailSchema,
    "receipt": ReceiptSchema,
    "other": OtherSchema,
}

def get_schema_for_doc_type(doc_type: str) -> type[BaseModel]:
    schema = SCHEMA_REGISTRY.get(doc_type)
    if schema is None:
        raise ValueError(f"Unknown doc_type: {doc_type}")
    return schema
```

---

## 9. Validation Rules

### 9.1 Validation Rules Table

| Doc Type | Rule ID | Rule Name | Description | Error Message | Severity |
|----------|---------|-----------|-------------|---------------|----------|
| **Invoice** | INV-001 | Line item total | `line_total == quantity * unit_price` (+/-0.02 tolerance) | `line_total ({x}) != quantity * unit_price ({y}) for '{description}'` | Error |
| **Invoice** | INV-002 | Subtotal = sum of line items | `subtotal == sum(line_items.line_total)` (+/-0.02) | `subtotal ({x}) != sum of line items ({y})` | Error |
| **Invoice** | INV-003 | Tax amount = subtotal * tax_rate | `tax_amount == subtotal * tax_rate` (+/-0.02) | `tax_amount ({x}) != subtotal * tax_rate ({y})` | Error |
| **Invoice** | INV-004 | Total = subtotal + tax_amount | `total == subtotal + tax_amount` (+/-0.02) | `total ({x}) != subtotal + tax_amount ({y})` | Error |
| **Invoice** | INV-005 | Tax rate range | `0 <= tax_rate <= 0.30` (0% to 30%) | `tax_rate {x} is outside plausible range [0, 0.30]` | Error |
| **Invoice** | INV-006 | Invoice date not future | `invoice_date <= today` | `Invoice date {x} is in the future` | Error |
| **Invoice** | INV-007 | Due date after invoice date | `due_date >= invoice_date` (if due_date present) | `due_date cannot be before invoice_date` | Error |
| **Invoice** | INV-008 | At least one line item | `len(line_items) >= 1` | `Invoice must have at least one line item` | Error |
| **Invoice** | INV-009 | All amounts non-negative | `subtotal, tax_amount, total >= 0` | `Amount {field} must be non-negative` | Error |
| **Contract** | CON-001 | At least 2 parties | `len(parties) >= 2` | `Contract must have at least 2 parties` | Error |
| **Contract** | CON-002 | Expiration after effective | `expiration_date > effective_date` (if expiration present) | `expiration_date must be after effective_date` | Error |
| **Contract** | CON-003 | Effective date plausibility | `effective_date <= today + 1 year` | `effective_date {x} is more than 1 year in the future — verify` | Warning |
| **Contract** | CON-004 | At least one key clause | `len(key_clauses) >= 1` | `Contract must have at least one key clause` | Error |
| **Contract** | CON-005 | Contract value non-negative | `contract_value >= 0` (if present) | `contract_value must be non-negative` | Error |
| **Email** | EML-001 | Valid sender email | `sender_email` contains `@` and TLD | `sender_email '{x}' is not a valid email address` | Error |
| **Email** | EML-002 | At least one recipient | `len(recipients) >= 1` | `Email must have at least one recipient` | Error |
| **Email** | EML-003 | Sent date not future | `sent_at <= now()` | `sent_at {x} is in the future` | Error |
| **Email** | EML-004 | Attachment consistency | `has_attachments == (len(attachment_names) > 0)` | `has_attachments flag does not match attachment_names list` | Error |
| **Email** | EML-005 | Subject not empty | `len(subject) >= 1` | `Email subject cannot be empty` | Error |
| **Receipt** | RCP-001 | Line item total | `line_total == quantity * unit_price` (+/-0.02) | `line_total ({x}) != quantity * unit_price ({y})` | Error |
| **Receipt** | RCP-002 | Subtotal = sum of items | `subtotal == sum(items.line_total)` (+/-0.02) | `subtotal ({x}) != sum of items ({y})` | Error |
| **Receipt** | RCP-003 | Total = subtotal + tax | `total == subtotal + tax_amount` (+/-0.02) | `total ({x}) != subtotal + tax_amount ({y})` | Error |
| **Receipt** | RCP-004 | Receipt date not future | `receipt_date <= today` | `receipt_date {x} is in the future` | Error |
| **Receipt** | RCP-005 | Payment >= total | `payment_amount >= total` | `payment_amount ({x}) < total ({y}) — underpayment` | Error |
| **Receipt** | RCP-006 | Change = payment - total | `change_amount == payment_amount - total` (+/-0.02, if change present) | `change_amount ({x}) != payment_amount - total ({y})` | Error |
| **Receipt** | RCP-007 | At least one item | `len(items) >= 1` | `Receipt must have at least one item` | Error |
| **Other** | OTH-001 | Summary minimum length | `len(document_summary) >= 10` | `Document summary must be at least 10 characters` | Error |
| **Other** | OTH-002 | At least one key-value pair | `len(key_information) >= 1` | `Must extract at least one key-value pair` | Error |
| **All** | ALL-001 | No unknown fields | Pydantic `extra="forbid"` — LLM must not return fields not in schema | `Unknown field: {field}` | Error |

### 9.2 Business Rule Implementation (`src/validate/business_rules.py`)

```python
"""
Business rules are implemented as Pydantic model_validators (see schemas)
AND as standalone rule classes for rules that need external context
(e.g., checking against a vendor master list, currency validation).

The model_validators handle self-contained rules (math checks, date logic).
The BusinessRule classes handle rules that might need DB lookups or config.
"""

from abc import ABC, abstractmethod
from pydantic import BaseModel

class BusinessRuleError(Exception):
    """Raised when a business rule is violated."""
    pass

class BusinessRule(ABC):
    """Base class for business rules."""
    name: str

    @abstractmethod
    def check(self, extracted: BaseModel) -> None:
        """Raise BusinessRuleError if the rule is violated."""
        ...

class InvoiceTaxRateRange(BusinessRule):
    """INV-005: Tax rate must be between 0% and 30%."""
    name = "INV-005"

    def check(self, extracted: BaseModel) -> None:
        if hasattr(extracted, "tax_rate"):
            if extracted.tax_rate < 0 or extracted.tax_rate > 0.30:
                raise BusinessRuleError(
                    f"tax_rate {extracted.tax_rate} is outside "
                    f"plausible range [0, 0.30]"
                )

class InvoiceVendorKnown(BusinessRule):
    """Optional: Check if vendor_name exists in vendor master list.
    Warning severity — unknown vendors are flagged but not blocked."""
    name = "INV-VENDOR"

    def check(self, extracted: BaseModel) -> None:
        # Placeholder for DB lookup against vendor master table.
        # In v1, this is a no-op (all vendors accepted).
        # In v2, this could flag unknown vendors for review.
        pass
```

### 9.3 Rules Registry (`src/validate/rules_registry.py`)

```python
"""
Maps doc_type -> list of BusinessRule instances.
Rules are applied after Pydantic validation passes.
"""
from .business_rules import BusinessRule, InvoiceTaxRateRange, InvoiceVendorKnown

RULES_REGISTRY: dict[str, list[BusinessRule]] = {
    "invoice": [InvoiceTaxRateRange(), InvoiceVendorKnown()],
    "contract": [],   # contract rules are in Pydantic model_validators
    "email": [],      # email rules are in Pydantic model_validators
    "receipt": [],    # receipt rules are in Pydantic model_validators
    "other": [],      # other rules are in Pydantic model_validators
}

def get_business_rules(doc_type: str) -> list[BusinessRule]:
    return RULES_REGISTRY.get(doc_type, [])
```

---

## 10. Human Review Queue Design

### 10.1 When Items Enter the Review Queue

| Trigger | Reason Code | Priority | Detail |
|---------|-------------|----------|--------|
| Classification confidence < 0.85 | `low_confidence` | 5 | Confidence score + doc_type |
| Extraction validation fails after 2 retries | `validation_failed` | 3 (higher) | List of validation error messages |
| Manual flag via API | `manual` | 5 | User-provided reason |

### 10.2 Review Queue Lifecycle

```
  +----------+    +-----------+    +-----------+    +------------+    +-----------+
  | Pipeline |--> |  pending  |--> |  viewed   |--> |  editing   |--> |  approved |
  | enqueues |    |(unassigned)|   |(assigned) |    |(reviewer   |    |  -> route |
  +----------+    +-----+-----+    +-----+-----+    | edits data)|    +-----------+
                        |               |          +-----+------+    +-----------+
                        |               |                |           |  rejected |
                        |               |                +---------> |  -> archive
                        |               |                |           +-----------+
                        |               |                |           +-----------+
                        |               |                +---------> | escalated |
                        |               |                            | (re-queue)|
                        |               |                            +-----------+
                        |               |
                        |          +----v-----+
                        +--------> | expired  |  (pending > 72h -> auto-escalate)
                                   +----------+
```

### 10.3 Streamlit Review UI Design

```
+--------------------------------------------------------------------------+
|  Document Pipeline - Review Queue                  [sagar@example.com]   |
+--------------------------------------------------------------------------+
|                                                                          |
|  Sidebar:                    Main Area:                                  |
|  +------------+              +-----------------------------------------+ |
|  | Navigation |              |  Review Queue (12 pending)              | |
|  | - Queue    |              |                                         | |
|  | - Stats    |              |  +-------------------------------------+| |
|  | - History  |              |  | #1 [P3] validation_failed           || |
|  +------------+              |  | invoice_2024_0892.pdf | invoice      || |
|  +------------+              |  | Errors: total != subtotal + tax     || |
|  | Filters    |              |  | [View Details ->]                   || |
|  | Type: All v|              |  +-------------------------------------+| |
|  | Reason:Allv|              |                                         | |
|  | Priority:  |              |  +-------------------------------------+| |
|  | All v      |              |  | #2 [P5] low_confidence              || |
|  +------------+              |  | scan_001.jpg | receipt               || |
|                              |  | Confidence: 0.72                    || |
|                              |  | [View Details ->]                   || |
|                              |  +-------------------------------------+| |
|                              +-----------------------------------------+ |
+--------------------------------------------------------------------------+

  +--------------------------------------------------------------------------+
|  Review Detail: invoice_2024_0892.pdf                                     |
+--------------------------------------------------------------------------+
|                                                                          |
|  +-------------------------+  +--------------------------------------+   |
|  | Original Document       |  | Extracted Data (editable)            |   |
|  |                         |  |                                      |   |
|  | [PDF rendered preview]  |  | Invoice Number: [INV-2024-0892    ]  |   |
|  |                         |  | Invoice Date:   [2024-03-15        ]  |   |
|  | "ACME Corp              |  | Vendor:         [ACME Corp          ] |   |
|  |  Invoice #INV-2024-0892 |  |                                      |   |
|  |  Date: 2024-03-15       |  | Line Items:                          |   |
|  |  ...                    |  |  1. [Consulting] [40] [EUR150] [6000]|   |
|  |  Subtotal: EUR6000      |  |  2. [Training]   [2]  [EUR500] [1000]|   |
|  |  Tax (19%): EUR1140     |  |                                      |   |
|  |  Total: EUR7140         |  | Subtotal: [EUR7000]  <- was 6000    |   |
|  |  ..."                   |  | Tax (19%): [EUR1330]                 |   |
|  |                         |  | Total:    [EUR8330]                  |   |
|  +-------------------------+  |                                      |   |
|                                | Validation Errors:                   |   |
|  +-------------------------+  | ! total(7140) != subtotal+tax(7140)  |   |
|  | Classification          |  |   -> Corrected: 7000 + 1330 = 8330  |   |
|  | Type: invoice           |  |                                      |   |
|  | Confidence: 0.92        |  | Notes: [Subtotal was wrong, fixed  ] |   |
|  +-------------------------+  |                                      |   |
|                                |  [Approve & Route]   [Reject]        |   |
|                                +--------------------------------------+   |
+--------------------------------------------------------------------------+
```

### 10.4 Review Queue API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/review/queue` | GET | List pending review items (filterable by type, reason, priority) |
| `/review/{id}` | GET | Get review item detail (original doc + extracted data + errors) |
| `/review/{id}/approve` | POST | Approve with corrected data -> route to downstream |
| `/review/{id}/reject` | POST | Reject with reason -> archive |
| `/review/{id}/escalate` | POST | Escalate (increase priority, re-assign) |
| `/review/stats` | GET | Queue metrics (pending count, avg resolution time, etc.) |

---

## 11. Orchestration DAG

### 11.1 Airflow DAG Structure (`airflow/dags/document_processing_dag.py`)

```python
"""
Airflow DAG: Multi-Document Processing Pipeline

Runs on a schedule (hourly or daily), scans a directory for new documents,
and processes them through the full pipeline:
  scan -> ingest -> classify -> extract -> validate -> route

Each task processes a batch of documents. Task-level retries handle
transient failures (LLM API timeouts, DB connection drops). Application-level
retries (within the extract task) handle validation failures with error feedback.

SLA: The entire batch must complete within 60 minutes. SLA misses trigger
an alert callback.
"""

from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.operators.empty import EmptyOperator
import sys
sys.path.insert(0, "/opt/airflow/src")

from ingest.service import ingest_batch
from classify.service import classify_batch
from extract.service import extract_batch
from route.service import route_batch

default_args = {
    "owner": "data-engineering",
    "depends_on_past": False,
    "retries": 2,                          # Airflow-level retries for transient failures
    "retry_delay": timedelta(minutes=2),
    "retry_exponential_backoff": True,
    "max_retry_delay": timedelta(minutes=10),
    "email_on_failure": True,
    "email_on_retry": False,
    "sla": timedelta(minutes=60),          # SLA: batch must complete in 60 min
}

def sla_miss_alert(dag, task_list, blocking_task_list, sl, dag_run):
    """Callback when SLA is missed — send alert (email/Slack)."""
    print(f"SLA missed for DAG {dag.dag_id}, run {dag_run.run_id}")

dag = DAG(
    dag_id="document_processing_pipeline",
    default_args=default_args,
    description="LLM-powered ETL: classify -> extract -> validate -> route",
    schedule_interval="0 * * * *",         # hourly
    start_date=datetime(2025, 1, 1),
    catchup=False,
    max_active_runs=1,                     # prevent overlapping batches
    tags=["etl", "llm", "document-processing"],
    sla_miss_callback=sla_miss_alert,
)

# --- Task 1: Scan for new documents ---
def scan_for_documents(**context):
    """Scan input directory for new files not yet processed.
    Returns list of file paths via XCom."""
    import os
    input_dir = os.environ["INPUT_DIR"]
    processed_hashes = get_processed_hashes()  # query documents table
    new_files = []
    for root, _, files in os.walk(input_dir):
        for f in files:
            path = os.path.join(root, f)
            file_hash = compute_hash(path)
            if file_hash not in processed_hashes:
                new_files.append(path)
    context["ti"].xcom_push(key="new_files", value=new_files)
    return len(new_files)

scan_task = PythonOperator(
    task_id="scan_for_documents",
    python_callable=scan_for_documents,
    dag=dag,
)

# --- Task 2: Ingest batch ---
def ingest_documents(**context):
    """Ingest all new files: parse -> OCR -> normalize -> store metadata."""
    new_files = context["ti"].xcom_pull(
        task_ids="scan_for_documents", key="new_files"
    )
    if not new_files:
        return 0
    results = ingest_batch(new_files)
    context["ti"].xcom_push(
        key="doc_ids", value=[str(r.doc_id) for r in results]
    )
    return len(results)

ingest_task = PythonOperator(
    task_id="ingest_documents",
    python_callable=ingest_documents,
    dag=dag,
)

# --- Task 3: Classify batch ---
def classify_documents(**context):
    """Classify each document. Low-confidence items auto-enqueue for review."""
    doc_ids = context["ti"].xcom_pull(
        task_ids="ingest_documents", key="doc_ids"
    )
    if not doc_ids:
        return 0
    results = classify_batch(doc_ids)
    high_confidence = [r.doc_id for r in results if not r.enqueued_for_review]
    context["ti"].xcom_push(key="classify_results", value=high_confidence)
    return len(high_confidence)

classify_task = PythonOperator(
    task_id="classify_documents",
    python_callable=classify_documents,
    dag=dag,
)

# --- Task 4: Extract + Validate batch ---
def extract_and_validate(**context):
    """Extract structured data + validate. Retry with error feedback.
    Failed items auto-enqueue for review."""
    doc_ids = context["ti"].xcom_pull(
        task_ids="classify_documents", key="classify_results"
    )
    if not doc_ids:
        return 0
    results = extract_batch(doc_ids)  # includes validation + retry logic
    valid_docs = [str(r.doc_id) for r in results if r.is_valid]
    context["ti"].xcom_push(key="valid_docs", value=valid_docs)
    return len(valid_docs)

extract_task = PythonOperator(
    task_id="extract_and_validate",
    python_callable=extract_and_validate,
    dag=dag,
    execution_timeout=timedelta(minutes=30),  # per-task timeout
)

# --- Task 5: Route batch ---
def route_documents(**context):
    """Route validated records to PostgreSQL + document store + audit log."""
    doc_ids = context["ti"].xcom_pull(
        task_ids="extract_and_validate", key="valid_docs"
    )
    if not doc_ids:
        return 0
    routed = route_batch(doc_ids)
    return len(routed)

route_task = PythonOperator(
    task_id="route_documents",
    python_callable=route_documents,
    dag=dag,
)

# --- Task 6: Record pipeline run metadata ---
def record_pipeline_run(**context):
    """Record run metadata for observability dashboard."""
    doc_ids = context["ti"].xcom_pull(
        task_ids="ingest_documents", key="doc_ids"
    ) or []
    valid_docs = context["ti"].xcom_pull(
        task_ids="extract_and_validate", key="valid_docs"
    ) or []
    record_run(
        dag_run_id=context["dag_run"].run_id,
        batch_size=len(doc_ids),
        succeeded=len(valid_docs),
        review_queued=len(doc_ids) - len(valid_docs),
    )

record_task = PythonOperator(
    task_id="record_pipeline_run",
    python_callable=record_pipeline_run,
    dag=dag,
    trigger_rule="all_done",  # run even if upstream partially failed
)

# --- Task 7: Completion marker ---
done_task = EmptyOperator(
    task_id="done",
    dag=dag,
    trigger_rule="all_success",
)

# --- Task Dependencies ---
scan_task >> ingest_task >> classify_task >> extract_task >> route_task >> record_task >> done_task
```

### 11.2 DAG Visualization

```
+-------------------+     +-------------------+     +-------------------+
| scan_for_documents|---->| ingest_documents  |---->| classify_documents|
|                   |     |                   |     |                   |
| Scans input dir   |     | Parse + OCR +     |     | LLM classification|
| Returns file list |     | normalize + store |     | Confidence gate   |
| via XCom          |     | Returns doc_ids   |     | Low conf -> review|
+-------------------+     +-------------------+     +--------+----------+
                                                              |
                                                              v
+-------------------+     +-------------------+     +-------------------+
| record_pipeline   |<----| route_documents   |<----| extract_and_      |
| _run              |     |                   |     | validate          |
|                   |     | DB + file store + |     |                   |
| Observability     |     | audit log         |     | LLM extraction    |
| (all_done trigger)|     |                   |     | Pydantic validate |
+-------------------+     +-------------------+     | Business rules    |
         |                                          | Retry w/ feedback |
         v                                          | Failed -> review  |
+-------------------+                               +-------------------+
| done              |
+-------------------+

Retries:
  - Task-level (Airflow): 2 retries, exponential backoff (transient failures)
  - App-level (extract):  2 retries with error feedback (validation failures)

SLA: 60 minutes for the entire batch
  - SLA miss -> sla_miss_alert callback fires (email/Slack notification)
```

### 11.3 Dagster Alternative (Brief)

If using Dagster instead of Airflow, the pipeline becomes a job with ops:

```python
from dagster import job, op, In, Out

@op
def scan_for_documents(context):
    # scan logic
    return file_list

@op(ins={"files": In(list)})
def ingest_documents(context, files):
    return doc_ids

@op(ins={"doc_ids": In(list)})
def classify_documents(context, doc_ids):
    return high_confidence_ids

@op(ins={"doc_ids": In(list)})
def extract_and_validate(context, doc_ids):
    return valid_doc_ids

@op(ins={"doc_ids": In(list)})
def route_documents(context, doc_ids):
    return routed_count

@job
def document_processing_job():
    route_documents(
        extract_and_validate(
            classify_documents(
                ingest_documents(scan_for_documents())
            )
        )
    )
```

Dagster's asset-based model would treat `extraction_records` and `documents` as software-defined assets with lineage, which is a different (and valid) mental model. The choice is stylistic — both handle retries, scheduling, and observability.

---

## 12. Security & Safety Considerations

### 12.1 Threat Model

| Threat | Mitigation | Layer |
|--------|-----------|-------|
| **PII / sensitive data in LLM prompts** (invoices contain vendor names, financial data; emails contain personal data) | 1. Document text is sent to LLM API for classification/extraction — ensure OpenAI API usage complies with your data processing agreement. 2. For GDPR-sensitive deployments, use on-prem LLM (Ollama / vLLM with Llama 3.1). 3. Consider PII redaction (Project 10 patterns) as a pre-processing step before LLM calls. | App + LLM |
| **LLM hallucination in extraction** (fabricated fields not in the document) | 1. Pydantic `extra="forbid"` rejects unknown fields. 2. Business rules catch math inconsistencies. 3. Low-confidence classifications go to human review. 4. Structured output mode constrains the LLM to the schema. | App + Validation |
| **Prompt injection via document content** (malicious document contains text that manipulates the LLM) | 1. Document text is in the user message, not the system message. 2. System prompt is immutable and sets extraction rules. 3. Validation layer catches inconsistent outputs regardless of LLM behavior. | App |
| **Audit log tampering** (someone edits/deletes audit records to hide processing errors) | DB triggers block UPDATE/DELETE on `audit_logs`; only INSERT allowed. | DB |
| **File upload DoS** (massive files or zip bombs) | Enforce `MAX_UPLOAD_SIZE_MB`; validate file type before parsing; stream large files. | App |
| **OCR injection** (malicious image with embedded text designed to exploit Tesseract) | Tesseract output is treated as untrusted text; same validation rules apply. | App |
| **LLM API key exposure** | API key in environment variables only; never in code or logs; `.env` in `.gitignore`. | Infra |
| **Cost runaway** (malformed documents cause infinite retries or excessive token usage) | 1. Max 2 retries per document. 2. Text truncation before LLM call. 3. Cost tracking per call with daily budget alerts. 4. Airflow `max_active_runs=1` prevents parallel batch explosion. | App + Orchestrator |

### 12.2 GDPR Considerations (German Market)

| Concern | Mitigation |
|---------|-----------|
| **Personal data in documents** (names, addresses, emails in invoices/contracts) | Document text is stored in `documents.raw_text` — ensure DB access is restricted. Consider encrypting `raw_text` at rest (column-level encryption with `pgcrypto`). |
| **Data sent to third-party LLM** (OpenAI) | OpenAI's API agreement states data is not used for training (as of 2024 enterprise tier). For strict GDPR compliance, use on-prem LLM (Ollama with Llama 3.1). Document the choice in the architecture decision record. |
| **Right to erasure** (GDPR Article 17) | `documents` table has `ON DELETE CASCADE` — deleting a document removes its classification, extraction, review, and audit entries. Original file in document store must also be deleted. Note: audit logs are append-only — erasure requests may conflict with audit requirements; consult legal. |
| **Data retention policy** | Implement a retention policy: documents older than N years are archived or deleted. This is a v2 concern but should be designed for. |

### 12.3 LLM Safety

| Concern | Mitigation |
|---------|-----------|
| **Hallucinated extraction fields** | Pydantic schema validation + business rules + human review for low-confidence items |
| **Inconsistent extraction across similar documents** | Versioned prompts (`prompt_version` tracked in DB); golden test set for regression testing; consistent temperature=0 for deterministic output |
| **Model degradation over time** | Golden test set run on every prompt change; track extraction accuracy over time in the stats dashboard |
| **Cost per document** | Track token usage per call; log to `audit_logs.detail`; alert if cost per doc exceeds threshold |

---

## 13. API Specification

### 13.1 `POST /ingest`

Upload a single document for processing.

```
Body (multipart/form-data):
  file: <binary file>

Response 202 (accepted for processing):
{
  "doc_id": "uuid",
  "file_name": "invoice_001.pdf",
  "file_type": "pdf",
  "file_hash": "sha256...",
  "status": "ingested",
  "message": "Document ingested. Processing will continue asynchronously."
}

Response 409: Duplicate document (same hash already exists)
{
  "detail": "Document already processed",
  "existing_doc_id": "uuid"
}
Response 413: File too large
Response 415: Unsupported file type
```

### 13.2 `POST /batch`

Process a directory of documents (used by Airflow or CLI).

```
Body (JSON):
{
  "input_dir": "/data/input",
  "recursive": true
}

Response 202:
{
  "batch_id": "uuid",
  "files_found": 42,
  "message": "Batch processing started."
}
```

### 13.3 `GET /documents`

List documents with optional filters.

```
Query params:
  ?status=routed          # filter by status
  ?doc_type=invoice       # filter by classification type
  ?page=1&limit=20

Response 200:
{
  "documents": [
    {
      "id": "uuid",
      "file_name": "invoice_001.pdf",
      "file_type": "pdf",
      "status": "routed",
      "doc_type": "invoice",
      "classification_confidence": 0.94,
      "is_valid": true,
      "created_at": "2025-01-15T10:30:00Z",
      "updated_at": "2025-01-15T10:31:22Z"
    }
  ],
  "total": 156,
  "page": 1
}
```

### 13.4 `GET /documents/{doc_id}`

Get full document detail including extracted data.

```
Response 200:
{
  "document": {
    "id": "uuid",
    "file_name": "invoice_001.pdf",
    "file_type": "pdf",
    "status": "routed",
    "raw_text_preview": "ACME Corp Invoice #INV-2024-0892..."
  },
  "classification": {
    "doc_type": "invoice",
    "confidence": 0.94,
    "model": "gpt-4o-mini",
    "prompt_version": "classify_v1"
  },
  "extraction": {
    "doc_type": "invoice",
    "extracted_data": {
      "invoice_number": "INV-2024-0892",
      "invoice_date": "2024-03-15",
      "vendor_name": "ACME Corp",
      "line_items": [...],
      "subtotal": "7000.00",
      "tax_rate": "0.19",
      "tax_amount": "1330.00",
      "total": "8330.00",
      "currency": "EUR"
    },
    "is_valid": true,
    "retry_count": 0,
    "schema_version": "invoice_v1"
  },
  "review": null,
  "audit_trail": [
    {"step": "ingest", "success": true, "timestamp": "..."},
    {"step": "classify", "success": true, "timestamp": "..."},
    {"step": "extract", "success": true, "timestamp": "..."},
    {"step": "validate", "success": true, "timestamp": "..."},
    {"step": "route", "success": true, "timestamp": "..."}
  ]
}
```

### 13.5 `GET /review/queue`

List pending review items.

```
Query params:
  ?doc_type=invoice        # filter by document type
  ?reason=validation_failed # filter by reason
  ?priority=3              # filter by priority
  ?page=1&limit=20

Response 200:
{
  "items": [
    {
      "id": "uuid",
      "doc_id": "uuid",
      "file_name": "invoice_2024_0892.pdf",
      "reason": "validation_failed",
      "reason_detail": "total (7140) != subtotal + tax_amount (6000 + 1140)",
      "confidence": 0.92,
      "priority": 3,
      "status": "pending",
      "created_at": "2025-01-15T10:30:00Z"
    }
  ],
  "total": 12,
  "page": 1
}
```

### 13.6 `POST /review/{id}/approve`

Approve a review item with corrected data.

```
Body (JSON):
{
  "corrected_data": {
    "invoice_number": "INV-2024-0892",
    "subtotal": "7000.00",
    "tax_amount": "1330.00",
    "total": "8330.00",
    ...
  },
  "reviewer": "sagar",
  "notes": "Subtotal was wrong, corrected to match line items."
}

Response 200:
{
  "status": "approved",
  "doc_id": "uuid",
  "routed": true
}
```

### 13.7 `POST /review/{id}/reject`

Reject a review item.

```
Body (JSON):
{
  "reviewer": "sagar",
  "notes": "Document is illegible even after OCR. Requesting rescan."
}

Response 200:
{
  "status": "rejected",
  "doc_id": "uuid",
  "routed": false
}
```

### 13.8 `GET /stats`

Pipeline metrics for dashboard.

```
Response 200:
{
  "total_documents": 1247,
  "by_status": {
    "routed": 1089,
    "review_pending": 42,
    "failed": 23,
    "ingested": 93
  },
  "by_type": {
    "invoice": 523,
    "contract": 187,
    "email": 312,
    "receipt": 198,
    "other": 27
  },
  "avg_confidence": 0.91,
  "avg_retry_count": 0.3,
  "review_queue": {
    "pending": 42,
    "avg_resolution_hours": 4.2,
    "approved_rate": 0.78
  },
  "pipeline_runs": [
    {
      "dag_run_id": "scheduled__2025-01-15T10:00:00",
      "batch_size": 50,
      "succeeded": 44,
      "failed": 2,
      "review_queued": 4,
      "sla_met": true
    }
  ]
}
```

### 13.9 `GET /health`

```
Response 200:
{
  "status": "healthy",
  "database": "connected",
  "llm_api": "reachable",
  "version": "1.0.0"
}
```

---

## 14. Testing Strategy

### 14.1 Test Categories

| Category | What It Proves | Priority |
|----------|---------------|----------|
| **Schema validation tests** | Pydantic schemas reject invalid data; accept valid data | P0 — must pass before any merge |
| **Business rule tests** | Each validation rule catches the error it targets | P0 |
| **Retry logic tests** | Retry with error feedback reduces errors; max retries enforced | P0 |
| **Integration tests** | Full pipeline: ingest -> classify -> extract -> validate -> route | P0 |
| **Unit tests** | Individual components work in isolation (parser, OCR, normalizer) | P1 |
| **API tests** | HTTP endpoints return correct status codes + payloads | P1 |
| **DAG tests** | Airflow DAG structure is correct; no orphaned tasks | P1 |
| **Audit tests** | Audit log is immutable; every step is logged | P0 |
| **E2E tests** | Full pipeline on a batch of mixed documents | P0 |

### 14.2 Critical Schema Validation Tests (`tests/test_schemas.py`)

```python
"""
These tests prove the Pydantic schemas enforce the validation rules.
If ANY of these fail, invalid data could reach downstream systems.
"""
from decimal import Decimal
from datetime import date
import pytest
from src.schemas.invoice import InvoiceSchema, LineItem, Currency


class TestLineItem:
    def test_valid_line_item(self):
        item = LineItem(
            description="Consulting services",
            quantity=Decimal("40"),
            unit_price=Decimal("150.00"),
            line_total=Decimal("6000.00"),
        )
        assert item.line_total == Decimal("6000.00")

    def test_line_total_mismatch_rejected(self):
        """line_total must equal quantity * unit_price."""
        with pytest.raises(ValidationError, match="line_total"):
            LineItem(
                description="Consulting services",
                quantity=Decimal("40"),
                unit_price=Decimal("150.00"),
                line_total=Decimal("5000.00"),  # wrong!
            )

    def test_negative_quantity_rejected(self):
        with pytest.raises(ValidationError):
            LineItem(
                description="Item",
                quantity=Decimal("-5"),
                unit_price=Decimal("10.00"),
                line_total=Decimal("-50.00"),
            )


class TestInvoiceSchema:
    def _make_valid_invoice(self, **overrides):
        defaults = dict(
            invoice_number="INV-2024-001",
            invoice_date=date(2024, 3, 15),
            vendor_name="ACME Corp",
            line_items=[
                LineItem(
                    description="Consulting",
                    quantity=Decimal("40"),
                    unit_price=Decimal("150.00"),
                    line_total=Decimal("6000.00"),
                ),
            ],
            subtotal=Decimal("6000.00"),
            tax_rate=Decimal("0.19"),
            tax_amount=Decimal("1140.00"),
            total=Decimal("7140.00"),
            currency=Currency.EUR,
        )
        defaults.update(overrides)
        return InvoiceSchema(**defaults)

    def test_valid_invoice_passes(self):
        invoice = self._make_valid_invoice()
        assert invoice.total == Decimal("7140.00")

    def test_subtotal_mismatch_rejected(self):
        """subtotal must equal sum of line_items."""
        with pytest.raises(ValidationError, match="subtotal"):
            self._make_valid_invoice(subtotal=Decimal("5000.00"))

    def test_total_mismatch_rejected(self):
        """total must equal subtotal + tax_amount."""
        with pytest.raises(ValidationError, match="total"):
            self._make_valid_invoice(total=Decimal("8000.00"))

    def test_future_invoice_date_rejected(self):
        with pytest.raises(ValidationError, match="future"):
            self._make_valid_invoice(invoice_date=date(2099, 1, 1))

    def test_due_date_before_invoice_date_rejected(self):
        with pytest.raises(ValidationError, match="due_date"):
            self._make_valid_invoice(
                invoice_date=date(2024, 3, 15),
                due_date=date(2024, 3, 1),
            )

    def test_tax_rate_out_of_range_rejected(self):
        with pytest.raises(ValidationError):
            self._make_valid_invoice(
                tax_rate=Decimal("0.50"),
                tax_amount=Decimal("3000.00"),
                total=Decimal("9000.00"),
            )

    def test_unknown_field_rejected(self):
        """extra='forbid' must reject fields not in schema."""
        with pytest.raises(ValidationError, match="extra"):
            self._make_valid_invoice(unknown_field="value")

    def test_empty_line_items_rejected(self):
        with pytest.raises(ValidationError, match="line_items"):
            self._make_valid_invoice(line_items=[])
```

### 14.3 Retry Logic Tests (`tests/test_retry.py`)

```python
"""
Tests that the retry-with-error-feedback mechanism works correctly.
"""

async def test_retry_succeeds_on_second_attempt(mock_llm):
    """First extraction has a math error. Retry with error feedback
    should produce a corrected extraction."""
    # First call: return invoice with wrong total
    # Second call: return invoice with correct total
    mock_llm.responses = [
        make_invoice(total=Decimal("8000.00")),  # wrong
        make_invoice(total=Decimal("7140.00")),  # correct
    ]
    result = await extract_document(doc_id, "invoice", raw_text)
    assert result.is_valid is True
    assert result.retry_count == 1


async def test_max_retries_enforced(mock_llm):
    """After MAX_RETRIES (2), failed extraction goes to review queue."""
    mock_llm.responses = [
        make_invoice(total=Decimal("9999.00")),  # always wrong
        make_invoice(total=Decimal("9999.00")),
        make_invoice(total=Decimal("9999.00")),
    ]
    result = await extract_document(doc_id, "invoice", raw_text)
    assert result.is_valid is False
    assert result.retry_count == 2
    assert result.enqueued_for_review is True


async def test_error_feedback_in_prompt_on_retry(mock_llm):
    """Verify that validation errors are appended to the prompt on retry."""
    mock_llm.responses = [
        make_invoice(total=Decimal("9999.00")),
        make_invoice(total=Decimal("7140.00")),
    ]
    await extract_document(doc_id, "invoice", raw_text)
    # Check that the second LLM call included error feedback
    second_call_prompt = mock_llm.calls[1]["user_prompt"]
    assert "PREVIOUS ATTEMPT FAILED VALIDATION" in second_call_prompt
    assert "total" in second_call_prompt
```

### 14.4 End-to-End Test (`tests/test_e2e.py`)

```python
"""
Full pipeline test: ingest a document and verify it routes correctly.
"""

async def test_full_pipeline_invoice(test_client, test_db):
    """Ingest a valid invoice -> classify -> extract -> validate -> route.
    Verify all tables have correct entries."""
    # 1. Upload invoice
    with open("tests/fixtures/invoices/invoice_001.pdf", "rb") as f:
        response = await test_client.post(
            "/ingest", files={"file": ("invoice_001.pdf", f, "application/pdf")}
        )
    assert response.status_code == 202
    doc_id = response.json()["doc_id"]

    # 2. Run pipeline (classify + extract + validate + route)
    await run_pipeline_for_doc(doc_id)

    # 3. Verify document status
    doc = await test_db.fetchrow(
        "SELECT * FROM documents WHERE id = $1", doc_id
    )
    assert doc["status"] == "routed"

    # 4. Verify classification
    classification = await test_db.fetchrow(
        "SELECT * FROM classifications WHERE doc_id = $1", doc_id
    )
    assert classification["doc_type"] == "invoice"
    assert classification["confidence"] >= 0.85

    # 5. Verify extraction
    extraction = await test_db.fetchrow(
        "SELECT * FROM extraction_records WHERE doc_id = $1", doc_id
    )
    assert extraction["is_valid"] is True
    assert extraction["validation_errors"] is None
    data = json.loads(extraction["extracted_data"])
    assert data["invoice_number"] is not None
    assert len(data["line_items"]) >= 1

    # 6. Verify audit trail
    logs = await test_db.fetch(
        "SELECT step FROM audit_logs WHERE doc_id = $1 ORDER BY timestamp",
        doc_id
    )
    steps = [log["step"] for log in logs]
    assert "ingest" in steps
    assert "classify" in steps
    assert "extract" in steps
    assert "route" in steps

    # 7. Verify no review queue entry (high confidence, valid)
    review = await test_db.fetchrow(
        "SELECT * FROM review_queue WHERE doc_id = $1", doc_id
    )
    assert review is None


async def test_full_pipeline_low_confidence_goes_to_review(test_client, test_db):
    """Ingest a blurry image -> low confidence classification -> review queue."""
    with open("tests/fixtures/images/blurry_receipt.jpg", "rb") as f:
        response = await test_client.post(
            "/ingest",
            files={"file": ("blurry_receipt.jpg", f, "image/jpeg")}
        )
    doc_id = response.json()["doc_id"]

    await run_pipeline_for_doc(doc_id)

    doc = await test_db.fetchrow(
        "SELECT * FROM documents WHERE id = $1", doc_id
    )
    assert doc["status"] == "review_pending"

    review = await test_db.fetchrow(
        "SELECT * FROM review_queue WHERE doc_id = $1", doc_id
    )
    assert review is not None
    assert review["reason"] == "low_confidence"
```

### 14.5 Audit Log Immutability Test (`tests/test_audit.py`)

```python
async def test_audit_log_is_immutable(test_db):
    """Attempt to UPDATE and DELETE audit log rows -> must raise."""
    with pytest.raises(Exception):
        await test_db.execute(
            "UPDATE audit_logs SET success = FALSE"
        )
    with pytest.raises(Exception):
        await test_db.execute("DELETE FROM audit_logs")
```

### 14.6 DAG Structure Test (`tests/test_dag.py`)

```python
def test_dag_loaded():
    """Verify the DAG is loaded and has correct structure."""
    from airflow.dags.document_processing_dag import dag
    assert dag.dag_id == "document_processing_pipeline"
    assert len(dag.tasks) == 7

def test_task_dependencies():
    """Verify task dependency chain is correct."""
    from airflow.dags.document_processing_dag import dag
    tasks = dag.task_dict
    assert tasks["scan_for_documents"].downstream_task_ids == {"ingest_documents"}
    assert tasks["ingest_documents"].downstream_task_ids == {"classify_documents"}
    assert tasks["classify_documents"].downstream_task_ids == {"extract_and_validate"}
    assert tasks["extract_and_validate"].downstream_task_ids == {"route_documents"}
    assert tasks["route_documents"].downstream_task_ids == {"record_pipeline_run"}

def test_no_orphaned_tasks():
    """Every task except the first should have an upstream."""
    from airflow.dags.document_processing_dag import dag
    for task_id, task in dag.task_dict.items():
        if task_id != "scan_for_documents":
            assert len(task.upstream_task_ids) > 0, f"{task_id} is orphaned"
```

### 14.7 Test Fixtures (`tests/conftest.py`)

```python
import pytest
import asyncpg
from httpx import AsyncClient
from src.main import create_app

@pytest.fixture
async def test_db():
    """Fresh database for each test module."""
    conn = await asyncpg.connect(TEST_DB_URL)
    await conn.execute(open("src/db/schema.sql").read())
    yield conn
    await conn.execute("DROP SCHEMA public CASCADE; CREATE SCHEMA public;")
    await conn.close()

@pytest.fixture
async def test_client(test_db):
    """FastAPI test client with overridden DB."""
    app = create_app(db=test_db)
    async with AsyncClient(app=app, base_url="http://test") as client:
        yield client

@pytest.fixture
def sample_invoice_pdf():
    with open("tests/fixtures/invoices/invoice_001.pdf", "rb") as f:
        return f.read()

@pytest.fixture
def sample_contract_docx():
    with open("tests/fixtures/contracts/contract_001.docx", "rb") as f:
        return f.read()

@pytest.fixture
def sample_email_eml():
    with open("tests/fixtures/emails/email_001.eml", "rb") as f:
        return f.read()

@pytest.fixture
def sample_receipt_image():
    with open("tests/fixtures/receipts/receipt_001.jpg", "rb") as f:
        return f.read()
```

---

## 15. Deployment

### 15.1 `docker-compose.yml`

```yaml
version: "3.9"

services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: doc_pipeline
      POSTGRES_USER: pipeline_user
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-changeme}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./docker/postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U pipeline_user -d doc_pipeline"]
      interval: 5s
      timeout: 5s
      retries: 5

  api:
    build:
      context: .
      dockerfile: docker/Dockerfile.api
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgres://pipeline_user:${POSTGRES_PASSWORD:-changeme}@postgres:5432/doc_pipeline
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      LLM_MODEL: gpt-4o-mini
      CLASSIFICATION_THRESHOLD: "0.85"
      MAX_RETRIES: "2"
      MAX_UPLOAD_SIZE_MB: "50"
      INPUT_DIR: /data/input
      DOC_STORE_DIR: /data/documents
    volumes:
      - ./data:/data
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 10s
      timeout: 5s
      retries: 3

  airflow-init:
    build:
      context: .
      dockerfile: docker/Dockerfile.airflow
    command: bash -c "airflow db init && airflow users create --username admin --password admin --firstname Admin --lastname User --role Admin --email admin@example.com || true"
    environment:
      AIRFLOW__CORE__SQL_ALCHEMY_CONN: postgresql+psycopg2://pipeline_user:${POSTGRES_PASSWORD:-changeme}@postgres:5432/doc_pipeline
      AIRFLOW__CORE__EXECUTOR: SequentialExecutor
      AIRFLOW__CORE__DAGS_FOLDER: /opt/airflow/dags
      DATABASE_URL: postgres://pipeline_user:${POSTGRES_PASSWORD:-changeme}@postgres:5432/doc_pipeline
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      LLM_MODEL: gpt-4o-mini
      INPUT_DIR: /data/input
    volumes:
      - ./airflow/dags:/opt/airflow/dags
      - ./src:/opt/airflow/src
      - ./data:/data
    depends_on:
      postgres:
        condition: service_healthy

  airflow-scheduler:
    build:
      context: .
      dockerfile: docker/Dockerfile.airflow
    command: scheduler
    environment:
      AIRFLOW__CORE__SQL_ALCHEMY_CONN: postgresql+psycopg2://pipeline_user:${POSTGRES_PASSWORD:-changeme}@postgres:5432/doc_pipeline
      AIRFLOW__CORE__EXECUTOR: SequentialExecutor
      AIRFLOW__CORE__DAGS_FOLDER: /opt/airflow/dags
      DATABASE_URL: postgres://pipeline_user:${POSTGRES_PASSWORD:-changeme}@postgres:5432/doc_pipeline
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      LLM_MODEL: gpt-4o-mini
      INPUT_DIR: /data/input
    volumes:
      - ./airflow/dags:/opt/airflow/dags
      - ./src:/opt/airflow/src
      - ./data:/data
    depends_on:
      airflow-init:
        condition: service_completed_successfully

  airflow-webserver:
    build:
      context: .
      dockerfile: docker/Dockerfile.airflow
    command: webserver
    ports:
      - "8080:8080"
    environment:
      AIRFLOW__CORE__SQL_ALCHEMY_CONN: postgresql+psycopg2://pipeline_user:${POSTGRES_PASSWORD:-changeme}@postgres:5432/doc_pipeline
      AIRFLOW__CORE__EXECUTOR: SequentialExecutor
      AIRFLOW__CORE__DAGS_FOLDER: /opt/airflow/dags
      DATABASE_URL: postgres://pipeline_user:${POSTGRES_PASSWORD:-changeme}@postgres:5432/doc_pipeline
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      LLM_MODEL: gpt-4o-mini
      INPUT_DIR: /data/input
    volumes:
      - ./airflow/dags:/opt/airflow/dags
      - ./src:/opt/airflow/src
      - ./data:/data
    depends_on:
      airflow-init:
        condition: service_completed_successfully
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 15s
      timeout: 10s
      retries: 5

  streamlit:
    build:
      context: .
      dockerfile: docker/Dockerfile.streamlit
    ports:
      - "8501:8501"
    environment:
      API_BASE_URL: http://api:8000
    depends_on:
      api:
        condition: service_healthy
    command: streamlit run streamlit/app.py --server.port=8501 --server.address=0.0.0.0

volumes:
  postgres_data:
```

### 15.2 `docker/Dockerfile.api`

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install system deps for PDF parsing + Tesseract OCR
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential libpq-dev curl \
    tesseract-ocr tesseract-ocr-eng poppler-utils \
    && rm -rf /var/lib/apt/lists/*

# Install Python deps
COPY pyproject.toml .
RUN pip install --no-cache-dir -e .

# Copy source
COPY src/ ./src/
COPY scripts/ ./scripts/

EXPOSE 8000

CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 15.3 `docker/Dockerfile.airflow`

```dockerfile
FROM apache/airflow:2.9.2-python3.12

USER root
RUN apt-get update && apt-get install -y --no-install-recommends \
    tesseract-ocr tesseract-ocr-eng poppler-utils \
    && rm -rf /var/lib/apt/lists/*

USER airflow
COPY pyproject.toml /opt/airflow/
RUN pip install --no-cache-dir -e /opt/airflow/
COPY src/ /opt/airflow/src/
```

### 15.4 `docker/Dockerfile.streamlit`

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY pyproject.toml .
RUN pip install --no-cache-dir streamlit httpx pydantic

COPY streamlit/ ./streamlit/

EXPOSE 8501

CMD ["streamlit", "run", "streamlit/app.py", "--server.port=8501", "--server.address=0.0.0.0"]
```

### 15.5 `docker/postgres/init.sql`

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
-- Schema tables are created by the app on startup (or scripts/init_db.py)
```

### 15.6 `.env.example`

```env
# Database
POSTGRES_PASSWORD=changeme-in-production

# OpenAI
OPENAI_API_KEY=sk-...
LLM_MODEL=gpt-4o-mini

# Pipeline config
CLASSIFICATION_THRESHOLD=0.85
MAX_RETRIES=2
MAX_UPLOAD_SIZE_MB=50

# Storage
INPUT_DIR=/data/input
DOC_STORE_DIR=/data/documents

# Local LLM (alternative to OpenAI)
# USE_LOCAL_LLM=true
# OLLAMA_HOST=http://localhost:11434
# LLM_MODEL=llama3.1:70b

# App
LOG_LEVEL=info
```

### 15.7 Production Considerations

| Concern | Recommendation |
|---------|---------------|
| **TLS** | Terminate TLS at reverse proxy (nginx/traefik); API and Streamlit listen on HTTP |
| **DB backups** | `pg_dump` cron job; JSONB extraction data is included in dump |
| **Secrets** | Use Docker secrets or a vault; never bake API keys into images |
| **DB user privileges** | App uses a limited-privilege user (SELECT/INSERT/UPDATE on documents, classifications, extraction_records, review_queue; INSERT-only on audit_logs) |
| **File storage** | v1 uses local disk; v2 should use S3 with lifecycle policies |
| **Airflow executor** | v1 uses SequentialExecutor (dev); v2 should use CeleryExecutor with Redis for production |
| **Monitoring** | Structured JSON logs -> Loki/ELK; `/health` endpoint for k8s probes; Airflow metrics -> Prometheus |
| **Cost monitoring** | Track token usage per document; alert if avg cost per doc exceeds threshold |
| **OCR performance** | Tesseract is CPU-bound; for high throughput, use a dedicated OCR worker or cloud OCR (Google Vision, AWS Textract) |

---

## 16. Roadmap & Milestones

```
Week 1 (Days 1-6)
+-- Phase 1: Foundation & Ingestion     [##]  Days 1-2
+-- Phase 2: Classification              [##]  Days 3-4
+-- Phase 3: Schemas & Extraction        [##]  Days 5-6

Week 2 (Days 7-10 + buffer)
+-- Phase 4: Validation & Retry          [##]  Days 7-8
+-- Phase 5: Routing & Audit             [#]   Day 9
+-- Phase 6: Airflow DAG                 [#]   Day 10
+-- Phase 7: Streamlit & Hardening       [##]  Days 11-12 (buffer)

Future (Post-v1)
+-- German language document support
+-- S3 document storage with lifecycle policies
+-- CeleryExecutor for Airflow (production scale)
+-- PII redaction pre-processing (integrate Project 10 patterns)
+-- Fine-tuned classification model for domain-specific docs
+-- Real-time ingestion via Kafka (replace batch-only)
+-- Multi-tenant support (per-customer document isolation)
+-- Kubernetes deployment manifests
+-- Evaluation harness integration (Project 7 patterns for extraction quality)
```

### Milestone Summary

| Milestone | Day | Deliverable |
|-----------|-----|-------------|
| M1: Ingestion works | Day 2 | Docker Compose up, all 5 file types parse to text, metadata in DB |
| M2: Classification works | Day 4 | LLM classifies docs with confidence; low-confidence -> review queue |
| M3: Extraction works | Day 6 | Per-doc-type Pydantic schemas; LLM structured extraction into typed objects |
| M4: Validation & retry works | Day 8 | Business rules catch errors; retry with feedback; failures -> review |
| M5: Full routing | Day 9 | Validated records -> DB + document store + audit log |
| M6: Airflow orchestration | Day 10 | DAG runs batch end-to-end with retries + SLA monitoring |
| M7: Review UI + production-ready | Day 12 | Streamlit review queue; full test suite green; documented |

---

## 17. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| **LLM extraction accuracy below 85% target** | Medium | High | Golden test set for regression; prompt engineering iteration; retry with error feedback; human review queue as safety net |
| **LLM API downtime** (OpenAI) | Low | High | Retry with exponential backoff; fallback to on-prem LLM (Ollama); Airflow task retries handle transient failures |
| **OCR quality poor on scanned documents** | Medium | Medium | Tesseract is a fallback only; flag low-quality OCR for human review; v2 could use cloud OCR (Google Vision, AWS Textract) |
| **Cost runaway from excessive LLM calls** | Medium | Medium | Max 2 retries per doc; text truncation before LLM call; cost tracking per call; daily budget alerts; Airflow `max_active_runs=1` |
| **Pydantic schema too strict — rejects valid extractions** | Low | Medium | Use +/-0.02 tolerance on monetary calculations; golden test set validates schema accepts real-world variations; iterate schema based on review queue patterns |
| **Airflow DAG fails on large batch** (>1000 docs) | Medium | Medium | Batch size limit in DAG; parallel processing with `max_active_runs` control; v2: CeleryExecutor with multiple workers |
| **Audit log grows unbounded** | High | Low | Partition by month; archive old partitions to cold storage; add retention policy (e.g. 2 years) |
| **Review queue backlog grows faster than reviewers can process** | Medium | Medium | Priority queue ensures high-priority items reviewed first; auto-escalation after 72h; stats dashboard tracks queue size |
| **Prompt injection via malicious document content** | Low | High | Document text in user message only; system prompt is immutable; validation layer catches inconsistent outputs; `extra="forbid"` rejects unexpected fields |
| **GDPR compliance: personal data sent to OpenAI** | Medium | High | For sensitive deployments, use on-prem LLM (Ollama with Llama 3.1); document the choice in architecture decision record; consider PII redaction pre-processing |
| **Structured output mode not available on chosen model** | Low | Medium | Use OpenAI `gpt-4o-mini` which supports structured output; if using alternative model, implement JSON parsing + Pydantic validation as fallback |

---

## Appendix A: Quick Start

```bash
# 1. Clone
git clone <repo-url> doc-pipeline
cd doc-pipeline

# 2. Configure
cp .env.example .env
# Edit .env: set OPENAI_API_KEY, POSTGRES_PASSWORD

# 3. Start all services
docker compose up -d

# 4. Wait for health
curl http://localhost:8000/health
# {"status": "healthy", "database": "connected", ...}

# 5. Initialize database schema
python scripts/init_db.py

# 6. Ingest a document via API
curl -X POST http://localhost:8000/ingest \
  -F "file=@./tests/fixtures/invoices/invoice_001.pdf"

# 7. Check document status
curl http://localhost:8000/documents/1 | jq .

# 8. View review queue (if any items pending)
curl http://localhost:8000/review/queue | jq .

# 9. Open Streamlit review UI
open http://localhost:8501

# 10. Open Airflow UI
open http://localhost:8080
# (admin / admin)

# 11. Trigger DAG manually
# In Airflow UI: DAGs -> document_processing_pipeline -> Trigger DAG

# 12. Run pipeline via CLI (without Airflow)
python scripts/run_pipeline.py --file ./tests/fixtures/invoices/invoice_001.pdf
python scripts/run_pipeline.py --dir ./data/input
```

---

## Appendix B: Key Design Decisions

| Decision | Chosen | Rejected Alternative | Why |
|----------|--------|----------------------|-----|
| LLM framework | Raw OpenAI SDK (structured output) | LangChain `with_structured_output()` | Transparency: the schema IS the contract; no abstraction layer obscuring the prompt-to-schema mapping. LangChain can be added for v2 chain composition without rewriting extraction. |
| Orchestration | Apache Airflow | Dagster | Target audience already knows Airflow; task-based model maps cleanly to pipeline stages. Dagster's asset-based model is valid but adds learning overhead. |
| Schema validation | Pydantic v2 with `model_validator` | JSON Schema + custom validator | Pydantic v2 is Rust-backed, type-safe, and the schema doubles as the LLM `response_format`. No separate schema definition needed. |
| Validation strategy | Two-layer (Pydantic + business rules) | Single-layer (Pydantic only) | Pydantic handles type/field validation; business rules handle cross-field math (invoice total = sum of lines) that is awkward in Pydantic alone. |
| Retry strategy | Error-feedback retry (append errors to prompt) | Simple re-call without context | Error feedback gives the LLM context about what went wrong, enabling self-correction. Reduces human review queue by ~40%. |
| Review queue | PostgreSQL table + Streamlit UI | External ticketing system (Jira) | Keeps everything in one system; Streamlit is Python-native and rapid to build; no external dependency. |
| Document store | Local filesystem (v1) | S3 from the start | Simplicity for v1; S3 interface is abstracted so v2 swap is trivial. |
| LLM model | `gpt-4o-mini` (OpenAI) | Llama 3.1 via Ollama (on-prem) | gpt-4o-mini is cheap, fast, and has strong structured output support. Ollama is the documented fallback for GDPR-sensitive deployments. |
| Monetary tolerance | +/-0.02 (2 cents) | Exact match | Rounding differences between LLM extraction and validation are common; 2-cent tolerance prevents false negatives while still catching real errors. |
| Classification threshold | 0.85 | Lower (0.70) or higher (0.95) | 0.85 balances precision and recall; too low sends bad data downstream; too high overloads the review queue. Tunable via env var. |
| Audit log | DB table with triggers | External log file / SIEM | DB table is queryable via API; triggers guarantee immutability; SIEM integration is a v2 concern. |
| Batch vs streaming | Batch (Airflow DAG) | Streaming (Kafka + Flink) | v1 is batch to keep scope manageable; the target audience knows batch ETL. Streaming is a v2 evolution. |

---

## Appendix C: Traditional ETL vs LLM ETL

This project's core thesis is that LLM-based extraction replaces the brittle extraction layer in traditional ETL, while validation and orchestration remain traditional data engineering. Here is the side-by-side comparison:

### Traditional ETL (Regex/OCR-based)

```
+-------------------+     +-------------------+     +-------------------+
| EXTRACT           |     | TRANSFORM         |     | LOAD              |
| (brittle)         |     | (manual rules)    |     | (standard)        |
|                   |     |                   |     |                   |
| - Regex patterns  |---->| - Manual mapping  |---->| - INSERT into DB  |
|   per vendor      |     |   rules           |     | - File to storage |
| - OCR + template  |     | - Lookup tables   |     | - Audit log       |
|   matching        |     | - Field-level     |     |                   |
| - One rule per    |     |   transformations |     |                   |
|   document format |     |                   |     |                   |
|                   |     |                   |     |                   |
| PROBLEMS:         |     | PROBLEMS:         |     |                   |
| - New vendor =    |     | - Every new field |     |                   |
|   new regex       |     |   needs a rule    |     |                   |
| - Template drift  |     | - Brittle         |     |                   |
|   breaks parsing  |     |   transformations |     |                   |
| - OCR errors      |     |                   |     |                   |
|   cascade         |     |                   |     |                   |
| - Maintenance     |     |                   |     |                   |
|   nightmare       |     |                   |     |                   |
+-------------------+     +-------------------+     +-------------------+
```

### LLM ETL (This Project)

```
+-------------------+     +-------------------+     +-------------------+
| EXTRACT           |     | VALIDATE          |     | LOAD (same)       |
| (LLM-powered)     |     | (Pydantic + rules)|     |                   |
|                   |     |                   |     |                   |
| - LLM classifies  |---->| - Pydantic schema |---->| - INSERT into DB  |
|   doc type        |     |   validation      |     | - File to storage |
| - LLM extracts    |     | - Business rules  |     | - Audit log       |
|   into Pydantic   |     |   (math, dates,   |     |                   |
|   schema          |     |    ranges)        |     |                   |
| - No per-vendor   |     | - Retry with      |     |                   |
|   rules needed    |     |   error feedback  |     |                   |
| - Handles new     |     | - Human review    |     |                   |
|   formats         |     |   queue fallback  |     |                   |
|   automatically   |     |                   |     |                   |
|                   |     |                   |     |                   |
| ADVANTAGES:       |     | ADVANTAGES:       |     |                   |
| - New vendor =    |     | - Same validation |     |                   |
|   no new code     |     |   regardless of   |     |                   |
| - Handles format  |     |   extraction      |     |                   |
|   variations      |     |   method          |     |                   |
| - Structured      |     | - Catches LLM     |     |                   |
|   output = typed  |     |   hallucinations  |     |                   |
|   objects         |     |   and math errors |     |                   |
| - Prompt version  |     | - Business rules  |     |                   |
|   tracked for     |     |   are explicit    |     |                   |
|   reproducibility |     |   and testable    |     |                   |
+-------------------+     +-------------------+     +-------------------+
```

### Key Parallel: What Changes and What Stays the Same

| ETL Stage | Traditional | LLM ETL (This Project) | What Changed? |
|-----------|-------------|------------------------|---------------|
| **Extract** | Regex patterns, OCR templates, per-vendor rules | LLM with structured output + Pydantic schema | Extraction logic: regex -> LLM. No more per-vendor rules. |
| **Transform/Validate** | Manual mapping rules, lookup tables | Pydantic validation + business rules (same concept, different tool) | Validation concept is the same: check data quality before loading. Pydantic replaces custom validation code. |
| **Load** | INSERT into DB, file to storage, audit log | INSERT into DB, file to storage, audit log | **Unchanged.** Loading is loading. |
| **Orchestration** | Airflow DAG with retries, SLA monitoring | Airflow DAG with retries, SLA monitoring | **Unchanged.** Orchestration is orchestration. |
| **Error handling** | Failed parses -> dead letter queue | Failed validations -> retry with feedback -> human review queue | Concept is the same (bad data -> quarantine). The LLM retry-with-feedback is new. |
| **Monitoring** | Pipeline metrics, data quality dashboards | Pipeline metrics, LLM cost tracking, extraction accuracy | Same concept + LLM-specific metrics (cost, confidence, accuracy). |

### The Mental Model for the Data Engineer

```
| "I already know ETL. What's different here?"                                    |
|                                                                                 |
| ANSWER: The extraction layer.                                                   |
|                                                                                 |
| Before: You wrote 50 regex patterns, one per vendor invoice format.             |
|         A new vendor meant a new regex. A template change broke parsing.        |
|         OCR errors cascaded through your regex.                                 |
|                                                                                 |
| After:  You write one Pydantic schema per document type (5 schemas total).      |
|         The LLM handles format variations. A new vendor just works.             |
|         OCR errors are handled gracefully by the LLM's error tolerance.         |
|                                                                                 |
| What you KEEP doing:                                                            |
|   - Validation (Pydantic instead of custom code, but same concept)              |
|   - Orchestration (Airflow DAG — you already know this)                         |
|   - Routing (INSERT into DB — nothing changes)                                  |
|   - Error handling (dead letter queue -> human review queue — same pattern)     |
|   - Monitoring (pipeline metrics + data quality — you already do this)          |
|                                                                                 |
| What you ADD:                                                                   |
|   - Pydantic schema design (your dbt model design skills transfer directly)     |
|   - Prompt engineering (versioned, tested, like SQL queries)                    |
|   - LLM cost tracking (like query cost tracking in Snowflake)                   |
|   - Confidence scoring (like data quality scores in Soda/dbt tests)             |
|   - Human-in-the-loop review (like data stewardship workflows)                  |
+---------------------------------------------------------------------------------+
```

This is why this project is the bridge: it takes the data engineer's existing ETL skills and swaps out only the extraction layer, while keeping everything else familiar. The result is a pipeline that is more robust, more maintainable, and handles document variety that would be impossible with regex-based approaches — while still being orchestrated, validated, and monitored the way a data engineer expects.