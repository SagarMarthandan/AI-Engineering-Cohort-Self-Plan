# Agent Orchestration Guide — LLM Serving Gateway

> **Tier 4 — AI Infrastructure & Production · Project 9**
> Herdr multi-agent runbook for building a production-grade LLM proxy with semantic caching, model routing, cost tracking, budgets, rate limiting, and a Streamlit admin dashboard.
>
> **Source of truth:** `IMPLEMENTATION_PLAN.md` — all file paths, schemas, pseudocode, and phase definitions referenced below are from that document.

---

## 1. Agent Roster

| Agent | Kind | Responsibility | Owned Files / Directories |
|-------|------|----------------|---------------------------|
| **billing** | codex | Foundation, DB schema, config, auth, rate limiting, budget enforcement, cost tracking, structured logging, main endpoint wiring, integration tests, load tests, Docker/Makefile | `gateway/config.py`, `gateway/main.py`, `gateway/dependencies.py`, `gateway/db/` (all), `gateway/middleware/` (all), `gateway/tracking/` (all), `gateway/models/` (all), `gateway/api/` (all), `docker-compose.yml`, `Dockerfile`, `pyproject.toml`, `.env.example`, `Makefile`, `scripts/seed_tenants.py`, `scripts/cost_report.py`, `tests/conftest.py`, `tests/unit/test_rate_limiter.py`, `tests/unit/test_budget_enforcer.py`, `tests/unit/test_cost_calculator.py`, `tests/integration/` (all), `tests/load/locustfile.py` |
| **cache** | codex | Semantic cache: embedding client, Redis vector index, cache lookup/store/invalidate, cache benchmarking | `gateway/cache/` (all), `scripts/benchmark_cache.py`, `tests/unit/test_semantic_cache.py` |
| **routing** | codex | Provider adapters, failover chain, circuit breaker, task classifier, confidence scorer, model router | `gateway/providers/` (all), `gateway/routing/` (all), `tests/unit/test_router.py`, `tests/unit/test_confidence.py`, `tests/unit/test_failover.py`, `tests/unit/test_circuit_breaker.py` |
| **dashboard** | codex | Streamlit admin dashboard: overview, cost-by-tenant, model distribution, budget status, rate limits, chart helpers | `dashboard/` (all), `Dockerfile.dashboard` |
| **reviewer** | codex | Code review after each phase: correctness, security, tenant isolation, test coverage, plan adherence | Reviews all files; owns none |

---

## 2. Pane Layout

```
┌─────────────────────────────────────────────────────────────┐
│                      Herdr Pane Layout                       │
├───────────────────────┬─────────────────────────────────────┤
│   pane: billing       │   pane: cache                       │
│   (Billing Agent)     │   (Cache Agent)                     │
│   Foundation, DB,     │   Semantic cache,                   │
│   middleware, cost,   │   embeddings, Redis                 │
│   integration         │   vector index                      │
├───────────────────────┼─────────────────────────────────────┤
│   pane: routing       │   pane: dashboard                   │
│   (Routing Agent)     │   (Dashboard Agent)                 │
│   Providers, failover,│   Streamlit admin                   │
│   classifier, router  │   dashboard pages                   │
├───────────────────────┴─────────────────────────────────────┤
│   pane: reviewer                                            │
│   (Reviewer Agent)                                         │
│   Phase reviews, security audit, plan adherence             │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Setup Commands

```bash
# ── Split panes (all in project root) ──────────────────────────
herdr split --id billing    --cwd "$PWD" --no-focus
herdr split --id cache      --cwd "$PWD" --no-focus
herdr split --id routing    --cwd "$PWD" --no-focus
herdr split --id dashboard  --cwd "$PWD" --no-focus
herdr split --id reviewer   --cwd "$PWD" --no-focus

# ── Start agents ───────────────────────────────────────────────
herdr agent start billing    --kind codex --pane billing
herdr agent start cache      --kind codex --pane cache
herdr agent start routing    --kind codex --pane routing
herdr agent start dashboard  --kind codex --pane dashboard
herdr agent start reviewer   --kind codex --pane reviewer

# ── Verify all agents are running ──────────────────────────────
herdr agent list
```

---

## 4. Phase-by-Phase Orchestration

### Phase 1: Foundation — Scaffold, Docker, DB, Auth (Days 1–2)

> **Plan reference:** Section 9, Phase 1. Section 6 (Project Structure), Section 5 (Database Schema), Section 12 (Deployment).
>
> **Strategy:** billing builds all foundation infrastructure solo. No other agent can start until the project scaffold, Docker Compose, config, DB schema, and auth are in place. This is inherently sequential.

#### Day 1 — Project scaffold + Docker Compose + config

**Sequential (billing only):**

```
billing: Create the full project structure from IMPLEMENTATION_PLAN.md Section 6. Start with these files:

1. pyproject.toml — dependencies: fastapi, uvicorn[standard], redis[asyncio], asyncpg, openai, anthropic, structlog, pydantic-settings, streamlit, plotly, pandas, pytest, pytest-asyncio, fakeredis, httpx, apscheduler, pytest-postgresql.

2. docker-compose.yml — per Section 12. Services: gateway (FastAPI :8000), dashboard (Streamlit :8501), redis (redis/redis-stack:7.4.0-v4 for RediSearch vector search, :6379 + :8001 RedisInsight), postgres (postgres:16-alpine, DB=llm_gateway, user=gateway), ollama (ollama/ollama:latest, :11434). Volumes: redis-data, postgres-data, ollama-data. Healthchecks for redis and postgres. Gateway depends_on redis+postgres with service_healthy condition. Mount ./gateway:/app/gateway for hot reload.

3. gateway/config.py — Pydantic Settings class reading from .env. Include ALL env vars from Section 12 .env.example: GATEWAY_HOST, GATEWAY_PORT, LOG_LEVEL, ENVIRONMENT, REDIS_URL, CACHE_INDEX_NAME, CACHE_KEY_PREFIX, CACHE_SIMILARITY_THRESHOLD (0.95), CACHE_TTL_SECONDS (3600), CACHE_TOP_K (5), DATABASE_URL, OPENAI_API_KEY, ANTHROPIC_API_KEY, OLLAMA_BASE_URL, FAILOVER_ORDER (comma-separated list), PROVIDER_TIMEOUT_SECONDS (10), CIRCUIT_BREAKER_FAILURE_THRESHOLD (5), CIRCUIT_BREAKER_COOLDOWN_SECONDS (30), ROUTING_MODEL_MAP (simple=gpt-4o-mini, complex=gpt-4o, escalate=gpt-4o), ROUTING_CONFIDENCE_CHECK_ENABLED (true), ROUTING_CONFIDENCE_THRESHOLD (7.0), CONFIDENCE_STRATEGY (heuristic), CONFIDENCE_MIN_RESPONSE_LENGTH (50), EMBEDDING_MODEL (text-embedding-3-small), EMBEDDING_DIMENSIONS (1536), BUDGET_WARNING_THRESHOLD (0.80), DASHBOARD_REFRESH_SECONDS (30).

4. .env.example — copy all env vars from Section 12 .env.example verbatim.

5. gateway/__init__.py, gateway/api/__init__.py, gateway/api/v1/__init__.py, gateway/cache/__init__.py, gateway/routing/__init__.py, gateway/providers/__init__.py, gateway/middleware/__init__.py, gateway/tracking/__init__.py, gateway/models/__init__.py, gateway/db/__init__.py, dashboard/__init__.py, tests/__init__.py — all empty init files.

6. gateway/main.py — FastAPI app factory with lifespan handler that initializes Redis connection pool (redis.asyncio) and asyncpg connection pool. Mount health endpoints: GET /health → {"status": "ok"}, GET /ready → checks Redis ping + PG connection + at least one provider reachable. Include CORS middleware. Read settings from gateway.config.Settings.

7. Dockerfile — per Section 12: FROM python:3.11-slim, install build-essential libpq-dev curl, COPY pyproject.toml, pip install -e ., COPY . ., CMD runs migrations then uvicorn.

8. gateway/api/v1/health.py — GET /health and GET /ready endpoints per Section 8.2.

Deliverable: docker-compose up starts all services; GET /health returns 200.
```

#### Day 2 — Database schema + migrations + tenant auth

**Sequential (billing only, continues from Day 1):**

```
billing: Implement the database layer and tenant authentication. Reference IMPLEMENTATION_PLAN.md Section 5 (Database & Schema Design) and Section 7.10 (main flow, auth step).

