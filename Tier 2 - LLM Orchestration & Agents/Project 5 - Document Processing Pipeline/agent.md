# Herdr Multi-Agent Orchestration Guide — Document Processing Pipeline

> **Project:** Multi-Document Processing Pipeline (LLM-powered ETL)
> **Plan:** `IMPLEMENTATION_PLAN.md` — 7 phases, 10 days + 2 buffer days
> **Stack:** Python 3.11+, FastAPI, Pydantic v2, OpenAI SDK (structured output), PostgreSQL 16, Airflow 2.9+, Streamlit, Docker Compose

---

## 1. Agent Roster

| Agent | Kind | Responsibility | Owns |
|-------|------|----------------|------|
| **pipeline** | codex | Core pipeline path: ingestion (PDF/DOCX/TXT/EML + OCR), LLM classification, structured extraction, retry logic, LLM client, audit logger, DB layer, config, FastAPI app shell, ingest/batch/health API routes | `src/ingest/`, `src/classify/`, `src/extract/`, `src/llm/`, `src/audit/`, `src/db/`, `src/config.py`, `src/main.py`, `src/api/dependencies.py`, `src/api/routes/ingest.py`, `src/api/routes/batch.py`, `src/api/routes/health.py`, `tests/test_ingest.py`, `tests/test_classify.py`, `tests/test_extract.py`, `tests/test_retry.py`, `tests/conftest.py` |
| **schemas** | codex | Pydantic v2 schemas for all document types (Invoice, Contract, Email, Receipt, Other), schema registry, validation layer (Pydantic wrapper + business rules + rules registry), schema/validation tests | `src/schemas/`, `src/validate/`, `tests/test_schemas.py`, `tests/test_validate.py` |
| **orchestration** | codex | Airflow DAG, Streamlit review UI, routing layer, review queue CRUD + API, document/stats API routes, scripts, Docker infrastructure, docker-compose, pyproject.toml, routing/audit/DAG/E2E/API/review-queue tests, test fixtures | `airflow/`, `streamlit/`, `src/route/`, `src/review/`, `src/api/routes/review.py`, `src/api/routes/documents.py`, `src/api/routes/stats.py`, `scripts/`, `docker/`, `docker-compose.yml`, `.env.example`, `pyproject.toml`, `tests/test_route.py`, `tests/test_audit.py`, `tests/test_dag.py`, `tests/test_e2e.py`, `tests/test_review_queue.py`, `tests/test_api.py`, `tests/fixtures/` |
| **reviewer** | codex | Code review after each phase: correctness, adherence to plan specs, file ownership boundaries, test coverage, no stubs/placeholders | _(no permanent file ownership — reviews diffs)_ |

---

## 2. Pane Layout

```
+---------------------------+---------------------------+
|                           |                           |
|        pipeline           |         schemas           |
|   ingest / classify /     |   Pydantic models /       |
|   extract / llm / audit   |   validation rules        |
|                           |                           |
+---------------------------+---------------------------+
|                           |                           |
|     orchestration         |        reviewer           |
|   Airflow / Streamlit /   |   code review /           |
|   route / review queue    |   phase gate checks       |
|                           |                           |
+---------------------------+---------------------------+
```

---

## 3. Setup Commands

```bash
# --- Create 4 panes in a 2x2 grid ---
# Pane 1: pipeline (top-left)
herdr pane split --cwd "$PWD" --no-focus

# Pane 2: schemas (top-right) — split right from pane 1
herdr pane split --direction right --cwd "$PWD" --no-focus

# Pane 3: orchestration (bottom-left) — split down from pane 1
herdr pane focus 1
herdr pane split --direction down --cwd "$PWD" --no-focus

# Pane 4: reviewer (bottom-right) — split down from pane 2
herdr pane focus 2
herdr pane split --direction down --cwd "$PWD" --no-focus

# --- Start agents ---
herdr agent start pipeline  --kind codex --pane 1
herdr agent start schemas   --kind codex --pane 2
herdr agent start orchestration --kind codex --pane 3
herdr agent start reviewer  --kind codex --pane 4
```

> **Pane IDs:** `1` = pipeline, `2` = schemas, `3` = orchestration, `4` = reviewer.
> Adjust IDs if your Herdr session numbers differently — verify with `herdr pane list`.

---

## 4. Phase-by-Phase Orchestration

### Phase 1: Foundation & Ingestion (Days 1-2)

**Goal:** Docker Compose stack running (Postgres + API), document parsing for all formats, files stored with metadata.

**Plan reference:** Section 6 Phase 1 (steps 1.1–1.11), Section 7.1 (Ingestion Service pseudocode), Section 4 (Database Schema DDL), Section 15 (Deployment/Docker).

#### Parallel Work

**orchestration** (infra setup — no dependencies):
```
Read IMPLEMENTATION_PLAN.md Section 15 (Deployment) and Section 4 (Database Schema).

Create the project infrastructure:
1. `pyproject.toml` — Python 3.11+, deps: fastapi, uvicorn, asyncpg, pydantic, pydantic-settings, openai, tenacity, pypdf, python-docx, mailparser, pytesseract, Pillow, streamlit, httpx, apache-airflow, ruff, pytest, pytest-asyncio. Use src/ layout.
2. `docker-compose.yml` — follow Section 15.1 exactly: services postgres (image postgres:16, healthcheck), api (build from docker/Dockerfile.api, port 8000, env vars from Section 15.1), airflow-init, airflow-scheduler, airflow-webserver (port 8080), streamlit (port 8501). Include `postgres_data` volume.
3. `docker/Dockerfile.api` — follow Section 15.2: python:3.12-slim, install tesseract-ocr + poppler-utils, pip install -e ., CMD uvicorn.
4. `docker/Dockerfile.airflow` — follow Section 15.3: apache/airflow:2.9.2-python3.12, install tesseract + poppler, copy src + pyproject.
5. `docker/Dockerfile.streamlit` — follow Section 15.4: python:3.12-slim, pip install streamlit httpx pydantic, copy streamlit/.
6. `docker/postgres/init.sql` — follow Section 15.5: CREATE EXTENSION uuid-ossp, pgcrypto.
7. `.env.example` — follow Section 15.6: POSTGRES_PASSWORD, OPENAI_API_KEY, LLM_MODEL, CLASSIFICATION_THRESHOLD, MAX_RETRIES, MAX_UPLOAD_SIZE_MB, INPUT_DIR, DOC_STORE_DIR, LOG_LEVEL.

Do NOT create src/ files — those are owned by pipeline and schemas agents.
```

