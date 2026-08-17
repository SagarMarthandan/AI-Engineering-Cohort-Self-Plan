# LLM Serving Gateway with Caching + Cost Control

## Implementation Plan

> **Tier 4 — AI Infrastructure & Production · Project 9**
> A production-grade proxy layer in front of LLM APIs that provides semantic caching, model routing, per-request cost tracking, per-tenant token budgets, rate limiting, structured logging, an admin dashboard, and multi-provider fallback.
>
> **Audience:** Data engineer / analytics engineer transitioning to AI engineering. Assumes working knowledge of Python, SQL, dbt, Airflow, Docker, and data quality tooling. Introduces LLM-specific infrastructure patterns: embedding-based caching, confidence-based routing, token economics, and provider failover.

---

## Table of Contents

1. [Project Goals & Non-Goals](#1-project-goals--non-goals)
2. [The Business Case — Cost Math](#2-the-business-case--cost-math)
3. [Architecture Overview](#3-architecture-overview)
4. [Tech Stack with Rationale](#4-tech-stack-with-rationale)
5. [Database & Schema Design](#5-database--schema-design)
6. [Project Structure](#6-project-structure)
7. [Component Specifications](#7-component-specifications)
8. [API Specification](#8-api-specification)
9. [Implementation Phases (10 Working Days)](#9-implementation-phases-10-working-days)
10. [Testing Strategy](#10-testing-strategy)
11. [Security & Safety Considerations](#11-security--safety-considerations)
12. [Deployment](#12-deployment)
13. [Roadmap & Milestones](#13-roadmap--milestones)
14. [Risk Register](#14-risk-register)
15. [Appendix](#15-appendix)

---

## 1. Project Goals & Non-Goals

### Goals

| # | Goal | Measurable Success Criterion |
|---|------|------------------------------|
| G1 | **Semantic cache** that returns cached responses for similar prompts (cosine similarity ≥ threshold) | ≥ 50% cache hit rate on repeated workloads; sub-10 ms cache lookup latency |
| G2 | **Model routing** that tries a cheap model first and escalates to an expensive model on low confidence | ≥ 70% of requests served by the cheap model; no measurable quality regression on golden Q&A set |
| G3 | **Per-request cost tracking** with aggregation by tenant, endpoint, and day | Every request has a cost row in PostgreSQL; daily cost report matches provider invoices within ±2% |
| G4 | **Per-tenant monthly token budgets** with rejection at limit and warning at 80% | Budget-exceeded requests return HTTP 429; 80% warning emitted to admin dashboard and webhook |
| G5 | **Rate limiting** via sliding-window in Redis (per-minute, per-hour, per-day) | Over-limit requests return HTTP 429 with `Retry-After` header; no burst-through under concurrent load |
| G6 | **Structured request/response logging** | Every request emits a JSON log line with tenant, model, tokens, latency, cache hit/miss, cost |
| G7 | **Admin dashboard** (Streamlit) | Shows cache hit rate, cost per tenant, model distribution, budget usage, rate-limit stats — live-updating |
| G8 | **Multi-provider fallback** (OpenAI → Anthropic → Ollama) | Automatic failover on provider 5xx/timeout; zero downtime during a simulated primary outage |

### Non-Goals

- **Not** a fine-tuning or training platform. The gateway is inference-only.
- **Not** a prompt management / versioning system (use LangSmith or Promptflow for that).
- **Not** a vector database for RAG. The semantic cache uses Redis only; it does not serve as a knowledge base.
- **Not** a multi-tenant SaaS control plane. Tenants are identified by API key; there is no self-service signup, billing, or UI for tenant management.
- **Not** streaming-response optimization. v1 buffers full responses (needed for caching and cost tracking). Streaming is a v2 concern.
- **Not** model hosting. Ollama is used only as a last-resort fallback; the gateway does not manage GPU infrastructure.
- **Not** a replacement for provider-native features (OpenAI's own caching, Anthropic's prompt caching). The gateway complements them.

---

## 2. The Business Case — Cost Math

This project exists because **companies burn money on LLM APIs without caching or routing**. The math is simple and brutal.

### Scenario: Customer Support Q&A Bot

| Parameter | Value |
|-----------|-------|
| Queries per day | 10,000 |
| Avg input tokens | 500 |
| Avg output tokens | 200 |
| Model (naive) | GPT-4o (`$5.00/M input`, `$15.00/M output`) |
| Cost per query (naive) | `500×$5/M + 200×$15/M = $0.0025 + $0.003 = $0.0055` |
| **Daily cost (naive)** | `10,000 × $0.0055 = $55.00/day` |
| **Monthly cost (naive)** | `$55 × 30 = $1,650/month` |

### With the Gateway (60% cache hit + routing)

| Optimization | Effect |
|--------------|--------|
| **Semantic cache (60% hit rate)** | 6,000 queries served from cache → $0. Only 4,000 call the LLM. |
| **Model routing (70% to GPT-4o-mini)** | Of the 4,000 uncached: 2,800 go to GPT-4o-mini (`$0.15/M in`, `$0.60/M out`). 1,200 escalate to GPT-4o. |
| **Cost — mini tier** | `2,800 × (500×$0.15/M + 200×$0.60/M) = 2,800 × $0.000195 = $0.546/day` |
| **Cost — full tier** | `1,200 × $0.0055 = $6.60/day` |
| **Daily cost (gateway)** | `$0.546 + $6.60 = $7.15/day` |
| **Monthly cost (gateway)** | `$7.15 × 30 = $214.46/month` |
| **Savings** | `$1,650 − $214 = $1,436/month` (**87% reduction**) |

### Embedding Cost (cache overhead)

| Parameter | Value |
|-----------|-------|
| Embedding model | `text-embedding-3-small` (`$0.02/M tokens`) |
| Embedding calls/day | 4,000 (only cache misses need embedding lookup) |
| Embedding tokens/day | `4,000 × 500 = 2M tokens` |
| **Embedding cost/day** | `2M × $0.02/M = $0.04/day` |

**Net savings: $1,436/month. The gateway pays for itself in hour one.**

> The dashboard makes this visible. Without it, the cost saving is invisible to finance — which is why teams get cut when budgets tighten.

---

## 3. Architecture Overview

```
                              ┌─────────────────────────────────────────────────────────┐
                              │                    ADMIN DASHBOARD                       │
                              │                     (Streamlit :8501)                    │
                              │  Cache hit rate · Cost/tenant · Model dist · Budgets    │
                              └──────────────────────┬──────────────────────────────────┘
                                                     │ SQL queries (read-only)
                              ┌──────────────────────▼──────────────────────────────────┐
                              │                   POSTGRESQL                             │
                              │  cost_records · tenants · budget_usage · model_pricing  │
                              └──────────────────────▲──────────────────────────────────┘
                                                     │ cost writes
┌──────────┐    HTTPS    ┌───────────────────────────┴──────────────────────────────────┐
│  Client  │────────────▶│                    FASTAPI GATEWAY (:8000)                   │
│ (SDK /   │             │                                                            │
│  curl)   │             │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│          │             │  │  Auth &  │─▶│  Rate    │─▶│  Budget  │─▶│ Semantic │     │
│          │◀────────────│  │  Tenant  │  │ Limiter  │  │  Check   │  │  Cache   │     │
│          │  response   │  │  Resolve │  │ (Redis)  │  │ (PG+Redis)│  │ (Redis)  │     │
└──────────┘             │  └──────────┘  └──────────┘  └──────────┘  └────┬─────┘     │
                         │                                                  │           │
                         │                                          HIT ┌───┴───┐       │
                         │                                              │  YES  │       │
                         │                                              │return │       │
                         │                                              │cached │       │
                         │                                              │response│      │
                         │                                              └───┬───┘       │
                         │                                          MISS │   NO        │
                         │                                              ▼             │
                         │  ┌──────────────────────────────────────────────────────┐   │
                         │  │              MODEL ROUTING ENGINE                     │   │
                         │  │                                                       │   │
                         │  │   task_type? ──▶ simple ──▶ GPT-4o-mini               │   │
                         │  │                complex ─▶ GPT-4o                      │   │
                         │  │                                                       │   │
                         │  │   confidence_check:                                   │   │
                         │  │     if mini_response.confidence < threshold:          │   │
                         │  │         retry with GPT-4o                             │   │
                         │  └──────────────────────┬───────────────────────────────┘   │
                         │                         │                                  │
                         │  ┌──────────────────────▼───────────────────────────────┐  │
                         │  │              PROVIDER FAILOVER CHAIN                  │  │
                         │  │                                                       │  │
                         │  │   OpenAI ──(5xx/timeout)──▶ Anthropic ──(fail)──▶    │  │
                         │  │                                                       │  │
                         │  │   ──▶ Ollama (local, degraded)                        │  │
                         │  └──────────────────────┬───────────────────────────────┘  │
                         │                         │                                  │
                         │  ┌──────────────────────▼───────────────────────────────┐  │
                         │  │         COST TRACKER + STRUCTURED LOGGER              │  │
                         │  │  tokens(model, in, out) → cost → PG cost_records      │  │
                         │  │  JSON log: tenant, model, tokens, latency, cache, $   │  │
                         │  │  Redis: budget_used += tokens; rate window += 1       │  │
                         │  └──────────────────────┬───────────────────────────────┘  │
                         │                         │                                  │
                         │  ┌──────────────────────▼───────────────────────────────┐  │
                         │  │         CACHE WRITE (on miss only)                    │  │
                         │  │  embedding(prompt) → store in Redis with TTL          │  │
                         │  └──────────────────────────────────────────────────────┘  │
                         └──────────────────────────────────────────────────────────┘
                                                    │
                              ┌─────────────────────▼────────────────────────────┐
                              │                    REDIS                          │
                              │  semantic_cache (vector + payload, TTL)          │
                              │  rate_limit:{tenant}:{window} (sliding window)   │
                              │  budget:{tenant}:{month} (token counter)         │
                              └──────────────────────────────────────────────────┘
                                                    │
                              ┌─────────────────────▼────────────────────────────┐
                              │              EXTERNAL LLM PROVIDERS               │
                              │  OpenAI API · Anthropic API · Ollama (local)     │
                              └──────────────────────────────────────────────────┘
```

### Semantic Cache Architecture (Detail)

```
   INCOMING PROMPT "How do I reset my password?"
              │
              ▼
   ┌─────────────────────────────────────────────────────┐
   │  1. EMBED  prompt → text-embedding-3-small          │
   │     → 1536-dim float vector                         │
   └──────────────────────┬──────────────────────────────┘
                          │
                          ▼
   ┌─────────────────────────────────────────────────────┐
   │  2. SEARCH  Redis vector index (HNSW or FLAT)       │
   │     query: find top-k nearest by cosine similarity  │
   │     filter: same model_family, within TTL           │
   │     k = 5                                           │
   └──────────────────────┬──────────────────────────────┘
                          │
                    top-k candidates
                          │
                          ▼
   ┌─────────────────────────────────────────────────────┐
   │  3. THRESHOLD  if max_similarity >= 0.95:           │
   │     CACHE HIT → return stored response              │
   │                                                     │
   │     else: CACHE MISS → proceed to model routing     │
   └──────────────────────┬──────────────────────────────┘
                          │ MISS
                          ▼
   ┌─────────────────────────────────────────────────────┐
   │  4. EXECUTE  call LLM, get response                 │
   └──────────────────────┬──────────────────────────────┘
                          │
                          ▼
   ┌─────────────────────────────────────────────────────┐
   │  5. STORE  embedding(prompt) + response + metadata  │
   │     key: cache:{sha256(prompt)[:16]}                │
   │     vector: stored in Redis vector index            │
   │     TTL: configurable (default 1 hour)              │
   │     tags: model_family, tenant, task_type           │
   └─────────────────────────────────────────────────────┘
```

### Model Routing Decision Tree

```
                        ┌─────────────────────┐
                        │  Incoming Request    │
                        │  (cache miss)        │
                        └──────────┬──────────┘
                                   │
                          ┌────────▼────────┐
                          │ task_type known? │
                          └────────┬────────┘
                              │         │
                            YES        NO (auto-detect)
                              │         │
                              │    ┌────▼───────────────────┐
                              │    │ Heuristic classify:    │
                              │    │ - len(prompt) > 2000   │
                              │    │   → complex            │
                              │    │ - contains "explain",  │
                              │    │   "analyze","compare"  │
                              │    │   → complex            │
                              │    │ - else → simple        │
                              │    └────┬───────────────────┘
                              │         │
                    ┌─────────▼─────────▼─────────┐
                    │      task_type == ?          │
                    └─────────┬───────────────────┘
                              │
              ┌───────────────┼───────────────┐
              │                               │
          simple                           complex
              │                               │
              ▼                               ▼
    ┌─────────────────┐             ┌─────────────────┐
    │  GPT-4o-mini    │             │     GPT-4o      │
    │  (cheap tier)   │             │  (expensive)    │
    └────────┬────────┘             └────────┬────────┘
             │                               │
             ▼                               │
    ┌─────────────────┐                      │
    │ confidence_check│                      │
    │ enabled?        │                      │
    └────────┬────────┘                      │
        YES  │  NO ───────────────────────────┘
             │  │                               │
             ▼  ▼                               ▼
    ┌─────────────────────┐             ┌──────────────┐
    │ self-evaluate:      │             │  return      │
    │ "Rate confidence    │             │  response    │
    │ 1-10 for this       │             │              │
    │  answer"            │             └──────────────┘
    └────────┬────────────┘
             │
             ▼
    ┌─────────────────────┐
    │ confidence < 7?     │
    │  OR len(response)   │
    │  < 50 chars?        │
    └────────┬────────────┘
        YES  │   NO
             │    │
             ▼    └──────────────────▶ return mini response
    ┌─────────────────────┐
    │ ESCALATE to GPT-4o  │
    │ (re-run with full   │
    │  model)             │
    └────────┬────────────┘
             │
             ▼
    ┌─────────────────────┐
    │ return GPT-4o       │
    │ response            │
    │ (log escalation)    │
    └─────────────────────┘
```

### Fallback Chain Diagram

```
   ┌─────────────────────────────────────────────────────────┐
   │                  PROVIDER FAILOVER                       │
   └─────────────────────────────────────────────────────────┘

   request ──────▶ ┌─────────────┐
                   │  Provider 1 │  OpenAI (primary)
                   │  OpenAI     │
                   └──────┬──────┘
                          │
              ┌───────────┼───────────┐
              │           │           │
          200 OK      5xx/429    timeout (>10s)
              │           │           │
              │           └──────┬────┘
              │                  │
              │                  ▼
              │          ┌─────────────┐
              │          │  Provider 2 │  Anthropic (secondary)
              │          │  Anthropic  │
              │          └──────┬──────┘
              │                 │
              │         ┌───────┼───────┐
              │         │       │       │
              │      200 OK   5xx   timeout
              │         │       │       │
              │         │       └───┬───┘
              │         │           │
              │         │           ▼
              │         │   ┌─────────────┐
              │         │   │  Provider 3 │  Ollama (local, degraded)
              │         │   │  Ollama     │  — smaller model, may be lower quality
              │         │   └──────┬──────┘
              │         │          │
              │         │     ┌────┼────┐
              │         │     │    │    │
              │         │   200   err  timeout
              │         │     │    │    │
              │         │     │    └──┬─┘
              │         │     │       │
              │         │     │       ▼
              │         │     │  ┌─────────────┐
              │         │     │  │  RETURN 503 │  All providers failed
              │         │     │  │  (log alert)│
              │         │     │  └─────────────┘
              ▼         ▼     ▼
   ┌─────────────────────────────┐
   │  RETURN RESPONSE + LOG      │
   │  (which provider served it) │
   └─────────────────────────────┘

   Circuit Breaker: if a provider fails 5× in 60s, mark it "open"
   (skip it) for 30s before retrying — prevents cascading failures.
```

---

## 4. Tech Stack with Rationale

| Layer | Technology | Why This Choice | Alternatives Considered |
|-------|-----------|-----------------|------------------------|
| **Gateway API** | FastAPI (Python 3.11+) | Async-native (handles concurrent LLM calls), automatic OpenAPI docs, Pydantic validation, ecosystem familiarity for the target audience | Flask (sync, poor concurrency), Litestar (smaller community) |
| **Semantic cache** | Redis 7.4+ with `RediSearch` / vector similarity | Sub-ms vector search at scale; TTL natively supported; already needed for rate limiting so no new infra | Qdrant/Pinecone (separate infra, cost), pgvector (slower for high QPS) |
| **Rate limiting** | Redis (sliding window via sorted sets) | Atomic Lua scripts, sub-ms operations, already in stack | In-memory (not distributed), NGINX (inflexible per-tenant logic) |
| **Cost tracking / budgets** | PostgreSQL 16 | ACID for financial-grade cost records; SQL aggregation for reporting; tenant budget as a column with atomic `UPDATE ... RETURNING` | DynamoDB (no SQL aggregation), MongoDB (weaker consistency) |
| **LLM providers** | OpenAI (`openai` SDK) + Anthropic (`anthropic` SDK) + Ollama (`ollama` REST) | Market leaders + local fallback; SDKs handle retries/streaming/typed responses | LiteLLM (adds abstraction layer we don't need for 3 providers) |
| **Embeddings** | OpenAI `text-embedding-3-small` (1536-dim) | Cheapest production embedding model ($0.02/M tokens), good quality, already have OpenAI key | Sentence-Transformers local (no API cost but needs GPU/CPU and lower throughput) |
| **Admin dashboard** | Streamlit 1.40+ | Fastest path to internal dashboard for a data engineer; SQL + pandas is native; no frontend build | Grafana (ops-focused, less SQL-native), Metabase (heavier) |
| **Background tasks** | FastAPI `BackgroundTasks` + APScheduler | Cache writes and cost recording are fire-and-forget; budget reset cron via APScheduler | Celery (overkill for this scale), Airflow (wrong tool for sub-second tasks) |
| **Testing** | pytest + pytest-asyncio + httpx (AsyncClient) + `fakeredis` + `pytest-postgresql` | Standard Python testing stack; async support; in-memory Redis/PG for fast isolated tests | unittest (verbose), locust (load testing — separate concern) |
| **Containerization** | Docker Compose | Single-command local dev with all services; matches the target audience's existing Docker skills | Kubernetes (premature for this stage), bare metal (no reproducibility) |
| **Logging** | `structlog` (JSON output) | Structured logging that's easy to ship to ELK/Datadog; contextvars for per-request tenant/model binding | stdlib `logging` (not structured by default), loguru (less ecosystem integration) |

### Why Not LiteLLM / LangChain?

The gateway intentionally avoids LLM abstraction frameworks:

- **LiteLLM** wraps 100+ providers behind one interface — useful, but it hides the provider-specific features (Anthropic prompt caching, OpenAI structured outputs) that we need for cost optimization. For 3 providers, a thin adapter pattern is clearer and more debuggable.
- **LangChain** adds orchestration abstractions (chains, agents) that this gateway doesn't need. The gateway is a proxy, not an agent framework. Adding LangChain would mean debugging two layers when the LLM call fails.

**Principle: own the provider interface. It's 3 adapter classes, not a framework dependency.**

---

## 5. Database & Schema Design

### PostgreSQL Schema

```sql
-- ============================================================
-- TENANTS: API-keyed customers with budgets and rate limits
-- ============================================================
CREATE TABLE tenants (
    tenant_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_name        VARCHAR(255) NOT NULL,
    api_key_hash       VARCHAR(255) NOT NULL UNIQUE,   -- SHA-256 of API key
    api_key_prefix     VARCHAR(8) NOT NULL,             -- first 8 chars for identification
    monthly_token_budget BIGINT NOT NULL DEFAULT 1_000_000,  -- tokens/month
    rate_limit_per_minute  INT NOT NULL DEFAULT 60,
    rate_limit_per_hour    INT NOT NULL DEFAULT 3600,
    rate_limit_per_day     INT NOT NULL DEFAULT 86400,
    is_active          BOOLEAN NOT NULL DEFAULT TRUE,
    created_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at         TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tenants_api_key_hash ON tenants(api_key_hash);
CREATE INDEX idx_tenants_active ON tenants(is_active) WHERE is_active = TRUE;

-- ============================================================
-- MODEL_PRICING: cost per 1M tokens, versioned
-- ============================================================
CREATE TABLE model_pricing (
    pricing_id         SERIAL PRIMARY KEY,
    provider           VARCHAR(50) NOT NULL,             -- 'openai', 'anthropic', 'ollama'
    model_name         VARCHAR(100) NOT NULL,            -- 'gpt-4o', 'gpt-4o-mini', 'claude-3-5-sonnet'
    input_cost_per_million   NUMERIC(10,4) NOT NULL,     -- USD per 1M input tokens
    output_cost_per_million  NUMERIC(10,4) NOT NULL,     -- USD per 1M output tokens
    effective_from     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    effective_to       TIMESTAMPTZ,                      -- NULL = currently active
    UNIQUE(provider, model_name, effective_from)
);

-- Seed data (prices as of 2025-08; update via admin API)
INSERT INTO model_pricing (provider, model_name, input_cost_per_million, output_cost_per_million) VALUES
    ('openai',    'gpt-4o',         5.00,  15.00),
    ('openai',    'gpt-4o-mini',    0.15,   0.60),
    ('anthropic', 'claude-3-5-sonnet', 3.00, 15.00),
    ('anthropic', 'claude-3-5-haiku',  0.80,  4.00),
    ('ollama',    'llama3.1:8b',    0.00,   0.00);   -- self-hosted, no API cost

-- ============================================================
-- COST_RECORDS: one row per LLM call (cache misses only)
-- ============================================================
CREATE TABLE cost_records (
    record_id          BIGSERIAL PRIMARY KEY,
    tenant_id          UUID NOT NULL REFERENCES tenants(tenant_id),
    request_id         UUID NOT NULL,                   -- correlates with logs
    provider           VARCHAR(50) NOT NULL,
    model_name         VARCHAR(100) NOT NULL,
    input_tokens       INT NOT NULL,
    output_tokens      INT NOT NULL,
    total_tokens       INT NOT NULL,                    -- input + output (generated)
    cost_usd           NUMERIC(10,6) NOT NULL,          -- computed at request time
    cache_hit          BOOLEAN NOT NULL DEFAULT FALSE,
    cache_key          VARCHAR(64),                     -- sha256 prefix if cache hit
    task_type          VARCHAR(20) NOT NULL DEFAULT 'simple',  -- 'simple' | 'complex'
    escalated          BOOLEAN NOT NULL DEFAULT FALSE,  -- did routing escalate?
    latency_ms         INT NOT NULL,                    -- end-to-end gateway latency
    endpoint           VARCHAR(100) NOT NULL,           -- '/v1/chat/completions'
    created_at         TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Aggregation-friendly indexes
CREATE INDEX idx_cost_tenant_date    ON cost_records(tenant_id, created_at DESC);
CREATE INDEX idx_cost_model_date     ON cost_records(model_name, created_at DESC);
CREATE INDEX idx_cost_endpoint_date  ON cost_records(endpoint, created_at DESC);
CREATE INDEX idx_cost_created_at     ON cost_records(created_at DESC);

-- ============================================================
-- BUDGET_USAGE: materialized monthly token usage per tenant
-- (Redis holds the live counter; this is the durable record)
-- ============================================================
CREATE TABLE budget_usage (
    tenant_id          UUID NOT NULL REFERENCES tenants(tenant_id),
    budget_month       DATE NOT NULL,                   -- always 1st of month
    tokens_used        BIGINT NOT NULL DEFAULT 0,
    estimated_cost_usd NUMERIC(12,2) NOT NULL DEFAULT 0,
    requests_count     INT NOT NULL DEFAULT 0,
    last_updated       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (tenant_id, budget_month)
);

-- ============================================================
-- RATE_LIMIT_EVENTS: audit log of rejected requests (optional)
-- ============================================================
CREATE TABLE rate_limit_events (
    event_id           BIGSERIAL PRIMARY KEY,
    tenant_id          UUID NOT NULL REFERENCES tenants(tenant_id),
    window             VARCHAR(10) NOT NULL,            -- 'minute', 'hour', 'day'
    limit_value        INT NOT NULL,
    attempted_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rate_limit_tenant_date ON rate_limit_events(tenant_id, attempted_at DESC);

-- ============================================================
-- VIEWS for the dashboard
-- ============================================================
CREATE OR REPLACE VIEW v_daily_cost_summary AS
SELECT
    tenant_id,
    DATE(created_at) AS day,
    model_name,
    COUNT(*) AS request_count,
    SUM(input_tokens) AS total_input_tokens,
    SUM(output_tokens) AS total_output_tokens,
    SUM(total_tokens) AS total_tokens,
    SUM(cost_usd) AS total_cost_usd,
    SUM(CASE WHEN cache_hit THEN 1 ELSE 0 END) AS cache_hits,
    ROUND(AVG(CASE WHEN cache_hit THEN 1 ELSE 0 END) * 100, 1) AS cache_hit_rate_pct
FROM cost_records
GROUP BY tenant_id, DATE(created_at), model_name;

CREATE OR REPLACE VIEW v_tenant_monthly_budget AS
SELECT
    t.tenant_id,
    t.tenant_name,
    t.monthly_token_budget,
    DATE_TRUNC('month', NOW())::DATE AS budget_month,
    COALESCE(bu.tokens_used, 0) AS tokens_used,
    COALESCE(bu.estimated_cost_usd, 0) AS estimated_cost,
    ROUND(COALESCE(bu.tokens_used, 0)::NUMERIC / NULLIF(t.monthly_token_budget, 0) * 100, 1) AS budget_used_pct
FROM tenants t
LEFT JOIN budget_usage bu ON t.tenant_id = bu.tenant_id
    AND bu.budget_month = DATE_TRUNC('month', NOW())::DATE
WHERE t.is_active = TRUE;
```

### Redis Key Layout

| Key Pattern | Type | TTL | Purpose |
|-------------|------|-----|---------|
| `semantic_cache:{sha256[:16]}` | Hash (vector + payload) | configurable (default 3600s) | Cache entry: embedding vector + cached response + metadata |
| `semantic_cache:index` | RediSearch index (HNSW) | — | Vector similarity index for cosine search |
| `rate:{tenant_id}:minute` | Sorted set (ZSET) | 60s | Sliding-window rate limit (per-minute) |
| `rate:{tenant_id}:hour` | ZSET | 3600s | Sliding-window rate limit (per-hour) |
| `rate:{tenant_id}:day` | ZSET | 86400s | Sliding-window rate limit (per-day) |
| `budget:{tenant_id}:{YYYY-MM}` | String (integer) | 35 days | Live token budget counter for current month |
| `circuit:{provider}` | String ("open"/"closed") | 30s | Circuit breaker state per provider |
| `budget_warned:{tenant_id}:{YYYY-MM}` | String | 35 days | Flag: 80% warning already sent this month |

---

## 6. Project Structure

```
llm-gateway/
├── docker-compose.yml
├── .env.example
├── Makefile
├── pyproject.toml
├── README.md
├── agent.md                      # Herdr multi-agent orchestration guide
│
├── gateway/                          # Main FastAPI application
│   ├── __init__.py
│   ├── main.py                       # FastAPI app factory, lifespan, middleware
│   ├── config.py                     # Pydantic Settings (env-driven)
│   ├── dependencies.py               # FastAPI dependency injection (tenant, budget, etc.)
│   │
│   ├── api/                          # API routes
│   │   ├── __init__.py
│   │   ├── v1/
│   │   │   ├── __init__.py
│   │   │   ├── chat.py               # POST /v1/chat/completions (main endpoint)
│   │   │   ├── embeddings.py         # POST /v1/embeddings
│   │   │   └── health.py             # GET /health, GET /ready
│   │   └── admin.py                  # Admin endpoints (tenants, pricing, cache stats)
│   │
│   ├── cache/                        # Semantic cache
│   │   ├── __init__.py
│   │   ├── semantic_cache.py         # Embed, search, store, TTL logic
│   │   └── embedding_client.py       # OpenAI embedding API wrapper
│   │
│   ├── routing/                      # Model routing engine
│   │   ├── __init__.py
│   │   ├── router.py                 # Task classification + confidence escalation
│   │   ├── classifier.py             # Heuristic task-type classifier
│   │   └── confidence.py             # Self-evaluation confidence scorer
│   │
│   ├── providers/                    # LLM provider adapters
│   │   ├── __init__.py
│   │   ├── base.py                   # Abstract LLMProvider interface
│   │   ├── openai_provider.py        # OpenAI adapter
│   │   ├── anthropic_provider.py     # Anthropic adapter
│   │   ├── ollama_provider.py        # Ollama (local) adapter
│   │   └── failover.py               # Failover chain + circuit breaker
│   │
│   ├── middleware/                   # Cross-cutting concerns
│   │   ├── __init__.py
│   │   ├── auth.py                   # API key → tenant resolution
│   │   ├── rate_limiter.py           # Sliding-window rate limit (Redis ZSET)
│   │   ├── budget_enforcer.py        # Token budget check + deduction
│   │   ├── request_logger.py         # Structured JSON logging
│   │   └── error_handler.py          # Consistent error responses
│   │
│   ├── tracking/                     # Cost tracking
│   │   ├── __init__.py
│   │   ├── cost_calculator.py        # Tokens × pricing → USD
│   │   ├── cost_store.py             # PostgreSQL cost_records writer
│   │   └── budget_store.py           # Redis budget counter + PG sync
│   │
│   ├── models/                       # Pydantic schemas
│   │   ├── __init__.py
│   │   ├── chat.py                   # ChatCompletionRequest / Response
│   │   ├── tenant.py                 # Tenant, BudgetUsage
│   │   └── admin.py                  # Admin API schemas
│   │
│   └── db/
│       ├── __init__.py
│       ├── session.py                # asyncpg connection pool
│       ├── migrations/
│       │   ├── 001_initial.sql       # Schema creation
│       │   └── 002_seed_pricing.sql  # Model pricing seed data
│       └── repositories/
│           ├── tenant_repo.py
│           ├── cost_repo.py
│           └── budget_repo.py
│
├── dashboard/                        # Streamlit admin dashboard
│   ├── __init__.py
│   ├── app.py                        # Streamlit entry point
│   ├── pages/
│   │   ├── overview.py               # Cache hit rate, total cost, request volume
│   │   ├── cost_by_tenant.py         # Cost breakdown per tenant
│   │   ├── model_distribution.py     # Which models are serving what
│   │   ├── budget_status.py          # Budget usage bars + warnings
│   │   └── rate_limits.py            # Rate limit events + current windows
│   └── charts.py                     # Reusable chart helpers
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py                   # Fixtures: fakeredis, pytest-postgresql, mock providers
│   ├── unit/
│   │   ├── test_semantic_cache.py
│   │   ├── test_rate_limiter.py
│   │   ├── test_budget_enforcer.py
│   │   ├── test_router.py
│   │   ├── test_confidence.py
│   │   ├── test_cost_calculator.py
│   │   ├── test_failover.py
│   │   └── test_circuit_breaker.py
│   ├── integration/
│   │   ├── test_chat_endpoint.py     # Full request lifecycle with fakeredis + test PG
│   │   ├── test_cache_hit_miss.py
│   │   ├── test_budget_rejection.py
│   │   └── test_provider_failover.py
│   └── load/
│       └── locustfile.py             # Load test: 100 concurrent users, mixed cache hit/miss
│
├── scripts/
│   ├── seed_tenants.py               # Create test tenants with API keys
│   ├── benchmark_cache.py            # Measure cache hit rate on a Q&A dataset
│   └── cost_report.py                # CLI: print monthly cost report
│
├── Dockerfile                        # Gateway image
├── Dockerfile.dashboard              # Streamlit dashboard image
└── .env.example
```

---

## 7. Component Specifications

### 7.1 Semantic Cache (`gateway/cache/semantic_cache.py`)

```python
"""
Semantic cache: stores prompt embeddings in Redis and retrieves cached
responses when a new prompt's embedding is within a cosine similarity
threshold of an existing entry.

Key design decisions:
- Embedding is done via OpenAI text-embedding-3-small (1536-dim, $0.02/M tokens)
- Vector search uses Redis RediSearch HNSW index (sub-ms at <100k entries)
- Cache entries are namespaced by model_family (gpt-4o vs claude) to avoid
  cross-model response contamination
- TTL is per-entry (default 1h); entries auto-expire from Redis
- Cache is bypassed for requests with temperature > 0 or seed set (non-deterministic)
"""

import hashlib
import time
import json
from dataclasses import dataclass
from typing import Optional

import redis.asyncio as redis
from redis.commands.search.query import Query

from gateway.cache.embedding_client import EmbeddingClient
from gateway.config import Settings


@dataclass
class CacheEntry:
    cache_key: str
    response: dict          # The full LLM response JSON
    model_name: str
    tenant_id: str
    created_at: float
    similarity: float       # Only set on retrieval


class SemanticCache:
    VECTOR_FIELD = "embedding"
    PAYLOAD_FIELD = "response"
    METADATA_FIELDS = ["model_name", "tenant_id", "created_at"]

    def __init__(
        self,
        redis_client: redis.Redis,
        embedding_client: EmbeddingClient,
        settings: Settings,
    ):
        self._redis = redis_client
        self._embedder = embedding_client
        self._settings = settings
        self._index_name = settings.cache_index_name       # "semantic_cache:index"
        self._key_prefix = settings.cache_key_prefix       # "semantic_cache"
        self._similarity_threshold = settings.cache_similarity_threshold  # 0.95
        self._ttl_seconds = settings.cache_ttl_seconds     # 3600
        self._top_k = settings.cache_top_k                 # 5

    async def lookup(
        self,
        prompt: str,
        model_family: str,
        tenant_id: str,
    ) -> Optional[CacheEntry]:
        """
        Search the cache for a semantically similar prompt.
        Returns a CacheEntry if similarity >= threshold, else None.
        """
        # 1. Embed the incoming prompt
        embedding = await self._embedder.embed(prompt)  # List[float], 1536-dim

        # 2. Build the RediSearch vector query
        #    Filter by model_family and tenant_id (tenants don't share cache)
        #    Use cosine similarity (Redis returns distance = 1 - cosine_sim)
        query = (
            Query(f"(@model_family:{{{model_family}}} @tenant_id:{{{tenant_id}}})")
            .add_vector_param(
                self.VECTOR_FIELD,
                embedding,
            )
            .return_fields(*self.METADATA_FIELDS, self.PAYLOAD_FIELD, "__vector_score")
            .limit(0, self._top_k)
            .dialect(2)
        )

        # 3. Execute search
        results = await self._redis.ft(self._index_name).search(query)

        if not results.docs:
            return None

        # 4. Redis returns distance (1 - cosine_similarity); convert back
        best = results.docs[0]
        cosine_distance = float(best.__vector_score)
        cosine_similarity = 1.0 - cosine_distance

        if cosine_similarity < self._similarity_threshold:
            return None  # Closest match still too far → cache miss

        # 5. Deserialize the cached response
        response = json.loads(best.response)
        return CacheEntry(
            cache_key=best.id,
            response=response,
            model_name=best.model_name,
            tenant_id=best.tenant_id,
            created_at=float(best.created_at),
            similarity=cosine_similarity,
        )

    async def store(
        self,
        prompt: str,
        response: dict,
        model_name: str,
        model_family: str,
        tenant_id: str,
    ) -> str:
        """
        Store a prompt-response pair in the cache.
        Returns the cache key.
        """
        embedding = await self._embedder.embed(prompt)
        prompt_hash = hashlib.sha256(prompt.encode()).hexdigest()[:16]
        cache_key = f"{self._key_prefix}:{prompt_hash}"

        entry = {
            self.VECTOR_FIELD: embedding,          # stored as vector
            self.PAYLOAD_FIELD: json.dumps(response),
            "model_name": model_name,
            "model_family": model_family,
            "tenant_id": tenant_id,
            "created_at": str(time.time()),
        }

        # Store as a Redis hash with TTL
        await self._redis.hset(cache_key, mapping=entry)
        await self._redis.expire(cache_key, self._ttl_seconds)

        return cache_key

    async def invalidate_tenant(self, tenant_id: str) -> int:
        """Flush all cache entries for a tenant (admin operation)."""
        # Scan and delete keys matching tenant_id in the index
        # In production, use a secondary index on tenant_id for efficient deletion
        count = 0
        async for key in self._redis.scan_iter(
            match=f"{self._key_prefix}:*", count=100
        ):
            entry_tenant = await self._redis.hget(key, "tenant_id")
            if entry_tenant == tenant_id:
                await self._redis.delete(key)
                count += 1
        return count

    def should_cache(self, request: dict) -> bool:
        """
        Determine if a request is cacheable.
        Non-deterministic requests (temperature > 0, seed set) are not cached.
        """
        if request.get("temperature", 0) > 0:
            return False
        if "seed" in request:
            return False
        if request.get("stream", False):
            return False  # v1 doesn't cache streaming responses
        return True
```

### 7.2 Model Routing Engine (`gateway/routing/router.py`)

```python
"""
Model routing: classifies the task type and selects the appropriate model.
If confidence checking is enabled, a cheap-model response that scores low
confidence is escalated to the expensive model.

Routing rules (configurable in settings):
  - task_type == "simple"  → GPT-4o-mini
  - task_type == "complex" → GPT-4o
  - confidence < threshold → escalate to GPT-4o

The classifier uses heuristics (prompt length, keyword matching) by default.
A learned classifier (fine-tuned embedding classifier) can be plugged in later.
"""

from dataclasses import dataclass
from typing import Optional

from gateway.routing.classifier import TaskClassifier
from gateway.routing.confidence import ConfidenceScorer
from gateway.providers.base import LLMProvider, LLMResponse
from gateway.providers.failover import FailoverChain
from gateway.config import Settings


@dataclass
class RoutingDecision:
    primary_model: str
    escalated_model: Optional[str]   # None if no escalation needed
    task_type: str                   # "simple" | "complex"
    confidence_score: Optional[float]
    escalated: bool = False


class ModelRouter:
    def __init__(
        self,
        failover_chain: FailoverChain,
        classifier: TaskClassifier,
        confidence_scorer: ConfidenceScorer,
        settings: Settings,
    ):
        self._failover = failover_chain
        self._classifier = classifier
        self._confidence = confidence_scorer
        self._settings = settings

        # Model mapping from config
        self._model_map = settings.routing_model_map
        # e.g. {"simple": "gpt-4o-mini", "complex": "gpt-4o",
        #        "escalate": "gpt-4o"}

    async def route_and_execute(
        self,
        prompt: str,
        request: dict,
        tenant_id: str,
    ) -> tuple[LLMResponse, RoutingDecision]:
        """
        Classify the task, select a model, execute, and optionally escalate.
        Returns the final response and the routing decision (for logging).
        """
        # 1. Classify task type
        task_type = self._classifier.classify(prompt, request)
        primary_model = self._model_map[task_type]

        # 2. Execute on the primary (cheap) model via failover chain
        response = await self._failover.execute(
            model=primary_model,
            request=request,
            tenant_id=tenant_id,
        )

        decision = RoutingDecision(
            primary_model=primary_model,
            escalated_model=None,
            task_type=task_type,
            confidence_score=None,
            escalated=False,
        )

        # 3. Confidence check (only for simple tasks that used the cheap model)
        if (
            task_type == "simple"
            and self._settings.routing_confidence_check_enabled
            and primary_model == self._model_map["simple"]
        ):
            confidence = await self._confidence.score(
                prompt=prompt,
                response=response.content,
                model=primary_model,
            )
            decision.confidence_score = confidence

            if confidence < self._settings.routing_confidence_threshold:
                # 4. Escalate to the expensive model
                escalate_model = self._model_map["escalate"]
                response = await self._failover.execute(
                    model=escalate_model,
                    request=request,
                    tenant_id=tenant_id,
                )
                decision.escalated_model = escalate_model
                decision.escalated = True

        return response, decision
```

### 7.3 Task Classifier (`gateway/routing/classifier.py`)

```python
"""
Heuristic task classifier: determines if a prompt is 'simple' or 'complex'.

Rules (in priority order):
1. If the request explicitly sets "model" → respect it, skip classification
2. If prompt length > 2000 chars → "complex" (long context needs reasoning)
3. If prompt contains reasoning keywords → "complex"
4. If prompt contains code/math keywords → "complex"
5. Otherwise → "simple"
"""

import re

COMPLEX_KEYWORDS = [
    "explain", "analyze", "compare", "contrast", "synthesize",
    "reason", "derive", "prove", "design", "architect", "refactor",
    "debug", "optimize", "evaluate", "critique", "summarize long",
]
CODE_MATH_KEYWORDS = [
    "function", "algorithm", "equation", "theorem", "complexity",
    "recursive", "concurrent", "distributed", "sql query", "code review",
]
COMPLEX_PATTERN = re.compile(
    r"\b(" + "|".join(COMPLEX_KEYWORDS + CODE_MATH_KEYWORDS) + r")\b",
    re.IGNORECASE,
)


class TaskClassifier:
    def __init__(self, long_prompt_threshold: int = 2000):
        self._threshold = long_prompt_threshold

    def classify(self, prompt: str, request: dict) -> str:
        # Respect explicit model choice
        if request.get("model"):
            # If user specified a model, infer task type from it
            model = request["model"].lower()
            if "mini" in model or "haiku" in model or "8b" in model:
                return "simple"
            return "complex"

        # Long prompts need reasoning
        if len(prompt) > self._threshold:
            return "complex"

        # Keyword-based detection
        if COMPLEX_PATTERN.search(prompt):
            return "complex"

        return "simple"
```

### 7.4 Confidence Scorer (`gateway/routing/confidence.py`)

```python
"""
Confidence scorer: evaluates whether a cheap-model response is good enough
or should be escalated to a more expensive model.

Two strategies (configurable):
1. "self_eval" — ask the same model to rate its own confidence (1-10)
2. "heuristic" — length + keyword checks (no extra LLM call, free)

The self_eval strategy costs one extra LLM call per cheap-model response,
but catches low-quality answers. The heuristic strategy is free but less
accurate. Default: "heuristic" for cost; switch to "self_eval" for quality.
"""

from gateway.providers.base import LLMProvider
from gateway.config import Settings


SELF_EVAL_PROMPT = """\
You are evaluating the quality of an AI response.
Rate your confidence that this response fully and correctly answers the
user's question on a scale of 1-10.

User question: {question}

Response to evaluate: {response}

Respond with ONLY a single integer 1-10. No explanation.
"""


class ConfidenceScorer:
    def __init__(self, provider: LLMProvider, settings: Settings):
        self._provider = provider
        self._strategy = settings.confidence_strategy  # "self_eval" | "heuristic"
        self._min_response_length = settings.confidence_min_response_length  # 50

    async def score(self, prompt: str, response: str, model: str) -> float:
        if self._strategy == "self_eval":
            return await self._self_eval(prompt, response, model)
        else:
            return self._heuristic(response)

    async def _self_eval(self, prompt: str, response: str, model: str) -> float:
        """Ask the model to rate its own confidence. Returns 1.0-10.0."""
        eval_prompt = SELF_EVAL_PROMPT.format(question=prompt, response=response)
        eval_response = await self._provider.chat(
            model=model,
            messages=[{"role": "user", "content": eval_prompt}],
            temperature=0,
            max_tokens=5,
        )
        try:
            score = float(eval_response.content.strip())
            return max(1.0, min(10.0, score))
        except ValueError:
            return 5.0  # If we can't parse, assume medium confidence

    def _heuristic(self, response: str) -> float:
        """
        Free heuristic: short responses and hedging language indicate
        low confidence. Returns a pseudo-score 1-10.
        """
        score = 7.0  # Default: assume acceptable

        # Very short responses are suspicious
        if len(response.strip()) < self._min_response_length:
            score -= 3.0

        # Hedging language lowers confidence
        hedging = ["i'm not sure", "i cannot", "i can't help",
                   "as an ai", "i don't have access"]
        response_lower = response.lower()
        for phrase in hedging:
            if phrase in response_lower:
                score -= 2.0
                break

        # "I don't know" style responses
        if any(phrase in response_lower for phrase in
               ["unable to", "no information", "cannot determine"]):
            score -= 3.0

        return max(1.0, min(10.0, score))
```

### 7.5 Rate Limiter — Sliding Window (`gateway/middleware/rate_limiter.py`)

```python
"""
Sliding-window rate limiter using Redis sorted sets (ZSET).

Algorithm:
  - Each request adds a member (unique ID) with score = current timestamp
    to a ZSET keyed by tenant + window
  - Before adding, remove all members with score < (now - window_seconds)
  - Count remaining members; if count >= limit, reject
  - The ZSET auto-expires via Redis TTL set to window_seconds

This is a true sliding window (not fixed window), so it handles bursts
at window boundaries correctly. The entire check is atomic via a Lua script
to prevent race conditions under concurrent load.
"""

import time
import uuid
from typing import Optional

import redis.asyncio as redis


# Atomic Lua script: prune + count + add in one round trip
# Returns: 1 if allowed, 0 if rate-limited
LUA_SLIDING_WINDOW = """
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])
local member = ARGV[4]

-- Remove entries outside the window
redis.call('ZREMRANGEBYSCORE', key, 0, now - window)

-- Count current entries
local count = redis.call('ZCARD', key)

if count >= limit then
    return 0
end

-- Add the new request
redis.call('ZADD', key, now, member)

-- Set TTL to window size (so the key cleans up when idle)
redis.call('EXPIRE', key, window)

return 1
"""


class RateLimiter:
    def __init__(self, redis_client: redis.Redis):
        self._redis = redis_client
        self._script = redis_client.register_script(LUA_SLIDING_WINDOW)

    async def check(
        self,
        tenant_id: str,
        limits: dict[str, int],   # {"minute": 60, "hour": 3600, "day": 86400}
    ) -> tuple[bool, Optional[str]]:
        """
        Check all rate limit windows for a tenant.
        Returns (allowed, rejected_window) — rejected_window is None if allowed.
        """
        now = int(time.time())
        request_id = str(uuid.uuid4())

        window_seconds = {"minute": 60, "hour": 3600, "day": 86400}

        for window_name, limit in limits.items():
            key = f"rate:{tenant_id}:{window_name}"
            window = window_seconds[window_name]

            allowed = await self._script(
                keys=[key],
                args=[now, window, limit, request_id],
            )

            if not allowed:
                return False, window_name

        return True, None

    async def current_usage(
        self, tenant_id: str
    ) -> dict[str, int]:
        """Return current request counts per window (for dashboard)."""
        now = int(time.time())
        usage = {}
        for window_name, window_secs in [
            ("minute", 60), ("hour", 3600), ("day", 86400)
        ]:
            key = f"rate:{tenant_id}:{window_name}"
            # Prune and count
            await self._redis.zremrangebyscore(key, 0, now - window_secs)
            count = await self._redis.zcard(key)
            usage[window_name] = count
        return usage
```

### 7.6 Budget Enforcer (`gateway/middleware/budget_enforcer.py`)

```python
"""
Per-tenant monthly token budget enforcement.

Flow:
  1. PRE-CHECK: before the LLM call, check if the tenant has budget remaining.
     If budget exhausted → reject with HTTP 429 (budget_exceeded).
     If budget > 80% used → add warning header to response.
  2. POST-DEDUCT: after the LLM call, deduct actual tokens used from the
     Redis counter and sync to PostgreSQL.

The Redis counter is the source of truth for fast checks. PostgreSQL
budget_usage table is the durable record, synced asynchronously.

Budget is reset on the 1st of each month (APScheduler cron job clears
the Redis key and creates a new budget_usage row).
"""

import time
from datetime import datetime
from typing import Optional

import redis.asyncio as redis

from gateway.db.repositories.budget_repo import BudgetRepository


class BudgetEnforcer:
    WARNING_THRESHOLD = 0.80  # 80%

    def __init__(
        self,
        redis_client: redis.Redis,
        budget_repo: BudgetRepository,
    ):
        self._redis = redis_client
        self._repo = budget_repo

    def _budget_key(self, tenant_id: str) -> str:
        month = datetime.utcnow().strftime("%Y-%m")
        return f"budget:{tenant_id}:{month}"

    def _warned_key(self, tenant_id: str) -> str:
        month = datetime.utcnow().strftime("%Y-%m")
        return f"budget_warned:{tenant_id}:{month}"

    async def check_budget(
        self,
        tenant_id: str,
        monthly_limit: int,
        estimated_tokens: int,
    ) -> tuple[bool, Optional[float], Optional[str]]:
        """
        Pre-request budget check.
        Returns:
          - allowed (bool): True if budget permits this request
          - usage_pct (float|None): current usage percentage
          - warning (str|None): "80% budget warning" if at threshold
        """
        key = self._budget_key(tenant_id)
        current = int(await self._redis.get(key) or 0)

        # Check if adding estimated tokens would exceed budget
        if current + estimated_tokens > monthly_limit:
            usage_pct = (current / monthly_limit) * 100 if monthly_limit > 0 else 100
            return False, usage_pct, None

        usage_pct = (current / monthly_limit) * 100 if monthly_limit > 0 else 0

        # Check 80% warning (only emit once per month per tenant)
        warning = None
        if usage_pct >= self.WARNING_THRESHOLD * 100:
            warned_key = self._warned_key(tenant_id)
            already_warned = await self._redis.exists(warned_key)
            if not already_warned:
                await self._redis.set(warned_key, "1", ex=35 * 86400)  # 35 days TTL
                warning = f"Budget warning: {usage_pct:.1f}% of monthly limit used"

        return True, usage_pct, warning

    async def deduct_tokens(
        self,
        tenant_id: str,
        tokens_used: int,
        cost_usd: float,
        monthly_limit: int,
    ) -> int:
        """
        Post-request: deduct actual tokens from the Redis counter.
        Also sync to PostgreSQL (async, non-blocking).
        Returns the new total tokens used.
        """
        key = self._budget_key(tenant_id)
        # Atomic increment; set 35-day TTL if key is new
        new_total = await self._redis.incrby(key, tokens_used)
        if new_total == tokens_used:
            # Key was new — set TTL
            await self._redis.expire(key, 35 * 86400)

        # Async sync to PostgreSQL (fire and forget via background task)
        await self._repo.update_budget_usage(
            tenant_id=tenant_id,
            tokens_added=tokens_used,
            cost_added=cost_usd,
            requests_added=1,
        )

        return new_total

    async def reset_monthly_budget(self, tenant_id: str) -> None:
        """Called by APScheduler on the 1st of each month."""
        # The old key will expire naturally; we just stop using it.
        # The new month's key is created on first request.
        await self._repo.archive_month_and_create_new(tenant_id)
```

### 7.7 Provider Failover + Circuit Breaker (`gateway/providers/failover.py`)

```python
"""
Provider failover chain with circuit breaker.

Failover order: OpenAI → Anthropic → Ollama
  - On 5xx, 429, or timeout → try next provider
  - Circuit breaker: if a provider fails N times in M seconds, mark it
    "open" (skip it) for a cooldown period, then try again ("half-open")

The circuit breaker prevents cascading failures: if OpenAI is down, we
don't waste 10s timeouts on every request before falling back.
"""

import asyncio
import time
from dataclasses import dataclass
from typing import Optional

import redis.asyncio as redis

from gateway.providers.base import LLMProvider, LLMResponse
from gateway.config import Settings


@dataclass
class CircuitState:
    provider_name: str
    state: str           # "closed", "open", "half_open"
    failure_count: int
    opened_at: float


class CircuitBreaker:
    """Per-provider circuit breaker backed by Redis for distributed state."""

    def __init__(
        self,
        redis_client: redis.Redis,
        failure_threshold: int = 5,
        cooldown_seconds: int = 30,
        window_seconds: int = 60,
    ):
        self._redis = redis_client
        self._failure_threshold = failure_threshold
        self._cooldown = cooldown_seconds
        self._window = window_seconds

    def _key(self, provider: str) -> str:
        return f"circuit:{provider}"

    async def is_open(self, provider: str) -> bool:
        """Check if the circuit breaker is open (provider should be skipped)."""
        key = self._key(provider)
        state = await self._redis.hgetall(key)

        if not state:
            return False  # No state → closed

        current_state = state.get(b"state", b"closed").decode()
        if current_state == "open":
            # Check if cooldown has elapsed → move to half_open
            opened_at = float(state.get(b"opened_at", 0))
            if time.time() - opened_at > self._cooldown:
                await self._redis.hset(key, "state", "half_open")
                return False  # Allow one trial request
            return True

        return False  # closed or half_open → allow

    async def record_success(self, provider: str) -> None:
        """Reset the circuit breaker on a successful call."""
        key = self._key(provider)
        await self._redis.delete(key)

    async def record_failure(self, provider: str) -> None:
        """Record a failure; open the circuit if threshold is reached."""
        key = self._key(provider)
        failures = await self._redis.incr(f"{key}:failures")
        await self._redis.expire(f"{key}:failures", self._window)

        if failures >= self._failure_threshold:
            await self._redis.hset(
                key,
                mapping={
                    "state": "open",
                    "opened_at": str(time.time()),
                    "failure_count": str(failures),
                },
            )
            await self._redis.expire(key, self._cooldown + 60)


class FailoverChain:
    def __init__(
        self,
        providers: dict[str, LLMProvider],   # {"openai": ..., "anthropic": ..., "ollama": ...}
        circuit_breaker: CircuitBreaker,
        settings: Settings,
    ):
        self._providers = providers
        self._circuit = circuit_breaker
        self._order = settings.failover_order  # ["openai", "anthropic", "ollama"]
        self._timeout_seconds = settings.provider_timeout_seconds  # 10

    async def execute(
        self,
        model: str,
        request: dict,
        tenant_id: str,
    ) -> LLMResponse:
        """
        Try providers in failover order. If all fail, raise the last error.
        """
        last_error = None

        for provider_name in self._order:
            # Skip if circuit breaker is open
            if await self._circuit.is_open(provider_name):
                continue

            provider = self._providers[provider_name]

            # Check if this provider serves the requested model
            if not provider.supports_model(model):
                continue

            try:
                response = await asyncio.wait_for(
                    provider.chat(model=model, request=request),
                    timeout=self._timeout_seconds,
                )
                await self._circuit.record_success(provider_name)
                response.provider = provider_name  # Tag for logging
                return response

            except Exception as e:
                await self._circuit.record_failure(provider_name)
                last_error = e
                continue  # Try next provider

        # All providers failed
        raise AllProvidersFailedError(
            f"All providers failed for model {model}. Last error: {last_error}"
        )


class AllProvidersFailedError(Exception):
    pass
```

### 7.8 Cost Calculator (`gateway/tracking/cost_calculator.py`)

```python
"""
Cost calculator: converts token counts to USD using the model_pricing table.
Caches pricing in-memory (refreshed every 5 minutes) to avoid a DB hit per request.
"""

import time
from typing import Optional

from gateway.db.repositories.cost_repo import CostRepository


class CostCalculator:
    REFRESH_INTERVAL = 300  # 5 minutes

    def __init__(self, cost_repo: CostRepository):
        self._repo = cost_repo
        self._pricing_cache: dict[tuple[str, str], tuple[float, float]] = {}
        self._last_refresh = 0.0

    async def _ensure_pricing_fresh(self) -> None:
        if time.time() - self._last_refresh > self.REFRESH_INTERVAL:
            rows = await self._repo.get_active_pricing()
            self._pricing_cache = {
                (r["provider"], r["model_name"]): (
                    float(r["input_cost_per_million"]),
                    float(r["output_cost_per_million"]),
                )
                for r in rows
            }
            self._last_refresh = time.time()

    async def calculate_cost(
        self,
        provider: str,
        model_name: str,
        input_tokens: int,
        output_tokens: int,
    ) -> float:
        """
        Calculate the USD cost of a request.
        cost = (input_tokens / 1M * input_price) + (output_tokens / 1M * output_price)
        """
        await self._ensure_pricing_fresh()

        key = (provider, model_name)
        if key not in self._pricing_cache:
            # Unknown model — log warning, return 0 (don't block the request)
            return 0.0

        input_price, output_price = self._pricing_cache[key]
        cost = (input_tokens / 1_000_000 * input_price) + \
               (output_tokens / 1_000_000 * output_price)
        return round(cost, 6)
```

### 7.9 Structured Request Logger (`gateway/middleware/request_logger.py`)

```python
"""
Structured JSON logger: one log line per request with full context.
Uses structlog with contextvars for per-request tenant/model binding.

Output format (JSON, one line per request):
{
  "timestamp": "2026-08-10T12:34:56.789Z",
  "level": "info",
  "event": "llm_request",
  "request_id": "uuid",
  "tenant_id": "uuid",
  "tenant_name": "acme-corp",
  "endpoint": "/v1/chat/completions",
  "model": "gpt-4o-mini",
  "provider": "openai",
  "task_type": "simple",
  "escalated": false,
  "cache_hit": false,
  "cache_similarity": null,
  "input_tokens": 500,
  "output_tokens": 200,
  "total_tokens": 700,
  "cost_usd": 0.000195,
  "latency_ms": 845,
  "budget_used_pct": 23.5,
  "rate_limited": false,
  "status": "success"
}
"""

import time
import structlog
from contextvars import ContextVar

logger = structlog.get_logger("gateway")

# Context vars for per-request binding
request_id_ctx: ContextVar[str] = ContextVar("request_id", default="")
tenant_id_ctx: ContextVar[str] = ContextVar("tenant_id", default="")


def bind_request_context(request_id: str, tenant_id: str) -> None:
    request_id_ctx.set(request_id)
    tenant_id_ctx.set(tenant_id)


async def log_request(
    event: str,
    *,
    model: str,
    provider: str,
    task_type: str,
    escalated: bool,
    cache_hit: bool,
    cache_similarity: float | None,
    input_tokens: int,
    output_tokens: int,
    cost_usd: float,
    latency_ms: int,
    budget_used_pct: float,
    endpoint: str,
    status: str = "success",
    error: str | None = None,
) -> None:
    logger.info(
        event,
        request_id=request_id_ctx.get(),
        tenant_id=tenant_id_ctx.get(),
        endpoint=endpoint,
        model=model,
        provider=provider,
        task_type=task_type,
        escalated=escalated,
        cache_hit=cache_hit,
        cache_similarity=cache_similarity,
        input_tokens=input_tokens,
        output_tokens=output_tokens,
        total_tokens=input_tokens + output_tokens,
        cost_usd=cost_usd,
        latency_ms=latency_ms,
        budget_used_pct=budget_used_pct,
        status=status,
        error=error,
    )
```

### 7.10 Main Gateway Flow (`gateway/api/v1/chat.py`)

```python
"""
POST /v1/chat/completions — the main gateway endpoint.

Full request lifecycle:
  1. Auth: resolve tenant from API key
  2. Rate limit check (Redis sliding window)
  3. Budget pre-check (Redis counter)
  4. Semantic cache lookup (Redis vector search)
     → HIT: return cached response, log, done
     → MISS: continue
  5. Model routing (classify + execute + maybe escalate)
  6. Provider failover (OpenAI → Anthropic → Ollama)
  7. Cost calculation (tokens × pricing)
  8. Budget deduction (Redis + PostgreSQL sync)
  9. Cost record write (PostgreSQL)
  10. Cache write (if cacheable)
  11. Structured log
  12. Return response to client
"""

import time
import uuid
from fastapi import APIRouter, Depends, HTTPException, Request, Response

from gateway.models.chat import ChatCompletionRequest, ChatCompletionResponse
from gateway.dependencies import (
    get_tenant,
    get_rate_limiter,
    get_budget_enforcer,
    get_semantic_cache,
    get_router,
    get_cost_calculator,
    get_cost_store,
)
from gateway.middleware.request_logger import bind_request_context, log_request

router = APIRouter()


@router.post("/v1/chat/completions", response_model=ChatCompletionResponse)
async def chat_completions(
    request_body: ChatCompletionRequest,
    request: Request,
    response: Response,
    tenant=Depends(get_tenant),
    rate_limiter=Depends(get_rate_limiter),
    budget_enforcer=Depends(get_budget_enforcer),
    cache=Depends(get_semantic_cache),
    router_engine=Depends(get_router),
    cost_calculator=Depends(get_cost_calculator),
    cost_store=Depends(get_cost_store),
):
    request_id = str(uuid.uuid4())
    bind_request_context(request_id, str(tenant.tenant_id))
    start_time = time.time()
    endpoint = "/v1/chat/completions"

    # --- 2. Rate limit check ---
    allowed, rejected_window = await rate_limiter.check(
        tenant_id=str(tenant.tenant_id),
        limits={
            "minute": tenant.rate_limit_per_minute,
            "hour": tenant.rate_limit_per_hour,
            "day": tenant.rate_limit_per_day,
        },
    )
    if not allowed:
        response.headers["Retry-After"] = "60"
        await log_request(
            "llm_request", model="", provider="", task_type="",
            escalated=False, cache_hit=False, cache_similarity=None,
            input_tokens=0, output_tokens=0, cost_usd=0,
            latency_ms=int((time.time() - start_time) * 1000),
            budget_used_pct=0, endpoint=endpoint,
            status="rate_limited", error=f"window:{rejected_window}",
        )
        raise HTTPException(
            status_code=429,
            detail=f"Rate limit exceeded for window: {rejected_window}",
        )

    # --- 3. Budget pre-check ---
    estimated_tokens = len(request_body.messages[-1].content.split()) * 2  # rough
    budget_ok, usage_pct, warning = await budget_enforcer.check_budget(
        tenant_id=str(tenant.tenant_id),
        monthly_limit=tenant.monthly_token_budget,
        estimated_tokens=estimated_tokens,
    )
    if not budget_ok:
        await log_request(
            "llm_request", model="", provider="", task_type="",
            escalated=False, cache_hit=False, cache_similarity=None,
            input_tokens=0, output_tokens=0, cost_usd=0,
            latency_ms=int((time.time() - start_time) * 1000),
            budget_used_pct=usage_pct or 100, endpoint=endpoint,
            status="budget_exceeded",
        )
        raise HTTPException(
            status_code=429,
            detail="Monthly token budget exceeded. Contact admin to increase budget.",
        )

    if warning:
        response.headers["X-Budget-Warning"] = warning

    # --- 4. Semantic cache lookup ---
    prompt = request_body.messages[-1].content
    model_family = "openai"  # simplified; derive from routing config

    if cache.should_cache(request_body.model_dump()):
        cache_entry = await cache.lookup(
            prompt=prompt,
            model_family=model_family,
            tenant_id=str(tenant.tenant_id),
        )
        if cache_entry:
            # CACHE HIT — return immediately
            latency_ms = int((time.time() - start_time) * 1000)
            await log_request(
                "llm_request",
                model=cache_entry.model_name,
                provider="cache",
                task_type="",
                escalated=False,
                cache_hit=True,
                cache_similarity=cache_entry.similarity,
                input_tokens=0,
                output_tokens=0,
                cost_usd=0,
                latency_ms=latency_ms,
                budget_used_pct=usage_pct or 0,
                endpoint=endpoint,
                status="cache_hit",
            )
            return ChatCompletionResponse(**cache_entry.response)

    # --- 5-6. Model routing + failover ---
    try:
        llm_response, routing_decision = await router_engine.route_and_execute(
            prompt=prompt,
            request=request_body.model_dump(),
            tenant_id=str(tenant.tenant_id),
        )
    except Exception as e:
        latency_ms = int((time.time() - start_time) * 1000)
        await log_request(
            "llm_request", model="", provider="", task_type="",
            escalated=False, cache_hit=False, cache_similarity=None,
            input_tokens=0, output_tokens=0, cost_usd=0,
            latency_ms=latency_ms, budget_used_pct=usage_pct or 0,
            endpoint=endpoint, status="error", error=str(e),
        )
        raise HTTPException(status_code=503, detail="All LLM providers unavailable")

    # --- 7. Cost calculation ---
    cost_usd = await cost_calculator.calculate_cost(
        provider=llm_response.provider,
        model_name=llm_response.model,
        input_tokens=llm_response.input_tokens,
        output_tokens=llm_response.output_tokens,
    )

    # --- 8. Budget deduction ---
    await budget_enforcer.deduct_tokens(
        tenant_id=str(tenant.tenant_id),
        tokens_used=llm_response.total_tokens,
        cost_usd=cost_usd,
        monthly_limit=tenant.monthly_token_budget,
    )

    # --- 9. Cost record write ---
    await cost_store.record(
        tenant_id=str(tenant.tenant_id),
        request_id=request_id,
        provider=llm_response.provider,
        model_name=llm_response.model,
        input_tokens=llm_response.input_tokens,
        output_tokens=llm_response.output_tokens,
        total_tokens=llm_response.total_tokens,
        cost_usd=cost_usd,
        cache_hit=False,
        task_type=routing_decision.task_type,
        escalated=routing_decision.escalated,
        latency_ms=int((time.time() - start_time) * 1000),
        endpoint=endpoint,
    )

    # --- 10. Cache write ---
    if cache.should_cache(request_body.model_dump()):
        await cache.store(
            prompt=prompt,
            response=llm_response.to_dict(),
            model_name=llm_response.model,
            model_family=model_family,
            tenant_id=str(tenant.tenant_id),
        )

    # --- 11. Structured log ---
    latency_ms = int((time.time() - start_time) * 1000)
    await log_request(
        "llm_request",
        model=llm_response.model,
        provider=llm_response.provider,
        task_type=routing_decision.task_type,
        escalated=routing_decision.escalated,
        cache_hit=False,
        cache_similarity=None,
        input_tokens=llm_response.input_tokens,
        output_tokens=llm_response.output_tokens,
        cost_usd=cost_usd,
        latency_ms=latency_ms,
        budget_used_pct=usage_pct or 0,
        endpoint=endpoint,
        status="success",
    )

    # --- 12. Return response ---
    response.headers["X-Model-Used"] = llm_response.model
    response.headers["X-Provider"] = llm_response.provider
    response.headers["X-Cache"] = "MISS"
    response.headers["X-Cost-USD"] = f"{cost_usd:.6f}"
    response.headers["X-Escalated"] = str(routing_decision.escalated)

    return ChatCompletionResponse(
        id=request_id,
        model=llm_response.model,
        choices=[{"message": {"role": "assistant", "content": llm_response.content}}],
        usage={
            "input_tokens": llm_response.input_tokens,
            "output_tokens": llm_response.output_tokens,
            "total_tokens": llm_response.total_tokens,
            "cost_usd": cost_usd,
        },
    )
```

---

## 8. API Specification

### 8.1 Chat Completions

```
POST /v1/chat/completions
Authorization: Bearer <api_key>
Content-Type: application/json

Request:
{
  "model": "auto",                    // "auto" = gateway decides; or explicit model
  "messages": [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "How do I reset my password?"}
  ],
  "temperature": 0,                   // 0 = cacheable; >0 = bypass cache
  "max_tokens": 1000,
  "task_type": null                   // null = auto-classify; or "simple"/"complex"
}

Response (200):
{
  "id": "uuid",
  "model": "gpt-4o-mini",
  "choices": [
    {"message": {"role": "assistant", "content": "To reset your password..."}}
  ],
  "usage": {
    "input_tokens": 45,
    "output_tokens": 120,
    "total_tokens": 165,
    "cost_usd": 0.000080
  }
}

Response Headers:
  X-Model-Used: gpt-4o-mini
  X-Provider: openai
  X-Cache: HIT | MISS
  X-Cost-USD: 0.000080
  X-Escalated: false
  X-Budget-Warning: (only present at 80%+ usage)

Error Responses:
  401 Unauthorized       — invalid or missing API key
  429 Too Many Requests  — rate limit exceeded (Retry-After header set)
  429 Budget Exceeded    — monthly token budget exhausted
  503 Service Unavailable — all LLM providers failed
```

### 8.2 Health

```
GET /health          → 200 {"status": "ok"}
GET /ready           → 200 if Redis + PG + at least one provider reachable
```

### 8.3 Admin Endpoints

```
POST   /admin/tenants                  — create tenant (returns API key once)
GET    /admin/tenants                  — list tenants
GET    /admin/tenants/{id}             — tenant detail + budget usage
PATCH  /admin/tenants/{id}             — update budget / rate limits
DELETE /admin/tenants/{id}             — deactivate tenant

GET    /admin/cache/stats              — cache hit rate, size, eviction count
DELETE /admin/cache/tenant/{id}        — flush tenant's cache entries

GET    /admin/cost/summary?tenant=&from=&to=&group_by=day|model|endpoint
GET    /admin/budget/status            — all tenants' budget usage
GET    /admin/rate-limit/status        — current rate window usage per tenant

GET    /admin/pricing                  — current model pricing table
PUT    /admin/pricing/{id}             — update pricing (versioned)
```

---

## 9. Implementation Phases (10 Working Days)

### Phase 1: Foundation (Days 1–2)

**Day 1 — Project scaffold + Docker Compose + config**

| Task | Detail |
|------|--------|
| Create project structure | All directories and `__init__.py` files per Section 6 |
| `docker-compose.yml` | FastAPI, Redis (with RediSearch), PostgreSQL, Streamlit, Ollama |
| `pyproject.toml` | Dependencies: fastapi, uvicorn, redis, asyncpg, openai, anthropic, structlog, pydantic-settings, streamlit, pytest, pytest-asyncio, fakeredis, httpx |
| `gateway/config.py` | Pydantic Settings class reading from `.env` |
| `.env.example` | All env vars documented |
| `gateway/main.py` | App factory, lifespan (Redis pool, PG pool), health endpoints |
| `Dockerfile` | Python 3.11-slim, install deps, run uvicorn |

**Deliverable:** `docker-compose up` starts all services; `GET /health` returns 200.
**Verification:** `curl localhost:8000/health` → `{"status":"ok"}`; `redis-cli ping` → `PONG`; `psql` connects.

**Day 2 — Database schema + migrations + tenant auth**

| Task | Detail |
|------|--------|
| `001_initial.sql` | All tables from Section 5 |
| `002_seed_pricing.sql` | Model pricing seed data |
| `gateway/db/session.py` | asyncpg connection pool (lifespan-managed) |
| `gateway/db/repositories/tenant_repo.py` | CRUD for tenants |
| `gateway/middleware/auth.py` | API key → tenant resolution (SHA-256 lookup) |
| `gateway/models/tenant.py` | Pydantic schemas |
| `scripts/seed_tenants.py` | Create 3 test tenants with different budgets |
| `gateway/api/admin.py` | Tenant CRUD endpoints |

**Deliverable:** Can create tenants, get API keys, authenticate requests.
**Verification:** `POST /admin/tenants` → returns API key; `GET /health` with valid API key → 200; with invalid key → 401.

---

### Phase 2: Rate Limiting + Budget (Days 3–4)

**Day 3 — Rate limiter (Redis sliding window)**

| Task | Detail |
|------|--------|
| `gateway/middleware/rate_limiter.py` | Lua script + `check()` + `current_usage()` |
| `gateway/dependencies.py` | Rate limiter dependency injection |
| Wire into `/v1/chat/completions` | Rate limit check before cache/routing |
| `fakeredis` test setup | `tests/conftest.py` fixture |
| `test_rate_limiter.py` | Unit tests: under-limit, at-limit, over-limit, concurrent, window expiry |

**Deliverable:** Rate limiting enforced per-tenant with configurable windows.
**Verification:** Send 61 requests in 1 minute with limit=60 → 61st returns 429 with `Retry-After: 60`. Concurrent test (100 parallel) → exactly 60 succeed, 40 rejected.

**Day 4 — Budget enforcement**

| Task | Detail |
|------|--------|
| `gateway/db/repositories/budget_repo.py` | Budget usage CRUD + monthly archive |
| `gateway/middleware/budget_enforcer.py` | Pre-check + post-deduct + 80% warning |
| `gateway/tracking/budget_store.py` | Redis counter + PG sync |
| APScheduler cron | Reset budget on 1st of month |
| `test_budget_enforcer.py` | Under-budget, at-80%-warning, over-budget rejection, monthly reset |

**Deliverable:** Budget enforcement with Redis counter + PG sync + warning.
**Verification:** Tenant with 1000-token budget: send requests until 801 tokens used → response has `X-Budget-Warning` header. Continue until 1001 → 429 Budget Exceeded.

---

### Phase 3: Semantic Cache (Days 5–6)

**Day 5 — Embedding client + Redis vector index**

| Task | Detail |
|------|--------|
| `gateway/cache/embedding_client.py` | OpenAI `text-embedding-3-small` wrapper with batching + retry |
| Redis RediSearch index creation | HNSW index on 1536-dim vectors, cosine distance |
| `gateway/cache/semantic_cache.py` | `lookup()`, `store()`, `should_cache()`, `invalidate_tenant()` |
| `test_semantic_cache.py` | Store + retrieve exact match, similar match (sim > threshold), dissimilar (sim < threshold), TTL expiry, tenant isolation |

**Deliverable:** Semantic cache stores and retrieves by similarity.
**Verification:** Store response for "How do I reset my password?" → lookup "How can I reset my password?" (similar) → HIT with similarity > 0.95. Lookup "What's the weather?" → MISS.

**Day 6 — Cache integration + benchmarking**

| Task | Detail |
|------|--------|
| Wire cache into `/v1/chat/completions` | Cache lookup before routing; cache write after response |
| `scripts/benchmark_cache.py` | Run 100 Q&A pairs (50 unique, 50 paraphrased) → measure hit rate |
| Cache stats endpoint | `GET /admin/cache/stats` — hit count, miss count, hit rate, cache size |
| `should_cache()` logic | Bypass for temperature > 0, seed set, streaming |

**Deliverable:** End-to-end cache hit/miss flow with benchmark.
**Verification:** Benchmark shows ≥ 50% hit rate on paraphrased questions. `X-Cache: HIT` header on cached responses. Cache stats endpoint returns correct counts.

---

### Phase 4: Model Routing + Provider Failover (Days 7–8)

**Day 7 — Provider adapters + failover chain**

| Task | Detail |
|------|--------|
| `gateway/providers/base.py` | `LLMProvider` abstract class: `chat()`, `supports_model()` |
| `gateway/providers/openai_provider.py` | OpenAI adapter (chat completions, token counting from response) |
| `gateway/providers/anthropic_provider.py` | Anthropic adapter (messages API, token counting) |
| `gateway/providers/ollama_provider.py` | Ollama REST adapter (local fallback) |
| `gateway/providers/failover.py` | FailoverChain + CircuitBreaker (Redis-backed) |
| `test_failover.py` | Primary succeeds, primary 5xx → secondary, all fail → 503 |
| `test_circuit_breaker.py` | 5 failures → open → skip → cooldown → half-open → success → closed |

**Deliverable:** Multi-provider failover with circuit breaker.
**Verification:** Mock OpenAI returning 500 → request succeeds via Anthropic. Mock 5 consecutive OpenAI failures → circuit opens → subsequent requests skip OpenAI directly to Anthropic. After 30s cooldown → half-open trial.

**Day 8 — Model routing engine**

| Task | Detail |
|------|--------|
| `gateway/routing/classifier.py` | Heuristic task classifier (length + keywords) |
| `gateway/routing/confidence.py` | Self-eval + heuristic confidence scoring |
| `gateway/routing/router.py` | Route + execute + escalate logic |
| Wire router into `/v1/chat/completions` | Replace direct provider call with router |
| `test_router.py` | Simple → mini, complex → full, low confidence → escalate |
| `test_confidence.py` | Short response → low score, hedging → low score, good response → high score |

**Deliverable:** Model routing with confidence-based escalation.
**Verification:** "What is 2+2?" → served by gpt-4o-mini. "Analyze the architectural tradeoffs of microservices vs monolith for a 50-engineer team" → served by gpt-4o. Mock mini returning "I don't know" → escalation to gpt-4o, `X-Escalated: true` header.

---

### Phase 5: Cost Tracking + Logging (Day 9)

**Day 9 — Cost tracking + structured logging + dashboard**

| Task | Detail |
|------|--------|
| `gateway/tracking/cost_calculator.py` | Token × pricing → USD, in-memory pricing cache |
| `gateway/tracking/cost_store.py` | PostgreSQL `cost_records` writer (batched) |
| `gateway/middleware/request_logger.py` | structlog JSON output with contextvars |
| `gateway/api/admin.py` | Cost summary endpoint with group_by |
| SQL views | `v_daily_cost_summary`, `v_tenant_monthly_budget` |
| `dashboard/app.py` | Streamlit main page: overview metrics |
| `dashboard/pages/overview.py` | Cache hit rate, total cost, request volume (last 7 days) |
| `dashboard/pages/cost_by_tenant.py` | Bar chart: cost per tenant (last 30 days) |
| `dashboard/pages/model_distribution.py` | Pie chart: requests by model |
| `dashboard/pages/budget_status.py` | Progress bars: budget usage per tenant |
| `dashboard/pages/rate_limits.py` | Current rate window usage per tenant |
| `test_cost_calculator.py` | Known tokens × known pricing → exact USD |

**Deliverable:** Full cost tracking pipeline + live dashboard.
**Verification:** Send 10 requests → `cost_records` has 10 rows with correct USD. Dashboard shows: cache hit rate, cost per tenant, model distribution, budget bars. Cost summary endpoint matches sum of `cost_records`.

---

### Phase 6: Integration + Load Testing + Polish (Day 10)

**Day 10 — End-to-end integration + load test + documentation**

| Task | Detail |
|------|--------|
| `test_chat_endpoint.py` | Full lifecycle: auth → rate limit → budget → cache → route → cost → log |
| `test_cache_hit_miss.py` | Same question twice → second is cache hit |
| `test_budget_rejection.py` | Exhaust budget → 429 |
| `test_provider_failover.py` | Kill OpenAI mock → Anthropic serves |
| `locustfile.py` | 100 concurrent users, 60% repeated questions, 40% unique |
| `README.md` | Quick start, architecture summary, API docs link |
| `Makefile` | `make dev`, `make test`, `make seed`, `make dashboard`, `make benchmark` |
| `.env.example` | All env vars with comments |
| Run load test | 10k requests over 10 minutes; verify cache hit rate, cost savings, no errors |

**Deliverable:** Full test suite passing + load test report.
**Verification:** `pytest` — all tests green. Load test: 10k requests, ~60% cache hit, cost < $5 (vs ~$55 without gateway). Dashboard reflects load test results. No 5xx errors. Rate limiting and budget enforcement hold under concurrent load.

---

## 10. Testing Strategy

### Test Pyramid

```
        ┌───────────────┐
        │  Load (Locust) │   ← 100 concurrent users, 10k requests
        │   1 file       │
        └───────┬───────┘
        ┌───────▼───────┐
        │ Integration    │   ← Full request lifecycle with fakeredis + test PG
        │   4 files      │
        └───────┬───────┘
        ┌───────▼───────┐
        │    Unit        │   ← Each component in isolation, mocked deps
        │   8 files      │
        └───────────────┘
```

### Critical Test Pseudocode

#### test_rate_limiter.py — Concurrent burst doesn't exceed limit

```python
async def test_concurrent_burst_respects_limit(fakeredis, rate_limiter):
    """100 concurrent requests with limit=60 → exactly 60 allowed, 40 rejected."""
    tenant_id = "test-tenant"
    limits = {"minute": 60, "hour": 3600, "day": 86400}

    # Fire 100 requests concurrently
    results = await asyncio.gather(*[
        rate_limiter.check(tenant_id, limits) for _ in range(100)
    ])

    allowed_count = sum(1 for allowed, _ in results if allowed)
    rejected_count = sum(1 for allowed, _ in results if not allowed)

    assert allowed_count == 60    # Exactly the limit
    assert rejected_count == 40   # The rest
    # No burst-through: the Lua script is atomic
```

#### test_semantic_cache.py — Similarity threshold boundary

```python
async def test_cache_hit_on_paraphrased_question(semantic_cache, mock_embedder):
    """A paraphrased question hits the cache if similarity >= threshold."""
    # Store original
    await semantic_cache.store(
        prompt="How do I reset my password?",
        response={"content": "Click 'Forgot Password' on the login page."},
        model_name="gpt-4o-mini",
        model_family="openai",
        tenant_id="tenant-1",
    )

    # Lookup with a paraphrase (mock embedder returns similar vectors)
    mock_embedder.set_similarity(0.96)  # Above 0.95 threshold
    entry = await semantic_cache.lookup(
        prompt="How can I reset my password?",
        model_family="openai",
        tenant_id="tenant-1",
    )

    assert entry is not None
    assert entry.similarity >= 0.95
    assert entry.response["content"] == "Click 'Forgot Password' on the login page."


async def test_cache_miss_on_unrelated_question(semantic_cache, mock_embedder):
    """An unrelated question misses the cache."""
    await semantic_cache.store(
        prompt="How do I reset my password?",
        response={"content": "Click 'Forgot Password'."},
        model_name="gpt-4o-mini",
        model_family="openai",
        tenant_id="tenant-1",
    )

    mock_embedder.set_similarity(0.45)  # Below threshold
    entry = await semantic_cache.lookup(
        prompt="What is the capital of France?",
        model_family="openai",
        tenant_id="tenant-1",
    )

    assert entry is None  # Cache miss


async def test_cache_tenant_isolation(semantic_cache, mock_embedder):
    """Tenant A cannot hit tenant B's cache entries."""
    await semantic_cache.store(
        prompt="How do I reset my password?",
        response={"content": "Tenant A answer."},
        model_name="gpt-4o-mini",
        model_family="openai",
        tenant_id="tenant-A",
    )

    mock_embedder.set_similarity(0.99)  # Would hit if not tenant-filtered
    entry = await semantic_cache.lookup(
        prompt="How do I reset my password?",
        model_family="openai",
        tenant_id="tenant-B",  # Different tenant
    )

    assert entry is None  # Tenant isolation enforced
```

#### test_budget_enforcer.py — 80% warning and budget exhaustion

```python
async def test_80_percent_warning_emitted_once(fakeredis, budget_enforcer):
    """At 80% budget usage, a warning is emitted exactly once."""
    tenant_id = "test-tenant"
    monthly_limit = 1000

    # Use 790 tokens (79%)
    await budget_enforcer.deduct_tokens(
        tenant_id, tokens_used=790, cost_usd=0.5, monthly_limit=monthly_limit
    )

    # Next request at 79% → no warning yet
    ok, pct, warning = await budget_enforcer.check_budget(
        tenant_id, monthly_limit, estimated_tokens=20
    )
    assert ok is True
    assert warning is None  # Not yet at 80%

    # Deduct 20 more → 810 tokens (81%) → next check should warn
    await budget_enforcer.deduct_tokens(
        tenant_id, tokens_used=20, cost_usd=0.1, monthly_limit=monthly_limit
    )

    ok, pct, warning = await budget_enforcer.check_budget(
        tenant_id, monthly_limit, estimated_tokens=10
    )
    assert ok is True
    assert pct >= 80.0
    assert warning is not None
    assert "80%" in warning

    # Second check at 81% → warning NOT repeated
    ok, pct, warning = await budget_enforcer.check_budget(
        tenant_id, monthly_limit, estimated_tokens=10
    )
    assert warning is None  # Already warned this month


async def test_budget_exceeded_rejects_request(fakeredis, budget_enforcer):
    """When budget is exhausted, requests are rejected."""
    tenant_id = "test-tenant"
    monthly_limit = 100

    # Exhaust the budget
    await budget_enforcer.deduct_tokens(
        tenant_id, tokens_used=100, cost_usd=0.5, monthly_limit=monthly_limit
    )

    ok, pct, warning = await budget_enforcer.check_budget(
        tenant_id, monthly_limit, estimated_tokens=10
    )
    assert ok is False
    assert pct == 100.0
```

#### test_failover.py — Provider failover on 5xx

```python
async def test_failover_on_primary_5xx(failover_chain, mock_providers):
    """If OpenAI returns 500, the request succeeds via Anthropic."""
    mock_providers["openai"].set_response(
        status=500, error=Exception("Internal Server Error")
    )
    mock_providers["anthropic"].set_response(
        content="Answer from Anthropic",
        input_tokens=50,
        output_tokens=100,
    )

    response = await failover_chain.execute(
        model="claude-3-5-sonnet",
        request={"messages": [{"role": "user", "content": "Hello"}]},
        tenant_id="test-tenant",
    )

    assert response.content == "Answer from Anthropic"
    assert response.provider == "anthropic"


async def test_all_providers_fail_returns_503(failover_chain, mock_providers):
    """If all providers fail, the gateway returns 503."""
    for provider in mock_providers.values():
        provider.set_response(status=500, error=Exception("Down"))

    with pytest.raises(AllProvidersFailedError):
        await failover_chain.execute(
            model="gpt-4o",
            request={"messages": [{"role": "user", "content": "Hello"}]},
            tenant_id="test-tenant",
        )
```

#### test_circuit_breaker.py — Opens after N failures, closes after cooldown

```python
async def test_circuit_opens_after_threshold(fakeredis, circuit_breaker):
    """5 failures in 60s → circuit opens → provider is skipped."""
    provider = "openai"

    for _ in range(5):
        await circuit_breaker.record_failure(provider)

    assert await circuit_breaker.is_open(provider) is True


async def test_circuit_half_open_after_cooldown(fakeredis, circuit_breaker):
    """After cooldown, circuit moves to half-open (allows one trial)."""
    provider = "openai"
    circuit_breaker._cooldown = 1  # 1 second for test

    for _ in range(5):
        await circuit_breaker.record_failure(provider)

    assert await circuit_breaker.is_open(provider) is True

    await asyncio.sleep(1.1)  # Wait for cooldown

    # Now half-open → allows a trial request
    assert await circuit_breaker.is_open(provider) is False

    # Success closes the circuit
    await circuit_breaker.record_success(provider)
    assert await circuit_breaker.is_open(provider) is False
```

#### test_chat_endpoint.py — Full lifecycle integration test

```python
async def test_full_lifecycle_cache_miss_then_hit(
    client, test_tenant, seed_pricing
):
    """Same question twice: first MISS, second HIT."""
    headers = {"Authorization": f"Bearer {test_tenant.api_key}"}
    body = {
        "model": "auto",
        "messages": [{"role": "user", "content": "What is the capital of France?"}],
        "temperature": 0,
    }

    # First request → cache miss, calls LLM
    resp1 = await client.post("/v1/chat/completions", json=body, headers=headers)
    assert resp1.status_code == 200
    assert resp1.headers["X-Cache"] == "MISS"
    assert float(resp1.headers["X-Cost-USD"]) > 0

    # Second request → cache hit, no LLM call
    resp2 = await client.post("/v1/chat/completions", json=body, headers=headers)
    assert resp2.status_code == 200
    assert resp2.headers["X-Cache"] == "HIT"
    assert float(resp2.headers["X-Cost-USD"]) == 0.0  # No cost on cache hit
```

---

## 11. Security & Safety Considerations

### API Key Security

| Concern | Mitigation |
|---------|-----------|
| API keys in transit | HTTPS only (TLS termination at reverse proxy / load balancer) |
| API key storage | Store only SHA-256 hash in PostgreSQL; never log raw keys |
| API key generation | `secrets.token_urlsafe(32)` — 256 bits of entropy |
| Key prefix storage | First 8 chars stored for identification without exposing the key |
| Key rotation | Admin endpoint to issue new key + revoke old; old key immediately invalid |

### Tenant Isolation

| Concern | Mitigation |
|---------|-----------|
| Cache cross-contamination | Redis vector index filtered by `tenant_id` in every search query |
| Budget cross-deduction | Budget keys namespaced: `budget:{tenant_id}:{month}` |
| Rate limit cross-counting | Rate keys namespaced: `rate:{tenant_id}:{window}` |
| Cost record leakage | All admin endpoints scoped by `tenant_id`; dashboard queries filter by tenant |

### LLM-Specific Safety

| Concern | Mitigation |
|---------|-----------|
| Prompt injection in cached responses | Cache stores LLM responses, not user prompts; injection risk is in the prompt itself, not the cache. Validate cached responses haven't been tampered (integrity hash). |
| Cache poisoning | Cache writes only happen after a successful LLM response; no user-controlled cache writes. Admin can flush tenant cache. |
| Cost injection (token bombing) | `max_tokens` enforced per request (configurable per tenant); budget enforcement stops runaway costs. |
| Provider API key leakage | Provider keys (OpenAI, Anthropic) in environment variables only; never in the database; never in logs; never in response headers. |
| PII in cached prompts | Cached prompts are embeddings + hashed keys, not plaintext. The original prompt is NOT stored in Redis (only the embedding + response). Admin can configure `store_prompt_hash` but not plaintext. |
| Rate limit bypass via concurrent requests | Lua script is atomic; no race condition window. |
| Budget race condition | Redis `INCRBY` is atomic; budget deduction is post-request (optimistic). Pre-check is advisory; the post-deduct is authoritative. |

### Operational Safety

| Concern | Mitigation |
|---------|-----------|
| Redis failure | Gateway degrades gracefully: if Redis is down, skip cache (all misses), use in-memory rate limit fallback (per-process), reject budget checks (fail-closed for budget, fail-open for cache). |
| PostgreSQL failure | Cost records buffered in Redis list; background worker retries writes when PG recovers. Budget checks fail-open (allow request, log alert). |
| Provider key exhaustion | Circuit breaker prevents cascading; admin alerted on circuit open. |
| Runaway costs | Hard budget limit per tenant; global cost alert (admin webhook if daily spend > threshold). |
| Cache stampede | If many similar requests miss simultaneously, they all call the LLM. Mitigation: single-flight (if a prompt is in-flight, wait for its response instead of calling again). v2 concern. |

---

## 12. Deployment

### docker-compose.yml

```yaml
version: "3.9"

services:
  gateway:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    env_file: .env
    depends_on:
      redis:
        condition: service_healthy
      postgres:
        condition: service_healthy
    volumes:
      - ./gateway:/app/gateway  # Hot reload in dev
    command: uvicorn gateway.main:app --host 0.0.0.0 --port 8000 --reload
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 10s
      timeout: 5s
      retries: 3

  dashboard:
    build:
      context: .
      dockerfile: Dockerfile.dashboard
    ports:
      - "8501:8501"
    env_file: .env
    depends_on:
      - postgres
    volumes:
      - ./dashboard:/app/dashboard
    command: streamlit run dashboard/app.py --server.port 8501 --server.address 0.0.0.0

  redis:
    image: redis/redis-stack:7.4.0-v4  # Includes RediSearch for vector search
    ports:
      - "6379:6379"
      - "8001:8001"  # RedisInsight UI (dev only)
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  postgres:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: llm_gateway
      POSTGRES_USER: gateway
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-gateway_dev}
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./gateway/db/migrations:/docker-entrypoint-initdb.d  # Auto-run on first start
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U gateway -d llm_gateway"]
      interval: 5s
      timeout: 3s
      retries: 5

  ollama:
    image: ollama/ollama:latest
    ports:
      - "11434:11434"
    volumes:
      - ollama-data:/root/.ollama
    # For GPU support, add: deploy.resources.reservations.devices
    # This is the last-resort fallback provider

volumes:
  redis-data:
  postgres-data:
  ollama-data:
```

### Dockerfile (Gateway)

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# System deps for asyncpg
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential libpq-dev curl \
    && rm -rf /var/lib/apt/lists/*

# Python deps
COPY pyproject.toml ./
RUN pip install --no-cache-dir -e .

# App code
COPY . .

# Run migrations on startup
CMD ["sh", "-c", "python -m gateway.db.migrations && uvicorn gateway.main:app --host 0.0.0.0 --port 8000"]
```

### Dockerfile.dashboard

```dockerfile
FROM python:3.11-slim

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential libpq-dev \
    && rm -rf /var/lib/apt/lists/*

COPY pyproject.toml ./
RUN pip install --no-cache-dir streamlit pandas plotly asyncpg psycopg2-binary

COPY dashboard/ ./dashboard/
COPY gateway/ ./gateway/

CMD ["streamlit", "run", "dashboard/app.py", "--server.port", "8501", "--server.address", "0.0.0.0"]
```

### .env.example

```bash
# ============================================================
# LLM Gateway Environment Configuration
# ============================================================

# --- Gateway ---
GATEWAY_HOST=0.0.0.0
GATEWAY_PORT=8000
LOG_LEVEL=info
ENVIRONMENT=development   # development | staging | production

# --- Redis ---
REDIS_URL=redis://redis:6379/0
CACHE_INDEX_NAME=semantic_cache:index
CACHE_KEY_PREFIX=semantic_cache
CACHE_SIMILARITY_THRESHOLD=0.95
CACHE_TTL_SECONDS=3600
CACHE_TOP_K=5

# --- PostgreSQL ---
DATABASE_URL=postgresql://gateway:gateway_dev@postgres:5432/llm_gateway
POSTGRES_PASSWORD=gateway_dev

# --- LLM Providers ---
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
OLLAMA_BASE_URL=http://ollama:11434

# --- Provider Failover ---
FAILOVER_ORDER=openai,anthropic,ollama
PROVIDER_TIMEOUT_SECONDS=10
CIRCUIT_BREAKER_FAILURE_THRESHOLD=5
CIRCUIT_BREAKER_COOLDOWN_SECONDS=30

# --- Model Routing ---
ROUTING_MODEL_MAP_SIMPLE=gpt-4o-mini
ROUTING_MODEL_MAP_COMPLEX=gpt-4o
ROUTING_MODEL_MAP_ESCALATE=gpt-4o
ROUTING_CONFIDENCE_CHECK_ENABLED=true
ROUTING_CONFIDENCE_THRESHOLD=7.0
CONFIDENCE_STRATEGY=heuristic   # heuristic | self_eval
CONFIDENCE_MIN_RESPONSE_LENGTH=50

# --- Embeddings ---
EMBEDDING_MODEL=text-embedding-3-small
EMBEDDING_DIMENSIONS=1536

# --- Budget ---
BUDGET_WARNING_THRESHOLD=0.80

# --- Dashboard ---
DASHBOARD_REFRESH_SECONDS=30
```

### Makefile

```makefile
.PHONY: dev test seed dashboard benchmark clean

dev:
	docker compose up --build

test:
	docker compose exec gateway pytest -v

seed:
	docker compose exec gateway python scripts/seed_tenants.py

dashboard:
	docker compose up dashboard

benchmark:
	docker compose exec gateway python scripts/benchmark_cache.py

load-test:
	locust -f tests/load/locustfile.py --headless -u 100 -r 10 -t 10m

clean:
	docker compose down -v
```

---

## 13. Roadmap & Milestones

### v1.0 (This Plan — 10 Working Days)

| Milestone | Day | Deliverable |
|-----------|-----|-------------|
| M1: Scaffold + auth | Day 2 | Gateway running, tenants authenticated |
| M2: Rate limit + budget | Day 4 | Per-tenant limits enforced |
| M3: Semantic cache | Day 6 | ≥ 50% cache hit rate on benchmark |
| M4: Routing + failover | Day 8 | 70%+ requests on cheap model, failover works |
| M5: Cost tracking + dashboard | Day 9 | Live dashboard with cost metrics |
| M6: Integration + load test | Day 10 | 10k requests, 87% cost reduction demonstrated |

### v1.1 (Next Sprint — 1 Week)

- Streaming response support (SSE) with incremental cost tracking
- Single-flight cache (prevent cache stampede on concurrent identical prompts)
- Prometheus metrics endpoint (`/metrics`) for Grafana integration
- OpenTelemetry tracing (distributed trace per request across gateway → provider)

### v2.0 (Next Quarter)

- Learned task classifier (fine-tuned embedding classifier replacing heuristics)
- A/B testing framework for routing strategies (compare cost/quality across configs)
- Multi-region failover (US-East OpenAI → EU Anthropic)
- Prompt template registry (versioned prompt management)
- Per-tenant cache namespace isolation with configurable sharing
- WebSocket streaming for dashboard live updates
- GraphQL admin API

---

## 14. Risk Register

| # | Risk | Likelihood | Impact | Severity | Mitigation | Contingency |
|---|------|-----------|--------|----------|------------|-------------|
| R1 | **Semantic cache returns wrong answer** (false positive: similar prompt, different intent) | Medium | High | **High** | Conservative similarity threshold (0.95 default); tenant-scoped cache; `should_cache()` bypasses non-deterministic requests (temperature > 0) | Admin can flush tenant cache; lower threshold to 0.98; add "cache bypass" header for sensitive use cases |
| R2 | **Redis vector index grows unbounded** (memory exhaustion) | Medium | High | **High** | TTL on every cache entry (default 1h); max cache size configurable; monitor `used_memory` via RedisInsight | Eviction policy: `allkeys-lru` as fallback; scheduled cache flush; scale Redis vertically or shard |
| R3 | **Provider API key leaked** (in logs, error messages, or response headers) | Low | Critical | **High** | Keys in env vars only; never logged; structlog configured to redact `authorization` headers; no key in response headers | Immediate key rotation via provider dashboard; audit log access; switch to secrets manager (AWS Secrets Manager / Vault) |
| R4 | **Budget race condition** (concurrent requests exceed budget before deduction) | Medium | Medium | **Medium** | Redis `INCRBY` is atomic for deduction; pre-check is advisory (optimistic); hard limit enforced post-deduct with next-request rejection | Accept minor overage (≤ a few requests worth); alert admin if monthly overage > 1% of budget |
| R5 | **Circuit breaker stuck open** (provider recovers but breaker never closes) | Low | Medium | **Low** | Half-open state allows trial request after cooldown; success resets to closed; cooldown TTL on Redis key ensures auto-recovery | Manual reset via admin endpoint: `DELETE /admin/circuit/{provider}`; monitoring alert on breaker open > 5 min |
| R6 | **Cost calculation drift** (pricing table outdated after provider price change) | Medium | Medium | **Medium** | Pricing table versioned with `effective_from`/`effective_to`; refresh cache every 5 min; admin endpoint to update pricing | Monthly reconciliation script compares `cost_records` sum against provider invoice; alert if drift > 2% |
| R7 | **Embedding API failure** (OpenAI embeddings down → cache can't function) | Low | Medium | **Low** | Embedding client has retry + exponential backoff; if embeddings fail, skip cache (all misses, direct to LLM) | Fallback to local sentence-transformers model (lower quality but no API dependency); degraded mode flag in dashboard |
| R8 | **PostgreSQL connection pool exhaustion** under high load | Low | High | **Medium** | asyncpg pool with configurable max connections; cost writes batched (not one-per-request); budget sync is fire-and-forget | Connection pooler (PgBouncer); read replicas for dashboard queries; buffer cost records in Redis if PG is down |
| R9 | **Ollama fallback produces low-quality responses** (degraded mode not visible to user) | Medium | Medium | **Medium** | Response header `X-Provider: ollama` indicates fallback; admin dashboard shows provider distribution; log alert on any ollama usage | Configure ollama with the best available local model (e.g., `llama3.1:70b` if GPU permits); add `X-Quality-Warning` header |
| R10 | **Rate limiter Lua script blocked by Redis slowlog** | Very Low | Medium | **Low** | Lua script is O(log N) for ZSET operations; Redis single-threaded but script is sub-ms | Monitor Redis slowlog; switch to `redis-cell` (dedicated rate limiting module) if needed |
| R11 | **Tenant API key brute-force** | Low | High | **Medium** | Keys are 256-bit entropy (`secrets.token_urlsafe(32)`); rate limit on auth failures (5 per minute per IP); key hash stored (not plaintext) | IP-based ban after 10 failed auths; optional IP allowlist per tenant; Web Application Firewall (WAF) in production |
| R12 | **Cache stampede** (many concurrent identical misses all call LLM) | Medium | Medium | **Medium** | v1 accepts this (low probability for identical concurrent prompts); v1.1 adds single-flight pattern | Temporary: increase TTL so repeated misses are less likely; v1.1: `inflight:{prompt_hash}` lock with short TTL |

### Risk Review Cadence

- **Weekly during development:** review R1, R2, R6 (cache quality, memory, pricing)
- **On each incident:** add a new risk row; update mitigation if the existing one failed
- **Monthly in production:** full risk register review; retire risks that haven't materialized in 90 days

---

## 15. Appendix

### A. Quick Start

```bash
# 1. Clone and configure
git clone <repo-url> llm-gateway && cd llm-gateway
cp .env.example .env
# Edit .env: add OPENAI_API_KEY and ANTHROPIC_API_KEY

# 2. Start all services
make dev
# → Gateway on :8000, Dashboard on :8501, Redis on :6379, PG on :5432

# 3. Create a test tenant (get API key)
make seed
# → Prints: "Tenant 'acme-corp' API key: gw_abc123..."

# 4. Make a request
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Authorization: Bearer gw_abc123..." \
  -H "Content-Type: application/json" \
  -d '{"model":"auto","messages":[{"role":"user","content":"What is 2+2?"}],"temperature":0}'

# 5. Check the dashboard
open http://localhost:8501
# → See cache hit rate, cost, model distribution

# 6. Run the cache benchmark
make benchmark
# → Prints: "Cache hit rate: 58%, estimated savings: $1,400/month"

# 7. Run tests
make test
```

### B. Key Design Decisions

| Decision | Choice | Rationale | Alternative Rejected |
|----------|--------|-----------|---------------------|
| Cache storage | Redis with RediSearch (HNSW vector index) | Sub-ms vector search; TTL native; already needed for rate limiting → no new infra | Qdrant (separate service, more memory); pgvector (slower at high QPS) |
| Embedding model | OpenAI `text-embedding-3-small` | $0.02/M tokens (cheapest production option); 1536-dim good quality; no infra to manage | Local sentence-transformers (CPU cost, lower throughput, but no API dependency — used as fallback only) |
| Similarity metric | Cosine similarity via Redis vector distance | Standard for semantic similarity; Redis HNSW supports it natively; threshold 0.95 is conservative | Euclidean distance (less intuitive threshold); dot product (needs normalization) |
| Rate limit algorithm | Sliding window via Redis ZSET + Lua | True sliding window (not fixed window); atomic via Lua; handles bursts at boundaries correctly | Token bucket (more complex, allows bursts); fixed window (burst-at-boundary problem) |
| Budget enforcement | Optimistic (pre-check advisory, post-deduct authoritative) | Pre-check is fast (Redis GET); post-deduct is atomic (Redis INCRBY); minor overage acceptable | Pessimistic (lock + check + deduct) — adds latency and complexity for marginal accuracy gain |
| Provider abstraction | Thin adapter classes (3 providers) | Full control over provider-specific features; easy to debug; no framework lock-in | LiteLLM (hides provider features, adds dependency, harder to debug provider-specific issues) |
| Confidence checking | Heuristic by default, self-eval optional | Heuristic is free (no extra LLM call); catches obvious low-quality responses; self-eval costs 1 extra call but is more accurate | Always self-eval (doubles cost for every cheap-model call — defeats the purpose) |
| Dashboard | Streamlit | Data engineer's native tool (SQL + pandas); fastest path to internal dashboard; no frontend build | Grafana (ops-focused, less SQL-native); custom React app (overkill for internal dashboard) |
| Cost storage | PostgreSQL `cost_records` (one row per LLM call) | ACID guarantees for financial data; SQL aggregation for reporting; joins with tenants for per-tenant analysis | DynamoDB (no SQL aggregation); TimescaleDB (premature optimization — 10k rows/day is trivial for PG) |
| Fallback order | OpenAI → Anthropic → Ollama | OpenAI = primary (best quality); Anthropic = secondary (different failure domain); Ollama = last resort (local, always available if host is up) | Random order (no quality preference); round-robin (defeats cost optimization) |
| Circuit breaker | Redis-backed (distributed state) | All gateway instances share circuit state; if one instance detects OpenAI is down, all skip it | In-memory per-instance (each instance independently discovers failure — wastes timeouts) |
| Logging | structlog (JSON) | Structured logs ship to ELK/Datadog without parsing; contextvars bind per-request context automatically | stdlib logging (requires custom formatter for JSON); loguru (less ecosystem integration for structured output) |

### C. Cost Model Reference (Pricing as of 2025-08)

| Provider | Model | Input ($/M tokens) | Output ($/M tokens) | Use in Gateway |
|----------|-------|--------------------|---------------------|----------------|
| OpenAI | gpt-4o | $5.00 | $15.00 | Complex tasks, escalation target |
| OpenAI | gpt-4o-mini | $0.15 | $0.60 | Simple tasks (primary cheap tier) |
| OpenAI | text-embedding-3-small | $0.02 | — | Semantic cache embeddings |
| Anthropic | claude-3-5-sonnet | $3.00 | $15.00 | Secondary provider (failover) |
| Anthropic | claude-3-5-haiku | $0.80 | $4.00 | Secondary cheap tier (failover) |
| Ollama | llama3.1:8b | $0.00 | $0.00 | Last-resort fallback (self-hosted) |

> **Note:** Provider pricing changes frequently. Update `model_pricing` table via admin API when providers announce price changes. The cost calculator refreshes its in-memory cache every 5 minutes.

### D. Glossary

| Term | Definition |
|------|-----------|
| **Semantic cache** | A cache where the key is the embedding (vector) of the prompt, and lookup is by cosine similarity rather than exact string match |
| **Cosine similarity** | A measure of similarity between two vectors, ranging from -1 (opposite) to 1 (identical). For text embeddings, 0.95+ typically means semantically equivalent |
| **Model routing** | Selecting which LLM model to use for a request based on task complexity, with optional escalation from cheap to expensive |
| **Confidence escalation** | Evaluating a cheap model's response quality and re-running with an expensive model if confidence is low |
| **Sliding window rate limiting** | A rate limit algorithm that counts requests in a window that slides with time (vs fixed windows that reset at boundaries) |
| **Circuit breaker** | A pattern that stops sending requests to a failing service after N failures, allowing it to recover, then probes (half-open) before fully restoring |
| **Token budget** | A monthly limit on the number of LLM tokens a tenant can consume, enforced by the gateway |
| **Failover chain** | An ordered list of LLM providers; if the primary fails, the gateway tries the next, and so on |
| **HNSW** | Hierarchical Navigable Small World — an approximate nearest neighbor algorithm used by Redis RediSearch for fast vector similarity search |
| **Tenant** | An identified customer (by API key) whose requests are rate-limited and budget-tracked independently |