1. gateway/db/migrations/001_initial.sql — Create ALL tables from Section 5 verbatim:
   - tenants (tenant_id UUID PK, tenant_name, api_key_hash UNIQUE, api_key_prefix, monthly_token_budget BIGINT default 1000000, rate_limit_per_minute INT default 60, rate_limit_per_hour INT default 3600, rate_limit_per_day INT default 86400, is_active BOOLEAN default TRUE, created_at, updated_at)
   - model_pricing (pricing_id SERIAL PK, provider, model_name, input_cost_per_million NUMERIC(10,4), output_cost_per_million NUMERIC(10,4), effective_from, effective_to, UNIQUE(provider, model_name, effective_from))
   - cost_records (record_id BIGSERIAL PK, tenant_id UUID FK, request_id UUID, provider, model_name, input_tokens INT, output_tokens INT, total_tokens INT, cost_usd NUMERIC(10,6), cache_hit BOOLEAN, cache_key VARCHAR(64), task_type VARCHAR(20) default 'simple', escalated BOOLEAN, latency_ms INT, endpoint VARCHAR(100), created_at TIMESTAMPTZ)
   - budget_usage (tenant_id UUID FK, budget_month DATE, tokens_used BIGINT, estimated_cost_usd NUMERIC(12,2), requests_count INT, last_updated TIMESTAMPTZ, PK(tenant_id, budget_month))
   - rate_limit_events (event_id BIGSERIAL PK, tenant_id UUID FK, window VARCHAR(10), limit_value INT, attempted_at TIMESTAMPTZ)
   - All indexes from Section 5 (idx_tenants_api_key_hash, idx_tenants_active, idx_cost_tenant_date, idx_cost_model_date, idx_cost_endpoint_date, idx_cost_created_at, idx_rate_limit_tenant_date)
   - Views: v_daily_cost_summary, v_tenant_monthly_budget (exact SQL from Section 5)

2. gateway/db/migrations/002_seed_pricing.sql — INSERT model pricing rows from Section 5: openai/gpt-4o (5.00, 15.00), openai/gpt-4o-mini (0.15, 0.60), anthropic/claude-3-5-sonnet (3.00, 15.00), anthropic/claude-3-5-haiku (0.80, 4.00), ollama/llama3.1:8b (0.00, 0.00).

3. gateway/db/session.py — asyncpg connection pool, lifespan-managed. Create pool on startup, close on shutdown. Provide get_pool() dependency.

4. gateway/db/repositories/tenant_repo.py — TenantRepository class: create_tenant (generates API key via secrets.token_urlsafe(32), stores SHA-256 hash + first 8 chars prefix), get_by_api_key_hash, get_by_id, list_tenants, update_tenant, deactivate_tenant.

5. gateway/models/tenant.py — Pydantic schemas: TenantCreate, TenantResponse (includes api_key only on creation), TenantDetail (includes budget_usage).

6. gateway/middleware/auth.py — API key → tenant resolution. Extract Bearer token from Authorization header, SHA-256 hash it, lookup in tenants table by api_key_hash. Return tenant object or raise 401. Implement as FastAPI dependency.

7. gateway/api/admin.py — Tenant CRUD endpoints per Section 8.3: POST /admin/tenants (create, returns API key once), GET /admin/tenants (list), GET /admin/tenants/{id} (detail + budget), PATCH /admin/tenants/{id} (update budget/rate limits), DELETE /admin/tenants/{id} (deactivate).

8. scripts/seed_tenants.py — Create 3 test tenants with different budgets: 'acme-corp' (1M tokens), 'startup-io' (500K tokens), 'enterprise-ltd' (5M tokens). Print API keys.

9. gateway/dependencies.py — Start building the dependency injection module. Add get_tenant (auth dependency), get_redis (Redis client), get_db_pool (asyncpg pool). Other dependencies will be added in later phases.

Deliverable: Can create tenants, get API keys, authenticate requests.
Verification: POST /admin/tenants returns API key; request with valid key → 200; invalid key → 401.
```

#### Review

```
reviewer: Review Phase 1 deliverables against IMPLEMENTATION_PLAN.md:
1. Verify docker-compose.yml matches Section 12 exactly (services, ports, healthchecks, volumes, depends_on conditions).
2. Verify all 5 tables in 001_initial.sql match Section 5 schema (column names, types, defaults, constraints, indexes).
3. Verify 002_seed_pricing.sql has all 5 pricing rows with correct costs.
4. Verify gateway/config.py has ALL env vars from Section 12 .env.example with correct defaults.
5. Verify auth middleware: API key is SHA-256 hashed, never logged, key prefix stored for identification.
6. Verify tenant isolation: each tenant has independent api_key_hash, budget, rate limits.
7. Run: docker compose up → curl localhost:8000/health → {"status":"ok"}; redis-cli ping → PONG; psql connects.
8. Run: POST /admin/tenants → returns API key; auth with valid key → 200; invalid key → 401.
Report any deviations from the plan.
```

#### Commit

```bash
git add -A && git commit -m "Phase 1: Foundation — scaffold, Docker Compose, DB schema, tenant auth"
```

---

### Phase 2: Rate Limiting + Budget Enforcement (Days 3–4)

> **Plan reference:** Section 9, Phase 2. Section 7.5 (Rate Limiter), Section 7.6 (Budget Enforcer).
>
> **Strategy:** billing works solo on rate limiting and budget enforcement. These are middleware components that depend on the Phase 1 foundation (Redis, PG, tenant auth). No other agent has work in this phase.

#### Day 3 — Rate limiter (Redis sliding window)

**Sequential (billing only):**

```
billing: Implement the sliding-window rate limiter. Reference IMPLEMENTATION_PLAN.md Section 7.5 (full pseudocode).

1. gateway/middleware/rate_limiter.py — Implement RateLimiter class exactly per Section 7.5:
   - LUA_SLIDING_WINDOW script: ZREMRANGEBYSCORE to prune, ZCARD to count, if count >= limit return 0, ZADD new member with score=timestamp, EXPIRE key to window size, return 1. The script MUST be atomic (single Lua execution).
   - __init__(redis_client): register the Lua script via redis_client.register_script().
   - check(tenant_id, limits: dict[str, int]) → (bool, Optional[str]): iterate over windows (minute=60s, hour=3600s, day=86400s), run Lua script for each, return (False, window_name) on first rejection, (True, None) if all pass.
   - current_usage(tenant_id) → dict[str, int]: prune + ZCARD for each window, return counts (for dashboard).
   - Redis key pattern: rate:{tenant_id}:{window_name}

2. gateway/dependencies.py — Add get_rate_limiter dependency that returns the RateLimiter instance.

3. gateway/api/v1/chat.py — Create the initial chat endpoint skeleton per Section 7.10. For now, wire in: (1) auth via get_tenant, (2) rate limit check via get_rate_limiter. If rate limited, return 429 with Retry-After header and log the event. The rest of the flow (cache, routing, cost) will be wired in later phases. Include the request_id generation and bind_request_context call.

4. tests/conftest.py — Create test fixtures: fakeredis (async), pytest-postgresql fixture, mock tenant object, test client (httpx AsyncClient). Include a test_tenant fixture with known API key and budget settings.

5. tests/unit/test_rate_limiter.py — Tests per Section 10 pseudocode:
   - test_concurrent_burst_respects_limit: 100 concurrent requests with limit=60 → exactly 60 allowed, 40 rejected (the Lua script is atomic, no burst-through).
   - test_under_limit_allowed: 59 requests → all allowed.
   - test_at_limit_rejected: 60th request at limit → rejected.
   - test_window_expiry: requests in window, wait for expiry, new requests allowed.
   - test_multiple_windows: minute limit hit but hour/day not → rejected on minute only.

Deliverable: Rate limiting enforced per-tenant with configurable windows.
Verification: Send 61 requests in 1 minute with limit=60 → 61st returns 429 with Retry-After: 60.
```

#### Day 4 — Budget enforcement

**Sequential (billing only, continues from Day 3):**

```
billing: Implement per-tenant monthly token budget enforcement. Reference IMPLEMENTATION_PLAN.md Section 7.6 (full pseudocode).

1. gateway/db/repositories/budget_repo.py — BudgetRepository class:
   - update_budget_usage(tenant_id, tokens_added, cost_added, requests_added): UPSERT into budget_usage (tenant_id, budget_month) with atomic increment.
   - get_budget_usage(tenant_id): current month's usage.
   - archive_month_and_create_new(tenant_id): called on monthly reset — archive current month, create new row.
   - get_all_budget_status(): for admin endpoint — all tenants' budget usage (used by v_tenant_monthly_budget view).