**pipeline** (ingestion + DB + config — no dependencies on other agents):
```
Read IMPLEMENTATION_PLAN.md Section 4 (Database Schema — full DDL for documents, classifications, extraction_records, review_queue, audit_logs, pipeline_runs tables + indexes + triggers), Section 7.1 (Ingestion Service pseudocode), Section 7.8 (Audit Logger pseudocode), Section 5 (Project Structure).

Create the foundation and ingestion layer:
1. `src/__init__.py` — empty.
2. `src/config.py` — Pydantic Settings class reading env vars: DATABASE_URL, OPENAI_API_KEY, LLM_MODEL (default gpt-4o-mini), CLASSIFICATION_THRESHOLD (default 0.85), MAX_RETRIES (default 2), MAX_UPLOAD_SIZE_MB (default 50), INPUT_DIR, DOC_STORE_DIR, LOG_LEVEL.
3. `src/db/__init__.py` — empty.
4. `src/db/connection.py` — asyncpg connection pool. Init pool from settings.DATABASE_URL. Provide `get_db()` dependency for FastAPI and a module-level pool accessor.
5. `src/db/schema.sql` — Full DDL from Section 4.2: all 6 tables (documents, classifications, extraction_records, review_queue, audit_logs, pipeline_runs) with all indexes, the `prevent_audit_modification()` function, and both triggers (no_audit_update, no_audit_delete).
6. `src/db/migrations/001_initial.sql` — same DDL as schema.sql (for migration tooling).
7. `src/db/migrations/002_audit_triggers.sql` — the audit trigger function + triggers.
8. `src/audit/__init__.py` — empty.
9. `src/audit/logger.py` — follow Section 7.8 pseudocode: async `log()` function inserting into audit_logs (doc_id, step, success, duration_ms, detail JSONB, error_message). Use the DB pool from connection.py.
10. `src/ingest/__init__.py` — empty.
11. `src/ingest/parser.py` — `parse_file(file_content, file_name, file_type) -> str`. Use pypdf for PDF, python-docx for DOCX, raw decode for TXT, mailparser for EML. Each returns plain text. Include `detect_file_type(file_name, file_content) -> str` returning 'pdf'|'docx'|'txt'|'eml'|'image'.
12. `src/ingest/ocr.py` — `ocr_extract(file_content) -> str` using pytesseract + Pillow. Import pytesseract, open image via PIL.Image.open(io.BytesIO(file_content)), run pytesseract.image_to_string().
13. `src/ingest/normalizer.py` — `normalize_text(text) -> str`: collapse excessive whitespace, fix encoding, strip trailing spaces per line.
14. `src/ingest/service.py` — follow Section 7.1 pseudocode exactly: `async def ingest_document(file_content, file_name, source="api") -> IngestResult`. Steps: hash for dedup (SHA-256, check documents table), detect file type, extract text (OCR for images, parse_file otherwise), normalize, store original file, insert into documents table, audit log. Define IngestResult dataclass (doc_id, raw_text, file_type). Define DuplicateDocumentError. Also write `async def ingest_batch(file_paths) -> list[IngestResult]` for Airflow (reads files from disk, calls ingest_document for each).
15. `src/main.py` — FastAPI app factory `create_app()` with lifespan (init DB pool on startup, close on shutdown). Include API routers.
16. `src/api/__init__.py`, `src/api/routes/__init__.py` — empty.
17. `src/api/dependencies.py` — FastAPI dependency injection: `get_db()` yields a connection from the pool, `get_settings()` returns config.
18. `src/api/routes/ingest.py` — `POST /ingest` endpoint: accept multipart file upload, enforce MAX_UPLOAD_SIZE_MB, call ingest_document, return 202 with doc_id/file_name/file_type/file_hash/status. Return 409 for duplicates, 413 for too large, 415 for unsupported type. Follow Section 13.1 response format.
19. `src/api/routes/health.py` — `GET /health`: check DB connectivity, return {"status":"healthy","database":"connected","llm_api":"reachable","version":"1.0.0"}. Follow Section 13.9.
20. `src/api/routes/batch.py` — `POST /batch`: accept JSON {input_dir, recursive}, scan directory, return 202 with batch_id/files_found. Follow Section 13.2.
21. `tests/conftest.py` — follow Section 14.7: pytest fixtures test_db (asyncpg connect, run schema.sql, yield, drop schema), test_client (AsyncClient with create_app), sample fixtures for each doc type (sample_invoice_pdf, sample_contract_docx, sample_email_eml, sample_receipt_image). Create test fixture files in tests/fixtures/ (minimal valid PDF, DOCX, TXT, EML, and a small image).
22. `tests/test_ingest.py` — test each parser (PDF, DOCX, TXT, EML), OCR, normalizer, ingest_document end-to-end (file -> documents table row with raw_text), duplicate detection (same hash -> DuplicateDocumentError), file type detection. Follow Section 14 testing strategy.

Create test fixture files in tests/fixtures/invoices/, contracts/, emails/, receipts/, images/ — at least one minimal valid file per format.
```

**schemas** (base schema only — no dependencies):
```
Read IMPLEMENTATION_PLAN.md Section 8.1 (Base Schema).

Create the base Pydantic schema:
1. `src/schemas/__init__.py` — empty.
2. `src/schemas/base.py` — follow Section 8.1 exactly: BaseExtractionSchema(BaseModel) with doc_id: UUID|None=None, extracted_at: datetime|None=None, model_config = {"extra":"forbid", "str_strip_whitespace":True}.

This is the foundation that all other schemas (Phase 3) will inherit from. Do NOT create other schemas yet — those are Phase 3.
```

#### Sequential (after parallel)

Nothing sequential in Phase 1 — all three agents work independently.

#### Review

**reviewer**:
```
Review Phase 1 deliverables against IMPLEMENTATION_PLAN.md:

1. Check docker-compose.yml matches Section 15.1 — all 6 services (postgres, api, airflow-init, airflow-scheduler, airflow-webserver, streamlit), correct ports, healthchecks, env vars, volume mounts.
2. Check src/db/schema.sql matches Section 4.2 — all 6 tables with correct columns, types, constraints, indexes, audit triggers (prevent_audit_modification function + no_audit_update + no_audit_delete triggers).
3. Check src/ingest/service.py follows Section 7.1 pseudocode — hash dedup, file type detection, text extraction (OCR for images), normalization, document store save, DB insert, audit log.
4. Check src/audit/logger.py follows Section 7.8 — append-only insert into audit_logs.
5. Check src/api/routes/ingest.py returns 202/409/413/415 per Section 13.1.
6. Check src/schemas/base.py matches Section 8.1 — extra="forbid", str_strip_whitespace=True.
7. Verify no stubs, TODOs, or placeholder code.
8. Verify file ownership: pipeline owns src/ingest/, src/audit/, src/db/, src/config.py, src/main.py, src/api/routes/ingest.py|batch.py|health.py; schemas owns src/schemas/base.py; orchestration owns docker/, docker-compose.yml, pyproject.toml, .env.example.
9. Run: docker compose config (validate compose file), python -c "from src.schemas.base import BaseExtractionSchema" (validate base schema imports).
```

#### Commit

```bash
git add -A && git commit -m "Phase 1: Foundation, Docker stack, ingestion (PDF/DOCX/TXT/EML/OCR), DB schema, audit logger, base schema"
```

---

### Phase 2: Classification (Days 3-4)

**Goal:** LLM classifies each document with confidence score; low-confidence items enqueued for review.

**Plan reference:** Section 6 Phase 2 (steps 2.1–2.8), Section 7.2 (Classification Service pseudocode), Section 7.7 (LLM Client pseudocode), Section 7.6 (Review Queue pseudocode — enqueue only), Section 10 (Review Queue Design).

#### Parallel Work

**pipeline** (LLM client + classification — depends on Phase 1 DB/config):
```
Read IMPLEMENTATION_PLAN.md Section 7.7 (LLM Client pseudocode), Section 7.2 (Classification Service pseudocode), Section 6 Phase 2 steps 2.1-2.4.

Create the LLM client and classification layer:
1. `src/llm/__init__.py` — empty.
2. `src/llm/cost_tracker.py` — CostTracker class: `record(usage) -> CostRecord` that logs prompt_tokens, completion_tokens, total_tokens, estimated cost (based on gpt-4o-mini pricing: $0.15/1M input, $0.60/1M output). CostRecord is a dataclass with token counts + cost.
3. `src/llm/client.py` — follow Section 7.7 pseudocode: LLMClient class wrapping OpenAI client. `__init__(api_key, model, timeout=60)`. `structured_complete(response_format, system_prompt, user_prompt) -> tuple[BaseModel, CostRecord]` with tenacity retry (stop_after_attempt(3), wait_exponential(min=2,max=30), retry on RateLimitError/APITimeoutError/APIConnectionError). Uses `client.beta.chat.completions.parse()` with Pydantic response_format. Also provide an async wrapper `async structured_complete_async` for use in async services.
4. `src/classify/__init__.py` — empty.
5. `src/classify/prompts.py` — versioned classification prompt. CLASSIFICATION_PROMPT (system message instructing LLM to classify as invoice/contract/email/receipt/other with confidence 0.0-1.0). CLASSIFICATION_PROMPT_VERSION = "classify_v1". Define ClassificationResponse(BaseModel) with doc_type: str (enum: invoice/contract/email/receipt/other) and confidence: float (0.0-1.0).
6. `src/classify/classifier.py` — `async def classify(doc_id, raw_text) -> ClassificationResult`. Truncate text to ~12000 tokens (keep head + tail). Call llm_client.structured_complete with ClassificationResponse schema. Return doc_type + confidence.
7. `src/classify/service.py` — follow Section 7.2 pseudocode exactly: `async def classify_document(doc_id, raw_text) -> ClassificationResult`. Steps: truncate text, call LLM, store in classifications table (id, doc_id, doc_type, confidence, model, prompt_version, raw_response), update documents.status='classified', audit log, confidence gate (if confidence < CLASSIFICATION_THRESHOLD=0.85, enqueue for review with reason='low_confidence'). Define ClassificationResult dataclass (doc_type, confidence, enqueued_for_review). Also write `async def classify_batch(doc_ids) -> list[ClassificationResult]` for Airflow.
8. `tests/test_classify.py` — test classification with mock LLM (mock structured_complete to return known ClassificationResponse), confidence gate (high confidence -> proceeds, low confidence -> enqueued), classifications table insert, audit log entry. Follow Section 14 testing strategy.

The classify service calls enqueue_for_review from src/review/queue.py — that file is owned by orchestration agent and will be created in parallel. Use a lazy import or accept the function as a parameter to avoid import errors during testing. Coordinate with orchestration via the contract: enqueue_for_review(doc_id, reason, reason_detail, confidence, priority) -> UUID.
```

