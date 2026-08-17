# Herdr Multi-Agent Orchestration Guide — Customer Support AI System (P11)

> **Capstone project.** This guide orchestrates 5 Herdr agents to build the end-to-end
> Customer Support AI System described in `IMPLEMENTATION_PLAN.md`. Every phase, prompt,
> and command below is derived from that plan — references to sections, pseudocode, file
> paths, and table names are intentional and exact.

---

## 1. Agent Roster

| Agent Name   | Kind   | Responsibility                                                                                  | Owns (files / dirs)                                                                                                                                                             |
|--------------|--------|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `ingest`     | codex  | Ticket ingestion, PII redaction (Presidio), ticket classification, audit logger, API ticket routes | `src/ingest/`, `src/pii/`, `src/classify/`, `src/audit/`, `src/api/routes/tickets.py`, `src/api/routes/health.py`, `tests/test_pii.py`, `tests/test_classify.py`, `tests/test_audit.py` |
| `rag`        | codex  | KB document indexing, hybrid BM25+vector retrieval, cross-encoder re-ranking, permission filter, KB API routes | `src/rag/`, `src/api/routes/kb.py`, `tests/test_rag.py`, `tests/test_security.py`                                                                                                  |
| `pipeline`   | codex  | Response drafting, confidence scoring, routing, cost tracking, pipeline orchestrator, eval harness, eval API routes | `src/draft/`, `src/confidence/`, `src/cost/`, `src/pipeline/`, `src/eval/`, `src/api/routes/eval.py`, `tests/test_draft.py`, `tests/test_confidence.py`, `tests/test_cost.py`, `tests/test_pipeline.py`, `tests/test_eval.py` |
| `frontend`   | codex  | Human review queue (Streamlit), monitoring dashboard (Streamlit), feedback collector, review & metrics API routes | `src/review/`, `src/api/routes/review.py`, `src/api/routes/metrics.py`, `dashboards/review_queue/`, `dashboards/monitoring/`, `tests/test_review.py`, `tests/test_api.py`           |
| `reviewer`   | codex  | Code review after every phase: correctness, security (PII/tenant isolation), test coverage, plan adherence | `tests/` (read-only review), all `src/` (read-only review)                                                                                                                       |

### Shared infrastructure (Phase 1, owned by `ingest` with `pipeline` support)

| File                                | Owner      |
|-------------------------------------|------------|
| `docker-compose.yml`                | `ingest`   |
| `docker/postgres/init.sql`          | `ingest`   |
| `docker/Dockerfile.api`             | `ingest`   |
| `docker/Dockerfile.review`          | `frontend` |
| `docker/Dockerfile.monitor`         | `frontend` |
| `src/config.py`                     | `ingest`   |
| `src/db/schema.sql`                 | `ingest`   |
| `src/db/connection.py`              | `ingest`   |
| `src/db/migrations/`                | `ingest`   |
| `src/cache/redis_client.py`         | `pipeline` |
| `src/main.py`                       | `ingest`   |
| `src/api/dependencies.py`           | `ingest`   |
| `scripts/init_db.py`                | `ingest`   |
| `scripts/seed_test_data.py`         | `ingest`   |
| `scripts/seed_kb.py`                | `rag`      |
| `scripts/run_eval.py`               | `pipeline` |
| `scripts/generate_agent_token.py`   | `ingest`   |
| `tests/conftest.py`                 | `pipeline` |
| `pyproject.toml`                    | `ingest`   |
| `.env.example`                      | `ingest`   |
| `.github/workflows/ci.yml`          | `reviewer` |

---

## 2. Pane Layout

```
┌─────────────────────────────────────────────────────────────┐
│                     WORKSPACE: support-ai                    │
├────────────────────────────┬────────────────────────────────┤
│                            │                                │
│   p1 — ingest              │   p2 — rag                     │
│   (ticket ingestion,       │   (KB indexing,                │
│    PII redaction,          │    hybrid retrieval,           │
│    classification)         │    re-ranking)                 │
│                            │                                │
├────────────────────────────┼────────────────────────────────┤
│                            │                                │
│   p3 — pipeline            │   p4 — frontend                │
│   (drafting, confidence,   │   (review queue,               │
│    routing, cost, eval,    │    monitoring dashboard,       │
│    orchestrator)           │    feedback)                   │
│                            │                                │
├────────────────────────────┴────────────────────────────────┤
│                                                             │
│   p5 — reviewer                                             │
│   (code review, security audit, test verification)          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Setup Commands

> Run these from the project root (`support-ai/`). All splits use `--cwd "$PWD" --no-focus`
> to preserve the working directory and keep focus on the caller.

```bash
# ── Create the workspace and first tab (pane p1 is the root pane) ──
herdr workspace create --name support-ai
# Read workspace_id, tab_id, and root_pane_id from the JSON response.
# The root pane becomes p1 (ingest).

# ── Split p2 (rag) to the right of p1 ──
herdr pane split --current --direction right --cwd "$PWD" --no-focus
# Read .result.pane.pane_id → this is p2

# ── Split p3 (pipeline) down from p1 ──
herdr pane split --pane <p1-id> --direction down --cwd "$PWD" --no-focus
# Read .result.pane.pane_id → this is p3

# ── Split p4 (frontend) down from p2 ──
herdr pane split --pane <p2-id> --direction down --cwd "$PWD" --no-focus
# Read .result.pane.pane_id → this is p4

# ── Split p5 (reviewer) down from p3 (full width bottom) ──
herdr pane split --pane <p3-id> --direction down --cwd "$PWD" --no-focus
# Read .result.pane.pane_id → this is p5

# ── Start agents in each pane ──
herdr agent start ingest   --kind codex --pane <p1-id>
herdr agent start rag      --kind codex --pane <p2-id>
herdr agent start pipeline --kind codex --pane <p3-id>
herdr agent start frontend --kind codex --pane <p4-id>
herdr agent start reviewer --kind codex --pane <p5-id>
```

### Verification after setup

```bash
herdr agent list
# All 5 agents should show state: idle
```

---

## 4. Phase-by-Phase Orchestration

### Phase 1: Foundation & Infrastructure (Days 1–2)

**Goal:** Running Postgres + pgvector + Redis + FastAPI skeleton with auth and DB schema.
(IMPLEMENTATION_PLAN.md §8 Phase 1, §5 Tech Stack, §6 Database Schema, §16 Deployment)

#### Parallel Work

```
ingest:
Read IMPLEMENTATION_PLAN.md sections §5 (Tech Stack), §6 (Database Schema — all DDL
including tables: tenants, agents, tickets, classifications, kb_documents, kb_chunks,
draft_responses, review_queue, cost_ledger, audit_logs, eval_results, feedback,
pii_entities; RLS policies in §6.3; triggers prevent_audit_modification), §16.1
(docker-compose.yml), §16.5 (docker/postgres/init.sql), §16.6 (.env.example).

Create the following files exactly as specified in the plan:

1. docker-compose.yml — 5 services: postgres (pgvector/pgvector:pg16), redis
   (redis:7-alpine), api (FastAPI, port 8000), review_queue (Streamlit, port 8501),
   monitoring (Streamlit, port 8502). Use the YAML from §16.1 with healthchecks and
   depends_on conditions.

2. docker/postgres/init.sql — CREATE EXTENSION for uuid-ossp, vector, pg_trgm (§16.5).

3. docker/Dockerfile.api — Python 3.12-slim, install build-essential libpq-dev curl
   poppler-utils, pip install -e ., spacy download en_core_web_lg, pre-download
   cross-encoder/ms-marco-MiniLM-L-6-v2, CMD uvicorn src.main:app (§16.2).

4. pyproject.toml — dependencies: fastapi, uvicorn, asyncpg, pgvector, redis,
   presidio-analyzer, presidio-anonymizer, spacy, sentence-transformers, openai,
   langchain, langchain-community, pydantic, pydantic-settings, structlog,
   pytest, pytest-asyncio, httpx, ruff, streamlit, plotly, python-jose.

5. .env.example — all env vars from §16.6.

6. src/config.py — Pydantic Settings class reading all env vars: DATABASE_URL,
   REDIS_URL, JWT_SECRET, JWT_ALGORITHM, JWT_EXPIRY_HOURS, OPENAI_API_KEY,
   EMBEDDING_MODEL, LLM_MODEL, RERANKER_MODEL, AUTO_RESPONSE_THRESHOLD,
   PII_ENCRYPTION_KEY, LOG_LEVEL, MAX_TICKET_BODY_LENGTH, EVAL_JUDGE_MODEL,
   EVAL_SCHEDULE_HOURS.

7. src/db/schema.sql — Full DDL from §6.2: all 13 tables with columns, constraints,
   CHECK constraints (valid_urgency, valid_category, valid_sentiment, valid_confidence,
   valid_route, valid_review_status, valid_feedback_action), all indexes (HNSW on
   kb_chunks.embedding, GIN on allowed_roles and content trigram, B-tree on tenant_id),
   RLS policies from §6.3, and the prevent_audit_modification trigger function +
   no_audit_update + no_audit_delete triggers from §6.2.

8. src/db/connection.py — asyncpg connection pool, pgvector type registration,
   set_tenant_context(tenant_id) helper that executes SET LOCAL
   app.current_tenant_id.

9. src/db/migrations/001_initial.sql, 002_rls.sql, 003_indexes.sql — split schema.sql
   into ordered migrations.

10. src/api/dependencies.py — FastAPI DI: JWT decode (python-jose), get_current_agent
    (lookup in agents table), extract tenant_id from X-Tenant-ID header or JWT claim,
    set tenant DB context per request.

11. src/main.py — FastAPI app factory with lifespan (init DB pool, Redis connection),
    include health routes, mount all route modules. Background task runner for pipeline.

12. src/api/routes/health.py — GET /health returning {status, database, redis, openai,
    version} per §14.10.

13. scripts/init_db.py — run schema.sql against fresh database.
14. scripts/seed_test_data.py — insert 2 tenants (acme, globex), agents, sample tickets.
15. scripts/generate_agent_token.py — create JWT for review queue auth (CLI args:
    --tenant, --agent, --role).