2. gateway/middleware/budget_enforcer.py — Implement BudgetEnforcer class exactly per Section 7.6:
   - WARNING_THRESHOLD = 0.80
   - _budget_key(tenant_id): f"budget:{tenant_id}:{YYYY-MM}"
   - _warned_key(tenant_id): f"budget_warned:{tenant_id}:{YYYY-MM}"
   - check_budget(tenant_id, monthly_limit, estimated_tokens) → (allowed, usage_pct, warning): Redis GET current count, check if current + estimated > limit → reject. Check 80% warning (only emit once per month via budget_warned key with 35-day TTL).
   - deduct_tokens(tenant_id, tokens_used, cost_usd, monthly_limit) → new_total: Redis INCRBY (atomic), set 35-day TTL if key is new, async sync to PG via budget_repo.update_budget_usage.
   - reset_monthly_budget(tenant_id): called by APScheduler on 1st of month — archive_month_and_create_new.

3. gateway/tracking/budget_store.py — Thin wrapper that coordinates Redis counter + PG sync. May be absorbed into BudgetEnforcer if the plan's design has the enforcer doing both directly (Section 7.6 shows the enforcer doing Redis + PG directly — follow that pattern).

4. APScheduler setup in gateway/main.py lifespan: schedule monthly budget reset cron job (runs at midnight on 1st of each month) that calls budget_enforcer.reset_monthly_budget for all active tenants.

5. gateway/dependencies.py — Add get_budget_enforcer dependency.

6. Wire budget check into gateway/api/v1/chat.py: After rate limit check (step 2 in Section 7.10 flow), add budget pre-check (step 3). If budget exceeded → 429 with "Monthly token budget exceeded" detail. If warning → add X-Budget-Warning response header.

7. tests/unit/test_budget_enforcer.py — Tests per Section 10 pseudocode:
   - test_80_percent_warning_emitted_once: Use 790 tokens (79%) → no warning. Deduct 20 more (81%) → warning emitted. Second check → warning NOT repeated (budget_warned key prevents duplicate).
   - test_budget_exceeded_rejects_request: Exhaust 100-token budget → next check returns (False, 100.0, None).
   - test_monthly_reset: After reset_monthly_budget, budget counter starts fresh.

Deliverable: Budget enforcement with Redis counter + PG sync + 80% warning.
Verification: Tenant with 1000-token budget: use 801 tokens → X-Budget-Warning header. Continue to 1001 → 429 Budget Exceeded.
```

#### Review

```
reviewer: Review Phase 2 deliverables against IMPLEMENTATION_PLAN.md:
1. Verify LUA_SLIDING_WINDOW script in rate_limiter.py matches Section 7.5 exactly (ZREMRANGEBYSCORE, ZCARD, ZADD, EXPIRE, atomic return).
2. Verify rate limiter handles concurrent requests atomically — run the concurrent burst test (100 parallel, limit=60 → exactly 60 allowed).
3. Verify budget_enforcer.py matches Section 7.6: check_budget (pre-check with estimated tokens), deduct_tokens (post-deduct with actual tokens), 80% warning emitted once (budget_warned key with 35-day TTL).
4. Verify Redis key patterns: rate:{tenant_id}:{window}, budget:{tenant_id}:{YYYY-MM}, budget_warned:{tenant_id}:{YYYY-MM}.
5. Verify budget deduction uses Redis INCRBY (atomic) and syncs to PG asynchronously.
6. Verify APScheduler cron is configured for monthly budget reset.
7. Verify chat.py flow order: auth → rate limit → budget (per Section 7.10 steps 1-3).
8. Run all unit tests: pytest tests/unit/test_rate_limiter.py tests/unit/test_budget_enforcer.py -v
Report any deviations.
```

#### Commit

```bash
git add -A && git commit -m "Phase 2: Rate limiting (Redis sliding window) + budget enforcement with 80% warning"
```

---

### Phase 3: Semantic Cache (Days 5–6)

> **Plan reference:** Section 9, Phase 3. Section 7.1 (Semantic Cache), Section 3 (Semantic Cache Architecture detail).
>
> **Strategy:** cache agent builds the semantic cache components in parallel. billing wires the cache into the chat endpoint and dependencies sequentially after cache components are ready.

#### Day 5 — Embedding client + Redis vector index + cache logic

**Parallel Work:**

```
cache: Build the semantic cache. Reference IMPLEMENTATION_PLAN.md Section 7.1 (full pseudocode for SemanticCache class) and Section 3 (Semantic Cache Architecture detail).

1. gateway/cache/embedding_client.py — EmbeddingClient class:
   - Wraps OpenAI text-embedding-3-small API (1536-dim vectors).
   - async embed(text: str) → List[float]: call OpenAI embeddings API with model=text-embedding-3-small, return the embedding vector.
   - Include retry with exponential backoff (3 retries, 1s/2s/4s).
   - If embedding API fails, raise an exception that the cache layer catches to degrade gracefully (skip cache, all misses).

2. gateway/cache/semantic_cache.py — Implement SemanticCache class exactly per Section 7.1:
   - VECTOR_FIELD = "embedding", PAYLOAD_FIELD = "response", METADATA_FIELDS = ["model_name", "tenant_id", "created_at"]
   - __init__(redis_client, embedding_client, settings): store refs, read config (cache_index_name, cache_key_prefix, cache_similarity_threshold=0.95, cache_ttl_seconds=3600, cache_top_k=5).
   - async lookup(prompt, model_family, tenant_id) → Optional[CacheEntry]:
     Step 1: embed prompt via embedding_client.
     Step 2: Build RediSearch vector query — filter by @model_family:{...} @tenant_id:{...}, add_vector_param on "embedding" field, return_fields includes metadata + payload + __vector_score, limit(0, top_k), dialect(2).
     Step 3: Execute search via redis.ft(index_name).search(query).
     Step 4: If no results → None. Get best doc, convert Redis distance to cosine_similarity (1.0 - distance).
     Step 5: If cosine_similarity < threshold → None (miss). Else deserialize response JSON, return CacheEntry(cache_key, response, model_name, tenant_id, created_at, similarity).
   - async store(prompt, response, model_name, model_family, tenant_id) → cache_key:
     Embed prompt, compute sha256(prompt)[:16] hash, build cache_key = f"{prefix}:{hash}".
     Create entry dict: embedding vector, JSON-dumped response, model_name, model_family, tenant_id, created_at (str(time.time())).
     HSET the entry, EXPIRE to ttl_seconds. Return cache_key.
   - async invalidate_tenant(tenant_id) → int: scan_iter for prefix:*, check tenant_id field, delete matching keys, return count.
   - should_cache(request: dict) → bool: return False if temperature > 0, if "seed" in request, if stream=True. Else True.
   - CacheEntry dataclass: cache_key, response (dict), model_name, tenant_id, created_at (float), similarity (float, only on retrieval).
   - Also implement a method to create the RediSearch HNSW index on startup (1536-dim, cosine distance) if it doesn't exist.

3. tests/unit/test_semantic_cache.py — Tests per Section 10 pseudocode:
   - test_cache_hit_on_paraphrased_question: Store "How do I reset my password?" → lookup "How can I reset my password?" with mock_embedder.set_similarity(0.96) → HIT, similarity >= 0.95, correct response content.
   - test_cache_miss_on_unrelated_question: Store password question → lookup "What is the capital of France?" with similarity 0.45 → MISS (None).
   - test_cache_tenant_isolation: Store for tenant-A → lookup as tenant-B with similarity 0.99 → MISS (tenant filter in query).
   - test_ttl_expiry: Store entry, advance time past TTL, lookup → MISS.
   - test_should_cache_bypass: temperature > 0 → False, seed set → False, stream=True → False, temperature=0 no seed → True.

Use fakeredis with RediSearch support for tests. Use a mock embedder that returns controllable similarity scores.
```

```
billing: While cache agent works on cache components, prepare the integration points.

1. gateway/dependencies.py — Add get_semantic_cache dependency. It should construct SemanticCache with the Redis client, embedding client, and settings. The embedding client needs the OpenAI API key from settings.

2. gateway/api/v1/chat.py — Add the cache lookup step (step 4 in Section 7.10 flow) AFTER rate limit and budget checks:
   - Extract prompt from request_body.messages[-1].content
   - Determine model_family (e.g., "openai" — simplified per Section 7.10)
   - If cache.should_cache(request_body.model_dump()): call cache.lookup(prompt, model_family, tenant_id)
   - On HIT: return cached response immediately with X-Cache: HIT header, log as cache_hit, cost_usd=0. Skip routing, cost calculation, budget deduction.
   - On MISS: continue to routing (not yet implemented — leave a TODO comment placeholder for Phase 4).
   - Add X-Cache: MISS header on non-cached responses.