**orchestration** (review queue CRUD + API — depends on Phase 1 DB):
```
Read IMPLEMENTATION_PLAN.md Section 7.6 (Review Queue pseudocode — enqueue_for_review, approve_review, reject_review), Section 10 (Review Queue Design), Section 13.5-13.7 (Review API endpoints), Section 6 Phase 2 steps 2.5-2.6.

Create the review queue layer:
1. `src/review/__init__.py` — empty.
2. `src/review/queue.py` — follow Section 7.6 pseudocode:
   - `async def enqueue_for_review(doc_id, reason, reason_detail=None, confidence=None, priority=5) -> UUID`: insert into review_queue, update documents.status='review_pending', audit log (step='review', action='enqueued').
   - `async def approve_review(review_id, corrected_data, reviewer, notes=None) -> None`: update review_queue status='approved', update extraction_records with corrected_data + is_valid=True, update documents.status='routed', audit log.
   - `async def reject_review(review_id, reviewer, notes) -> None`: update review_queue status='rejected', update documents.status='failed', audit log.
   - `async def list_pending(doc_type=None, reason=None, priority=None, page=1, limit=20) -> dict`: query review_queue with filters, return {items, total, page}.
   - `async def get_review_item(review_id) -> dict`: join review_queue + documents + extraction_records + classifications for detail view.
   - `async def escalate_review(review_id) -> None`: increase priority, re-queue.
3. `src/api/routes/review.py` — follow Section 13.5-13.7 + Section 10.4:
   - `GET /review/queue` — list pending items with filters (doc_type, reason, priority, page, limit). Section 13.5 response format.
   - `GET /review/{id}` — get review item detail (original doc + extracted data + errors).
   - `POST /review/{id}/approve` — approve with corrected_data, reviewer, notes. Section 13.6 response format.
   - `POST /review/{id}/reject` — reject with reviewer, notes. Section 13.7 response format.
   - `POST /review/{id}/escalate` — escalate (increase priority).
   - `GET /review/stats` — queue metrics (pending count, avg resolution time, approved rate).
4. `tests/test_review_queue.py` — test enqueue (insert + status update + audit), list_pending with filters, approve (corrected_data routes to extraction_records + documents.status='routed'), reject (documents.status='failed'), status transitions. Follow Section 14 testing strategy.

Register the review router in src/main.py — coordinate with pipeline agent who owns main.py. Send a message to pipeline agent requesting they add `from src.api.routes.review import router as review_router` and `app.include_router(review_router)` in main.py.
```

#### Sequential (after parallel)

**pipeline** (wire review router into main.py — after orchestration creates review routes):
```
Orchestration agent has created src/review/queue.py and src/api/routes/review.py. Update src/main.py to include the review router:

1. Add import: `from src.api.routes.review import router as review_router`
2. Add: `app.include_router(review_router)`
3. Verify the enqueue_for_review function signature in src/review/queue.py matches what classify/service.py expects: enqueue_for_review(doc_id, reason, reason_detail, confidence, priority).
4. If classify/service.py used a lazy import or stub for enqueue_for_review, replace with the real import: `from src.review.queue import enqueue_for_review`.
5. Run tests/test_classify.py and tests/test_review_queue.py together to verify the integration.
```

#### Review

**reviewer**:
```
Review Phase 2 deliverables against IMPLEMENTATION_PLAN.md:

1. Check src/llm/client.py follows Section 7.7 — tenacity retry with correct exception types, structured_complete returns tuple[BaseModel, CostRecord], uses beta.chat.completions.parse().
2. Check src/classify/service.py follows Section 7.2 — truncation, LLM call, classifications table insert (all columns: id, doc_id, doc_type, confidence, model, prompt_version, raw_response), documents.status update, audit log, confidence gate at 0.85, enqueue_for_review with reason='low_confidence'.
3. Check src/review/queue.py follows Section 7.6 — enqueue_for_review inserts into review_queue, updates documents.status='review_pending', audit logs. approve_review updates extraction_records + routes. reject_review archives.
4. Check src/api/routes/review.py matches Section 13.5-13.7 endpoints and response formats.
5. Verify classify/service.py correctly imports and calls enqueue_for_review from src/review/queue.py (no stubs or lazy imports remaining).
6. Verify src/main.py includes both ingest and review routers.
7. Verify no stubs, TODOs, or placeholder code.
8. Run: pytest tests/test_classify.py tests/test_review_queue.py -v
```

#### Commit

```bash
git add -A && git commit -m "Phase 2: LLM classification with confidence gate, review queue CRUD + API, LLM client with retry"
```

---

### Phase 3: Pydantic Schemas & Structured Extraction (Days 5-6)

**Goal:** Per doc_type, LLM extracts structured data into Pydantic models with structured output.

**Plan reference:** Section 6 Phase 3 (steps 3.1–3.12), Section 8 (all Pydantic schemas), Section 7.3 (Extraction Service pseudocode — first half, without retry).

#### Parallel Work

**schemas** (all Pydantic schemas + registry — depends on Phase 1 base.py):
```
Read IMPLEMENTATION_PLAN.md Section 8 in full:
- Section 8.1: BaseExtractionSchema (already created in Phase 1 — verify it matches)
- Section 8.2: InvoiceSchema + LineItem + Currency enum
- Section 8.3: ContractSchema + Party + Clause + ContractType enum
- Section 8.4: EmailSchema + EmailPriority enum
- Section 8.5: ReceiptSchema + ReceiptItem + PaymentMethod enum
- Section 8.6: OtherSchema + KeyValuePair
- Section 8.7: Schema Registry

Create ALL schemas following the plan pseudocode EXACTLY:

1. `src/schemas/invoice.py` — follow Section 8.2 precisely:
   - Currency enum (USD, EUR, GBP, JPY, CAD, AUD, CHF, OTHER)
   - LineItem(BaseModel): description, quantity (gt=0), unit_price (ge=0), line_total (ge=0), model_validator check_line_total (line_total == quantity * unit_price, +/-0.02 tolerance)
   - InvoiceSchema(BaseExtractionSchema): invoice_number, invoice_date, due_date (optional), vendor_name, vendor_address (optional), vendor_tax_id (optional), customer_name (optional), customer_address (optional), line_items (min_length=1), subtotal, tax_rate (ge=0, le=1), tax_amount, total, currency, payment_terms (optional)
   - field_validator date_not_future for invoice_date + due_date
   - model_validator check_subtotal_equals_line_items (+/-0.02)
   - model_validator check_tax_amount (tax_amount == subtotal * tax_rate, +/-0.02)
   - model_validator check_total (total == subtotal + tax_amount, +/-0.02)
   - model_validator check_due_date_after_invoice_date

2. `src/schemas/contract.py` — follow Section 8.3 precisely:
   - ContractType enum (service_agreement, nda, employment, lease, purchase_agreement, partnership, other)
   - Party(BaseModel): name, role, address (optional), contact_email (optional)
   - Clause(BaseModel): clause_type, summary (min_length=10), page_reference (optional)
   - ContractSchema(BaseExtractionSchema): contract_title, contract_type, effective_date, expiration_date (optional), parties (min_length=2), contract_value (optional, ge=0), currency (optional), key_clauses (min_length=1), termination_notice_days (optional, ge=0), governing_law (optional)
   - field_validator effective_date_plausibility (>1 year future -> error)
   - model_validator check_expiration_after_effective

3. `src/schemas/email.py` — follow Section 8.4 precisely:
   - EmailPriority enum (high, normal, low)
   - EmailSchema(BaseExtractionSchema): sender_name (optional), sender_email, recipients (min_length=1), cc_recipients, bcc_recipients, subject, body, sent_at, priority, has_attachments, attachment_names
   - field_validator validate_email_format for sender_email
   - field_validator validate_recipient_list for recipients/cc/bcc
   - field_validator sent_at_not_future
   - model_validator check_attachments_consistency

4. `src/schemas/receipt.py` — follow Section 8.5 precisely:
   - PaymentMethod enum (cash, credit_card, debit_card, bank_transfer, mobile_payment, other)
   - ReceiptItem(BaseModel): description, quantity (gt=0), unit_price (ge=0), line_total (ge=0), model_validator check_line_total
   - ReceiptSchema(BaseExtractionSchema): merchant_name, merchant_address (optional), receipt_number (optional), receipt_date, receipt_time (optional), items (min_length=1), subtotal, tax_amount, total, payment_method, payment_amount, change_amount (optional), currency
   - field_validator date_not_future for receipt_date
   - field_validator validate_time_format (HH:MM) for receipt_time
   - model_validator check_subtotal (+/-0.02)
   - model_validator check_total (total == subtotal + tax_amount, +/-0.02)
   - model_validator check_payment (payment_amount >= total, change_amount == payment_amount - total)

5. `src/schemas/other.py` — follow Section 8.6 precisely:
   - KeyValuePair(BaseModel): key, value
   - OtherSchema(BaseExtractionSchema): document_summary (min_length=10), key_information (min_length=1), document_date (optional), mentioned_entities

6. `src/schemas/registry.py` — follow Section 8.7 precisely:
   - SCHEMA_REGISTRY dict mapping "invoice"->InvoiceSchema, "contract"->ContractSchema, "email"->EmailSchema, "receipt"->ReceiptSchema, "other"->OtherSchema
   - `get_schema_for_doc_type(doc_type) -> type[BaseModel]` with ValueError for unknown types

7. `tests/test_schemas.py` — follow Section 14.2 precisely:
   - TestLineItem: test_valid_line_item, test_line_total_mismatch_rejected, test_negative_quantity_rejected
   - TestInvoiceSchema: _make_valid_invoice helper, test_valid_invoice_passes, test_subtotal_mismatch_rejected, test_total_mismatch_rejected, test_future_invoice_date_rejected, test_due_date_before_invoice_date_rejected, test_tax_rate_out_of_range_rejected, test_unknown_field_rejected, test_empty_line_items_rejected
   - Add equivalent tests for ContractSchema (min 2 parties, expiration before effective, unknown field), EmailSchema (invalid email, empty recipients, future sent_at, attachment inconsistency), ReceiptSchema (line total, subtotal, total, payment < total, change mismatch), OtherSchema (short summary, empty key_information)

ALL model_config must use {"extra":"forbid", "str_strip_whitespace":True}. ALL monetary validators use +/-0.02 Decimal tolerance.
```