Verification gate: docker compose up → curl localhost:8000/health → 200 OK.
pytest tests/test_audit.py -k immutability passes (audit triggers work).
```

```
pipeline:
Read IMPLEMENTATION_PLAN.md §5 (Tech Stack — Redis for caching), §7 (Project Structure
— src/cache/redis_client.py), §15.6 (Test Fixtures — conftest.py).

Create:

1. src/cache/redis_client.py — Redis connection pool, helpers: get(key), setex(key,
   ttl, value), delete(key), exists(key). Connection from config.REDIS_URL. Include
   circuit breaker state helpers (get/set circuit_open flag per service).

2. tests/conftest.py — Test fixtures per §15.6:
   - db fixture: fresh asyncpg connection, run schema.sql, yield, close.
   - mock_llm fixture: MockLLMClient returning deterministic responses
     (classification_response: {urgency: "medium", category: "technical",
     sentiment: "neutral", confidence: 0.82}; draft_response: "Hi [NAME_1], to
     reset your password [1]...").
   - mock_embeddings fixture: deterministic 1536-dim vectors.
   - acme_token fixture: JWT for tenant acme, agent-001, role agent.
   - globex_token fixture: JWT for tenant globex, agent-002, role agent.
   - seed_data fixture: insert tenants, agents, KB docs for both tenants.

3. scripts/run_eval.py — CLI to trigger eval suite manually (calls POST /eval/run).
```

#### Sequential (after parallel)

```
ingest:
After pipeline completes conftest.py and redis_client.py, verify integration:
- Run: docker compose up -d
- Run: docker compose exec api python scripts/init_db.py
- Run: docker compose exec api python scripts/seed_test_data.py
- Run: curl http://localhost:8000/health
Confirm 200 OK with database: connected, redis: connected.
```

#### Review

```
reviewer:
Review Phase 1 deliverables against IMPLEMENTATION_PLAN.md §6 (Database Schema) and
§16 (Deployment). Check:
1. All 13 tables in schema.sql match §6.2 exactly (columns, types, constraints).
2. RLS policies enabled on all 10 tables listed in §6.3.
3. prevent_audit_modification trigger function + no_audit_update + no_audit_delete
   triggers present.
4. HNSW index on kb_chunks.embedding with vector_cosine_ops, m=16, ef_construction=64.
5. GIN indexes on allowed_roles and content gin_trgm_ops.
6. docker-compose.yml has all 5 services with correct ports and healthchecks.
7. config.py reads every env var listed in §16.6 .env.example.
8. dependencies.py implements JWT decode + tenant context setting.
9. conftest.py has all fixtures from §15.6 (db, mock_llm, acme_token, globex_token).
Report any discrepancies as a numbered list. Do NOT edit files — report only.
```

#### Commit

```bash
git add -A && git commit -m "Phase 1: Foundation & Infrastructure — Docker Compose, DB schema with RLS, FastAPI skeleton, Redis cache, test fixtures"
```

---

### Phase 2: Ticket Ingestion & PII Redaction (Days 3–4)

**Goal:** Tickets arrive via API, PII is detected and redacted before any LLM call.
(IMPLEMENTATION_PLAN.md §8 Phase 2, §9.1 PII Redactor, §9.2 PII Restorer, §13.2 PII Safety Flow)

#### Parallel Work

```
ingest:
Read IMPLEMENTATION_PLAN.md §8 Phase 2 (steps 2.1–2.9), §9.1 (PII Redactor pseudocode),
§9.2 (PII Restorer pseudocode), §13.2 (PII Safety Flow), §14.1 (POST /tickets API spec),
§14.2 (GET /tickets/{id} API spec), §15.2 (Critical PII Safety Tests).

Create:

1. src/pii/recognizers.py — Custom Presidio recognizers: IBANRecognizer (regex for
   IBAN patterns DE\d{20} etc.), GermanPhoneRecognizer (+49 patterns), AddressRecognizer.
   Each extends PatternRecognizer or EntityRecognizer.

2. src/pii/encryption.py — encrypt_value(plaintext) and decrypt_value(ciphertext) using
   Fernet symmetric encryption with PII_ENCRYPTION_KEY from config. Original PII values
   are encrypted before storing in pii_entities table.

3. src/pii/redactor.py — PIIRedactor class per §9.1 pseudocode:
   - __init__: AnalyzerEngine, register IBANRecognizer, GermanPhoneRecognizer.
   - async redact(text, ticket_id, tenant_id) → RedactionResult:
     a. analyzer.analyze with entities ["PERSON", "EMAIL_ADDRESS", "PHONE_NUMBER",
        "IBAN", "LOCATION", "US_SSN"], language="en".
     b. Sort results by start position descending (so replacements don't shift offsets).
     c. For each result: generate token [TYPE_N] (incrementing counter per type),
        replace in text, store {token, entity_type, encrypt_value(original), start, end}.
     d. Persist to pii_entities table.
     e. Audit log: stage="pii_scanned", event_data={entities_found, entity_types}.
     f. Return RedactionResult(redacted_text, entity_count, entity_types).

4. src/pii/restorer.py — restore_pii(text, ticket_id, tenant_id) per §9.2:
   - Load all pii_entities for ticket_id.
   - For each entity: decrypt original_value, replace token with original in text.
   - Return restored text.

5. src/ingest/ticket_receiver.py — receive_ticket(payload, tenant_id):
   - Validate schema (subject, body required; customer_email, customer_name optional).
   - Encrypt customer_email and customer_name at rest.
   - INSERT into tickets table (status='new').
   - Audit log: stage="ticket_received".
   - Trigger pipeline background task: process_ticket(ticket_id, tenant_id).
   - Return 202 {ticket_id, status, message}.

6. src/ingest/email_parser.py — IMAP poller stub (optional, per §1 Non-Goals: v1 uses
   API endpoint + optional IMAP poller stub). Skeleton class with connect/poll/parse
   methods that can be implemented in v2.

7. src/api/routes/tickets.py — POST /tickets (calls ticket_receiver), GET /tickets/{id}
   (returns ticket status, classification, draft, cost per §14.2 response shape).

8. tests/test_pii.py — Tests per §15.2:
   - test_no_real_pii_in_redacted_text: "Jane Doe", "jane@example.com", "+49 170 1234567"
     all replaced with tokens.
   - test_pii_restoration_replaces_all_tokens: every token replaced with original.
   - test_no_tokens_in_sent_response: end-to-end, no [PERSON_], [EMAIL_], [PHONE_] in
     sent text; real PII present.
   - test_pii_scan_failure_routes_to_human: mock Presidio failure → ticket status
     pending_review, mock_llm.call_count == 0.

9. tests/test_audit.py — Tests:
   - test_audit_log_immutable: UPDATE and DELETE on audit_logs raise Exception.
   - test_audit_completeness: after processing, audit_events contain all expected
     stages (ticket_received, pii_scanned at minimum for this phase).
```

#### Sequential (after parallel)

_(No sequential dependencies in Phase 2 — ingest owns all files.)_

#### Review

```
reviewer:
Review Phase 2 against IMPLEMENTATION_PLAN.md §9.1, §9.2, §13.2, §15.2. Check:
1. PIIRedactor sorts results descending by start position (critical for offset safety).
2. Original PII values are encrypted via encrypt_value before storing in pii_entities.
3. Redactor logs audit event "pii_scanned" with entity count and types.
4. Restorer decrypts and replaces ALL tokens — no token left behind.
5. ticket_receiver encrypts customer_email and customer_name at rest.
6. POST /tickets returns 202 with ticket_id per §14.1.
7. GET /tickets/{id} returns the response shape from §14.2.
8. test_pii.py covers all 4 test cases from §15.2 exactly.
9. test_audit.py verifies immutability (UPDATE + DELETE both raise).
10. INVARIANT from §13.2: no real PII reaches LLM; no tokens reach customer.
Report findings as numbered list. Do NOT edit files.
```

#### Commit

```bash
git add -A && git commit -m "Phase 2: Ticket ingestion, PII redaction (Presidio), PII restoration, audit logging, ticket API routes"
```

---

### Phase 3: Knowledge Base RAG (Days 5–6)

**Goal:** KB documents ingested, hybrid search + re-ranking working with tenant filtering.
(IMPLEMENTATION_PLAN.md §8 Phase 3, §9.4 RAG Service, §2 Component #4)

#### Parallel Work

```
rag:
Read IMPLEMENTATION_PLAN.md §8 Phase 3 (steps 3.1–3.10), §9.4 (RAG Service pseudocode),
§6.2 (kb_documents and kb_chunks table schemas — embedding vector(1536), HNSW index,
trigram index, allowed_roles TEXT[]), §14.3 (POST /kb/upload API spec), §15.3 (Security
Tests — cross-tenant KB leakage).

Create:

1. src/ingest/kb_loader.py — load_kb_document(file, tenant_id, source_type,
   allowed_roles):
   - Parse file (PDF, TXT, MD) using LangChain document loaders.
   - Chunk using RecursiveCharacterTextSplitter (chunk_size=512, overlap=64).
   - Embed each chunk with text-embedding-3-small (1536-dim).
   - INSERT into kb_documents (title, source_type, file_hash SHA-256 for dedup,
     chunk_count, allowed_roles).
   - INSERT into kb_chunks (doc_id, tenant_id, chunk_idx, content, embedding,
     allowed_roles).
   - Return {doc_id, chunk_count, title}.

2. src/api/routes/kb.py — POST /kb/upload (multipart/form-data: file, source_type,
   allowed_roles) per §14.3. Requires admin or editor role. GET /kb/documents lists
   documents for current tenant.

3. src/rag/query_formulator.py — build(ticket, classification) → str:
   - Combine ticket.subject + ticket.body_redacted + classification metadata into a
     RAG query string.
   - Example: "billing charges refund policy — customer is angry about unexpected
     charge" (from §9.4 comment).
   - Classification helps disambiguate: "billing" ticket about "charges" → retrieve
     billing FAQ, not technical docs.

4. src/rag/bm25_search.py — search(query, tenant_id, limit):
   - Use pg_trgm GIN index for keyword/trigram similarity search on kb_chunks.content.
   - Filter by tenant_id.
   - Return candidates with similarity scores.
   - (From P1: BM25-like keyword search.)

5. src/rag/vector_search.py — search(query, tenant_id, limit):
   - Embed query with text-embedding-3-small.
   - pgvector cosine similarity query: SELECT ... ORDER BY embedding <=> query_embedding.
   - Tenant pre-filter: WHERE tenant_id = $1 (from P2 tenant isolation).
   - Return candidates with cosine distance.
   - (From P1+P2: vector search + tenant filter.)

6. src/rag/reranker.py — rerank(query, candidates, top_n):
   - Load cross-encoder/ms-marco-MiniLM-L-6-v2 (sentence-transformers CrossEncoder).
   - Score each (query, candidate.content) pair.
   - Sort by score descending, return top_n.
   - (From P1: cross-encoder re-ranking.)

7. src/rag/permission_filter.py — filter_by_role(candidates, user_roles):
   - Keep only chunks where set(chunk.allowed_roles) & set(user_roles) is non-empty.
   - (From P2: role-based post-filter.)

8. src/rag/service.py — retrieve(ticket, classification, user_roles, top_k=5,
   oversample=3) per §9.4 pseudocode:
   a. query = query_formulator.build(ticket, classification).
   b. Parallel: bm25_search.search + vector_search.search (asyncio.gather), each with
      limit=top_k * oversample * 2.
   c. Merge via Reciprocal Rank Fusion (RRF) — from P1.
   d. reranker.rerank(query, merged, top_k * oversample).
   e. permission_filter.filter_by_role(reranked, user_roles).
   f. Take top_k.
   g. Audit log: stage="rag_retrieved", event_data={query, bm25_count,
      vector_count, merged_count, reranked_count, final_count, chunk_ids,
      avg_distance}.

9. scripts/seed_kb.py — Load sample support docs + FAQs into KB for both tenants
   (acme, globex). Include password reset FAQ, billing FAQ, technical docs.

10. tests/test_rag.py — Tests:
    - test_hybrid_search_returns_relevant_chunks: upload billing FAQ → query billing
      ticket → billing chunks in top-k.
    - test_reranking_improves_precision: compare top-k before vs after re-ranking;
      re-ranked order should be more relevant.
    - test_tenant_filter_isolation: tenant A query returns only tenant A chunks.
    - test_role_filter: chunks with allowed_roles=['supervisor'] filtered out for
      agent role.
    - test_empty_kb_returns_empty: no KB docs → empty list, no error.

11. tests/test_security.py — Tests per §15.3:
    - test_tenant_a_cannot_retrieve_tenant_b_kb: upload to tenant_b → retrieve as
      tenant_a → all results have tenant_id == tenant_a (or empty).
    - test_rls_blocks_cross_tenant_db: direct DB connection as tenant_a → SELECT *
      FROM tickets → all rows have tenant_id == tenant_a.
```

#### Sequential (after parallel)

_(No sequential dependencies — rag owns all Phase 3 files.)_

#### Review

```
reviewer:
Review Phase 3 against IMPLEMENTATION_PLAN.md §9.4, §6.2 (kb_chunks schema), §15.3. Check:
1. kb_loader computes SHA-256 file_hash for dedup (UNIQUE constraint on
   kb_documents(tenant_id, file_hash)).
2. query_formulator uses classification to disambiguate (not just raw ticket text).
3. bm25_search uses pg_trgm GIN index; vector_search uses pgvector cosine (<=>
   operator) with tenant pre-filter.
4. reranker uses cross-encoder/ms-marco-MiniLM-L-6-v2 from sentence-transformers.
5. service.py uses Reciprocal Rank Fusion (RRF) to merge BM25 + vector results.
6. permission_filter checks set intersection of allowed_roles and user_roles.
7. service.py logs audit event "rag_retrieved" with all counts and avg_distance.
8. POST /kb/upload returns {doc_id, chunk_count, title} per §14.3.
9. test_security.py has test_tenant_a_cannot_retrieve_tenant_b_kb and
   test_rls_blocks_cross_tenant_db from §15.3.
10. No cross-tenant leakage possible: tenant_id filter at SQL level + RLS as backstop.
Report findings as numbered list. Do NOT edit files.
```

#### Commit

```bash
git add -A && git commit -m "Phase 3: KB indexing, hybrid BM25+vector retrieval, cross-encoder re-ranking, tenant+role filtering, KB API routes, security tests"
```

---

### Phase 4: Classification & Response Drafting (Days 7–8)

**Goal:** LLM classifies tickets and drafts grounded responses with citations.
(IMPLEMENTATION_PLAN.md §8 Phase 4, §9.3 Ticket Classifier, §2 Components #3, #5)

#### Parallel Work

```
ingest:
Read IMPLEMENTATION_PLAN.md §8 Phase 4 (steps 4.1–4.3), §9.3 (Ticket Classifier
pseudocode with CLASSIFY_PROMPT), §6.2 (classifications table schema with CHECK
constraints: valid_urgency, valid_category, valid_sentiment, valid_confidence),
§15.1 (test categories).

Create:

1. src/classify/models.py — Pydantic models for structured LLM output:
   - ClassificationResult: urgency (Literal['critical','high','medium','low']),
     category (Literal['billing','technical','general','complaint']),
     sentiment (Literal['positive','neutral','negative','angry']),
     confidence (float 0.0–1.0), reasoning (str), model_used (str), tokens_used (int).
   - ClassificationResult matches the JSON schema in the CLASSIFY_PROMPT from §9.3.

2. src/classify/classifier.py — classify_ticket(subject, body, ticket_id, tenant_id)
   per §9.3 pseudocode:
   a. Check Redis cache: cache_key = "classify:" + sha256(body).hex(). If cached,
      return ClassificationResult.parse_raw(cached).
   b. Call LLM with CLASSIFY_PROMPT (the exact prompt from §9.3 — includes
      classification guidelines for urgency levels, complaint vs negative, angry
      vs negative). Use JSON mode / structured output with Pydantic validation.
   c. Validate with Pydantic (raises on schema mismatch → retry with error feedback).
   d. Cache for 24h: redis.setex(cache_key, 86400, result.json()).
   e. Store in classifications table.
   f. Cost tracking: cost_tracker.record(stage="classify", ...).
   g. Audit: stage="classified", event_data={urgency, category, sentiment, confidence}.

3. src/classify/cache.py — Redis classification cache helpers wrapping the cache
   logic from §9.3 (get_cached_classification, cache_classification with 24h TTL).

4. tests/test_classify.py — Tests:
   - test_structured_output_validation: LLM returns valid JSON matching schema →
     ClassificationResult created successfully.
   - test_invalid_urgency_raises: LLM returns urgency="urgent" (invalid) → Pydantic
     raises ValidationError.
   - test_cache_hit_avoids_llm_call: same body twice → second call uses cache,
     mock_llm.call_count == 1.
   - test_classification_stored_in_db: after classify, classifications table has row
     with correct urgency/category/sentiment/confidence.
   - test_classification_audit_logged: audit_logs has "classified" event.
```

```
pipeline:
Read IMPLEMENTATION_PLAN.md §8 Phase 4 (steps 4.4–4.7, 4.9), §9.4 (RAG Service —
draft uses retrieved context), §2 Component #5 (Response Drafting — tone adapted to
sentiment, citations for human reviewer, stripped from auto-response), §14.2 (draft
field in GET /tickets response).

Create:

1. src/draft/prompt_builder.py — build_prompt(ticket, classification, retrieved_chunks)
   → str:
   - System prompt: "You are a customer support agent drafting a response."
   - Include: redacted ticket subject + body, retrieved context chunks (numbered [1],
     [2], ...), classification (urgency, category, sentiment).
   - Tone adaptation: if sentiment="angry" → empathetic, apologetic tone; if
     sentiment="positive" → warm, appreciative; if "neutral" → professional, concise.
   - Citation instructions: "Cite sources using [1], [2] markers corresponding to the
     numbered context above."
   - Self-assessment: "At the end, include a line 'CONFIDENCE: X.XX' rating your
     confidence (0.00–1.00) that this response fully and correctly addresses the
     ticket."
   - Context-only: response must be grounded in retrieved context only (from P2).

2. src/draft/llm_client.py — LLMClient class:
   - Primary: OpenAI (gpt-4o-mini from config.LLM_MODEL).
   - Fallback: Ollama (Llama 3.1 8B via config.OLLAMA_HOST) — from P9 fallback pattern.
   - generate(prompt) → LLMResponse(text, prompt_tokens, completion_tokens, model).
   - generate_json(prompt, response_model) → parsed Pydantic model (for classification).
   - Retry 2× with exponential backoff on timeout/error.

3. src/draft/citation_extractor.py — extract_citations(draft_text, retrieved_chunks)
   → list[Citation]:
   - Parse [1], [2] markers from draft text.
   - Map each number to the corresponding retrieved chunk (by index).
   - Return structured citations: [{number, chunk_id, doc_title, content_snippet}].
   - Validate: every [N] marker has a matching chunk; flag orphan citations.

4. src/draft/service.py — generate(ticket, classification, retrieved_chunks) →
   DraftResult:
   a. prompt = prompt_builder.build_prompt(ticket, classification, retrieved_chunks).
   b. response = llm_client.generate(prompt).
   c. Parse self-assessment confidence from "CONFIDENCE: X.XX" line.
   d. draft_text_clean = strip citation markers [1], [2] from response text
      (customer-facing version).
   e. citations = citation_extractor.extract_citations(response.text, retrieved_chunks).
   f. Store in draft_responses table (draft_text, draft_text_clean, citations JSONB,
      retrieved_chunk_ids, model_used, tokens_used, cost_usd).
   g. Cost tracking: cost_tracker.record(stage="draft", ...).
   h. Audit: stage="response_drafted".
   i. Return DraftResult(draft_text, draft_text_clean, citations, self_confidence,
      retrieved_chunk_ids).

5. tests/test_draft.py — Tests:
   - test_draft_includes_citations: draft text contains [1] marker pointing to correct
     chunk.
   - test_draft_clean_strips_citations: draft_text_clean has no [1], [2] markers.
   - test_tone_adapts_to_sentiment: angry ticket → draft contains empathetic language;
     positive → warm language.
   - test_self_confidence_parsed: "CONFIDENCE: 0.85" line parsed to self_confidence=0.85.
   - test_citation_extractor_maps_markers: [1] maps to retrieved_chunks[0], [2] to [1].
   - test_llm_fallback_to_ollama: mock OpenAI failure → Ollama called.
   - test_redacted_pii_in_draft: draft contains [NAME_1] tokens, not real PII.
```

#### Sequential (after parallel)

_(Classification and drafting are independent in code; they connect in the pipeline
orchestrator in Phase 8. No sequential dependency needed here.)_

#### Review

```
reviewer:
Review Phase 4 against IMPLEMENTATION_PLAN.md §9.3 (Classifier), §2 Component #5
(Response Drafting), §6.2 (classifications + draft_responses schemas). Check:
1. CLASSIFY_PROMPT in classifier.py matches §9.3 exactly — includes classification
   guidelines for urgency levels, complaint vs negative, angry vs negative.
2. ClassificationResult Pydantic model enforces Literal types matching the CHECK
   constraints in §6.2 (valid_urgency, valid_category, valid_sentiment).
3. Classification cache uses sha256(body) as key, 24h TTL (86400 seconds).
4. Classification stores in classifications table and logs "classified" audit event.
5. prompt_builder adapts tone based on sentiment (angry→empathetic, positive→warm).
6. prompt_builder includes citation instructions [1], [2] and self-assessment
   "CONFIDENCE: X.XX" line.
7. citation_extractor maps [N] markers to retrieved chunks by index; validates no
   orphan citations.
8. draft_text_clean has citation markers stripped (customer-facing version).
9. llm_client has OpenAI primary + Ollama fallback with 2× retry + backoff.
10. Draft stores in draft_responses table with citations JSONB, retrieved_chunk_ids,
    cost_usd.
11. Redacted PII tokens ([NAME_1]) appear in draft, NOT real PII (§13.2 invariant).
Report findings as numbered list. Do NOT edit files.
```

#### Commit

```bash
git add -A && git commit -m "Phase 4: Ticket classification (structured LLM output + cache) and response drafting (citations, tone adaptation, LLM fallback)"
```

---

### Phase 5: Confidence Scoring, Routing & Cost Tracking (Day 9)

**Goal:** Composite confidence score drives routing; cost tracked per ticket and per tenant.
(IMPLEMENTATION_PLAN.md §8 Phase 5, §9.5 Confidence Scorer & Router, §9.7 Cost Tracker)

#### Parallel Work

```
pipeline:
Read IMPLEMENTATION_PLAN.md §8 Phase 5 (steps 5.1–5.7), §9.5 (Confidence Scorer &
Router pseudocode — WEIGHTS, AUTO_THRESHOLD, compute_confidence, should_auto_respond),
§9.7 (Cost Tracker pseudocode — PRICING, record, check_budget, aggregate_ticket),
§4.1 (Failure Modes & Fallbacks — budget_exceeded, confidence_defaulted), §6.2
(cost_ledger table schema).

Create:

1. src/confidence/scorer.py — compute_confidence(retrieved_chunks, classification,
   draft) per §9.5 pseudocode:
   - WEIGHTS = {"retrieval": 0.35, "classification": 0.20, "self_assess": 0.25,
     "coverage": 0.20}
   - retrieval_score = max(0.0, 1.0 - avg_distance) if chunks else 0.0
   - cls_score = float(classification.confidence)
   - self_score = draft.self_confidence or 0.0
   - coverage_score = 1.0 if chunks and chunks[0].distance < 0.3; 0.5 if chunks; 0.0
     if no chunks
   - score = weighted sum, rounded to 2 decimal places.
   - If all signals unavailable → default to 0.0 (route to human, audit
     "confidence_defaulted").

2. src/confidence/router.py — should_auto_respond(confidence, classification,
   tenant_budget_ok, pii_scan_ok, pipeline_ok) per §9.5 pseudocode:
   - Hard rules (always human, checked first):
     not pipeline_ok → human/pipeline_failure
     not pii_scan_ok → human/pii_scan_failed
     not tenant_budget_ok → human/budget_exceeded
     urgency == "critical" → human/critical_urgency
     sentiment == "angry" → human/angry_sentiment
     category == "complaint" → human/complaint_category
   - AUTO_THRESHOLD = 0.75 (from config.AUTO_RESPONSE_THRESHOLD)
   - confidence >= threshold → auto/confidence_above_threshold
   - else → human/confidence_below_threshold
   - Return RoutingDecision(route, reason).

3. src/cost/tracker.py — Per §9.7 pseudocode:
   - PRICING dict: gpt-4o-mini {input: 0.15e-6, output: 0.60e-6},
     text-embedding-3-small {input: 0.02e-6, output: 0.0},
     cross-encoder/ms-marco-MiniLM-L-6-v2 {input: 0.0, output: 0.0}.
   - record(tenant_id, ticket_id, stage, model, input_tokens, output_tokens):
     calculate cost, INSERT into cost_ledger, UPDATE tenants.current_spend_usd.
   - aggregate_ticket(ticket_id, tenant_id): SUM(cost_usd) from cost_ledger.

4. src/cost/budget.py — Per §9.7 + §4.1:
   - check_budget(tenant_id): SELECT current_spend_usd, monthly_budget_usd FROM
     tenants; return current_spend < monthly_budget.
   - reset_monthly_budget(tenant_id): if current date >= budget_reset_day, reset
     current_spend_usd to 0.
   - Budget checked BEFORE draft LLM call (per §4.1: budget exceeded → route to
     human, no LLM call, audit "budget_exceeded").

5. Wire cost tracking into classify (stage="classify"), draft (stage="draft"),
   and eval (stage="eval") — ensure every LLM call records cost. This means
   updating the imports/calls in src/classify/classifier.py and src/draft/service.py
   to use cost_tracker.record. Coordinate with ingest agent via hub message before
   editing src/classify/classifier.py.

6. tests/test_confidence.py — Tests per §15.1:
   - test_high_confidence_routes_auto: score >= 0.75, no hard rule triggers →
     route="auto".
   - test_low_confidence_routes_human: score < 0.75 → route="human".
   - test_critical_urgency_overrides_score: urgency="critical" + score=0.99 →
     route="human", reason="critical_urgency".
   - test_angry_sentiment_overrides: sentiment="angry" + score=0.99 → human.
   - test_complaint_category_overrides: category="complaint" + score=0.99 → human.
   - test_budget_exceeded_routes_human: tenant_budget_ok=False → human.
   - test_pii_scan_failed_routes_human: pii_scan_ok=False → human.
   - test_empty_retrieval_zero_confidence: no chunks → retrieval_score=0,
     coverage_score=0.
   - test_weighted_sum_correct: verify score = 0.35*ret + 0.20*cls + 0.25*self +
     0.20*cov.

7. tests/test_cost.py — Tests:
   - test_cost_calculation_gpt4o_mini: 1000 input + 500 output tokens → cost =
     1000*0.15e-6 + 500*0.60e-6 = 0.00045.
   - test_cost_recorded_in_ledger: after record(), cost_ledger has row with correct
     stage, model, tokens, cost.
   - test_tenant_spend_updated: after record(), tenants.current_spend_usd
     incremented.
   - test_budget_exceeded_returns_false: set current_spend > monthly_budget →
     check_budget returns False.
   - test_budget_not_exceeded_returns_true: current_spend < monthly_budget → True.
   - test_aggregate_ticket_sums_all_stages: 3 cost_ledger rows for ticket →
     aggregate returns sum.
   - test_budget_exceeded_no_draft_call: per §15.4 test_budget_exceeded_routes_to_human
     — cost_ledger stages do NOT include "draft".
```

#### Sequential (after parallel)

```
pipeline:
After parallel work, wire cost_tracker.record calls into classifier.py and
draft/service.py. Message the ingest agent via hub to coordinate the edit to
src/classify/classifier.py (ingest owns that file). The call pattern from §9.3:
  await cost_tracker.record(
      tenant_id=tenant_id, ticket_id=ticket_id, stage="classify",
      model=result.model_used, input_tokens=response.prompt_tokens,
      output_tokens=response.completion_tokens)
And from §9.8 for eval:
  await cost_tracker.record(
      tenant_id=tenant_id, ticket_id=ticket.id, stage="eval",
      model=judge.model_name, input_tokens=judge.last_input_tokens,
      output_tokens=judge.last_output_tokens)
```

#### Review

```
reviewer:
Review Phase 5 against IMPLEMENTATION_PLAN.md §9.5, §9.7, §4.1. Check:
1. WEIGHTS in scorer.py match §9.5 exactly: retrieval=0.35, classification=0.20,
   self_assess=0.25, coverage=0.20.
2. AUTO_THRESHOLD = 0.75 (or from config.AUTO_RESPONSE_THRESHOLD).
3. Hard rules in router.py checked BEFORE confidence threshold, in the order from
   §9.5: pipeline_ok → pii_scan_ok → tenant_budget_ok → critical → angry → complaint.
4. PRICING in tracker.py matches §9.7: gpt-4o-mini input=0.15e-6, output=0.60e-6;
   embedding input=0.02e-6; cross-encoder free.
5. Cost uses ACTUAL token counts from API response (not estimated) — per Appendix B
   design decision.
6. Budget checked BEFORE draft LLM call (pre-check, not post-check/refund) — per
   Appendix B.
7. Budget exceeded → route to human, NO draft LLM call, audit "budget_exceeded".
8. test_confidence.py covers all hard rule overrides + threshold boundary.
9. test_cost.py verifies calculation accuracy and budget enforcement.
10. All LLM-calling components (classify, draft, eval) wired to cost_tracker.record.
Report findings as numbered list. Do NOT edit files.
```

#### Commit

```bash
git add -A && git commit -m "Phase 5: Composite confidence scoring, routing with hard rules, per-call cost tracking, per-tenant budget enforcement"
```

---

### Phase 6: Human Review Queue (Days 10–11)

**Goal:** Streamlit dashboard where agents review, edit, approve, or reject drafts with feedback.
(IMPLEMENTATION_PLAN.md §8 Phase 6, §10 Human Review Queue Design, §9.9 Feedback Collector)

#### Parallel Work

```
frontend:
Read IMPLEMENTATION_PLAN.md §8 Phase 6 (steps 6.1–6.7), §10 (Human Review Queue Design
— §10.1 Queue Data Model, §10.2 Dashboard Layout, §10.3 Agent Workflow, §10.4 Queue API
Endpoints, §10.5 Concurrency & Claim Mechanism), §9.9 (Feedback Collector pseudocode),
§14.4–14.7 (Review API specs), §6.2 (review_queue + feedback table schemas).

Create:

1. src/review/queue_manager.py — Queue operations per §10.1 + §10.4:
   - enqueue(ticket_id, tenant_id, draft_id, priority, reason): INSERT into
     review_queue (status='pending', priority from urgency mapping: critical=4,
     high=3, medium=2, low=1).
   - list_queue(tenant_id, status, limit, page): SELECT from review_queue ORDER BY
     priority DESC, queued_at ASC. Returns items per §14.4 response shape.
   - get_review_item(review_id, tenant_id): full detail — ticket body (redacted),
     classification, retrieved chunks, draft text with citations, PII entity map.
   - claim(review_id, agent_id): if status=='pending' → set 'in_review',
     assigned_to=agent. If 'in_review' → return 409 Conflict (§10.5).
   - release(review_id): set 'pending', assigned_to=null.
   - approve(review_id, feedback_tags, feedback_text): set 'approved', call
     feedback_collector, restore PII, send response, update ticket status
     'human_resolved'.
   - edit(review_id, edited_text, feedback_tags, feedback_text): set 'edited',
     call feedback_collector with final_response, restore PII in edited text, send,
     update ticket 'human_resolved', store edit_diff.
   - reject(review_id, feedback_tags, feedback_text): set 'rejected', call
     feedback_collector, update ticket 'human_rejected', re-queue or escalate.

2. src/review/feedback_collector.py — capture_feedback(review_id, ticket_id,
   draft_id, agent_id, tenant_id, action, feedback_tags, feedback_text,
   final_response) per §9.9 pseudocode:
   - Get original draft from draft_responses.
   - INSERT into feedback table (original_draft, final_response, feedback_tags,
     feedback_text, action).
   - If action in ("rejected", "edited"): add to eval dataset via
     eval_dataset.add_example(...), set added_to_eval_dataset=TRUE,
     eval_dataset_version.
   - If action == "edited" and final_response: compute edit_diff, store in
     review_queue.edit_diff.
   - Audit: stage="reviewed", event_data={action, feedback_tags, added_to_eval}.

3. src/api/routes/review.py — Endpoints per §10.4 + §14.4–14.7:
   - GET /review/queue?status=pending&limit=20&page=1
   - GET /review/{review_id}
   - POST /review/{review_id}/claim
   - POST /review/{review_id}/release
   - POST /review/{review_id}/approve
   - POST /review/{review_id}/edit
   - POST /review/{review_id}/reject

4. dashboards/review_queue/app.py — Streamlit app per §10.2 Dashboard Layout:
   - Header: "Customer Support AI — Review Queue" + agent identity.
   - Left panel: QUEUE LIST (sorted by priority DESC, queued_at ASC). Each item
     shows ticket ID, urgency, category, sentiment.
   - Right panel: TICKET DETAIL + DRAFT EDITOR:
     - Subject, customer (tokens), received date.
     - Classification (urgency, category, sentiment, confidence).
     - Retrieved knowledge chunks with source links.
     - AI draft (editable text area — agent can modify).
     - Feedback section: checkbox tags (wrong_category, bad_tone, incorrect_info,
       missing_info, other) + free-text notes.
     - Buttons: [APPROVE AS-IS] [EDIT & APPROVE] [REJECT].
   - Footer: queue stats (pending, in review, resolved today).
   - Auto-refresh after action.

5. dashboards/review_queue/components.py — Reusable UI components:
   - ticket_card(ticket, classification): renders ticket metadata.
   - draft_editor(draft_text): editable text area with citation markers visible.
   - feedback_widget(): checkbox tags + text area.
   - queue_stats(stats): footer stats display.

6. dashboards/review_queue/api_client.py — HTTP client for API calls:
   - get_queue(status, limit), get_review_item(review_id), claim(review_id),
     approve(review_id, tags, text), edit(review_id, text, tags, text),
     reject(review_id, tags, text).
   - Auth: Bearer JWT token.

7. docker/Dockerfile.review — Per §16.3: Python 3.12-slim, pip install, copy
   dashboards/review_queue/ + src/, CMD streamlit run ... --server.port=8501.

8. tests/test_review.py — Tests:
   - test_enqueue_creates_pending_item: enqueue → review_queue has status='pending'.
   - test_queue_sorted_by_priority: critical before high before medium before low.
   - test_claim_sets_in_review: claim → status='in_review', assigned_to set.
   - test_claim_conflict_returns_409: second claim on in_review item → 409.
   - test_release_returns_to_pending: release → status='pending', assigned_to=null.
   - test_approve_sends_response_and_resolves: approve → ticket 'human_resolved',
     response sent, feedback action='approved', NOT added to eval dataset.
   - test_edit_stores_diff_and_adds_to_eval: edit → feedback action='edited',
     edit_diff stored, added_to_eval_dataset=TRUE.
   - test_reject_adds_to_eval: reject → feedback action='rejected',
     added_to_eval_dataset=TRUE, ticket 'human_rejected'.
   - test_pii_restored_in_approved_response: approved response has real PII, no
     tokens (per §10.3 APPROVE AS-IS behavior).
```

#### Sequential (after parallel)

_(No sequential dependencies — frontend owns all Phase 6 files. Feedback collector
calls eval_dataset.add_example which is created in Phase 7 by pipeline — for now,
stub the call with a try/except so it doesn't block.)_

#### Review

```
reviewer:
Review Phase 6 against IMPLEMENTATION_PLAN.md §10 (Human Review Queue Design), §9.9
(Feedback Collector), §14.4–14.7 (API specs). Check:
1. Priority mapping: critical=4, high=3, medium=2, low=1 (§10.1).
2. Queue sorted by priority DESC, queued_at ASC (FIFO within same priority).
3. Claim mechanism: pending→in_review (200 OK); in_review→409 Conflict (§10.5).
4. APPROVE AS-IS: PII restored, response sent, ticket→human_resolved, feedback
   action='approved', NOT added to eval dataset (§10.3).
5. EDIT & APPROVE: PII restored in edited version, response sent, ticket→
   human_resolved, feedback action='edited', edit_diff stored, added to eval
   dataset (§10.3).
6. REJECT: feedback required (tags + text), no response sent, ticket→
   human_rejected, feedback action='rejected', added to eval dataset (§10.3).
7. feedback_collector stores original_draft + final_response + feedback_tags +
   feedback_text in feedback table (§9.9).
8. feedback_collector adds to eval dataset only for rejected/edited (not approved)
   per §9.9.
9. Streamlit dashboard matches §10.2 layout: queue list (left) + ticket detail +
   draft editor + feedback + action buttons (right).
10. API endpoints match §14.4–14.7 response shapes exactly.
11. PII tokens restored before sending response (§13.2 invariant: no tokens reach
    customer).
Report findings as numbered list. Do NOT edit files.
```

#### Commit

```bash
git add -A && git commit -m "Phase 6: Human review queue (Streamlit), queue manager with claim mechanism, feedback collector, review API routes"
```

---

### Phase 7: Evaluation Harness & Feedback Loop (Days 12–13)

**Goal:** Eval suite runs on auto-responses; agent feedback feeds back into eval dataset.
(IMPLEMENTATION_PLAN.md §8 Phase 7, §9.8 Evaluation Harness, §12 Feedback Loop Architecture)

#### Parallel Work

```
pipeline:
Read IMPLEMENTATION_PLAN.md §8 Phase 7 (steps 7.1–7.3, 7.6–7.7), §9.8 (Evaluation
Harness pseudocode), §12 (Feedback Loop Architecture — §12.1 loop diagram, §12.2
feedback tags → improvement mapping, §12.3 eval dataset versioning), §14.9 (POST
/eval/run API spec), §6.2 (eval_results table schema).

Create:

1. src/eval/metrics.py — Metric computation functions:
   - faithfulness(draft_text, retrieved_chunks, ticket_body): score 0.0–1.0 — is
     the response grounded in retrieved context? No hallucination. Uses LLM-as-judge.
   - relevance(draft_text, ticket_subject, ticket_body): score 0.0–1.0 — does the
     response address the customer's actual question?
   - citation_accuracy(citations, retrieved_chunks): score 0.0–1.0 — do citations
     point to correct source chunks?
   - Each metric returns a float and token usage for cost tracking.

2. src/eval/judge.py — LLMAsJudge client:
   - Uses EVAL_JUDGE_MODEL from config (default gpt-4o-mini).
   - faithfulness(draft, context, question) → (score, input_tokens, output_tokens).
   - relevance(draft, subject, body) → (score, tokens).
   - citation_accuracy(citations, chunks) → (score, tokens).
   - Tracks last_token_count, last_cost, last_input_tokens, last_output_tokens for
     cost tracking.
   - Retry 2× on error; eval is non-blocking (§4.1: eval_skipped on failure).

3. src/eval/harness.py — evaluate_auto_response(ticket, draft, tenant_id) per §9.8
   pseudocode:
   a. Retrieve chunks used for the draft (draft.retrieved_chunk_ids).
   b. Run 3 judge metrics in parallel (asyncio.gather): faithfulness, relevance,
      citation_accuracy.
   c. overall = (faithfulness + relevance + citation_acc) / 3.
   d. Store in eval_results table (eval_run_id, all scores, eval_model_used,
      eval_tokens, eval_cost_usd).
   e. Cost tracking: cost_tracker.record(stage="eval", ...).
   f. Audit: stage="evaluated".
   g. On exception: audit "eval_skipped" with error, log warning, do NOT re-raise
      (eval is non-blocking per §4.1).

4. src/eval/dataset.py — EvalDataset class per §12.3:
   - __init__(base_path="data/eval_datasets/").
   - add_example(ticket_id, tenant_id, original_draft, corrected_response,
     feedback_tags, source) → new_version:
     Load current version, append example, save as new version (incremented).
   - load_latest() → list[dict]: load most recent version JSONL.
   - _get_latest_version(), _increment_version(), _load(version), _save(version,
     examples).
   - Example format per §12.3: {id, ticket_subject, ticket_body_redacted,
     classification, expected_response, feedback_tags, source, added_at}.
   - Initial seed: 20 synthetic examples in eval_dataset_v001.jsonl.

5. src/api/routes/eval.py — Endpoints per §14.9:
   - POST /eval/run: trigger eval suite (body: tenant_id, dataset_version="latest",
     limit). Returns 202 {eval_run_id, message, dataset_version, example_count}.
   - GET /eval/results?tenant_id=X&days=7: return eval results for dashboard.

6. tests/test_eval.py — Tests per §15.5:
   - test_rejected_draft_added_to_eval_dataset: reject draft → feedback
     added_to_eval_dataset=TRUE → eval_dataset.load_latest() contains example with
     ticket_id.
   - test_approved_draft_not_added_to_eval_dataset: approve →
     added_to_eval_dataset=FALSE.
   - test_eval_metrics_computed: run eval on auto-response → eval_results has
     faithfulness, relevance, citation_accuracy, overall_score.
   - test_eval_non_blocking_on_failure: mock judge failure → ticket still resolved,
     audit has "eval_skipped".
   - test_eval_dataset_versioning: add_example → version increments (v001→v002).
   - test_eval_cost_tracked: eval run → cost_ledger has stage="eval" row.
```

```
frontend:
Read IMPLEMENTATION_PLAN.md §8 Phase 7 (steps 7.4–7.5), §12.1 (Feedback Loop —
feedback collector → eval dataset), §9.9 (Feedback Collector — the add to eval
dataset step).

Update src/review/feedback_collector.py to connect the feedback loop:
- Replace the stubbed eval_dataset.add_example call with the real implementation
  from src/eval/dataset.py (now created by pipeline in parallel).
- Ensure: action in ("rejected", "edited") → eval_dataset.add_example(...) called,
  feedback.added_to_eval_dataset=TRUE, feedback.eval_dataset_version set.
- action == "approved" → NOT added to eval dataset.
- This closes the feedback loop: agent edits/rejections → eval dataset → eval
  harness → prompt improvement (§12.1).

Coordinate with pipeline via hub message before editing feedback_collector.py —
pipeline owns src/eval/dataset.py, frontend owns src/review/feedback_collector.py.
The interface contract: eval_dataset.add_example(ticket_id, tenant_id,
original_draft, corrected_response, feedback_tags, source) returns version string.
```

#### Sequential (after parallel)

```
pipeline:
After frontend connects feedback_collector to eval_dataset, verify the feedback
loop end-to-end:
1. Submit a ticket that routes to human review.
2. Reject the draft with feedback_tags=["incorrect_info"].
3. Verify feedback table has added_to_eval_dataset=TRUE.
4. Verify eval_dataset.load_latest() contains the new example.
5. Run eval suite → verify the new example is included.
```

#### Review

```
reviewer:
Review Phase 7 against IMPLEMENTATION_PLAN.md §9.8, §12 (Feedback Loop), §14.9. Check:
1. Eval harness runs 3 metrics in parallel (asyncio.gather) per §9.8.
2. overall_score = (faithfulness + relevance + citation_accuracy) / 3.
3. Eval is non-blocking: on failure, audit "eval_skipped", ticket still resolved
   (§4.1).
4. Eval results stored in eval_results table with all fields per §6.2.
5. Eval cost tracked via cost_tracker.record(stage="eval").
6. EvalDataset versioning: each add_example creates a new version (v001→v002→...)
   per §12.3.
7. Example format matches §12.3: {id, ticket_subject, ticket_body_redacted,
   classification, expected_response, feedback_tags, source, added_at}.
8. Feedback loop closed: rejected/edited drafts → eval dataset → eval harness
   includes them in next run (§12.1).
9. Approved drafts NOT added to eval dataset (no correction signal).
10. POST /eval/run returns 202 with eval_run_id, dataset_version, example_count
    per §14.9.
11. test_eval.py covers: rejected→eval dataset, approved→not in dataset, metrics
    computed, non-blocking on failure, versioning, cost tracked.
Report findings as numbered list. Do NOT edit files.
```

#### Commit

```bash
git add -A && git commit -m "Phase 7: Evaluation harness (faithfulness, relevance, citation accuracy), LLM-as-judge, versioned eval dataset, feedback loop closed"
```

---

### Phase 8: Monitoring Dashboard & Pipeline Integration (Day 14)

**Goal:** Monitoring dashboard shows all operational metrics; full pipeline runs end-to-end.
(IMPLEMENTATION_PLAN.md §8 Phase 8, §11 Monitoring Dashboard Layout, §9.6 Pipeline Orchestrator, §11.2 Metrics Endpoint)

#### Parallel Work

```
frontend:
Read IMPLEMENTATION_PLAN.md §8 Phase 8 (steps 8.1–8.3), §11 (Monitoring Dashboard
Layout — §11.1 all sections: KPI cards, ticket volume, auto-resolution rate, eval
scores, cost breakdown, queue depth, classification distribution, recent
auto-responses, feedback summary), §11.2 (Metrics Endpoint pseudocode), §14.8
(GET /metrics API spec).

Create:

1. src/api/routes/metrics.py — GET /metrics?tenant_id=X&days=7 per §11.2 pseudocode:
   Return single JSON with:
   - kpi_cards: ticket_volume, auto_resolution_rate, avg_response_time_min,
     avg_cost_per_ticket, queue_depth.
   - time_series: daily_ticket_counts, daily_auto_resolution, hourly_queue_depth,
     daily_eval_scores.
   - distributions: urgency, category, sentiment distributions.
   - cost: total_usd, by_stage, budget_remaining.
   - feedback: total, by_action, top_tags, eval_dataset_size.
   - recent_auto_responses: last 20 auto-resolved tickets with category, confidence,
     eval score, cost, status.
   All queries filtered by tenant_id (RLS + WHERE clause).

2. dashboards/monitoring/app.py — Streamlit app per §11.1 Dashboard Layout:
   - Header: "Customer Support AI — Monitoring Dashboard" + tenant selector +
     date range selector + refresh button.
   - KPI CARDS (top row): Ticket Volume, Auto-Resolution Rate, Avg Response Time,
     Cost per Ticket, Queue Depth. Each with trend indicator (↑/↓ vs previous
     period).
   - TICKET VOLUME (time series, stacked bar): auto-resolved, human-resolved,
     pending per day.
   - AUTO-RESOLUTION RATE (trend line): with target line at 65% (dashed).
   - EVAL SCORES (trend): faithfulness, relevance, citation_accuracy over time.
   - COST BREAKDOWN (pie/bar): by stage (classification, draft, embedding, eval,
     rerank). Total + budget + remaining.
   - REVIEW QUEUE DEPTH (time series): backlog trend.
   - CLASSIFICATION DISTRIBUTION: by urgency, category, sentiment (bar charts).
   - RECENT AUTO-RESPONSES (table): ticket, category, confidence, eval score,
     cost, status.
   - FEEDBACK SUMMARY: total, by action (approved/edited/rejected), top feedback
     tags, eval dataset size.
   - Use @st.cache_data for metrics fetching (per §18 risk mitigation for
     Streamlit performance).

3. dashboards/monitoring/charts.py — Chart definitions using Plotly:
   - ticket_volume_chart(time_series): stacked bar chart.
   - auto_resolution_rate_chart(time_series): line chart with target line.
   - eval_scores_chart(time_series): multi-line chart.
   - cost_breakdown_chart(cost_data): pie or bar chart.
   - queue_depth_chart(time_series): area chart.
   - classification_distribution_charts(distributions): horizontal bar charts.

4. dashboards/monitoring/api_client.py — HTTP client:
   - get_metrics(tenant_id, days): calls GET /metrics.
   - Auth: Bearer JWT token.

5. docker/Dockerfile.monitor — Per §16.4: Python 3.12-slim, pip install, copy
   dashboards/monitoring/ + src/, CMD streamlit run ... --server.port=8502.
```

```
pipeline:
Read IMPLEMENTATION_PLAN.md §8 Phase 8 (steps 8.4–8.6), §9.6 (Pipeline Orchestrator
pseudocode — full stages 1–9), §4.1 (Failure Modes & Fallbacks for every stage),
§4 (Data Flow: Single Ticket Lifecycle).

Create:

1. src/pipeline/orchestrator.py — process_ticket(ticket_id, tenant_id) per §9.6
   pseudocode (the complete 9-stage pipeline):
   STAGE 1: Get ticket from DB. Audit "ticket_received".
   STAGE 2: PII redaction (pii_redactor.redact). On failure:
     failure_handlers.handle_pii_failure → route to human, return (no LLM calls).
     Update ticket.body_redacted.
   STAGE 3: Classification (classifier.classify_ticket with body_redacted). On
     failure: failure_handlers.handle_classification_failure, classification=None.
   STAGE 4: RAG retrieval (rag_service.retrieve). On failure: retrieved=[],
     failure_handlers.handle_rag_failure.
   BUDGET CHECK: budget_checker.check(tenant_id). If not OK: audit
     "budget_exceeded", enqueue_for_review(draft=None, reason="budget_exceeded"),
     return.
   STAGE 5: Draft response (draft_service.generate). On failure: draft=None,
     failure_handlers.handle_draft_failure, enqueue_for_review(draft=None,
     reason="draft_failed"), return.
   STAGE 6: Confidence + routing (confidence_scorer.compute_confidence +
     should_auto_respond). Audit "confidence_scored" with score, route, reason.
   STAGE 7: Route:
     - auto: pii_restorer.restore(draft.draft_text_clean), send_response,
       update_ticket(status="auto_resolved", resolved_at=now), audit
       "auto_responded". Then asyncio.create_task(eval_harness.evaluate_auto_
       response(...)) — async, non-blocking.
     - human: enqueue_for_review(ticket, classification, draft, reason),
       audit "enqueued_for_review".
   STAGE 8: Cost aggregation (cost_tracker.aggregate_ticket). Audit
     "cost_recorded" with total_cost_usd.
   Exception handler: update_ticket(status="failed"), audit "pipeline_error"
     with error, log exception.

2. src/pipeline/failure_handlers.py — Fallback logic per §4.1 (Failure Modes &
   Fallbacks table):
   - handle_pii_failure: route to human review, flag "PII scan failed", audit
     "pii_scan_failed". No LLM calls.
   - handle_classification_failure: retry 2× with backoff; if still failing,
     route to human with classification=null, audit "classification_failed".
   - handle_rag_failure: if empty KB → confidence=0, route to human, draft=
     "We're reviewing your ticket and will respond shortly", audit "rag_empty".
     If vector timeout → fall back to BM25-only, audit "rag_degraded_bm25_only".
   - handle_draft_failure: retry 2×; if still failing, route to human with
     draft=null, audit "draft_failed".
   - handle_auto_send_failure: retry 3×; if still failing, enqueue for human
     review, audit "auto_send_failed".

3. tests/test_pipeline.py — End-to-end tests per §15.4:
   - test_full_pipeline_auto_resolve: seed KB with password reset FAQ → submit
     ticket "Can't reset my password" → process → verify status="auto_resolved",
     resolved_at set, classification.urgency in (high, medium), category=
     "technical", draft has citations, route_decision="auto", confidence >= 0.75,
     cost > 0, audit trail has ALL stages: ticket_received, pii_scanned,
     classified, rag_retrieved, response_drafted, confidence_scored,
     auto_responded, cost_recorded.
   - test_full_pipeline_human_review: submit complex billing dispute → process →
     status="pending_review", review_queue item exists with status="pending".
   - test_critical_urgency_always_routes_to_human: "PRODUCTION SYSTEM DOWN" →
     status="pending_review" (never auto_resolved).
   - test_budget_exceeded_routes_to_human: set budget to $0.001 → submit →
     status="pending_review", cost_ledger stages do NOT include "draft".
   - test_pii_scan_failure_routes_to_human: mock Presidio failure →
     status="pending_review", mock_llm.call_count == 0.
   - test_pipeline_failure_marks_failed: mock unhandled exception →
     status="failed", audit has "pipeline_error".
```

#### Sequential (after parallel)

```
pipeline:
After frontend completes metrics endpoint, verify full end-to-end flow:
1. docker compose up -d
2. docker compose exec api python scripts/init_db.py
3. docker compose exec api python scripts/seed_test_data.py
4. docker compose exec api python scripts/seed_kb.py
5. Submit 10 tickets with varying complexity via POST /tickets.
6. Some auto-resolve, some go to review queue.
7. Approve/reject in review queue dashboard (port 8501).
8. Run eval suite via POST /eval/run.
9. Open monitoring dashboard (port 8502) → verify all metrics display:
   ticket_volume=10, auto_resolution_rate, avg_response_time, cost/ticket,
   eval scores, queue depth.
10. Run: pytest tests/test_pipeline.py -v → all pass.
```

#### Review

```
reviewer:
Review Phase 8 against IMPLEMENTATION_PLAN.md §9.6 (Pipeline Orchestrator), §11
(Monitoring Dashboard), §4.1 (Failure Modes), §15.4 (Pipeline Tests). Check:
1. Orchestrator implements all 9 stages in order per §9.6.
2. Budget check happens BEFORE draft LLM call (between stage 4 and 5).
3. Auto-respond path: PII restored → response sent → ticket auto_resolved →
   eval triggered async (non-blocking).
4. Human path: enqueue_for_review with reason → audit "enqueued_for_review".
5. Cost aggregation after routing (stage 8).
6. Exception handler: ticket→failed, audit "pipeline_error".
7. failure_handlers cover all entries in §4.1 table: pii_scan_failed,
   classification_failed, rag_empty, rag_degraded_bm25_only, draft_failed,
   auto_send_failed, budget_exceeded.
8. Metrics endpoint returns all sections from §11.2: kpi_cards, time_series,
   distributions, cost, feedback, recent_auto_responses.
9. Monitoring dashboard renders all sections from §11.1: KPI cards, ticket
   volume, auto-resolution rate, eval scores, cost breakdown, queue depth,
   classification distribution, recent auto-responses, feedback summary.
10. test_pipeline.py covers: auto-resolve (full audit trail), human review,
    critical urgency override, budget exceeded (no draft call), PII failure
    (no LLM call), pipeline failure (status=failed).
11. Full audit trail for auto-resolved ticket has ALL 8 stages from §15.4:
    ticket_received, pii_scanned, classified, rag_retrieved, response_drafted,
    confidence_scored, auto_responded, cost_recorded.
Report findings as numbered list. Do NOT edit files.
```

#### Commit

```bash
git add -A && git commit -m "Phase 8: Pipeline orchestrator (9 stages with failure handling), monitoring dashboard (Streamlit), metrics endpoint, end-to-end pipeline tests"
```

---

### Phase 9: Hardening, Testing & Documentation (Day 15)

**Goal:** Production-ready, fully tested, documented.
(IMPLEMENTATION_PLAN.md §8 Phase 9, §15 Testing Strategy, §16.7 CI, §16.8 Production)

#### Parallel Work

```
reviewer:
Read IMPLEMENTATION_PLAN.md §8 Phase 9 (steps 9.1–9.8), §15 (Testing Strategy), §16.7
(ci.yml), §16.8 (Production Considerations), §13 (Security & Safety).

Create:

1. .github/workflows/ci.yml — Per §16.7:
   - on: [push, pull_request]
   - Services: postgres (pgvector/pgvector:pg16), redis (redis:7-alpine) with
     healthchecks.
   - Steps: checkout, setup-python 3.12, pip install -e ., spacy download
     en_core_web_lg, ruff check src/ tests/, pytest tests/ -v --tb=short.
   - Env: DATABASE_URL, REDIS_URL, OPENAI_API_KEY, PII_ENCRYPTION_KEY, JWT_SECRET.

2. Run full test suite end-to-end:
   - pytest tests/ -v --tb=short
   - Verify ALL tests green: test_pii, test_classify, test_rag, test_draft,
     test_confidence, test_review, test_eval, test_cost, test_audit,
     test_pipeline, test_security, test_api.
   - Report any failures with full traceback.
```

```
ingest:
Read IMPLEMENTATION_PLAN.md §8 Phase 9 (steps 9.4, 9.5, 9.7), §5 (structlog), §16.8
(Rate limiting, DB user privileges).

Create/update:

1. Add structured logging (structlog) across all components:
   - Configure structlog with JSON output, correlation IDs (ticket_id, tenant_id).
   - Update src/main.py to configure structlog at startup.
   - Add logging to key paths: ticket_received, pii_scanned, classified,
     rag_retrieved, response_drafted, confidence_scored, auto_responded,
     pipeline_error.
   - Log level from config.LOG_LEVEL.

2. Add rate limiting on API endpoints:
   - Use slowapi or custom middleware.
   - Limits: 100 tickets/min per tenant, 30 review actions/min per agent.
   - Return 429 Too Many Requests when exceeded.

3. Verify .env.example has all env vars from §16.6 (should already exist from
   Phase 1 — verify completeness).
```

```
pipeline:
Read IMPLEMENTATION_PLAN.md §8 Phase 9 (step 9.8), §17.2 (Milestone M9 verification).

Perform performance check:
1. Process 50 tickets with varying complexity.
2. Measure throughput (tickets/min) and total cost.
3. Record baseline metrics: avg processing time per ticket, avg cost per ticket,
   auto-resolution rate, eval scores.
4. Verify no memory leaks or connection pool exhaustion after 50 tickets.
5. Report baseline metrics for documentation.
```

```
frontend:
Read IMPLEMENTATION_PLAN.md §8 Phase 9 (step 9.3), §13 (Security), §15.2–15.3
(PII + Security tests).

Perform security review:
1. Re-run tests/test_security.py — verify all cross-tenant isolation tests pass.
2. Re-run tests/test_pii.py — verify all PII safety tests pass.
3. Manual PII leakage check: submit ticket with diverse PII (names, emails, phones,
   IBANs, addresses, SSNs) → verify body_redacted has zero real PII.
4. Verify no PII tokens in any sent response (auto or human-approved).
5. Verify RLS policies active on all 10 tables.
6. Report any security findings.
```

#### Sequential (after parallel)

```
ingest:
After all parallel work completes, write README.md per §8 Phase 9 step 9.6:
- Project overview (from §1 Goals).
- Architecture diagram (from §3.1).
- Quick start guide (from Appendix A).
- API documentation summary (from §14).
- Database schema overview (from §6).
- Testing instructions (from §15).
- Deployment instructions (from §16).
- Monitoring dashboard guide (from §11).
```

#### Review

```
reviewer:
Final production readiness review against IMPLEMENTATION_PLAN.md §8 Phase 9 verification
gate. Check:
1. CI pipeline (.github/workflows/ci.yml) runs ruff + pytest on push per §16.7.
2. Full test suite green: all 12 test files pass.
3. Security: no cross-tenant leakage, no PII leakage to LLM, no tokens in responses.
4. Structured logging (structlog) with JSON output and correlation IDs.
5. Rate limiting on API endpoints (100 tickets/min, 30 review actions/min).
6. README.md covers: setup, architecture, usage, metrics (per §8 step 9.6).
7. .env.example documents all env vars.
8. Performance baseline recorded (50 tickets, throughput, cost).
9. Fresh docker compose up → seed → submit → process → dashboard → all works
   (M9 verification from §17.2).
10. All 12 goals from §1 have corresponding implementation + tests:
    G1 end-to-end, G2 PII redaction, G3 classification, G4 hybrid RAG,
    G5 confidence-gated, G6 review queue, G7 eval harness, G8 cost tracking,
    G9 audit trail, G10 monitoring, G11 feedback loop, G12 production-deployable.
Report final sign-off or list of remaining issues. Do NOT edit files.
```

#### Commit

```bash
git add -A && git commit -m "Phase 9: Hardening (structlog, rate limiting), CI pipeline, full test suite, security review, README documentation, performance baseline"
```

---

## 5. Coordination Rules

### File Ownership Boundaries

| Directory / File              | Owner      | Others may read? | Others may edit? |
|-------------------------------|------------|------------------|------------------|
| `src/ingest/`                 | `ingest`   | yes              | no               |
| `src/pii/`                    | `ingest`   | yes              | no               |
| `src/classify/`               | `ingest`   | yes              | no (see §5.1)    |
| `src/audit/`                  | `ingest`   | yes              | no               |
| `src/rag/`                    | `rag`      | yes              | no               |
| `src/draft/`                  | `pipeline` | yes              | no               |
| `src/confidence/`             | `pipeline` | yes              | no               |
| `src/cost/`                   | `pipeline` | yes              | no               |
| `src/pipeline/`               | `pipeline` | yes              | no               |
| `src/eval/`                   | `pipeline` | yes              | no               |
| `src/review/`                 | `frontend` | yes              | no               |
| `src/cache/`                  | `pipeline` | yes              | no               |
| `src/api/routes/tickets.py`   | `ingest`   | yes              | no               |
| `src/api/routes/kb.py`        | `rag`      | yes              | no               |
| `src/api/routes/review.py`    | `frontend` | yes              | no               |
| `src/api/routes/metrics.py`   | `frontend` | yes              | no               |
| `src/api/routes/eval.py`      | `pipeline` | yes              | no               |
| `src/api/routes/health.py`    | `ingest`   | yes              | no               |
| `src/api/dependencies.py`     | `ingest`   | yes              | no               |
| `src/db/`                     | `ingest`   | yes              | no               |
| `src/config.py`               | `ingest`   | yes              | no               |
| `src/main.py`                 | `ingest`   | yes              | no               |
| `dashboards/review_queue/`    | `frontend` | yes              | no               |
| `dashboards/monitoring/`      | `frontend` | yes              | no               |
| `docker/Dockerfile.review`    | `frontend` | yes              | no               |
| `docker/Dockerfile.monitor`   | `frontend` | yes              | no               |
| `docker/Dockerfile.api`       | `ingest`   | yes              | no               |
| `tests/conftest.py`           | `pipeline` | yes              | no               |
| `tests/test_pii.py`           | `ingest`   | yes              | no               |
| `tests/test_classify.py`      | `ingest`   | yes              | no               |
| `tests/test_audit.py`         | `ingest`   | yes              | no               |
| `tests/test_rag.py`           | `rag`      | yes              | no               |
| `tests/test_security.py`      | `rag`      | yes              | no               |
| `tests/test_draft.py`         | `pipeline` | yes              | no               |
| `tests/test_confidence.py`    | `pipeline` | yes              | no               |
| `tests/test_cost.py`          | `pipeline` | yes              | no               |
| `tests/test_pipeline.py`      | `pipeline` | yes              | no               |
| `tests/test_eval.py`          | `pipeline` | yes              | no               |
| `tests/test_review.py`        | `frontend` | yes              | no               |
| `tests/test_api.py`           | `frontend` | yes              | no               |
| `.github/workflows/ci.yml`    | `reviewer` | yes              | no               |

### 5.1 Cross-Agent Edit Coordination

Two cases require an agent to edit a file owned by another agent:

1. **Phase 5 — pipeline wires cost_tracker into classifier.py (owned by ingest):**
   `pipeline` must send a hub message to `ingest` before editing
   `src/classify/classifier.py`. The edit adds `await cost_tracker.record(...)` after
   the classification LLM call (per §9.3 step 6). `ingest` should acknowledge before
   the edit proceeds.

2. **Phase 7 — frontend connects feedback_collector to eval_dataset (owned by pipeline):**
   `frontend` must send a hub message to `pipeline` before editing
   `src/review/feedback_collector.py` to call `eval_dataset.add_example(...)`. The
   interface contract: `add_example(ticket_id, tenant_id, original_draft,
   corrected_response, feedback_tags, source) → str (version)`. `pipeline` should
   confirm the interface before the edit proceeds.

### Conflict Avoidance Rules

- **Never edit a file you don't own** without messaging the owner agent first via hub.
- **Shared imports**: `src/audit/logger.py` (owned by `ingest`) is imported by all
  agents. If the interface changes, `ingest` must broadcast the change to all agents.
- **Shared config**: `src/config.py` (owned by `ingest`) — if a new env var is needed,
  message `ingest` to add it.
- **Database schema**: `src/db/schema.sql` (owned by `ingest`) — if a new table or
  column is needed, message `ingest`. Schema changes require migration files.
- **conftest.py**: `pipeline` owns test fixtures. If an agent needs a new fixture,
  message `pipeline`.

### Parallel vs Sequential

- **Parallel**: agents work on non-overlapping file sets with no data dependency.
  Phases 4, 7, 8, 9 use parallel work.
- **Sequential**: a phase depends on the output of a prior phase or another agent
  within the same phase. Phase 1 (foundation before everything), Phase 5 (cost wiring
  after parallel work), Phase 7 (feedback loop connection after parallel work), and
  all review checkpoints are sequential.
- **Review checkpoints are always sequential**: no phase proceeds until the reviewer
  signs off on the prior phase.

### Handling Blocked Agents

1. If an agent is blocked (e.g., needs a file from another agent that isn't ready),
   it should send a hub message to the owning agent requesting the prerequisite.
2. If the owning agent is also blocked, escalate to `reviewer` for coordination.
3. An agent should never wait idle — if blocked on one task, it can proceed with other
   independent tasks in its ownership scope.
4. If an agent encounters a design decision not covered by the plan, it should make
   the most conservative choice consistent with the plan's architecture and note it
   for the reviewer.
5. Use `herdr agent wait <name> --until blocked --timeout 120000` to detect when an
   agent needs input, then use `herdr agent prompt` to provide guidance.

---

## 6. Quick Reference

### Herdr Commands

```bash
# ── Discovery ──
herdr agent list                          # List all live agents + states
herdr pane list --workspace "$HERDR_WORKSPACE_ID"  # List all panes
herdr pane current --current              # Show current pane

# ── Prompting agents ──
herdr agent prompt ingest "..." --wait --timeout 120000
herdr agent prompt rag "..." --wait --timeout 120000
herdr agent prompt pipeline "..." --wait --timeout 120000
herdr agent prompt frontend "..." --wait --timeout 120000
herdr agent prompt reviewer "..." --wait --timeout 120000

# ── Reading agent output ──
herdr agent get ingest                    # Agent state + metadata
herdr agent read ingest --source recent-unwrapped --lines 120

# ── Waiting for states ──
herdr agent wait ingest --until blocked --timeout 120000
herdr agent wait ingest --until idle --timeout 120000

# ── Sending keys (interactive UI) ──
herdr agent send-key ingest esc
herdr agent send-key ingest ctrl+c

# ── Pane operations ──
herdr pane split --current --direction right --cwd "$PWD" --no-focus
herdr pane run <pane-id> "pytest tests/ -v"
herdr pane wait-output <pane-id> --match "test result" --timeout 120000
herdr pane read <pane-id> --source recent-unwrapped --lines 120
```

### Verification Commands (per phase)

```bash
# Phase 1: Infrastructure
docker compose up -d
curl http://localhost:8000/health  # → 200 OK
pytest tests/test_audit.py -k immutability

# Phase 2: PII
curl -X POST http://localhost:8000/tickets \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"subject":"Test","body":"Hi, I am Jane Doe, jane@example.com, IBAN DE89..."}'
# Check body_redacted has [PERSON_1], [EMAIL_1], [IBAN_1]
pytest tests/test_pii.py tests/test_audit.py -v

# Phase 3: RAG
pytest tests/test_rag.py tests/test_security.py -v

# Phase 4: Classification + Drafting
pytest tests/test_classify.py tests/test_draft.py -v

# Phase 5: Confidence + Cost
pytest tests/test_confidence.py tests/test_cost.py -v

# Phase 6: Review Queue
open http://localhost:8501  # Review queue dashboard
pytest tests/test_review.py -v

# Phase 7: Eval + Feedback
pytest tests/test_eval.py -v
curl -X POST http://localhost:8000/eval/run \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"tenant_id":"acme","dataset_version":"latest"}'

# Phase 8: Monitoring + Pipeline
open http://localhost:8502  # Monitoring dashboard
pytest tests/test_pipeline.py -v

# Phase 9: Full suite
pytest tests/ -v --tb=short
ruff check src/ tests/
```

### Git Workflow

```bash
# After each phase (only after reviewer sign-off):
git add -A && git commit -m "Phase N: <description>"

# Phase commit messages (copy-pasteable):
git add -A && git commit -m "Phase 1: Foundation & Infrastructure — Docker Compose, DB schema with RLS, FastAPI skeleton, Redis cache, test fixtures"
git add -A && git commit -m "Phase 2: Ticket ingestion, PII redaction (Presidio), PII restoration, audit logging, ticket API routes"
git add -A && git commit -m "Phase 3: KB indexing, hybrid BM25+vector retrieval, cross-encoder re-ranking, tenant+role filtering, KB API routes, security tests"
git add -A && git commit -m "Phase 4: Ticket classification (structured LLM output + cache) and response drafting (citations, tone adaptation, LLM fallback)"
git add -A && git commit -m "Phase 5: Composite confidence scoring, routing with hard rules, per-call cost tracking, per-tenant budget enforcement"
git add -A && git commit -m "Phase 6: Human review queue (Streamlit), queue manager with claim mechanism, feedback collector, review API routes"
git add -A && git commit -m "Phase 7: Evaluation harness (faithfulness, relevance, citation accuracy), LLM-as-judge, versioned eval dataset, feedback loop closed"
git add -A && git commit -m "Phase 8: Pipeline orchestrator (9 stages with failure handling), monitoring dashboard (Streamlit), metrics endpoint, end-to-end pipeline tests"
git add -A && git commit -m "Phase 9: Hardening (structlog, rate limiting), CI pipeline, full test suite, security review, README documentation, performance baseline"
```

### Key Plan References

| Topic                        | IMPLEMENTATION_PLAN.md Section |
|------------------------------|--------------------------------|
| Goals & Non-Goals            | §1                             |
| Component-to-Project Mapping | §2                             |
| Architecture Diagram         | §3.1                           |
| Ticket Lifecycle (9 stages)  | §4                             |
| Failure Modes & Fallbacks    | §4.1                           |
| Tech Stack                   | §5                             |
| Database Schema (all tables) | §6.2                           |
| Row-Level Security           | §6.3                           |
| Project Structure            | §7                             |
| Implementation Phases (1–9)  | §8                             |
| Component Pseudocode         | §9.1–9.9                       |
| Review Queue Design          | §10                            |
| Monitoring Dashboard Layout  | §11                            |
| Feedback Loop Architecture   | §12                            |
| Security & Safety            | §13                            |
| API Specification            | §14                            |
| Testing Strategy             | §15                            |
| Deployment (Docker)          | §16                            |
| Roadmap & Milestones         | §17                            |
| Risk Register                | §18                            |
| Quick Start                  | Appendix A                     |
| Key Design Decisions         | Appendix B                     |