3. gateway/api/admin.py — Add cache admin endpoints per Section 8.3:
   - GET /admin/cache/stats — hit count, miss count, hit rate, cache size (number of keys matching semantic_cache:*)
   - DELETE /admin/cache/tenant/{id} — flush tenant's cache entries via cache.invalidate_tenant()

Do NOT modify files in gateway/cache/ — that is the cache agent's domain. Only modify dependencies.py, chat.py, and admin.py.
```

#### Day 6 — Cache integration + benchmarking

**Sequential (after Day 5 parallel work):**

```
cache: Build the cache benchmark script and verify integration.

1. scripts/benchmark_cache.py — Benchmark script per Section 9 Phase 3 Day 6:
   - Prepare 100 Q&A pairs: 50 unique questions, 50 paraphrased versions of the first 50.
   - Run all 100 through the gateway endpoint (or directly through SemanticCache).
   - Measure: cache hit count, miss count, hit rate %, average lookup latency (ms).
   - Print summary: "Cache hit rate: X%, estimated savings: $Y/month" (use cost math from Section 2).
   - Use the gateway API endpoint (POST /v1/chat/completions) with a test tenant API key.

2. Verify the cache integration works end-to-end:
   - Start services: docker compose up
   - Seed tenants: make seed
   - Send a request with a simple question → X-Cache: MISS
   - Send the same question again → X-Cache: HIT
   - Send a paraphrased question → X-Cache: HIT (if similarity >= 0.95)
   - Run benchmark: make benchmark → hit rate >= 50% on paraphrased questions

3. Verify GET /admin/cache/stats returns correct counts after benchmark.

Report the benchmark results (hit rate, latency) to the reviewer.
```

#### Review

```
reviewer: Review Phase 3 deliverables against IMPLEMENTATION_PLAN.md:
1. Verify SemanticCache.lookup() matches Section 7.1: embed → RediSearch vector query (filtered by model_family + tenant_id) → threshold check (0.95) → return CacheEntry or None.
2. Verify SemanticCache.store() matches Section 7.1: embed → sha256[:16] key → HSET with vector + payload + metadata → EXPIRE TTL.
3. Verify should_cache() bypasses: temperature > 0, seed set, stream=True.
4. Verify tenant isolation: RediSearch query includes @tenant_id filter — tenant A cannot hit tenant B's cache.
5. Verify cache lookup happens AFTER rate limit + budget check, BEFORE routing (Section 7.10 step 4).
6. Verify cache HIT returns immediately with X-Cache: HIT, cost_usd=0, no LLM call.
7. Verify cache write happens only on MISS, only if should_cache() is true (Section 7.10 step 10).
8. Run benchmark: make benchmark → hit rate >= 50% on paraphrased questions (Goal G1).
9. Verify cache lookup latency < 10ms (Goal G1).
10. Run unit tests: pytest tests/unit/test_semantic_cache.py -v
Report any deviations.
```

#### Commit

```bash
git add -A && git commit -m "Phase 3: Semantic cache — embedding client, Redis vector index, cache hit/miss flow, benchmark"
```

---

### Phase 4: Model Routing + Provider Failover (Days 7–8)

> **Plan reference:** Section 9, Phase 4. Section 7.2 (Router), Section 7.3 (Classifier), Section 7.4 (Confidence), Section 7.7 (Failover + Circuit Breaker), Section 3 (Model Routing Decision Tree, Fallback Chain Diagram).
>
> **Strategy:** routing agent builds all provider adapters, failover chain, circuit breaker, classifier, confidence scorer, and model router in parallel. billing wires the router into the chat endpoint sequentially after routing components are ready.

#### Day 7 — Provider adapters + failover chain + circuit breaker

**Parallel Work:**

```
routing: Build provider adapters and failover infrastructure. Reference IMPLEMENTATION_PLAN.md Section 7.7 (full pseudocode for CircuitBreaker and FailoverChain) and Section 3 (Fallback Chain Diagram).

1. gateway/providers/base.py — Abstract LLMProvider interface:
   - LLMResponse dataclass: content (str), model (str), provider (str), input_tokens (int), output_tokens (int), total_tokens (int), to_dict() method.
   - LLMProvider abstract class: async chat(model, request, tenant_id) → LLMResponse, supports_model(model) → bool.

2. gateway/providers/openai_provider.py — OpenAIProvider(LLMProvider):
   - Uses openai SDK (AsyncOpenAI).
   - chat(): call client.chat.completions.create(), extract content from choices[0].message.content, extract token counts from usage (prompt_tokens, completion_tokens, total_tokens). Return LLMResponse.
   - supports_model(): True for models starting with "gpt-".
   - Handle API errors (raise exceptions for 5xx/429/timeout to trigger failover).

3. gateway/providers/anthropic_provider.py — AnthropicProvider(LLMProvider):
   - Uses anthropic SDK (AsyncAnthropic).
   - chat(): call client.messages.create(), extract content from response.content[0].text, extract token counts from response.usage (input_tokens, output_tokens). Return LLMResponse.
   - supports_model(): True for models starting with "claude-".
   - Handle API errors similarly.