**pipeline** (extraction prompts + extractor — depends on Phase 2 LLM client):
```
Read IMPLEMENTATION_PLAN.md Section 7.3 (Extraction Service pseudocode — focus on the extraction part, NOT retry yet), Section 6 Phase 3 steps 3.8-3.10.

Create the extraction layer (WITHOUT retry — retry is Phase 4):
1. `src/extract/__init__.py` — empty.
2. `src/extract/prompts.py` — versioned extraction prompt templates per doc_type. One system prompt per type (invoice, contract, email, receipt, other) instructing the LLM to extract all fields according to the schema. Include EXTRACTION_PROMPT_VERSION = "extract_v1". `get_extraction_prompt(doc_type) -> str` returns the appropriate template.
3. `src/extract/extractor.py` — `async def extract(doc_type, raw_text, schema, error_feedback=None) -> tuple[BaseModel, CostRecord]`. Build prompt (system = template, user = document text + "Extract all fields according to the schema."). If error_feedback is not None, append the error feedback section (but don't implement retry loop yet — that's Phase 4). Call llm_client.structured_complete_async with the schema as response_format. Return (parsed_pydantic_object, cost).
4. `src/extract/service.py` — follow Section 7.3 pseudocode but WITHOUT the retry loop (Phase 4 adds retry). `async def extract_document(doc_id, doc_type, raw_text) -> ExtractionResult`:
   - Get schema from registry: `schema = get_schema_for_doc_type(doc_type)` (import from src.schemas.registry — owned by schemas agent, will exist after parallel work completes)
   - Get prompt template: `prompt_template = get_extraction_prompt(doc_type)`
   - Call extractor.extract(doc_type, raw_text, schema)
   - Store extraction in extraction_records table (doc_id, doc_type, extracted_data as JSONB, schema_version, model, prompt_version, retry_count=0, is_valid=False initially — validation is Phase 4)
   - Update documents.status='extracted'
   - Audit log (step='extract')
   - Define ExtractionResult dataclass (doc_id, extracted_data, is_valid, retry_count, enqueued_for_review=False)
   - Also write `async def extract_batch(doc_ids) -> list[ExtractionResult]` for Airflow
5. `tests/test_extract.py` — test extraction with mock LLM (mock structured_complete to return a valid InvoiceSchema), verify extraction_records insert, verify documents.status update, verify audit log. Follow Section 14 testing strategy.

IMPORTANT: The schemas agent is creating src/schemas/registry.py in parallel. Your extract/service.py imports `from src.schemas.registry import get_schema_for_doc_type`. This import will resolve once both agents commit. Write the import correctly — it will work after merge.
```

#### Sequential (after parallel)

**pipeline** (verify schema registry integration):
```
Schemas agent has completed src/schemas/registry.py and all schema files. Verify integration:

1. Confirm `from src.schemas.registry import get_schema_for_doc_type` works — run: python -c "from src.schemas.registry import get_schema_for_doc_type; print(get_schema_for_doc_type('invoice'))"
2. Confirm extract/service.py correctly uses the registry to get the schema for each doc_type.
3. Run tests/test_extract.py with the real schemas (not mocks) to verify the LLM structured output path works end-to-end with a mock LLM returning schema-compliant data.
4. Run tests/test_schemas.py to verify all schema tests pass.
```

#### Review

**reviewer**:
```
Review Phase 3 deliverables against IMPLEMENTATION_PLAN.md Section 8:

1. Verify EACH schema matches its plan section EXACTLY:
   - invoice.py (Section 8.2): Currency enum values, LineItem fields + check_line_total validator, InvoiceSchema all fields + 4 model_validators + date_not_future field_validator
   - contract.py (Section 8.3): ContractType enum, Party, Clause, ContractSchema fields + effective_date_plausibility + check_expiration_after_effective
   - email.py (Section 8.4): EmailPriority, all fields + validate_email_format + validate_recipient_list + sent_at_not_future + check_attachments_consistency
   - receipt.py (Section 8.5): PaymentMethod, ReceiptItem + check_line_total, ReceiptSchema all fields + date_not_future + validate_time_format + check_subtotal + check_total + check_payment
   - other.py (Section 8.6): KeyValuePair, OtherSchema fields
   - registry.py (Section 8.7): SCHEMA_REGISTRY mapping, get_schema_for_doc_type with ValueError
2. Verify ALL model_config uses {"extra":"forbid", "str_strip_whitespace":True}.
3. Verify ALL monetary tolerance is +/-0.02 Decimal.
4. Verify extract/service.py follows Section 7.3 (extraction portion only — no retry loop yet).
5. Verify extraction_records insert includes all columns from Section 4 (doc_id, doc_type, extracted_data JSONB, schema_version, model, prompt_version, retry_count, is_valid).
6. Verify test_schemas.py covers the critical tests from Section 14.2 (line total mismatch, subtotal mismatch, total mismatch, future date, due_date ordering, tax rate range, unknown field, empty line items).
7. Verify no stubs, TODOs, or placeholder code.
8. Run: pytest tests/test_schemas.py tests/test_extract.py -v
```

#### Commit

```bash
git add -A && git commit -m "Phase 3: Pydantic schemas (Invoice/Contract/Email/Receipt/Other), schema registry, LLM structured extraction"
```

---

### Phase 4: Validation Layer & Retry Logic (Days 7-8)

**Goal:** Pydantic validation + business rules catch errors; retry with error feedback; failures -> human review.

**Plan reference:** Section 6 Phase 4 (steps 4.1–4.8), Section 7.4 (Validation Service pseudocode), Section 9 (Validation Rules table + business rules implementation), Section 7.3 (Extraction Service retry portion — second half of pseudocode).

#### Parallel Work

**schemas** (validation layer — depends on Phase 3 schemas):
```
Read IMPLEMENTATION_PLAN.md Section 7.4 (Validation Service pseudocode), Section 9 (Validation Rules table — ALL-001 through OTH-002, Section 9.2 business_rules.py, Section 9.3 rules_registry.py).

Create the validation layer:
1. `src/validate/__init__.py` — empty.
2. `src/validate/pydantic_validator.py` — `async def validate_pydantic(extracted: BaseModel) -> list[str]`: re-validate the extracted model via model_validate(extracted.model_dump()) to collect ALL errors at once (not just first). Return list of formatted error strings. Follow Section 7.4 Layer 1.
3. `src/validate/business_rules.py` — follow Section 9.2 precisely:
   - BusinessRuleError(Exception)
   - BusinessRule(ABC) with name: str and abstract check(extracted: BaseModel) -> None
   - InvoiceTaxRateRange(BusinessRule): INV-005, check tax_rate in [0, 0.30]
   - InvoiceVendorKnown(BusinessRule): INV-VENDOR, placeholder no-op for v1 (pass)
   - Note: most rules (INV-001 through INV-004, INV-006-009, CON-001-005, EML-001-005, RCP-001-007, OTH-001-002) are already implemented as Pydantic model_validators in the schemas. The BusinessRule classes are for rules needing external context (DB lookups, config).
4. `src/validate/rules_registry.py` — follow Section 9.3 precisely:
   - RULES_REGISTRY dict: "invoice" -> [InvoiceTaxRateRange(), InvoiceVendorKnown()], "contract" -> [], "email" -> [], "receipt" -> [], "other" -> []
   - `get_business_rules(doc_type) -> list[BusinessRule]`
5. `src/validate/service.py` — follow Section 7.4 pseudocode exactly:
   - `async def validate_extraction(doc_type, extracted: BaseModel) -> ValidationResult`
   - Layer 1: Pydantic validation (call validate_pydantic, collect errors)
   - Layer 2: Business rules (get_business_rules(doc_type), call rule.check(extracted) for each, catch BusinessRuleError, append "[{rule.name}] {e}" to errors)
   - Return ValidationResult(is_valid=len(errors)==0, errors=errors)
   - Define ValidationResult dataclass (is_valid: bool, errors: list[str])
6. `tests/test_validate.py` — test each business rule (pass + fail cases):
   - InvoiceTaxRateRange: valid rate (0.19) passes, out-of-range (0.50) fails
   - Pydantic validation: valid invoice passes, invalid (wrong total) fails with correct error message
   - Full validate_extraction: valid invoice -> is_valid=True, invalid invoice -> is_valid=False with errors list
   - Test that ALL errors are collected (not just first) — create an invoice with multiple violations and verify multiple error messages
   - Follow Section 14 testing strategy
```

**pipeline** (retry logic — depends on Phase 3 extract/service.py):
```
Read IMPLEMENTATION_PLAN.md Section 7.3 (Extraction Service pseudocode — FULL version with retry loop), Section 6 Phase 4 steps 4.5-4.6.

Add retry-with-error-feedback to the extraction service:
1. `src/extract/retry.py`:
   - `def build_extraction_prompt(template, raw_text, error_feedback=None) -> dict` — follow Section 7.3 build_extraction_prompt pseudocode: system = template, user = "Document text:\n\n{raw_text}\n\nExtract all fields according to the schema." If error_feedback: append "\n\n--- PREVIOUS ATTEMPT FAILED VALIDATION ---\nThe previous extraction had these validation errors:\n{error_feedback}\nPlease correct these errors and re-extract."
   - `def format_validation_errors(errors: list[str]) -> str` — join error messages with newlines.
2. Update `src/extract/service.py` — follow Section 7.3 FULL pseudocode (the while loop):
   - MAX_RETRIES = 2 (from settings)
   - Loop: retry_count = 0; while retry_count <= MAX_RETRIES:
     - Build prompt (with error_feedback on retry via build_extraction_prompt)
     - Call extractor.extract with schema
     - Validate: `validation_result = await validate_extraction(doc_type, extracted)` (import from src.validate.service — owned by schemas agent, will exist after parallel work)
     - If valid: store_extraction with is_valid=True, update documents.status='validated', audit log, return ExtractionResult(is_valid=True)
     - If invalid: error_feedback = format_validation_errors(validation_result.errors), retry_count += 1, continue loop
   - After loop exhausted: store_extraction with is_valid=False + validation_errors, enqueue_for_review(reason='validation_failed', reason_detail=errors), audit log (success=False), return ExtractionResult(is_valid=False, enqueued_for_review=True)
   - Import: `from src.validate.service import validate_extraction` (schemas agent creates this in parallel — write the import correctly, it will resolve after merge)
   - Import: `from src.review.queue import enqueue_for_review` (already exists from Phase 2)
3. `tests/test_retry.py` — follow Section 14.3 precisely:
   - test_retry_succeeds_on_second_attempt: mock LLM returns wrong total first, correct total second -> is_valid=True, retry_count=1
   - test_max_retries_enforced: mock LLM always returns wrong -> is_valid=False, retry_count=2, enqueued_for_review=True
   - test_error_feedback_in_prompt_on_retry: verify second LLM call prompt contains "PREVIOUS ATTEMPT FAILED VALIDATION" and the error detail
   - Use mock LLM that returns different InvoiceSchema objects on successive calls
```

#### Sequential (after parallel)

**pipeline** (verify validation integration):
```
Schemas agent has completed src/validate/service.py and all validation files. Verify integration:

1. Confirm `from src.validate.service import validate_extraction` works — run: python -c "from src.validate.service import validate_extraction; print(validate_extraction)"
2. Confirm extract/service.py retry loop correctly calls validate_extraction and handles ValidationResult.
3. Run tests/test_retry.py with the real validation layer (not mocks) to verify retry-with-error-feedback works end-to-end.
4. Run tests/test_validate.py to verify all validation tests pass.
5. Run the full integration: ingest -> classify -> extract (with retry) -> validate on a test document with an intentional math error to verify retry triggers and error feedback appears in the second LLM call.
```

#### Review

**reviewer**:
```
Review Phase 4 deliverables against IMPLEMENTATION_PLAN.md:

1. Check src/validate/service.py follows Section 7.4 — two-layer validation (Pydantic + business rules), collects ALL errors (not just first), returns ValidationResult(is_valid, errors).
2. Check src/validate/business_rules.py follows Section 9.2 — BusinessRule ABC, InvoiceTaxRateRange (INV-005: 0-0.30 range), InvoiceVendorKnown (no-op for v1).
3. Check src/validate/rules_registry.py follows Section 9.3 — correct mapping, get_business_rules returns empty list for unknown types.
4. Check src/extract/retry.py follows Section 7.3 — build_extraction_prompt appends error feedback section with "PREVIOUS ATTEMPT FAILED VALIDATION" header.
5. Check src/extract/service.py retry loop: MAX_RETRIES=2, loop condition (retry_count <= MAX_RETRIES), valid -> store + status='validated' + audit, invalid -> error_feedback + retry, exhausted -> store is_valid=False + enqueue_for_review(reason='validation_failed') + audit(success=False).
6. Verify test_validate.py covers each business rule (pass + fail) and tests multi-error collection.
7. Verify test_retry.py follows Section 14.3 — retry succeeds on second attempt, max retries enforced, error feedback in prompt.
8. Verify no stubs, TODOs, or placeholder code.
9. Run: pytest tests/test_validate.py tests/test_retry.py -v
```

#### Commit

```bash
git add -A && git commit -m "Phase 4: Validation layer (Pydantic + business rules), retry-with-error-feedback, validation failures -> review queue"
```

---

### Phase 5: Routing & Audit (Day 9)

**Goal:** Validated records route to PostgreSQL + document store + audit log.

**Plan reference:** Section 6 Phase 5 (steps 5.1–5.8), Section 7.5 (Routing Service pseudocode), Section 13.3-13.4 + 13.8 (Documents + Stats API).

#### Parallel Work