4. gateway/providers/ollama_provider.py — OllamaProvider(LLMProvider):
   - Uses HTTP REST calls to OLLAMA_BASE_URL (http://ollama:11434/api/chat).
   - chat(): POST to /api/chat with model + messages, parse response, extract content and token counts. Return LLMResponse.
   - supports_model(): True for models starting with "llama".
   - This is the last-resort fallback — lower quality, no API cost.

5. gateway/providers/failover.py — Implement CircuitBreaker and FailoverChain exactly per Section 7.7:
   - CircuitBreaker(redis_client, failure_threshold=5, cooldown_seconds=30, window_seconds=60):
     - _key(provider): f"circuit:{provider}"
     - is_open(provider): check Redis hgetall. If state=="open" and cooldown elapsed → set to "half_open", return False. If state=="open" and cooldown not elapsed → return True. Else return False.
     - record_success(provider): delete key (reset to closed).
     - record_failure(provider): INCR failures counter, EXPIRE to window. If failures >= threshold → HSET state="open", opened_at=time.time(), failure_count. EXPIRE key to cooldown+60.
   - FailoverChain(providers: dict, circuit_breaker, settings):
     - _order = settings.failover_order (["openai", "anthropic", "ollama"])
     - _timeout_seconds = settings.provider_timeout_seconds (10)
     - execute(model, request, tenant_id) → LLMResponse: iterate providers in order. Skip if circuit open. Skip if provider doesn't support model. Try with asyncio.wait_for(timeout). On success: record_success, tag response.provider, return. On failure: record_failure, continue. If all fail: raise AllProvidersFailedError.
   - AllProvidersFailedError(Exception).

6. tests/unit/test_failover.py — Tests per Section 10 pseudocode:
   - test_failover_on_primary_5xx: Mock OpenAI returns 500 → request succeeds via Anthropic. response.provider == "anthropic".
   - test_all_providers_fail_returns_503: All providers return 500 → AllProvidersFailedError raised.

7. tests/unit/test_circuit_breaker.py — Tests per Section 10 pseudocode:
   - test_circuit_opens_after_threshold: 5 failures in 60s → is_open returns True.
   - test_circuit_half_open_after_cooldown: 5 failures → open. Wait cooldown → is_open returns False (half-open, allows trial). record_success → is_open returns False (closed).

Use mock providers that can be configured to return success/failure/timeout.
```

```
billing: While routing agent builds providers, prepare the integration points.

1. gateway/dependencies.py — Add get_failover_chain and get_router dependencies. Construct providers dict with OpenAIProvider, AnthropicProvider, OllamaProvider using settings. Construct CircuitBreaker with Redis client. Construct FailoverChain with providers, circuit_breaker, settings.

2. Do NOT modify files in gateway/providers/ or gateway/routing/ — those are the routing agent's domain. Only modify dependencies.py.
```

#### Day 8 — Model routing engine (classifier + confidence + router)

**Parallel Work:**

```
routing: Build the model routing engine. Reference IMPLEMENTATION_PLAN.md Section 7.2 (full pseudocode for ModelRouter), Section 7.3 (TaskClassifier), Section 7.4 (ConfidenceScorer), and Section 3 (Model Routing Decision Tree).

1. gateway/routing/classifier.py — Implement TaskClassifier exactly per Section 7.3:
   - COMPLEX_KEYWORDS: ["explain", "analyze", "compare", "contrast", "synthesize", "reason", "derive", "prove", "design", "architect", "refactor", "debug", "optimize", "evaluate", "critique", "summarize long"]
   - CODE_MATH_KEYWORDS: ["function", "algorithm", "equation", "theorem", "complexity", "recursive", "concurrent", "distributed", "sql query", "code review"]
   - COMPLEX_PATTERN: regex matching all keywords (case-insensitive, word-boundary).
   - __init__(long_prompt_threshold=2000).
   - classify(prompt, request) → str:
     1. If request.get("model"): infer from model name (mini/haiku/8b → "simple", else "complex").
     2. If len(prompt) > threshold → "complex".
     3. If COMPLEX_PATTERN.search(prompt) → "complex".
     4. Else → "simple".

2. gateway/routing/confidence.py — Implement ConfidenceScorer exactly per Section 7.4:
   - SELF_EVAL_PROMPT template: "You are evaluating the quality of an AI response. Rate your confidence... Respond with ONLY a single integer 1-10."
   - __init__(provider, settings): store strategy ("self_eval" | "heuristic"), min_response_length (50).
   - async score(prompt, response, model) → float:
     - If strategy == "self_eval": call _self_eval (extra LLM call, max_tokens=5, parse integer, clamp 1-10, fallback 5.0 on parse error).
     - If strategy == "heuristic": call _heuristic (free, no LLM call).
   - _heuristic(response) → float: start at 7.0, -3.0 if len < min_response_length, -2.0 if hedging language found ("i'm not sure", "i cannot", "i can't help", "as an ai", "i don't have access"), -3.0 if "unable to", "no information", "cannot determine". Clamp 1-10.

3. gateway/routing/router.py — Implement ModelRouter exactly per Section 7.2:
   - RoutingDecision dataclass: primary_model, escalated_model (Optional), task_type, confidence_score (Optional), escalated (bool).
   - __init__(failover_chain, classifier, confidence_scorer, settings): store refs, read routing_model_map from settings.
   - async route_and_execute(prompt, request, tenant_id) → (LLMResponse, RoutingDecision):
     1. Classify task_type via classifier.classify(prompt, request).
     2. Select primary_model from model_map[task_type].
     3. Execute via failover_chain.execute(model=primary_model, request, tenant_id).
     4. If task_type == "simple" AND confidence_check_enabled AND primary_model == model_map["simple"]:
        a. Score confidence via confidence_scorer.score(prompt, response.content, primary_model).
        b. If confidence < threshold: escalate to model_map["escalate"], re-execute via failover, set decision.escalated=True.
     5. Return (response, decision).

4. tests/unit/test_router.py — Tests:
   - test_simple_task_uses_mini: "What is 2+2?" → task_type="simple", primary_model="gpt-4o-mini".
   - test_complex_task_uses_full: "Analyze the architectural tradeoffs of microservices vs monolith for a 50-engineer team" → task_type="complex", primary_model="gpt-4o".
   - test_low_confidence_escalates: Mock mini returning "I don't know" (short response) → heuristic confidence < 7.0 → escalate to gpt-4o, decision.escalated=True.
   - test_explicit_model_respected: request with model="gpt-4o" → classified as "complex", no escalation.
   - test_no_confidence_check_for_complex: Complex tasks skip confidence check entirely.

5. tests/unit/test_confidence.py — Tests:
   - test_short_response_low_score: Response < 50 chars → score < 7.0.
   - test_hedging_language_low_score: Response containing "I'm not sure" → score reduced by 2.0.
   - test_good_response_high_score: 200+ char response without hedging → score = 7.0.
   - test_self_eval_parses_integer: Mock provider returns "8" → score = 8.0.
   - test_self_eval_parse_failure_defaults_5: Mock provider returns "not a number" → score = 5.0.
```

```
billing: While routing agent builds the routing engine, wire the router into the chat endpoint.

1. gateway/api/v1/chat.py — Replace the TODO placeholder from Phase 3 with the full routing + cost flow (steps 5-12 in Section 7.10):
   - Step 5-6: Call router_engine.route_and_execute(prompt, request_body.model_dump(), tenant_id). Handle AllProvidersFailedError → 503.
   - Step 7: Cost calculation (will be fully implemented in Phase 5 — for now, use a placeholder that returns 0.0 or a simple calculation).
   - Step 8: Budget deduction via budget_enforcer.deduct_tokens(tenant_id, total_tokens, cost_usd, monthly_limit).
   - Step 9: Cost record write (will be fully implemented in Phase 5 — placeholder for now).
   - Step 10: Cache write via cache.store(prompt, response, model_name, model_family, tenant_id) if should_cache.
   - Step 11: Structured log (will be fully implemented in Phase 5 — placeholder for now).
   - Step 12: Return response with headers: X-Model-Used, X-Provider, X-Cache (MISS), X-Cost-USD, X-Escalated.

2. gateway/dependencies.py — Add get_cost_calculator and get_cost_store dependencies (will be implemented in Phase 5, but add the DI wiring now with placeholder implementations that return 0.0 cost and no-op record).

Do NOT modify files in gateway/routing/ or gateway/providers/ — only modify chat.py and dependencies.py.
```

#### Review

```
reviewer: Review Phase 4 deliverables against IMPLEMENTATION_PLAN.md:
1. Verify provider adapters: OpenAI (chat.completions.create, token extraction from usage), Anthropic (messages.create, token extraction from response.usage), Ollama (REST /api/chat). Each raises on 5xx/timeout for failover.
2. Verify FailoverChain.execute() matches Section 7.7: iterate in order, skip if circuit open, skip if model not supported, asyncio.wait_for with timeout, record_success/record_failure, AllProvidersFailedError if all fail.
3. Verify CircuitBreaker: 5 failures in 60s → open. 30s cooldown → half-open (allows trial). Success → closed. Redis-backed (distributed state).
4. Verify TaskClassifier.classify() matches Section 7.3: explicit model → infer, len > 2000 → complex, keyword match → complex, else simple.
5. Verify ConfidenceScorer: heuristic strategy (free, length + hedging checks), self_eval strategy (extra LLM call, parse 1-10, fallback 5.0). Default strategy from settings.
6. Verify ModelRouter.route_and_execute() matches Section 7.2: classify → select model → execute → confidence check (only for simple + cheap model) → escalate if confidence < threshold.
7. Verify chat.py flow order matches Section 7.10 steps 5-12: route → cost → budget deduct → cost record → cache write → log → return.
8. Verify response headers: X-Model-Used, X-Provider, X-Cache, X-Cost-USD, X-Escalated.
9. Run: "What is 2+2?" → served by gpt-4o-mini. "Analyze architectural tradeoffs..." → served by gpt-4o. Mock mini returning "I don't know" → X-Escalated: true.
10. Run all unit tests: pytest tests/unit/test_router.py tests/unit/test_confidence.py tests/unit/test_failover.py tests/unit/test_circuit_breaker.py -v
Report any deviations.
```

#### Commit

```bash
git add -A && git commit -m "Phase 4: Model routing (classifier + confidence + escalation) + provider failover with circuit breaker"
```

---

### Phase 5: Cost Tracking + Logging + Dashboard (Day 9)

> **Plan reference:** Section 9, Phase 5. Section 7.8 (Cost Calculator), Section 7.9 (Request Logger), Section 5 (SQL views), Section 8.3 (Admin endpoints).
>
> **Strategy:** billing and dashboard work in parallel — no file conflicts (billing owns gateway/tracking/ + gateway/middleware/request_logger.py + gateway/api/admin.py; dashboard owns dashboard/). billing implements cost tracking and structured logging; dashboard builds all Streamlit pages reading from the PG views and admin endpoints.

#### Parallel Work

```
billing: Implement cost tracking, structured logging, and admin cost endpoints. Reference IMPLEMENTATION_PLAN.md Section 7.8 (Cost Calculator), Section 7.9 (Request Logger), Section 5 (SQL views), Section 8.3 (Admin endpoints).

1. gateway/tracking/cost_calculator.py — Implement CostCalculator exactly per Section 7.8:
   - REFRESH_INTERVAL = 300 (5 minutes)
   - __init__(cost_repo): in-memory pricing cache dict, last_refresh timestamp.
   - _ensure_pricing_fresh(): if time since last_refresh > interval, fetch active pricing from cost_repo.get_active_pricing(), populate cache as {(provider, model_name): (input_cost, output_cost)}.
   - calculate_cost(provider, model_name, input_tokens, output_tokens) → float: ensure pricing fresh, lookup in cache, compute (input_tokens/1M * input_price) + (output_tokens/1M * output_price), round to 6 decimals. Unknown model → return 0.0 (don't block).

2. gateway/db/repositories/cost_repo.py — CostRepository class:
   - record(tenant_id, request_id, provider, model_name, input_tokens, output_tokens, total_tokens, cost_usd, cache_hit, task_type, escalated, latency_ms, endpoint): INSERT into cost_records.
   - get_active_pricing(): SELECT from model_pricing WHERE effective_to IS NULL.
   - get_cost_summary(tenant_id=None, from_date, to_date, group_by="day"|"model"|"endpoint"): aggregate query using GROUP BY.

3. gateway/tracking/cost_store.py — CostStore class wrapping CostRepository.record() with optional batching (buffer writes and flush periodically). For v1, direct writes are acceptable.

4. gateway/middleware/request_logger.py — Implement exactly per Section 7.9:
   - Use structlog with JSON output.
   - Context vars: request_id_ctx, tenant_id_ctx (ContextVar).
   - bind_request_context(request_id, tenant_id): set context vars.
   - async log_request(event, *, model, provider, task_type, escalated, cache_hit, cache_similarity, input_tokens, output_tokens, cost_usd, latency_ms, budget_used_pct, endpoint, status="success", error=None): emit structured JSON log line with ALL fields from Section 7.9 output format example.
   - Configure structlog in gateway/main.py lifespan: JSON renderer, timestamp processor, redact authorization headers.

5. gateway/api/admin.py — Add cost and budget admin endpoints per Section 8.3:
   - GET /admin/cost/summary?tenant=&from=&to=&group_by=day|model|endpoint — uses cost_repo.get_cost_summary.
   - GET /admin/budget/status — all tenants' budget usage (uses v_tenant_monthly_budget view or budget_repo.get_all_budget_status).
   - GET /admin/rate-limit/status — current rate window usage per tenant (uses rate_limiter.current_usage).
   - GET /admin/pricing — current model pricing table.
   - PUT /admin/pricing/{id} — update pricing (versioned: set effective_to on old row, insert new row).

6. Replace all placeholder cost/log calls in gateway/api/v1/chat.py with real implementations:
   - Step 7: cost_calculator.calculate_cost(provider, model, input_tokens, output_tokens).
   - Step 9: cost_store.record(...) with all fields.
   - Step 11: log_request(...) with all fields from the LLM response + routing decision.

7. tests/unit/test_cost_calculator.py — Tests:
   - test_known_tokens_known_pricing: 500 input × $5/M + 200 output × $15/M = $0.0055 (gpt-4o).
   - test_mini_model_pricing: 500 input × $0.15/M + 200 output × $0.60/M = $0.000195 (gpt-4o-mini).
   - test_unknown_model_returns_zero: Unknown model → 0.0.
   - test_pricing_cache_refresh: After REFRESH_INTERVAL, cache is refreshed from DB.

Deliverable: Full cost tracking pipeline with structured logging.
Verification: Send 10 requests → cost_records has 10 rows with correct USD. Cost summary endpoint matches sum of cost_records.
```

```
dashboard: Build the Streamlit admin dashboard. Reference IMPLEMENTATION_PLAN.md Section 9 Phase 5 (dashboard tasks), Section 5 (SQL views: v_daily_cost_summary, v_tenant_monthly_budget), Section 8.3 (admin endpoints), Section 12 (Dockerfile.dashboard).

1. dashboard/app.py — Streamlit entry point:
   - Page config: title "LLM Gateway Admin", wide layout.
   - Sidebar: tenant selector (dropdown from tenants table), date range picker.
   - Auto-refresh every DASHBOARD_REFRESH_SECONDS (30s) via st_autorefresh or st.rerun.
   - Navigation: overview, cost_by_tenant, model_distribution, budget_status, rate_limits.
   - Database connection: asyncpg or psycopg2 to read from PostgreSQL (DATABASE_URL from env). Use read-only queries.

2. dashboard/charts.py — Reusable chart helpers:
   - bar_chart(data, x, y, title): plotly bar chart.
   - pie_chart(data, values, names, title): plotly pie chart.
   - progress_bar(value, max, label): budget usage progress bar.
   - line_chart(data, x, y, title): plotly line chart for time series.
   - metric_card(label, value, delta=None): st.metric wrapper.

3. dashboard/pages/overview.py — Overview page (Goal G7):
   - Key metrics: total requests (last 7 days), total cost (last 7 days), cache hit rate %, avg latency.
   - Cache hit rate trend (line chart, last 7 days, from v_daily_cost_summary).
   - Daily cost trend (line chart, last 7 days).
   - Request volume trend (line chart, last 7 days).

4. dashboard/pages/cost_by_tenant.py — Cost breakdown per tenant:
   - Bar chart: cost per tenant (last 30 days) from v_daily_cost_summary.
   - Table: tenant, total cost, total requests, avg cost per request.
   - Filter by date range.

5. dashboard/pages/model_distribution.py — Model distribution:
   - Pie chart: requests by model (from cost_records, GROUP BY model_name).
   - Pie chart: requests by provider (from cost_records, GROUP BY provider).
   - Escalation rate: % of requests where escalated=True.
   - Bar chart: cost by model.

6. dashboard/pages/budget_status.py — Budget usage per tenant:
   - Progress bars: budget_used_pct for each tenant (from v_tenant_monthly_budget).
   - Warning indicators: tenants at 80%+ usage highlighted in orange, 100% in red.
   - Table: tenant, monthly budget, tokens used, estimated cost, usage %.

7. dashboard/pages/rate_limits.py — Rate limit status:
   - Current request counts per tenant per window (minute, hour, day).
   - Rate limit events log (from rate_limit_events table).
   - Utilization bars: current count / limit for each window.

8. Dockerfile.dashboard — Per Section 12: FROM python:3.11-slim, install build-essential libpq-dev, pip install streamlit pandas plotly asyncpg psycopg2-binary, COPY dashboard/ and gateway/, CMD streamlit run dashboard/app.py --server.port 8501 --server.address 0.0.0.0.

All dashboard pages read from PostgreSQL using the SQL views (v_daily_cost_summary, v_tenant_monthly_budget) and direct queries on cost_records, rate_limit_events, tenants tables. Use pandas DataFrames for data manipulation and plotly for charts.

Deliverable: Live-updating Streamlit dashboard with all 5 pages.
Verification: Dashboard shows cache hit rate, cost per tenant, model distribution, budget bars, rate limit stats.
```

#### Review

```
reviewer: Review Phase 5 deliverables against IMPLEMENTATION_PLAN.md:
1. Verify CostCalculator matches Section 7.8: in-memory pricing cache, 5-min refresh, formula (input/1M * input_price + output/1M * output_price), round 6 decimals, unknown model → 0.0.
2. Verify cost_records has one row per LLM call (cache misses only) with ALL fields from Section 5 schema (tenant_id, request_id, provider, model_name, input_tokens, output_tokens, total_tokens, cost_usd, cache_hit, cache_key, task_type, escalated, latency_ms, endpoint, created_at).
3. Verify request_logger.py matches Section 7.9: structlog JSON output, contextvars for request_id + tenant_id, ALL fields in log line (timestamp, level, event, request_id, tenant_id, tenant_name, endpoint, model, provider, task_type, escalated, cache_hit, cache_similarity, input_tokens, output_tokens, total_tokens, cost_usd, latency_ms, budget_used_pct, rate_limited, status).
4. Verify authorization headers are redacted in logs (Security Section 11 — API key leakage prevention).
5. Verify admin endpoints: GET /admin/cost/summary (with group_by), GET /admin/budget/status, GET /admin/rate-limit/status, GET /admin/pricing, PUT /admin/pricing/{id} (versioned).
6. Verify dashboard has all 5 pages: overview (cache hit rate, total cost, request volume), cost_by_tenant, model_distribution, budget_status (progress bars + warnings), rate_limits.
7. Verify dashboard reads from SQL views (v_daily_cost_summary, v_tenant_monthly_budget) — not direct table scans where views exist.
8. Verify Dockerfile.dashboard matches Section 12.
9. Run: send 10 requests → cost_records has 10 rows with correct USD. Dashboard reflects the data. Cost summary endpoint matches sum of cost_records.
10. Run: pytest tests/unit/test_cost_calculator.py -v
11. Verify daily cost report would match provider invoices within ±2% (Goal G3) — check cost calculation formula.
Report any deviations.
```

#### Commit

```bash
git add -A && git commit -m "Phase 5: Cost tracking (calculator + PG writer) + structured logging + Streamlit admin dashboard"
```

---

### Phase 6: Integration + Load Testing + Polish (Day 10)

> **Plan reference:** Section 9, Phase 6. Section 10 (Testing Strategy, integration test pseudocode).
>
> **Strategy:** billing runs solo — integration tests, load tests, and documentation. All other agents' components are complete and wired in. This is the final validation phase.

#### Sequential Work

```
billing: Final integration tests, load test, and documentation. Reference IMPLEMENTATION_PLAN.md Section 9 Phase 6, Section 10 (Testing Strategy, integration test pseudocode).

1. tests/integration/test_chat_endpoint.py — Full lifecycle integration test per Section 10 pseudocode:
   - test_full_lifecycle_cache_miss_then_hit: Same question twice — first X-Cache: MISS with cost > 0, second X-Cache: HIT with cost = 0.0. Use fakeredis + test PG + mock providers.
   - test_full_lifecycle_with_escalation: Simple question with mock mini returning low-confidence → X-Escalated: true, model is gpt-4o.
   - test_rate_limit_integration: 61 requests with limit=60 → 61st returns 429 with Retry-After.
   - test_budget_exceeded_integration: Exhaust budget → 429 with "Monthly token budget exceeded".
   - test_unauthorized_request: No API key → 401. Invalid API key → 401.

2. tests/integration/test_cache_hit_miss.py — Cache-specific integration:
   - test_same_question_cache_hit: Same question → HIT.
   - test_paraphrased_question_cache_hit: Paraphrased question (similarity >= 0.95) → HIT.
   - test_different_question_cache_miss: Unrelated question → MISS.
   - test_temperature_bypasses_cache: temperature > 0 → always MISS (no cache lookup).
   - test_tenant_isolation_integration: Tenant A's cached question → MISS for tenant B.

3. tests/integration/test_budget_rejection.py — Budget integration:
   - test_budget_warning_at_80_percent: Use 80% → X-Budget-Warning header present.
   - test_budget_exceeded_429: Exhaust budget → 429.
   - test_budget_warning_emitted_once: Warning header on first 80%+ request, not on subsequent.

4. tests/integration/test_provider_failover.py — Failover integration:
   - test_openai_down_anthropic_serves: Mock OpenAI 500 → response from Anthropic, X-Provider: anthropic.
   - test_all_providers_down_503: All providers fail → 503.
   - test_circuit_breaker_skips_open_provider: 5 OpenAI failures → circuit opens → next request skips directly to Anthropic.

5. tests/load/locustfile.py — Load test per Section 10:
   - 100 concurrent users, ramp rate 10/sec, 10 minute duration.
   - 60% repeated questions (cache hits), 40% unique questions (cache misses).
   - Mix of simple (70%) and complex (30%) prompts.
   - Track: cache hit rate, cost per request, latency percentiles (p50, p95, p99), error rate.
   - Verify: ~60% cache hit rate, cost < $5 (vs ~$55 without gateway), no 5xx errors.

6. README.md — Quick start guide per Section 15A:
   - Clone and configure (cp .env.example .env, add API keys).
   - make dev → starts all services.
   - make seed → creates test tenants.
   - curl example for /v1/chat/completions.
   - Dashboard URL (localhost:8501).
   - make benchmark, make test, make load-test.
   - Architecture summary (brief, link to IMPLEMENTATION_PLAN.md).

7. Makefile — Per Section 12:
   - dev: docker compose up --build
   - test: docker compose exec gateway pytest -v
   - seed: docker compose exec gateway python scripts/seed_tenants.py
   - dashboard: docker compose up dashboard
   - benchmark: docker compose exec gateway python scripts/benchmark_cache.py
   - load-test: locust -f tests/load/locustfile.py --headless -u 100 -r 10 -t 10m
   - clean: docker compose down -v

8. scripts/cost_report.py — CLI tool: print monthly cost report per tenant. Query v_daily_cost_summary, format as table.

9. Run full test suite: pytest -v (all unit + integration tests pass).
10. Run load test: make load-test (10k requests, verify 60% cache hit, 87% cost reduction, no 5xx).

Deliverable: Full test suite passing + load test report.
Verification: pytest — all green. Load test: 10k requests, ~60% cache hit, cost < $5, no 5xx errors. Dashboard reflects load test results.
```

#### Review

```
reviewer: Final comprehensive review against ALL goals in IMPLEMENTATION_PLAN.md Section 1:

1. G1 — Semantic cache: ≥ 50% cache hit rate on repeated workloads; sub-10ms cache lookup latency. Verify via benchmark results.
2. G2 — Model routing: ≥ 70% of requests served by cheap model; no quality regression. Verify via load test model distribution.
3. G3 — Cost tracking: Every request has a cost row in PostgreSQL; daily cost report matches provider invoices within ±2%. Verify cost_records completeness.
4. G4 — Per-tenant budgets: Budget-exceeded → HTTP 429; 80% warning to dashboard + webhook. Verify integration tests.
5. G5 — Rate limiting: Over-limit → HTTP 429 with Retry-After; no burst-through under concurrent load. Verify concurrent burst test.
6. G6 — Structured logging: Every request emits JSON log with tenant, model, tokens, latency, cache hit/miss, cost. Verify log output format.
7. G7 — Admin dashboard: Shows cache hit rate, cost per tenant, model distribution, budget usage, rate-limit stats — live-updating. Verify all 5 pages.
8. G8 — Multi-provider fallback: Automatic failover on 5xx/timeout; zero downtime during simulated outage. Verify failover tests.

9. Security audit (Section 11):
   - API keys: SHA-256 hashed in DB, never logged, never in response headers.
   - Tenant isolation: cache filtered by tenant_id, budget/rate keys namespaced.
   - Provider keys: in env vars only, never in DB/logs/headers.
   - PII: cached prompts are embeddings + hashed keys, not plaintext.

10. Verify all file paths match Section 6 Project Structure — no missing or extra files.
11. Run: pytest -v → all tests green.
12. Run: make load-test → 10k requests, 60% cache hit, 87% cost reduction, no 5xx.
13. Verify Makefile targets work: make dev, make test, make seed, make benchmark, make dashboard, make load-test, make clean.

Report any deviations, remaining risks, or unmet goals.
```

#### Commit

```bash
git add -A && git commit -m "Phase 6: Integration tests, load test, documentation — 87% cost reduction demonstrated"
```

---

## 5. Coordination Rules

### File Ownership Boundaries

| Agent | Can Create / Modify | Cannot Touch |
|-------|---------------------|--------------|
| **billing** | `gateway/config.py`, `gateway/main.py`, `gateway/dependencies.py`, `gateway/db/**`, `gateway/middleware/**`, `gateway/tracking/**`, `gateway/models/**`, `gateway/api/**`, `docker-compose.yml`, `Dockerfile`, `pyproject.toml`, `.env.example`, `Makefile`, `scripts/seed_tenants.py`, `scripts/cost_report.py`, `tests/conftest.py`, `tests/unit/test_rate_limiter.py`, `tests/unit/test_budget_enforcer.py`, `tests/unit/test_cost_calculator.py`, `tests/integration/**`, `tests/load/**`, `README.md` | `gateway/cache/**`, `gateway/routing/**`, `gateway/providers/**`, `dashboard/**`, `Dockerfile.dashboard`, `scripts/benchmark_cache.py`, `tests/unit/test_semantic_cache.py`, `tests/unit/test_router.py`, `tests/unit/test_confidence.py`, `tests/unit/test_failover.py`, `tests/unit/test_circuit_breaker.py` |
| **cache** | `gateway/cache/**`, `scripts/benchmark_cache.py`, `tests/unit/test_semantic_cache.py` | All other files |
| **routing** | `gateway/providers/**`, `gateway/routing/**`, `tests/unit/test_router.py`, `tests/unit/test_confidence.py`, `tests/unit/test_failover.py`, `tests/unit/test_circuit_breaker.py` | All other files |
| **dashboard** | `dashboard/**`, `Dockerfile.dashboard` | All other files |
| **reviewer** | None (read-only review) | All files (no writes) |

### Conflict Avoidance Rules

1. **The chat endpoint (`gateway/api/v1/chat.py`) is owned by billing.** Other agents create their components in their own directories. billing wires them in via `gateway/dependencies.py` and `gateway/api/v1/chat.py`. No other agent modifies these two files.
2. **`gateway/dependencies.py` is owned by billing.** All dependency injection wiring goes through billing. Other agents expose their components via well-defined class interfaces; billing imports and wires them.
3. **`tests/conftest.py` is owned by billing.** Shared fixtures (fakeredis, test PG, mock providers, test tenant) are defined here. Other agents may add fixtures to their own test files if needed, but shared fixtures live in conftest.py.
4. **`pyproject.toml` is owned by billing.** If cache, routing, or dashboard need additional dependencies, they message billing via Herdr to add them. No agent edits pyproject.toml directly except billing.
5. **`docker-compose.yml` is owned by billing.** Dashboard agent creates `Dockerfile.dashboard` but does not modify docker-compose.yml — billing adds the dashboard service definition.

### Parallel vs Sequential Rules

| Scenario | Mode | Reason |
|----------|------|--------|
| Phase 1 (foundation) | Sequential (billing only) | All other agents depend on scaffold, config, DB, auth |
| Phase 2 (rate limit + budget) | Sequential (billing only) | Middleware depends on Phase 1 foundation |
| Phase 3 Day 5 (cache components + DI wiring) | Parallel (cache + billing) | cache builds gateway/cache/, billing prepares dependencies.py + chat.py wiring — no file overlap |
| Phase 3 Day 6 (benchmark + verification) | Sequential (cache, then review) | Cache benchmark depends on integration being complete |
| Phase 4 Day 7 (providers + DI wiring) | Parallel (routing + billing) | routing builds gateway/providers/, billing prepares dependencies.py — no file overlap |
| Phase 4 Day 8 (routing engine + chat wiring) | Parallel (routing + billing) | routing builds gateway/routing/, billing wires into chat.py — no file overlap |
| Phase 5 (cost tracking + dashboard) | Parallel (billing + dashboard) | billing owns gateway/tracking/ + middleware + admin; dashboard owns dashboard/ — zero file overlap |
| Phase 6 (integration + load test) | Sequential (billing only) | All components complete; billing validates end-to-end |

### Blocked Agent Protocol

1. **If an agent is blocked on a dependency from another agent:** Send a message via `herdr agent send <blocking-agent> --message "<description of what you need>"`. Do NOT modify the other agent's files.
2. **If billing needs an interface from cache/routing:** billing creates a placeholder interface in dependencies.py and wires the real implementation when the agent delivers. The agent must export their class with the expected constructor signature (matching the pseudocode in the plan).
3. **If an agent discovers a missing dependency in pyproject.toml:** Message billing with the package name and version. billing adds it and commits.
4. **If an agent discovers a schema change is needed:** Message billing. billing modifies the migration SQL and commits. The agent waits for the migration update before proceeding.
5. **If reviewer finds a critical defect:** reviewer messages the responsible agent with the specific file, line, and issue. The agent fixes it and re-notifies reviewer for re-review before the phase commit.

### Inter-Agent Interface Contracts

These contracts MUST be followed so billing can wire components without seeing the implementation:

| Component | Exported Class | Constructor Signature | Key Methods |
|-----------|---------------|----------------------|-------------|
| Semantic Cache | `SemanticCache` | `(redis_client, embedding_client, settings)` | `lookup(prompt, model_family, tenant_id) → Optional[CacheEntry]`, `store(prompt, response, model_name, model_family, tenant_id) → str`, `should_cache(request: dict) → bool`, `invalidate_tenant(tenant_id) → int` |
| Embedding Client | `EmbeddingClient` | `(settings)` | `embed(text: str) → List[float]` |
| Cache Entry | `CacheEntry` (dataclass) | — | Fields: `cache_key`, `response` (dict), `model_name`, `tenant_id`, `created_at`, `similarity` |
| Model Router | `ModelRouter` | `(failover_chain, classifier, confidence_scorer, settings)` | `route_and_execute(prompt, request, tenant_id) → (LLMResponse, RoutingDecision)` |
| Task Classifier | `TaskClassifier` | `(long_prompt_threshold=2000)` | `classify(prompt, request) → str` |
| Confidence Scorer | `ConfidenceScorer` | `(provider, settings)` | `score(prompt, response, model) → float` |
| Failover Chain | `FailoverChain` | `(providers: dict, circuit_breaker, settings)` | `execute(model, request, tenant_id) → LLMResponse` |
| Circuit Breaker | `CircuitBreaker` | `(redis_client, failure_threshold=5, cooldown_seconds=30, window_seconds=60)` | `is_open(provider) → bool`, `record_success(provider)`, `record_failure(provider)` |
| LLM Provider | `LLMProvider` (abstract) | — | `chat(model, request, tenant_id) → LLMResponse`, `supports_model(model) → bool` |
| LLM Response | `LLMResponse` (dataclass) | — | Fields: `content`, `model`, `provider`, `input_tokens`, `output_tokens`, `total_tokens`; method `to_dict()` |
| Routing Decision | `RoutingDecision` (dataclass) | — | Fields: `primary_model`, `escalated_model`, `task_type`, `confidence_score`, `escalated` |

---

## 6. Quick Reference

### Common Herdr Commands

```bash
# ── Pane Management ────────────────────────────────────────────
herdr split --id <name> --cwd "$PWD" --no-focus    # Create a new pane
herdr panes                                         # List all panes
herdr focus <pane-id>                               # Focus a pane
herdr kill <pane-id>                                # Kill a pane

# ── Agent Management ───────────────────────────────────────────
herdr agent start <name> --kind codex --pane <id>   # Start an agent in a pane
herdr agent list                                     # List all agents
herdr agent stop <name>                              # Stop an agent
herdr agent send <name> --message "<text>"           # Send a message to an agent

# ── Project Workflow ───────────────────────────────────────────
docker compose up --build                            # Start all services
docker compose exec gateway pytest -v               # Run tests
docker compose exec gateway python scripts/seed_tenants.py  # Seed test data
docker compose exec gateway python scripts/benchmark_cache.py  # Cache benchmark
docker compose up dashboard                          # Start dashboard only
locust -f tests/load/locustfile.py --headless -u 100 -r 10 -t 10m  # Load test
docker compose down -v                               # Stop and clean up

# ── Quick Verification ─────────────────────────────────────────
curl localhost:8000/health                           # Health check
curl localhost:8000/ready                            # Readiness check
redis-cli ping                                       # Redis check
psql -U gateway -d llm_gateway -c "SELECT COUNT(*) FROM cost_records;"  # PG check
open http://localhost:8501                           # Dashboard

# ── Git Commits (per phase) ────────────────────────────────────
git add -A && git commit -m "Phase 1: Foundation — scaffold, Docker, DB, auth"
git add -A && git commit -m "Phase 2: Rate limiting + budget enforcement"
git add -A && git commit -m "Phase 3: Semantic cache — embedding, vector index, benchmark"
git add -A && git commit -m "Phase 4: Model routing + provider failover"
git add -A && git commit -m "Phase 5: Cost tracking + logging + dashboard"
git add -A && git commit -m "Phase 6: Integration tests + load test + docs"
```

### Phase Summary

| Phase | Days | Agents Active | Parallel? | Key Deliverable |
|-------|------|---------------|-----------|-----------------|
| 1 — Foundation | 1–2 | billing | No (solo) | Scaffold, Docker, DB schema, tenant auth |
| 2 — Rate Limit + Budget | 3–4 | billing | No (solo) | Sliding window rate limiter, budget enforcer with 80% warning |
| 3 — Semantic Cache | 5–6 | cache + billing | Yes (Day 5) | Embedding client, Redis vector index, cache hit/miss flow |
| 4 — Routing + Failover | 7–8 | routing + billing | Yes (Days 7–8) | Provider adapters, circuit breaker, classifier, confidence, router |
| 5 — Cost + Dashboard | 9 | billing + dashboard | Yes | Cost calculator, structured logging, 5-page Streamlit dashboard |
| 6 — Integration + Load | 10 | billing | No (solo) | Integration tests, load test (10k req, 87% cost reduction), docs |

### Goal Verification Checklist

| Goal | Phase | Verification Method |
|------|-------|-------------------|
| G1 — Semantic cache ≥ 50% hit, < 10ms | Phase 3 | `make benchmark` → hit rate ≥ 50%, lookup < 10ms |
| G2 — Routing ≥ 70% cheap model | Phase 4 | Load test model distribution → ≥ 70% gpt-4o-mini |
| G3 — Cost tracking ± 2% accuracy | Phase 5 | `cost_records` sum vs provider invoice |
| G4 — Budget 429 + 80% warning | Phase 2 | Integration test: exhaust budget → 429, 80% → warning header |
| G5 — Rate limit 429 + Retry-After | Phase 2 | 61st request with limit=60 → 429, concurrent burst → no leak |
| G6 — Structured JSON logging | Phase 5 | Every request → JSON log with all fields |
| G7 — Streamlit dashboard | Phase 5 | All 5 pages live-updating |
| G8 — Provider failover | Phase 4 | Mock OpenAI 500 → Anthropic serves, zero downtime |