**orchestration** (routing + document store + API routes — depends on Phase 1-4 DB + extraction):
```
Read IMPLEMENTATION_PLAN.md Section 7.5 (Routing Service pseudocode), Section 6 Phase 5 steps 5.1-5.6, Section 13.3 (GET /documents), Section 13.4 (GET /documents/{id}), Section 13.8 (GET /stats).

Create the routing layer and additional API routes:
1. `src/route/__init__.py` — empty.
2. `src/route/document_store.py` — `class DocumentStore`: local filesystem abstraction with S3-ready interface. `async def save(file_content, doc_id, file_name) -> str` (store file, return path). `async def retrieve(file_path) -> bytes`. `async def delete(file_path) -> bool`. Use DOC_STORE_DIR from config. Path format: {DOC_STORE_DIR}/{doc_id}/{file_name}.
3. `src/route/router.py` — follow Section 7.5 pseudocode: `async def route_document(doc_id, extraction_result) -> None`. Update extraction_records.is_valid=True, update documents.status='routed', audit log (step='route'). Also write `async def route_batch(doc_ids) -> list[UUID]` for Airflow.
4. `src/route/service.py` — orchestrator: calls router.route_document for each doc. Handles errors, audit logs failures.
5. `src/api/routes/documents.py` — follow Section 13.3-13.4:
   - `GET /documents` — list with filters (status, doc_type, page, limit). Return {documents, total, page}. Section 13.3 response format.
   - `GET /documents/{doc_id}` — full detail: document metadata + classification + extraction + review + audit_trail. Section 13.4 response format. Join documents + classifications + extraction_records + review_queue + audit_logs.
6. `src/api/routes/stats.py` — follow Section 13.8: `GET /stats` — pipeline metrics: total_documents, by_status, by_type, avg_confidence, avg_retry_count, review_queue stats, pipeline_runs. Aggregate queries across all tables.
7. Register documents + stats routers in src/main.py — message pipeline agent to add these imports.
8. `tests/test_route.py` — test routing: valid extraction -> extraction_records.is_valid=True, documents.status='routed', audit_logs has 'route' entry. Test document_store save/retrieve/delete. Follow Section 14 testing strategy.
9. `tests/test_audit.py` — follow Section 14.5: test audit_log_is_immutable (UPDATE and DELETE must raise due to triggers). Test that every pipeline step (ingest, classify, extract, validate, route) produces an audit log entry.
```

**pipeline** (wire audit logging into all steps + integrate route API):
```
Read IMPLEMENTATION_PLAN.md Section 6 Phase 5 step 5.4 (wire audit logging into all pipeline steps).

1. Review all pipeline services (ingest/service.py, classify/service.py, extract/service.py) and verify each calls audit_logger.log at the appropriate point. Most should already have audit logging from Phases 1-4 — verify completeness:
   - ingest: audit log step='ingest' with detail {file_type, ocr_used, text_length}
   - classify: audit log step='classify' with detail {doc_type, confidence}
   - extract: audit log step='extract' with detail {retry_count, doc_type, cost} (success) or {retry_count, errors} (failure)
   - validate: audit log step='validate' with detail {is_valid, error_count}
2. Add a validate audit log call in extract/service.py if not already present (after validation, before routing decision).
3. Update src/main.py to include the documents and stats routers from orchestration:
   - `from src.api.routes.documents import router as documents_router`
   - `from src.api.routes.stats import router as stats_router`
   - `app.include_router(documents_router)`
   - `app.include_router(stats_router)`
4. Write `scripts/init_db.py` — run src/db/schema.sql against the database. Read schema.sql, execute via asyncpg.
5. Write `scripts/seed_test_data.py` — load sample documents from tests/fixtures/ into the database via the ingest API/service.
```

#### Sequential (after parallel)

**orchestration** (verify full routing integration):
```
Pipeline agent has wired audit logging and added route API routers. Verify:

1. Run scripts/init_db.py to initialize the database with schema.sql.
2. Process a test document end-to-end: ingest -> classify -> extract -> validate -> route.
3. Verify documents.status='routed', extraction_records.is_valid=True.
4. Verify audit_logs has entries for every step: ingest, classify, extract, validate, route — query: SELECT step FROM audit_logs WHERE doc_id = $1 ORDER BY timestamp.
5. Verify GET /documents/{doc_id} returns full detail with audit_trail.
6. Verify GET /stats returns pipeline metrics.
7. Run tests/test_route.py and tests/test_audit.py.
```

#### Review

**reviewer**:
```
Review Phase 5 deliverables against IMPLEMENTATION_PLAN.md:

1. Check src/route/router.py follows Section 7.5 — update extraction_records.is_valid, update documents.status='routed', audit log step='route'.
2. Check src/route/document_store.py — save/retrieve/delete with local FS, S3-ready interface.
3. Check src/api/routes/documents.py matches Section 13.3-13.4 — list with filters, detail with full join (document + classification + extraction + review + audit_trail).
4. Check src/api/routes/stats.py matches Section 13.8 — total_documents, by_status, by_type, avg_confidence, avg_retry_count, review_queue stats, pipeline_runs.
5. Verify audit logging is wired into ALL pipeline steps (ingest, classify, extract, validate, route).
6. Verify test_audit.py tests immutability (UPDATE/DELETE raise) and completeness (every step logged).
7. Verify src/main.py includes all routers: ingest, batch, health, review, documents, stats.
8. Verify no stubs, TODOs, or placeholder code.
9. Run: pytest tests/test_route.py tests/test_audit.py -v
```

#### Commit

```bash
git add -A && git commit -m "Phase 5: Routing layer (DB + document store + audit), documents/stats API, audit logging wired into all steps"
```

---

### Phase 6: Airflow DAG & Orchestration (Day 10)

**Goal:** Airflow DAG runs the full pipeline as a batch with retries and SLA monitoring.

**Plan reference:** Section 6 Phase 6 (steps 6.1–6.7), Section 11 (Orchestration DAG — full DAG code + visualization).

#### Sequential (depends on all prior phases)

**orchestration** (Airflow DAG + CLI + E2E tests):
```
Read IMPLEMENTATION_PLAN.md Section 11.1 (full Airflow DAG code), Section 11.2 (DAG visualization), Section 6 Phase 6.

Create the Airflow DAG and CLI:
1. `airflow/dags/document_processing_dag.py` — follow Section 11.1 pseudocode EXACTLY:
   - DAG id="document_processing_pipeline", schedule="0 * * * *" (hourly), catchup=False, max_active_runs=1
   - default_args: owner="data-engineering", retries=2, retry_delay=timedelta(minutes=2), retry_exponential_backoff=True, max_retry_delay=timedelta(minutes=10), email_on_failure=True, sla=timedelta(minutes=60)
   - sla_miss_callback function
   - Task 1: scan_for_documents — scan INPUT_DIR for new files (dedup via file_hash vs documents table), push file list via XCom
   - Task 2: ingest_documents — pull file list from XCom, call ingest_batch, push doc_ids via XCom
   - Task 3: classify_documents — pull doc_ids, call classify_batch, push high_confidence doc_ids
   - Task 4: extract_and_validate — pull doc_ids, call extract_batch (includes validation + retry), push valid_doc_ids. execution_timeout=timedelta(minutes=30)
   - Task 5: route_documents — pull valid_doc_ids, call route_batch
   - Task 6: record_pipeline_run — record run metadata (dag_run_id, batch_size, succeeded, failed, review_queued). trigger_rule="all_done"
   - Task 7: done — EmptyOperator, trigger_rule="all_success"
   - Dependencies: scan >> ingest >> classify >> extract >> route >> record >> done
   - sys.path.insert(0, "/opt/airflow/src") for imports
2. `airflow/plugins/doc_pipeline_plugin.py` — custom operators if needed (can be minimal/empty if PythonOperator suffices).
3. `airflow/config/airflow.cfg.template` — template config with DAGs folder, executor, SQL alchemy conn placeholders.
4. `scripts/run_pipeline.py` — CLI for single-file or directory processing:
   - `python scripts/run_pipeline.py --file <path>` — ingest + classify + extract + validate + route a single file
   - `python scripts/run_pipeline.py --dir <path>` — process all files in a directory
   - Uses the same service functions as the DAG (ingest_document, classify_document, extract_document, route_document)
5. `scripts/generate_golden_set.py` — create golden test set with expected extractions for accuracy measurement.
6. `tests/test_dag.py` — follow Section 14.6 precisely:
   - test_dag_loaded: dag.dag_id == "document_processing_pipeline", len(dag.tasks) == 7
   - test_task_dependencies: verify downstream chain (scan->ingest->classify->extract->route->record)
   - test_no_orphaned_tasks: every task except scan_for_documents has upstream
7. `tests/test_e2e.py` — follow Section 14.4 precisely:
   - test_full_pipeline_invoice: upload invoice PDF -> run pipeline -> verify documents.status='routed', classification.doc_type='invoice', extraction.is_valid=True, audit trail has all steps, no review queue entry
   - test_full_pipeline_low_confidence_goes_to_review: upload blurry image -> low confidence -> documents.status='review_pending', review_queue.reason='low_confidence'
   - Test with mixed document batch (invoice, contract, email, receipt) -> all route correctly
```

#### Review

**reviewer**:
```
Review Phase 6 deliverables against IMPLEMENTATION_PLAN.md Section 11:

1. Check airflow/dags/document_processing_dag.py follows Section 11.1 EXACTLY:
   - DAG id, schedule, catchup=False, max_active_runs=1
   - default_args: retries=2, exponential backoff, sla=60 min, email_on_failure
   - sla_miss_callback defined
   - 7 tasks with correct task_ids: scan_for_documents, ingest_documents, classify_documents, extract_and_validate, route_documents, record_pipeline_run, done
   - XCom push/pull keys match: new_files, doc_ids, classify_results, valid_docs
   - Dependencies: scan >> ingest >> classify >> extract >> route >> record >> done
   - record_pipeline_run has trigger_rule="all_done"
   - extract_and_validate has execution_timeout=timedelta(minutes=30)
2. Check scripts/run_pipeline.py supports --file and --dir modes.
3. Check test_dag.py follows Section 14.6 — dag loaded, task dependencies, no orphaned tasks.
4. Check test_e2e.py follows Section 14.4 — full pipeline invoice (all table assertions), low confidence -> review queue, mixed batch.
5. Verify no stubs, TODOs, or placeholder code.
6. Run: pytest tests/test_dag.py tests/test_e2e.py -v
```

#### Commit

```bash
git add -A && git commit -m "Phase 6: Airflow DAG (scan->ingest->classify->extract->route), CLI runner, DAG structure tests, E2E tests"
```

---

### Phase 7: Streamlit Review UI & Hardening (Days 11-12, buffer)

**Goal:** Human review queue UI works; full system hardened and documented.

**Plan reference:** Section 6 Phase 7 (steps 7.1–7.10), Section 10.3 (Streamlit Review UI Design), Section 10.4 (Review Queue API Endpoints).

#### Parallel Work

**orchestration** (Streamlit UI + README + final tests):
```
Read IMPLEMENTATION_PLAN.md Section 10.3 (Streamlit Review UI Design — ASCII mockup), Section 10.4 (Review Queue API Endpoints), Section 6 Phase 7.

Create the Streamlit review UI:
1. `streamlit/app.py` — main Streamlit app with sidebar navigation (Queue, Stats, History). Uses httpx to call the FastAPI API (API_BASE_URL from env). Follow Section 10.3 layout: sidebar with navigation + filters (type, reason, priority), main area with queue list or detail view.
2. `streamlit/components/queue_view.py` — pending review items list, sortable by priority. Each item shows: priority badge, reason, file_name, doc_type, confidence/errors, [View Details] button. Filters from sidebar applied. Calls GET /review/queue.
3. `streamlit/components/detail_view.py` — single item detail view following Section 10.3 mockup: left panel shows original document (rendered text/preview), right panel shows extracted data in editable form fields. Validation errors displayed with red highlighting. Notes field. [Approve & Route] and [Reject] buttons. Calls GET /review/{id}, POST /review/{id}/approve, POST /review/{id}/reject.
4. `streamlit/components/stats_view.py` — pipeline metrics dashboard. Calls GET /stats. Display: total documents, by status (bar chart), by type (pie chart), avg confidence, review queue stats, pipeline runs table.
5. `streamlit/styles.css` — custom CSS for the review UI (priority badges, error highlighting, layout).
6. `README.md` — setup, usage, architecture, metrics. Follow Quick Start (Appendix A): clone, configure, docker compose up, init_db, ingest, check status, view review queue, open Streamlit, open Airflow, trigger DAG, run CLI.
7. `tests/test_api.py` — test all HTTP endpoints: POST /ingest (202, 409, 413, 415), POST /batch, GET /documents (with filters), GET /documents/{id}, GET /review/queue (with filters), POST /review/{id}/approve, POST /review/{id}/reject, GET /stats, GET /health. Follow Section 14 testing strategy.
8. Run the full test suite end-to-end and fix any failures.
9. Manual smoke test: drop 5 docs in watched dir -> check full pipeline + review queue.
```

**pipeline** (structured logging):
```
Read IMPLEMENTATION_PLAN.md Section 6 Phase 7 step 7.6 (add structured logging).

Add structured logging:
1. Add structlog to pyproject.toml dependencies (message orchestration agent to add it).
2. Configure structlog in src/config.py or src/main.py: JSON output, log level from settings.LOG_LEVEL.
3. Add structured log calls in key pipeline services (ingest, classify, extract, route) with contextual fields: doc_id, step, duration_ms, doc_type, confidence, retry_count.
4. Ensure LLM cost tracking logs are structured (from src/llm/cost_tracker.py).
```

**schemas** (final validation hardening):
```
Review all schema tests and validation tests for completeness:

1. Verify tests/test_schemas.py covers ALL validation rules from Section 9.1 table:
   - Invoice: INV-001 through INV-009 (line total, subtotal, tax amount, total, tax rate range, date not future, due date ordering, min line items, non-negative amounts)
   - Contract: CON-001 through CON-005 (min 2 parties, expiration after effective, effective date plausibility, min 1 clause, non-negative value)
   - Email: EML-001 through EML-005 (valid email, min 1 recipient, sent not future, attachment consistency, subject not empty)
   - Receipt: RCP-001 through RCP-007 (line total, subtotal, total, date not future, payment >= total, change correct, min 1 item)
   - Other: OTH-001 through OTH-002 (summary min length, min 1 key-value)
   - All: ALL-001 (no unknown fields via extra="forbid")
2. Add any missing test cases.
3. Verify edge cases: empty strings, None values where optional, Decimal precision, date boundaries.
4. Run pytest tests/test_schemas.py tests/test_validate.py -v and ensure all pass.
```

#### Sequential (after parallel)

**orchestration** (final integration + full test suite):
```
All agents have completed their Phase 7 work. Final integration:

1. Verify structlog is in pyproject.toml (pipeline agent requested it).
2. Run the FULL test suite: pytest -v
   - tests/test_ingest.py
   - tests/test_classify.py
   - tests/test_extract.py
   - tests/test_schemas.py
   - tests/test_validate.py
   - tests/test_retry.py
   - tests/test_route.py
   - tests/test_review_queue.py
   - tests/test_audit.py
   - tests/test_api.py
   - tests/test_dag.py
   - tests/test_e2e.py
3. Fix any failures (coordinate with the owning agent if the failure is in their code).
4. Verify docker compose up starts all services and health checks pass:
   - curl http://localhost:8000/health -> {"status":"healthy",...}
   - curl http://localhost:8080/health (Airflow)
   - curl http://localhost:8501 (Streamlit)
5. Manual smoke test: drop 5 mixed docs in INPUT_DIR -> trigger DAG -> verify all process -> check review queue for any low-confidence items -> open Streamlit -> approve/reject from UI.
```

#### Review

**reviewer**:
```
Final review — complete system hardening check:

1. Check streamlit/app.py has sidebar navigation (Queue, Stats, History) per Section 10.3.
2. Check streamlit/components/queue_view.py — pending items list, sortable by priority, filters applied.
3. Check streamlit/components/detail_view.py — original doc + editable extracted data + validation errors + Approve/Reject buttons per Section 10.3 mockup.
4. Check streamlit/components/stats_view.py — metrics dashboard from GET /stats.
5. Check README.md covers Quick Start (Appendix A): setup, docker compose, init_db, ingest, check, review, Streamlit, Airflow, CLI.
6. Check structured logging is configured (structlog JSON output) and used in pipeline services.
7. Check tests/test_api.py covers all endpoints with correct status codes.
8. Run the FULL test suite: pytest -v — ALL tests must pass.
9. Verify docker compose up works and all health checks pass.
10. Verify no stubs, TODOs, placeholders, or missing implementations anywhere in the codebase.
11. Verify file ownership boundaries were respected throughout all phases.
```

#### Commit

```bash
git add -A && git commit -m "Phase 7: Streamlit review UI (queue/detail/stats), structured logging, README, full test suite green, production hardening"
```

---

## 5. Coordination Rules

### File Ownership Boundaries

| Directory/File | Owner | Notes |
|----------------|-------|-------|
| `src/ingest/` | pipeline | Parser, OCR, normalizer, ingest service |
| `src/classify/` | pipeline | Classifier, prompts, classify service |
| `src/extract/` | pipeline | Extractor, prompts, retry, extract service |
| `src/llm/` | pipeline | OpenAI client wrapper, cost tracker |
| `src/audit/` | pipeline | Audit logger |
| `src/db/` | pipeline | Connection pool, schema.sql, migrations |
| `src/config.py` | pipeline | Pydantic Settings |
| `src/main.py` | pipeline | FastAPI app factory — pipeline adds routers from other agents |
| `src/api/dependencies.py` | pipeline | FastAPI DI |
| `src/api/routes/ingest.py` | pipeline | POST /ingest |
| `src/api/routes/batch.py` | pipeline | POST /batch |
| `src/api/routes/health.py` | pipeline | GET /health |
| `src/schemas/` | schemas | All Pydantic schemas + registry |
| `src/validate/` | schemas | Validation layer (Pydantic + business rules) |
| `src/route/` | orchestration | Router, document store, route service |
| `src/review/` | orchestration | Review queue CRUD + API |
| `src/api/routes/review.py` | orchestration | Review queue endpoints |
| `src/api/routes/documents.py` | orchestration | Document list/detail endpoints |
| `src/api/routes/stats.py` | orchestration | Stats endpoint |
| `airflow/` | orchestration | DAGs, plugins, config |
| `streamlit/` | orchestration | Review UI app + components |
| `scripts/` | orchestration | init_db, seed, generate_golden_set, run_pipeline |
| `docker/` | orchestration | Dockerfiles, postgres init |
| `docker-compose.yml` | orchestration | Multi-service compose |
| `.env.example` | orchestration | Environment variable template |
| `pyproject.toml` | orchestration | Python project config + dependencies |
| `tests/conftest.py` | pipeline | Shared pytest fixtures |
| `tests/fixtures/` | pipeline | Sample test documents |
| `tests/test_ingest.py` | pipeline | Ingestion tests |
| `tests/test_classify.py` | pipeline | Classification tests |
| `tests/test_extract.py` | pipeline | Extraction tests |
| `tests/test_retry.py` | pipeline | Retry logic tests |
| `tests/test_schemas.py` | schemas | Schema validation tests |
| `tests/test_validate.py` | schemas | Business rule tests |
| `tests/test_route.py` | orchestration | Routing tests |
| `tests/test_audit.py` | orchestration | Audit immutability tests |
| `tests/test_review_queue.py` | orchestration | Review queue tests |
| `tests/test_api.py` | orchestration | API endpoint tests |
| `tests/test_dag.py` | orchestration | DAG structure tests |
| `tests/test_e2e.py` | orchestration | End-to-end tests |

### Conflict Avoidance Rules

1. **`src/main.py` is owned by pipeline** but needs routers from orchestration (review, documents, stats). Orchestration agent messages pipeline agent when a new router needs registration. Pipeline agent adds the import + include_router.
2. **`pyproject.toml` is owned by orchestration** but pipeline may need to add dependencies (e.g., structlog in Phase 7). Pipeline agent messages orchestration agent to add the dependency.
3. **Cross-agent imports** (e.g., pipeline's extract/service.py importing from schemas' validate/service.py) are resolved after parallel work completes. Write imports correctly during parallel work — they resolve after merge.
4. **Never edit another agent's files.** If a change is needed in another agent's file, message them via `hub send` with the specific change request.
5. **`tests/conftest.py` is owned by pipeline** but all agents use its fixtures. If schemas or orchestration needs a new fixture, message pipeline agent to add it.

### Parallel vs Sequential

- **Parallel:** Phases 1, 2, 3, 4, and 7 have parallel work where agents touch disjoint file sets with no import dependencies between the parallel tasks.
- **Sequential:** Phase 6 is fully sequential (DAG depends on all prior services). Within each phase, the sequential step runs after parallel work completes to verify cross-agent integration.
- **Rule:** Run in parallel when agents touch disjoint directories AND the parallel work doesn't require importing each other's new files. Run sequentially when B imports from A's new code.

### Blocked Agent Protocol

1. If an agent is blocked waiting for another agent's file (e.g., pipeline needs `src/schemas/registry.py` from schemas), the agent should:
   - Write their code with the correct import (it will resolve after merge).
   - Use a mock/stub for local testing only (clearly marked, removed before commit).
   - Message the blocking agent via `hub send` to check status.
   - Proceed with other work that doesn't depend on the blocker.
2. If an agent encounters a conflict (another agent edited their file), message the other agent immediately via `hub send` to resolve.
3. If the reviewer blocks a phase, the owning agent(s) must fix the issues before the commit.

---

## 6. Quick Reference

### Herdr Commands

```bash
# Pane management
herdr pane list                          # list all panes with IDs
herdr pane split --cwd "$PWD" --no-focus # split pane (new pane doesn't steal focus)
herdr pane split --direction right       # split horizontally
herdr pane split --direction down        # split vertically
herdr pane focus <id>                    # focus a specific pane
herdr pane close <id>                    # close a pane

# Agent management
herdr agent start <name> --kind codex --pane <id>  # start agent in pane
herdr agent list                                    # list running agents
herdr agent stop <name>                             # stop an agent
herdr agent send <name> "<message>"                 # send message to agent

# Monitoring
herdr agent logs <name>                  # view agent output
herdr agent status <name>                # check agent status
```

### Phase Summary

| Phase | Days | Parallel Agents | Key Deliverable |
|-------|------|-----------------|-----------------|
| 1: Foundation & Ingestion | 1-2 | pipeline, schemas, orchestration | Docker stack, ingestion (5 formats), DB schema |
| 2: Classification | 3-4 | pipeline, orchestration | LLM classification, confidence gate, review queue |
| 3: Schemas & Extraction | 5-6 | schemas, pipeline | Pydantic schemas (5 types), structured extraction |
| 4: Validation & Retry | 7-8 | schemas, pipeline | Validation layer, retry-with-error-feedback |
| 5: Routing & Audit | 9 | orchestration, pipeline | Routing to DB + store + audit, documents/stats API |
| 6: Airflow DAG | 10 | orchestration (sequential) | DAG with retries + SLA, CLI, E2E tests |
| 7: Streamlit & Hardening | 11-12 | orchestration, pipeline, schemas | Review UI, structured logging, full test suite |

### Critical Integration Points

| From | To | What | Phase |
|------|----|------|-------|
| pipeline (classify/service.py) | orchestration (review/queue.py) | `enqueue_for_review()` call | 2 |
| pipeline (extract/service.py) | schemas (schemas/registry.py) | `get_schema_for_doc_type()` import | 3 |
| pipeline (extract/service.py) | schemas (validate/service.py) | `validate_extraction()` import | 4 |
| pipeline (extract/service.py) | orchestration (review/queue.py) | `enqueue_for_review()` for validation failures | 4 |
| orchestration (route/router.py) | pipeline (extract/service.py) | reads extraction_records | 5 |
| orchestration (api/routes/*.py) | pipeline (main.py) | router registration | 2, 5 |
| orchestration (airflow DAG) | pipeline (all services) | calls ingest/classify/extract/route batch functions | 6 |
| orchestration (streamlit) | orchestration (api/routes/review.py) | HTTP calls to review API | 7 |

### Test Commands

```bash
# Per-phase test runs
pytest tests/test_ingest.py -v                    # Phase 1
pytest tests/test_classify.py tests/test_review_queue.py -v  # Phase 2
pytest tests/test_schemas.py tests/test_extract.py -v        # Phase 3
pytest tests/test_validate.py tests/test_retry.py -v         # Phase 4
pytest tests/test_route.py tests/test_audit.py -v            # Phase 5
pytest tests/test_dag.py tests/test_e2e.py -v                # Phase 6
pytest tests/test_api.py -v                                  # Phase 7

# Full suite
pytest -v

# Docker verification
docker compose up -d
curl http://localhost:8000/health
python scripts/init_db.py
```
