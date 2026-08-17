# Agent Orchestration Guide — Prompt Versioning & A/B Testing System

> **Runbook for Herdr multi-agent execution of IMPLEMENTATION_PLAN.md.**
> 5 phases over 7 days. 4 agents: core, api, dashboard, reviewer.

---

## 1. Agent Roster

| Agent | Kind | Responsibility | Owns (files/dirs) |
|-------|------|----------------|-------------------|
| **core** | codex | PostgreSQL schema, DB connection pool, repository layer, Jinja2 templating engine, LLM client/judge/cost, A/B test runner, statistical significance engine, metric aggregation, seed/init scripts, Docker infrastructure | `src/db/`, `src/templating/`, `src/llm/`, `src/testing/`, `scripts/`, `docker/`, `docker-compose.yml`, `pyproject.toml`, `.env.example` |
| **api** | codex | FastAPI app factory, Pydantic request/response models, dependency injection, all REST route handlers (prompts, eval datasets, ab tests, promote/rollback, SDK endpoint), Python SDK client (`PromptClient`) | `src/api/`, `src/sdk/`, `src/main.py`, `src/config.py` |
| **dashboard** | codex | Streamlit entry point, 4 dashboard pages (overview, A/B test results, version history, metric trends), Plotly chart builders, styled table renderers, SQL queries for dashboard data | `dashboard/` |
| **reviewer** | codex | Read-only code review after each phase: verify file ownership boundaries, check test coverage against plan's testing strategy (Section 10), validate API contracts against Section 9, confirm DB schema matches Section 4, run `ruff check` and `pytest` | *(read-only — no file ownership)* |

---

## 2. Pane Layout

```
┌──────────────────────────────────┬──────────────────────────────────┐
│                                  │                                  │
│  Pane 1: core                    │  Pane 2: api                     │
│  DB schema, repos, templating,   │  FastAPI routes, Pydantic        │
│  LLM client, A/B runner, stats   │  models, SDK client, config      │
│                                  │                                  │
├──────────────────────────────────┼──────────────────────────────────┤
│                                  │                                  │
│  Pane 3: dashboard               │  Pane 4: reviewer                │
│  Streamlit app, 4 pages,         │  Code review, test verification, │
│  Plotly charts, SQL queries      │  ruff + pytest gate per phase    │
│                                  │                                  │
└──────────────────────────────────┴──────────────────────────────────┘
```

---

## 3. Setup Commands

```bash
# --- Initialize Herdr session in project root ---
cd "$PWD"

# --- Split into 4 panes (2x2 grid) ---
herdr pane split --direction vertical --cwd "$PWD" --no-focus          # Pane 2 (right of Pane 1)
herdr pane select 0
herdr pane split --direction horizontal --cwd "$PWD" --no-focus        # Pane 3 (below Pane 1)
herdr pane select 1
herdr pane split --direction horizontal --cwd "$PWD" --no-focus        # Pane 4 (below Pane 2)

# --- Start agents ---
herdr agent start core      --kind codex --pane 0
herdr agent start api       --kind codex --pane 1
herdr agent start dashboard --kind codex --pane 2
herdr agent start reviewer  --kind codex --pane 3

# --- Verify all agents are running ---
herdr agent list
```

---

## 4. Phase-by-Phase Orchestration

### Phase 1: Foundation — DB + Prompt Store + Templating (Days 1-2)

**Goal:** PostgreSQL running, schema deployed, prompt CRUD API works, Jinja2 templating validated.

**Milestone M1:** Docker Compose up, prompt CRUD API works, Jinja2 templates render + validate.

#### Parallel Work

**Agent core (Pane 0):**

```
You are building the data layer foundation for a Prompt Versioning & A/B Testing system. Read IMPLEMENTATION_PLAN.md Sections 4 (Database Schema), 5 (Project Structure), and 7.1 (Jinja2 Templating Engine).

Create these files in order:

1. `docker-compose.yml` — Follow Section 11.1 exactly. Three services: postgres (image: postgres:16, DB: prompt_ab, user: prompt_user, healthcheck pg_isready), api (build from docker/Dockerfile.api, port 8000, depends_on postgres healthy), dashboard (build from docker/Dockerfile.dashboard, port 8501, depends_on postgres + api healthy). Include postgres_data volume.

2. `docker/postgres/init.sql` — Follow Section 11.4: `CREATE EXTENSION IF NOT EXISTS "uuid-ossp";`

3. `docker/Dockerfile.api` — Follow Section 11.2: python:3.12-slim, install build-essential libpq-dev curl, pip install -e ., CMD uvicorn src.main:app.

4. `docker/Dockerfile.dashboard` — Follow Section 11.3: python:3.12-slim, pip install -e ., CMD streamlit run dashboard/app.py.

5. `pyproject.toml` — Dependencies: fastapi, uvicorn, asyncpg, jinja2, openai, scipy, statsmodels, numpy, httpx, pydantic-settings, streamlit, plotly, pandas, pytest, pytest-asyncio, ruff. Python 3.11+.

6. `.env.example` — Follow Section 11.5: POSTGRES_PASSWORD, OPENAI_API_KEY, OPENAI_BASE_URL, DEFAULT_TARGET_MODEL, DEFAULT_JUDGE_MODEL, LOG_LEVEL.

7. `src/db/schema.sql` — Implement ALL 7 tables from Section 4.2 verbatim: prompts, prompt_versions (with partial unique index idx_pv_one_production for one-production invariant), eval_datasets, eval_items, ab_tests, test_results, metric_history, promotions. Include all CHECK constraints, indexes, and the prevent_promotion_modification trigger function + triggers (no_promo_update, no_promo_delete).

8. `src/db/connection.py` — asyncpg connection pool. Create a `get_pool()` function and a `close_pool()` function. Pool size 10, configured via DATABASE_URL env var.

9. `src/templating/renderer.py` — Follow Section 7.1 pseudocode exactly: `create_env()` returns SandboxedEnvironment(undefined=StrictUndefined, autoescape=False, trim_blocks=True, lstrip_blocks=True). `render_template(template_str, variables)` renders and returns string. `validate_template(template_str, variable_defs)` compile-checks, extracts referenced vars via jinja2.meta.find_undeclared_variables, returns list of error strings.

10. `src/templating/validator.py` — Wrapper around validate_template that raises ValueError with joined error messages if any errors found. Used by API layer on version creation.

11. `src/db/repositories/prompt_repo.py` — Async functions using asyncpg:
    - `create_prompt(conn, prompt_key, name, description, tags)` → inserts into prompts, returns row
    - `create_version(conn, prompt_id, template, variables, message_type, model_hint, temperature, max_tokens, metadata, changelog, created_by)` → auto-increments version (SELECT MAX(version)+1 for prompt_id), inserts into prompt_versions with status='draft', returns row
    - `get_prompt_by_key(conn, prompt_key)` → returns prompt row
    - `get_prompt_by_id(conn, prompt_id)` → returns prompt row
    - `get_version(conn, prompt_id, version)` → returns prompt_versions row
    - `list_versions(conn, prompt_id)` → returns all versions ordered by version DESC
    - `get_production_version(conn, prompt_id)` → returns the row with status='production'

12. `scripts/init_db.py` — Reads src/db/schema.sql and executes it against the database (uses DATABASE_URL env var). Print "Schema deployed" on success.

13. `scripts/seed_data.py` — Follow Section 10.5 seed pattern: create 2 prompts (summarizer, qa-extractor), 3 versions each (v1/v2/v3 with different templates), 1 eval dataset "summarizer-golden-v1" with 10 items (each has input_variables with article+max_words, expected_output). Print "Seed data loaded" on success.

14. `tests/test_templating.py` — Follow Section 10.4 exactly: TestRendering class (test_simple_variable, test_conditional, test_missing_variable_raises, test_sandbox_blocks_code_execution with "{{ ''.__class__.__mro__[1].__subclasses__() }}"), TestValidation class (test_valid_template_passes, test_undeclared_variable_caught, test_syntax_error_caught).

15. `tests/test_prompt_repo.py` — Test prompt CRUD: create_prompt returns row with prompt_key, create_version auto-increments version (v1→v2→v3), list_versions returns descending, get_version returns correct template, versions are immutable (no update function exists).

16. `tests/conftest.py` — Follow Section 10.5: db fixture (connect to test DB, run schema.sql, yield conn, drop all tables after), client fixture (FastAPI TestClient with overridden DB pool using httpx.AsyncClient), seeded_data fixture (2 prompts, 3 versions each, 1 dataset with 10 items).

Do NOT touch src/api/, src/sdk/, src/main.py, src/config.py, or dashboard/ — those are owned by other agents.
```

**Agent api (Pane 1):**

```
You are building the FastAPI application layer for a Prompt Versioning & A/B Testing system. Read IMPLEMENTATION_PLAN.md Sections 5 (Project Structure), 9.1-9.3 (API Spec: POST /prompts, POST /prompts/{id}/versions, GET /prompts/{id}/versions), and 9.11 (GET /health).

Create these files:

1. `src/config.py` — Pydantic Settings class reading env vars: DATABASE_URL (str, required), OPENAI_API_KEY (str, default ""), OPENAI_BASE_URL (str, default "https://api.openai.com/v1"), DEFAULT_TARGET_MODEL (str, default "gpt-4o-mini"), DEFAULT_JUDGE_MODEL (str, default "gpt-4o-mini"), LOG_LEVEL (str, default "info"). Use pydantic_settings.BaseSettings with model_config = SettingsConfigDict(env_file=".env").

2. `src/api/models.py` — Pydantic request/response models for prompt management:
   - CreatePromptRequest: prompt_key (str), name (str), description (str|None), tags (list[str] = [])
   - CreateVersionRequest: template (str), variables (list[dict]), message_type (str = "user"), model_hint (str|None), temperature (float = 0.0), max_tokens (int = 1024), changelog (str|None), metadata (dict = {})
   - PromptResponse: id (str), prompt_key (str), name (str), description (str|None), tags (list[str]), created_at (str)
   - VersionResponse: id (str), prompt_id (str), version (int), status (str), changelog (str|None), created_by (str), created_at (str)
   - VersionListResponse: prompt_id (str), versions (list[VersionResponse])

3. `src/api/dependencies.py` — FastAPI dependency `get_db_pool()` that returns the asyncpg pool from src/db/connection.py. Import the pool getter. Use `Depends()` pattern.

4. `src/api/routes/prompts.py` — FastAPI APIRouter with prefix="/prompts":
   - POST "/" — create prompt. Body: CreatePromptRequest. Calls prompt_repo.create_prompt. Returns 201 PromptResponse. 409 if prompt_key exists.
   - POST "/{prompt_id}/versions" — create version. Body: CreateVersionRequest. Validates template via templating.validator.validate_template (raise 400 on errors). Calls prompt_repo.create_version. Returns 201 VersionResponse.
   - GET "/{prompt_id}/versions" — list versions. Calls prompt_repo.list_versions. Returns 200 VersionListResponse.
   - GET "/{prompt_id}" — get prompt by ID. Returns PromptResponse.
   Import prompt_repo from src.db.repositories.prompt_repo and templating validator from src.templating.validator. These files are being built by the core agent in parallel — write the imports assuming they exist with the function signatures described above.

5. `src/main.py` — FastAPI app factory:
   - `create_app(db_pool=None)` function that creates FastAPI instance
   - Lifespan context: init db pool on startup, close on shutdown
   - Include prompts router from src.api.routes.prompts
   - GET "/health" endpoint returning {"status": "healthy", "database": "connected", "version": "1.0.0"} (Section 9.11)
   - Set title="Prompt A/B Testing API", version="1.0.0"

6. `src/api/routes/__init__.py` and `src/api/__init__.py` — empty init files.

Do NOT touch src/db/, src/templating/, src/llm/, src/testing/, scripts/, docker/, docker-compose.yml, pyproject.toml, or dashboard/ — those are owned by other agents.
```

#### Sequential (after parallel)

**Agent core (Pane 0):**

```
Phase 1 integration check. The api agent has created src/api/routes/prompts.py which imports from your src.db.repositories.prompt_repo and src.templating.validator. Verify the function signatures match:

- prompt_repo.create_prompt(conn, prompt_key, name, description, tags) → row dict
- prompt_repo.create_version(conn, prompt_id, template, variables, message_type, ...) → row dict
- prompt_repo.list_versions(conn, prompt_id) → list of row dicts
- prompt_repo.get_prompt_by_id(conn, prompt_id) → row dict
- templating.validator.validate_template(template_str, variable_defs) → raises ValueError on errors

If any signature mismatch is found, adjust your repo functions to match what api expects. Then run: `python scripts/init_db.py` to deploy schema, and `python scripts/seed_data.py` to load seed data. Report success or errors.
```

#### Review

**Agent reviewer (Pane 3):**

```
Review Phase 1 deliverables against IMPLEMENTATION_PLAN.md:

1. Verify docker-compose.yml matches Section 11.1 (3 services: postgres, api, dashboard with correct ports, healthchecks, depends_on).
2. Verify src/db/schema.sql has all 7 tables from Section 4.2 with correct columns, constraints, and indexes. Check specifically: idx_pv_one_production partial unique index on prompt_versions WHERE status='production', and the prevent_promotion_modification trigger on promotions table.
3. Verify src/templating/renderer.py uses SandboxedEnvironment + StrictUndefined (Section 7.1).
4. Verify src/api/routes/prompts.py implements POST /prompts (Section 9.1), POST /prompts/{id}/versions with template validation (Section 9.2), GET /prompts/{id}/versions (Section 9.3).
5. Verify src/main.py has GET /health (Section 9.11).
6. Run `ruff check src/ tests/` and `pytest tests/test_templating.py tests/test_prompt_repo.py -v`. Report any failures.
7. Check file ownership: core should NOT have touched src/api/ or src/main.py. api should NOT have touched src/db/ or src/templating/.
Report PASS or FAIL with specific issues.
```

#### Commit

```bash
git add -A && git commit -m "Phase 1: Foundation — DB schema, prompt store, Jinja2 templating, FastAPI prompt CRUD"
```

---

### Phase 2: A/B Test Runner + LLM-as-Judge (Days 3-4)

**Goal:** Run an A/B test comparing two prompt versions on an eval dataset, collect per-item metrics.

**Milestone M2:** Run A/B test on eval dataset, collect per-item metrics (accuracy, latency, cost, tokens).

#### Parallel Work

**Agent core (Pane 0):**

```
You are building the LLM integration and A/B test runner. Read IMPLEMENTATION_PLAN.md Sections 7.2 (A/B Test Runner), 7.4 (LLM-as-Judge), 8.3 (Cost Safety), and 8.2 (Safe Defaults).

Create these files:

1. `src/llm/client.py` — OpenAI-compatible async LLM client:
   - `LLMClient` class initialized with api_key, base_url (from config)
   - `async generate(model, messages, temperature, max_tokens)` → returns a dataclass with: content (str), prompt_tokens (int), completion_tokens (int), total_tokens (int)
   - Uses the openai Python SDK with AsyncOpenAI
   - Retry with exponential backoff: 3 attempts, 1s/2s/4s delays (Section 8.2)
   - Supports configurable base_url for Ollama/local LLMs

2. `src/llm/cost.py` — Token cost estimation:
   - `PRICING` dict mapping model names to {prompt: float per 1K tokens, completion: float per 1K tokens}
   - Include gpt-4o-mini ($0.00015/1K prompt, $0.0006/1K completion), gpt-4o ($0.0025/1K, $0.01/1K), gpt-3.5-turbo ($0.0005/1K, $0.0015/1K)
   - `estimate_cost(model, prompt_tokens, completion_tokens)` → float (USD)
   - `estimate_test_cost(n_items, n_variants, target_model, judge_model, avg_prompt_tokens=500, avg_output_tokens=200)` → float (Section 8.3)

3. `src/llm/judge.py` — Follow Section 7.4 pseudocode exactly:
   - JUDGE_META_PROMPT constant with the scoring rubric (0.0-1.0 scale, SCORE:/REASONING: output format)
   - JudgeResult dataclass: score (float), reasoning (str), raw (str)
   - `async score(judge_model, prompt, output, expected)` → JudgeResult. Truncates prompt to 500 chars. Uses temperature=0.0, max_tokens=256.
   - `_parse_judge_response(raw)` → parses "SCORE: 0.85\nREASONING: ..." format, clamps score to [0.0, 1.0], defaults to 0.5 on parse failure

4. `src/db/repositories/eval_repo.py` — Async functions:
   - `create_dataset(conn, name, description, metadata, created_by)` → inserts eval_datasets, returns row
   - `add_item(conn, dataset_id, item_idx, input_variables, expected_output, metadata)` → inserts eval_items
   - `get_items(conn, dataset_id)` → returns all eval_items ordered by item_idx
   - `get_dataset(conn, dataset_id)` → returns dataset row
   - `update_item_count(conn, dataset_id, count)` → updates item_count

5. `src/db/repositories/test_repo.py` — Async functions:
   - `create_test(conn, prompt_id, version_a, version_b, dataset_id, judge_model, target_model, sample_size, confidence_level, created_by)` → inserts ab_tests with status='pending', returns row with test_id
   - `update_status(conn, test_id, status)` → updates ab_tests.status
   - `insert_result(conn, test_id, result)` → inserts one test_results row from ItemResult dataclass (fields: eval_item_id, variant, version_number, rendered_prompt, raw_output, output_length, accuracy_score, latency_ms, prompt_tokens, completion_tokens, total_tokens, cost_usd, judge_reasoning, error)
   - `complete_test(conn, test_id, winner, p_value)` → sets status='completed', winner, p_value, completed_at=NOW()
   - `get_test(conn, test_id)` → returns ab_tests row
   - `get_results(conn, test_id)` → returns all test_results for a test
   - `get_item_result(conn, test_id, item_idx)` → returns A and B results for a specific eval item (joined with eval_items for input_variables + expected_output)

6. `src/testing/runner.py` — Follow Section 7.2 pseudocode exactly:
   - ItemResult dataclass with all fields from the plan
   - `async run_ab_test(test_id, prompt_id, version_a, version_b, dataset_id, target_model, judge_model, sample_size)` → orchestrates the full test:
     a. Load both prompt versions via prompt_repo.get_version
     b. Load eval items via eval_repo.get_items (slice to sample_size if provided)
     c. Mark test as running via test_repo.update_status
     d. Run both variants concurrently with asyncio.Semaphore(5) (Section 8.2)
     e. Store per-item results via test_repo.insert_result
     f. Compute aggregate metrics via testing.metrics.compute_aggregate_metrics
     g. Compute significance via testing.stats.compute_significance (will be built in Phase 3 — import it, wrap in try/except for now)
     h. Declare winner via _declare_winner(stats)
     i. Store aggregated metrics via metric_repo.insert_history (Phase 3 — try/except)
     j. Update ab_tests row via test_repo.complete_test
   - `async _run_single_item(item, version_row, variant_label, target_model, judge_model)` → ItemResult. Renders template, calls LLM, measures latency, estimates cost, calls judge if expected_output exists. Catches exceptions → ItemResult with error field.
   - `_declare_winner(stats)` → "A" if stats.p_value < 0.05 and stats.win_rate_a > 0.5, "B" if stats.p_value < 0.05 and stats.win_rate_a < 0.5, "inconclusive" otherwise

7. `src/testing/metrics.py` — Aggregate per-variant metrics:
   - `compute_aggregate_metrics(results: list[ItemResult])` → dict with: mean_accuracy, median_accuracy, std_accuracy, mean_latency_ms, median_latency_ms, mean_cost_usd, total_cost_usd, mean_output_length, mean_total_tokens, win_rate, wins, losses, ties, sample_size
   - Use numpy for mean/median/std. Filter out None accuracy_scores and error results for accuracy metrics.

8. `scripts/run_sample_test.py` — End-to-end demo: load seed data, create A/B test (summarizer v1 vs v2 on summarizer-golden-v1), run it, print results. Follow Section Appendix A steps 10-11.

9. `tests/test_ab_runner.py` — Test with mocked LLM client (mock llm_client.generate to return predetermined outputs). Verify: runner collects results for all items, metrics are computed correctly, winner is declared based on mock scores.

10. `tests/test_judge.py` — Test _parse_judge_response with: valid "SCORE: 0.85\nREASONING: Good" → score=0.85, reasoning="Good"; missing SCORE line → defaults to 0.5; score out of range "SCORE: 1.5" → clamped to 1.0; empty response → defaults to 0.5.

Do NOT touch src/api/, src/sdk/, src/main.py, src/config.py, or dashboard/.
```

**Agent api (Pane 1):**

```
You are adding A/B test and eval dataset API endpoints. Read IMPLEMENTATION_PLAN.md Sections 9.6 (POST /eval-datasets), 9.7 (POST /ab-tests), 9.8 (GET /ab-tests/{id}), 9.9 (GET /ab-tests/{id}/items/{item_idx}).

Add these files/routes:

1. `src/api/routes/eval_datasets.py` — FastAPI APIRouter with prefix="/eval-datasets":
   - POST "/" — create dataset. Body: {name, description, items: [{input_variables, expected_output, metadata}]}. Calls eval_repo.create_dataset + eval_repo.add_item for each item. Updates item_count. Returns 201 {id, name, item_count, created_at} (Section 9.6).
   - GET "/{dataset_id}" — get dataset metadata + item count.
   - GET "/{dataset_id}/items" — list all items in dataset.
   Import eval_repo from src.db.repositories.eval_repo (being built by core in parallel — assume these signatures: create_dataset(conn, name, description, metadata, created_by), add_item(conn, dataset_id, item_idx, input_variables, expected_output, metadata), get_items(conn, dataset_id), get_dataset(conn, dataset_id), update_item_count(conn, dataset_id, count)).

2. `src/api/routes/ab_tests.py` — FastAPI APIRouter with prefix="/ab-tests":
   - POST "/" — create and start A/B test. Body: {prompt_id, version_a, version_b, dataset_id, target_model?, judge_model?, sample_size?, confidence_level?}. Calls test_repo.create_test, then launches runner.run_ab_test as a background task (asyncio.create_task). Returns 202 {test_id, status: "pending", estimated_cost_usd, estimated_items} (Section 9.7). Use llm.cost.estimate_test_cost for the estimate.
   - GET "/{test_id}" — get test status + results. If running: {test_id, status, progress, completed_items, total_items}. If completed: full results with metrics per variant, winner, p_value, sample_size, required_sample_size (Section 9.8).
   - GET "/{test_id}/items/{item_idx}" — get per-item results for side-by-side comparison. Returns {item_idx, input_variables, expected_output, results: {A: {rendered_prompt, raw_output, accuracy_score, latency_ms, judge_reasoning}, B: {...}}} (Section 9.9).
   Import test_repo from src.db.repositories.test_repo and runner from src.testing.runner (being built by core in parallel).

3. Update `src/main.py` — Add: include_router(eval_datasets_router), include_router(ab_tests_router). Import from src.api.routes.eval_datasets and src.api.routes.ab_tests.

4. Add Pydantic models to `src/api/models.py`:
   - CreateDatasetRequest, CreateABTestRequest, ABTestResponse, ABTestRunningResponse, ItemResultResponse, ItemComparisonResponse

Do NOT touch src/db/, src/templating/, src/llm/, src/testing/, scripts/, docker/, or dashboard/.
```

#### Sequential (after parallel)

**Agent core (Pane 0):**

```
Phase 2 integration. The api agent created routes that import from your eval_repo, test_repo, and runner. Verify signatures match:
- eval_repo.create_dataset, add_item, get_items, get_dataset, update_item_count
- test_repo.create_test, update_status, get_test, get_results, get_item_result
- runner.run_ab_test

Fix any mismatches. Then run: `python scripts/run_sample_test.py` to verify the end-to-end flow works. Report results.
```

#### Review

**Agent reviewer (Pane 3):**

```
Review Phase 2 deliverables against IMPLEMENTATION_PLAN.md:

1. Verify src/llm/client.py uses AsyncOpenAI with configurable base_url and retry with exponential backoff (3 attempts, 1s/2s/4s per Section 8.2).
2. Verify src/llm/judge.py matches Section 7.4: JUDGE_META_PROMPT with scoring rubric, temperature=0.0, max_tokens=256, _parse_judge_response clamps to [0,1].
3. Verify src/testing/runner.py follows Section 7.2: loads versions, loads eval items, runs concurrently with Semaphore(5), stores results, computes metrics, declares winner.
4. Verify src/api/routes/ab_tests.py implements POST /ab-tests (returns 202 with estimated_cost_usd per Section 9.7), GET /ab-tests/{id} (Section 9.8 with running/completed variants), GET /ab-tests/{id}/items/{idx} (Section 9.9).
5. Verify src/api/routes/eval_datasets.py implements POST /eval-datasets (Section 9.6).
6. Run `ruff check src/ tests/` and `pytest tests/test_ab_runner.py tests/test_judge.py -v`. Report failures.
7. Check file ownership boundaries maintained.
Report PASS or FAIL with specific issues.
```

#### Commit

```bash
git add -A && git commit -m "Phase 2: A/B test runner, LLM-as-judge, eval dataset + ab-test API endpoints"
```

---

### Phase 3: Statistical Significance + Metric Tracking (Day 5)

**Goal:** Compute win-rate, confidence intervals, p-values, and sample size requirements. Store aggregated metrics for trend tracking.

**Milestone M3:** Win-rate, Wilson CI, z-test p-value, power analysis; winner declared only if significant.

#### Parallel Work

**Agent core (Pane 0):**

```
You are building the statistical significance engine and metric history persistence. Read IMPLEMENTATION_PLAN.md Sections 7.3 (Statistical Significance) and 10.2 (Critical Stats Tests) VERY carefully — the math must be exact.

Create these files:

1. `src/testing/stats.py` — Follow Section 7.3 pseudocode EXACTLY:
   - SignificanceResult dataclass with ALL fields: win_rate_a, win_rate_b, tie_rate, wins_a, wins_b, ties, sample_size, ci_lower, ci_upper, p_value, z_score, required_sample_size, is_significant, effect_size
   - `compute_significance(results_a, results_b, confidence_level=0.95)` → SignificanceResult:
     a. Pair results by eval_item_id via _pair_by_item
     b. Count wins_a, wins_b, ties based on accuracy_score (None scores → tie)
     c. Compute effective_wins_a = wins_a + ties * 0.5, win_rate_a = effective_wins_a / n
     d. Wilson CI via wilson_confidence_interval(effective_wins_a, n, z) where z = scipy_stats.norm.ppf((1+confidence_level)/2)
     e. Binomial z-test via binomial_z_test(effective_wins_a, n, p0=0.5)
     f. Required sample size via required_sample_size_for_power(effect_size, alpha=1-confidence_level, power=0.80)
     g. is_significant = p_value < (1-confidence_level) AND n >= required_sample_size
     h. Handle n=0 edge case (return defaults)
   - `wilson_confidence_interval(successes, n, z)` → (lower, upper): Wilson score interval formula from Section 7.3. center = (p + z²/(2n)) / (1 + z²/n), spread = (z/(1+z²/n)) * sqrt(p(1-p)/n + z²/(4n²)). Clamp to [0, 1].
   - `binomial_z_test(successes, n, p0=0.5)` → (p_value, z_score): z = (p_hat - p0) / sqrt(p0*(1-p0)/n), p_value = 2*(1 - norm.cdf(|z|)). Handle n=0 and se=0.
   - `required_sample_size_for_power(effect_size, alpha=0.05, power=0.80)` → int: n = ((z_alpha + z_power)² * p0*(1-p0)) / (p1-p0)² where p0=0.5, p1=0.5+effect_size. Return ceil(n). Return inf if effect_size <= 0.
   - `_pair_by_item(results_a, results_b)` → dict mapping eval_item_id → (result_a, result_b)

2. Update `src/testing/runner.py` — Remove the try/except wrappers around stats imports from Phase 2. Wire in the real calls:
   - After computing aggregate metrics, call compute_significance(results_a, results_b, confidence_level=0.95)
   - Call _declare_winner(stats) with the real stats object
   - Call metric_repo.insert_history for both variants with metrics + stats

3. `src/db/repositories/metric_repo.py` — Async functions:
   - `insert_history(conn, prompt_id, version_number, test_id, metrics_dict, stats, variant_label)` → inserts into metric_history with all fields: mean_accuracy, median_accuracy, std_accuracy, mean_latency_ms, median_latency_ms, mean_cost_usd, total_cost_usd, mean_output_length, mean_total_tokens, win_rate, wins, losses, ties, ci_lower, ci_upper, sample_size. test_date = NOW(). Uses UNIQUE(prompt_id, version_number, test_id) constraint.
   - `get_history_by_prompt(conn, prompt_id)` → returns all metric_history rows for a prompt, ordered by test_date DESC
   - `get_history_by_version(conn, prompt_id, version_number)` → returns metric_history for a specific version
   - `get_metric_trends(conn, prompt_id, metric_name)` → returns rows with test_date + the requested metric column, ordered by test_date ASC (for dashboard line charts)

4. Update `src/db/repositories/test_repo.py` — Ensure complete_test also stores p_value. The ab_tests row should be updated with: status='completed', winner, p_value, completed_at=NOW().

5. `tests/test_stats.py` — Follow Section 10.2 EXACTLY. These are known-answer tests:
   - TestWilsonConfidenceInterval: test_50_percent_100_samples (50/100 → CI contains 0.5, lower in [0.35,0.45], upper in [0.55,0.65]), test_100_percent_small_sample (10/10 → upper < 1.0, lower > 0.5), test_zero_successes (0/10 → lower > 0.0, upper < 0.5)
   - TestBinomialZTest: test_equal_to_null (50/100 → |z| < 0.01, p > 0.99), test_highly_significant (90/100 → z > 7.0, p < 0.001), test_not_significant_small_sample (6/10 → p > 0.05)
   - TestRequiredSampleSize: test_large_effect_needs_few_samples (effect=0.30 → 20 < n < 50), test_small_effect_needs_many_samples (effect=0.05 → n > 500), test_zero_effect_needs_infinite (effect=0.0 → inf)
   - TestComputeSignificance: test_clear_winner_declared (40/50 wins → significant, p < 0.05, win_rate_a > 0.7), test_inconclusive_with_small_sample (7/10 → not significant), test_ties_handled_correctly (all ties → win_rate=0.5, not significant)
   - Create a _make_mock_results helper that generates lists of mock ItemResult objects with given win/loss/tie counts.

Do NOT touch src/api/, src/sdk/, src/main.py, src/config.py, or dashboard/.
```

**Agent api (Pane 1):**

```
Update the A/B test API to include statistical significance results in responses. Read IMPLEMENTATION_PLAN.md Section 9.8 (GET /ab-tests/{id} completed response).

1. Update `src/api/routes/ab_tests.py` — The GET /{test_id} endpoint for completed tests must return the full stats payload per Section 9.8:
   {
     "test_id", "status": "completed", "winner", "p_value", "confidence_level",
     "version_a", "version_b",
     "metrics": {
       "A": {mean_accuracy, mean_latency_ms, mean_cost_usd, mean_output_length, win_rate, ci_lower, ci_upper},
       "B": {same fields}
     },
     "sample_size", "required_sample_size", "completed_at"
   }
   Query metric_history for both variants (via metric_repo.get_history_by_version or test_repo.get_results + compute). The simplest approach: query metric_history WHERE test_id = test_id, which gives both variant rows.

2. Update `src/api/models.py` — Add/Update ABTestResponse to include: winner (str|None), p_value (float|None), confidence_level (float), metrics (dict with A/B sub-dicts), sample_size (int), required_sample_size (int), completed_at (str|None).

Import metric_repo from src.db.repositories.metric_repo (being built by core in parallel — assume insert_history, get_history_by_prompt, get_history_by_version, get_metric_trends signatures).

Do NOT touch src/db/, src/templating/, src/llm/, src/testing/, scripts/, docker/, or dashboard/.
```

#### Review

**Agent reviewer (Pane 3):**

```
Review Phase 3 deliverables against IMPLEMENTATION_PLAN.md:

1. CRITICAL: Verify src/testing/stats.py math is correct by checking against Section 7.3 formulas:
   - Wilson CI: center = (p + z²/(2n)) / (1 + z²/n), spread = (z/(1+z²/n)) * sqrt(p(1-p)/n + z²/(4n²))
   - Binomial z-test: z = (p_hat - p0) / sqrt(p0*(1-p0)/n), p_value = 2*(1 - norm.cdf(|z|))
   - Sample size: n = ((z_alpha + z_power)² * 0.25) / effect_size²
2. Verify SignificanceResult dataclass has ALL 14 fields from Section 7.3.
3. Verify _declare_winner returns "inconclusive" when p_value >= 0.05 OR sample_size < required_sample_size.
4. Verify metric_repo.insert_history stores all metric_history fields including win_rate, ci_lower, ci_upper, wins, losses, ties.
5. Verify GET /ab-tests/{id} completed response matches Section 9.8 structure exactly.
6. Run `ruff check src/ tests/` and `pytest tests/test_stats.py -v`. These are known-answer tests — ALL must pass. Report any failures with the expected vs actual values.
7. Check file ownership boundaries maintained.
Report PASS or FAIL with specific issues.
```

#### Commit

```bash
git add -A && git commit -m "Phase 3: Statistical significance (Wilson CI, z-test, power analysis), metric history tracking"
```

---

### Phase 4: Promotion / Rollback + Python SDK (Day 6)

**Goal:** Promote prompt versions to production, roll back, and load production prompts via SDK.

**Milestone M4:** Promote/rollback via API, one-production invariant enforced, SDK loads production prompt.

#### Parallel Work

**Agent core (Pane 0):**

```
You are building the transactional promotion/rollback repository. Read IMPLEMENTATION_PLAN.md Section 7.5 (Promotion / Rollback) VERY carefully — atomicity is critical.

Create this file:

1. `src/db/repositories/promotion_repo.py` — Follow Section 7.5 pseudocode EXACTLY:
   - `async promote(conn, prompt_id, to_version, promoted_by, reason)` → dict:
     a. Begin transaction (async with conn.transaction())
     b. Find current production version: SELECT version FROM prompt_versions WHERE prompt_id=$1 AND status='production'
     c. If from_version == to_version → return {from_version, to_version, action: "no-op"}
     d. Archive current production: UPDATE prompt_versions SET status='archived' WHERE prompt_id=$1 AND status='production'
     e. Promote target: UPDATE prompt_versions SET status='production' WHERE prompt_id=$1 AND version=$2
     f. Determine action: "rollback" if to_version < from_version, else "promote"
     g. Insert into promotions: (prompt_id, from_version, to_version, action, reason, promoted_by)
     h. Return {from_version, to_version, action}
   - `async get_promotion_history(conn, prompt_id)` → returns all promotions for a prompt, ordered by created_at DESC

The one-production invariant is enforced at TWO levels (Section 7.5):
  1. DB: partial unique index idx_pv_one_production (already in schema.sql from Phase 1)
  2. App: this function uses a transaction to atomically archive + promote

Do NOT touch src/api/, src/sdk/, src/main.py, src/config.py, or dashboard/.
```

**Agent api (Pane 1):**

```
You are building the promotion/rollback API endpoints, the SDK endpoint, and the Python SDK client. Read IMPLEMENTATION_PLAN.md Sections 9.4 (POST /prompts/{id}/promote), 9.5 (POST /prompts/{id}/rollback), 9.10 (GET /sdk/prompts/{key}/production), and 7.6 (Python SDK).

Create/update these files:

1. Update `src/api/routes/prompts.py` — Add two endpoints:
   - POST "/{prompt_id}/promote" — Body: {version: int, reason: str}. Calls promotion_repo.promote(conn, prompt_id, version, promoted_by="api-user", reason). Returns 200 {prompt_id, from_version, to_version, action, promoted_at} (Section 9.4).
   - POST "/{prompt_id}/rollback" — Body: {version: int, reason: str}. Same as promote but the repo function automatically detects rollback (to_version < from_version). Returns 200 with action: "rollback" (Section 9.5).
   Import promotion_repo from src.db.repositories.promotion_repo (being built by core in parallel — assume promote(conn, prompt_id, to_version, promoted_by, reason) and get_promotion_history(conn, prompt_id) signatures).

2. `src/api/routes/sdk.py` — FastAPI APIRouter with prefix="/sdk":
   - GET "/prompts/{prompt_key}/production" — looks up prompt by prompt_key, gets production version, returns {prompt_key, version, template, variables, message_type, model_hint, temperature, max_tokens} (Section 9.10). 404 if no production version.
   - GET "/prompts/{prompt_key}/versions/{version}" — returns specific version details (for SDK get_version method).
   Import prompt_repo from src.db.repositories.prompt_repo.

3. Update `src/main.py` — Add: include_router(sdk_router). Import from src.api.routes.sdk.

4. `src/sdk/client.py` — Follow Section 7.6 pseudocode EXACTLY:
   - PromptVersion dataclass: prompt_key, version, template, variables, message_type, model_hint, temperature, max_tokens. Methods: render(variables) using SandboxedEnvironment+StrictUndefined, to_messages(variables) → [{"role": message_type, "content": rendered}].
   - PromptClient class:
     - __init__(base_url="http://localhost:8000", cache_ttl_seconds=300, timeout=10.0)
     - get_production_prompt(prompt_key) → PromptVersion: checks in-memory cache with TTL, fetches from GET /sdk/prompts/{key}/production, caches result. On httpx.HTTPError: fallback to stale cache if available, else re-raise.
     - get_version(prompt_key, version) → PromptVersion: fetches from GET /sdk/prompts/{key}/versions/{version}
     - render(prompt_key, variables) → str: gets production prompt, renders client-side
     - to_messages(prompt_key, variables) → list[dict]: gets production prompt, renders, returns messages
     - clear_cache() → clears in-memory cache
   - Uses httpx.Client (sync, not async — SDK is sync for simplicity of use in applications)

5. `tests/test_api_sdk.py` — Test the SDK endpoint: GET /sdk/prompts/{key}/production returns correct fields after promotion, returns 404 if no production version.

6. `tests/test_sdk_client.py` — Test PromptClient with mocked httpx responses: get_production_prompt returns PromptVersion with correct fields, cache works (second call doesn't hit API within TTL), stale cache fallback on HTTP error, render() produces correct output, to_messages() returns correct format.

Do NOT touch src/db/, src/templating/, src/llm/, src/testing/, scripts/, docker/, or dashboard/.
```

#### Sequential (after parallel)

**Agent core (Pane 0):**

```
Phase 4 integration + promotion tests. The api agent created routes that import from your promotion_repo. Verify:
- promotion_repo.promote(conn, prompt_id, to_version, promoted_by, reason) → dict with from_version, to_version, action
- promotion_repo.get_promotion_history(conn, prompt_id) → list of promotion rows

Create `tests/test_promotion.py` — Follow Section 10.3 EXACTLY:
- test_promote_sets_production_status: promote v2 → v2 has status='production'
- test_promote_archives_previous_production: promote v1 then v2 → v1 archived, v2 production
- test_rollback_to_previous_version: promote v3, then promote v2 → v2 production, v3 archived
- test_one_production_invariant_at_db_level: after promoting v1, directly UPDATE v2 to production must raise (partial unique index)
- test_promotion_audit_trail: 3 promotions (promote v1, promote v2, rollback v1) → promotions table has 3 rows with correct actions
- test_promotions_table_is_immutable: UPDATE promotions and DELETE promotions must raise (trigger blocks)

Run: `pytest tests/test_promotion.py -v`. Report results.
```

#### Review

**Agent reviewer (Pane 3):**

```
Review Phase 4 deliverables against IMPLEMENTATION_PLAN.md:

1. CRITICAL: Verify src/db/repositories/promotion_repo.py uses a transaction (async with conn.transaction()) that atomically archives old production + promotes new + inserts audit row (Section 7.5).
2. Verify POST /prompts/{id}/promote returns {prompt_id, from_version, to_version, action, promoted_at} (Section 9.4).
3. Verify POST /prompts/{id}/rollback works (Section 9.5) — same function, action auto-detected as "rollback" when to_version < from_version.
4. Verify GET /sdk/prompts/{key}/production returns {prompt_key, version, template, variables, message_type, model_hint, temperature, max_tokens} (Section 9.10).
5. Verify src/sdk/client.py PromptClient has: get_production_prompt with TTL cache + stale fallback, get_version, render, to_messages, clear_cache (Section 7.6).
6. Verify PromptVersion.render uses SandboxedEnvironment + StrictUndefined (same as server-side renderer).
7. Run `ruff check src/ tests/` and `pytest tests/test_promotion.py tests/test_api_sdk.py tests/test_sdk_client.py -v`. Report failures.
8. Verify one-production invariant: at most one version per prompt_id has status='production' (enforced by DB partial unique index).
9. Check file ownership boundaries maintained.
Report PASS or FAIL with specific issues.
```

#### Commit

```bash
git add -A && git commit -m "Phase 4: Transactional promotion/rollback, SDK endpoint, Python SDK client with caching"
```

---

### Phase 5: Streamlit Dashboard (Day 7)

**Goal:** Interactive dashboard showing A/B test results, version history, and metric trends.

**Milestone M5:** Streamlit dashboard: overview, A/B results, version history, metric trends.
**Milestone M6:** Full test suite green, documented, docker compose up starts API + DB + dashboard.

#### Parallel Work

**Agent dashboard (Pane 2):**

```
You are building the complete Streamlit dashboard. Read IMPLEMENTATION_PLAN.md Section 7.7 (Streamlit Dashboard Layout) VERY carefully — it contains the exact ASCII wireframe for all 4 pages.

Create these files:

1. `dashboard/app.py` — Follow Section 7.7 pseudocode:
   - st.set_page_config(page_title="Prompt A/B Testing Dashboard", page_icon="🧪", layout="wide")
   - Sidebar: title "🧪 Prompt A/B Testing", radio navigation with ["Overview", "A/B Test Results", "Version History", "Metric Trends"]
   - Sidebar: DB connection status indicator ("● Connected" or "● Disconnected")
   - Route to page render() function based on selection
   - Import pages: overview, ab_test_results, version_history, metric_trends

2. `dashboard/queries.py` — SQL queries for all dashboard data (uses asyncpg or psycopg2 sync):
   - `get_overview_stats(conn)` → {total_prompts, active_tests, prod_prompts} (count queries)
   - `get_prompt_table(conn)` → list of {prompt_key, name, version_count, prod_version, test_count, tags} (JOIN prompts + prompt_versions + ab_tests)
   - `get_all_tests(conn)` → list of ab_tests rows for test selector dropdown
   - `get_test_results_summary(conn, test_id)` → test metadata + per-variant aggregated metrics from metric_history
   - `get_test_item_results(conn, test_id, item_idx)` → per-item A/B results for side-by-side display (Section 9.9 structure)
   - `get_version_history(conn, prompt_id)` → all prompt_versions ordered by version DESC + promotion events from promotions table
   - `get_template_diff(conn, prompt_id, v1, v2)` → templates of two versions for diff display
   - `get_metric_trends(conn, prompt_id, metric_name)` → rows of (test_date, version_number, metric_value) for line charts, ordered by test_date ASC
   - `get_promotion_history(conn, prompt_id)` → promotions rows ordered by created_at DESC

3. `dashboard/pages/overview.py` — Follow Section 7.7 Overview wireframe:
   - `render()` function
   - Three metric cards: Total Prompts, Active Tests, Prod Prompts (st.columns(3) + st.metric)
   - Prompt table: st.dataframe with columns Prompt Key, Versions, Prod Ver, Tests, Tags
   - Use queries.get_overview_stats and queries.get_prompt_table

4. `dashboard/pages/ab_test_results.py` — Follow Section 7.7 A/B Test Results wireframe:
   - `render()` function
   - Test selector dropdown (st.selectbox with all tests)
   - Test info: "summarizer v2 vs v3", Status, Winner, p-value
   - Win-rate bar chart (Plotly go.Figure with horizontal bars for A%, B%, ties%)
   - Wilson 95% CI display: "[ci_lower, ci_upper]"
   - Metric comparison table: st.dataframe with columns Metric, v_A, v_B, Delta (accuracy, latency, cost, output len, total tokens)
   - Sample outputs side-by-side: st.columns(2) showing item selector, rendered output, judge score, judge reasoning for A and B
   - Item navigation: "◀ Prev Item  Item N of M  Next Item ▶"
   - Use queries.get_test_results_summary and queries.get_test_item_results

5. `dashboard/pages/version_history.py` — Follow Section 7.7 Version History wireframe:
   - `render()` function
   - Prompt selector (st.selectbox)
   - Version timeline: list of versions with status badges (draft/staging/production/archived), date, changelog. Highlight production version with "◀ prod"
   - Template diff: select two versions, show line-by-line diff (use difflib or st.code with - / + prefixes)
   - Promotion history: table of promotions (date, from→to, action, reason)
   - Use queries.get_version_history, queries.get_template_diff, queries.get_promotion_history

6. `dashboard/pages/metric_trends.py` — Follow Section 7.7 Metric Trends wireframe:
   - `render()` function
   - Prompt selector + metric selector (Accuracy / Latency / Cost)
   - Plotly line chart: x-axis = test_date, y-axis = selected metric, color = version_number
   - Each point = one A/B test run, line color = prompt version
   - Use queries.get_metric_trends

7. `dashboard/components/charts.py` — Plotly chart builders:
   - `win_rate_bar_chart(stats)` → go.Figure with horizontal bars for win_rate_a, win_rate_b, tie_rate
   - `metric_trend_line_chart(trend_data, metric_name)` → go.Figure line chart with per-version traces
   - `version_timeline_chart(versions)` → optional visual timeline

8. `dashboard/components/tables.py` — Styled dataframe renderers:
   - `render_prompt_table(prompts)` → styled st.dataframe
   - `render_metric_comparison(metrics_a, metrics_b)` → st.dataframe with delta column
   - `render_promotion_history(promotions)` → styled st.dataframe

9. `dashboard/__init__.py`, `dashboard/pages/__init__.py`, `dashboard/components/__init__.py` — empty init files.

Do NOT touch src/ (except dashboard reads from DB directly via queries.py), scripts/, docker/, or docker-compose.yml.
```

**Agent core (Pane 0):**

```
Add the dashboard service to docker-compose.yml. Read IMPLEMENTATION_PLAN.md Section 11.1 — the dashboard service is already defined there.

Update `docker-compose.yml` — Ensure the dashboard service is present (it should already be there from Phase 1, but verify):
  dashboard:
    build:
      context: .
      dockerfile: docker/Dockerfile.dashboard
    ports:
      - "8501:8501"
    environment:
      DATABASE_URL: postgres://prompt_user:${POSTGRES_PASSWORD:-changeme}@postgres:5432/prompt_ab
    depends_on:
      postgres:
        condition: service_healthy
      api:
        condition: service_healthy

Also verify docker/Dockerfile.dashboard copies dashboard/ directory and runs streamlit correctly (Section 11.3).

Then run the end-to-end smoke test (Section 5.9): seed data → run sample test → promote → verify dashboard can query the data. Run: `python scripts/run_sample_test.py` and verify test_results and metric_history have data. Report results.
```

#### Review

**Agent reviewer (Pane 3):**

```
Review Phase 5 deliverables against IMPLEMENTATION_PLAN.md Section 7.7:

1. Verify dashboard/app.py has sidebar navigation with 4 pages and DB connection status (Section 7.7).
2. Verify dashboard/pages/overview.py renders: 3 metric cards (Total Prompts, Active Tests, Prod Prompts) + prompt table with columns (Prompt Key, Versions, Prod Ver, Tests, Tags).
3. Verify dashboard/pages/ab_test_results.py renders: test selector, win-rate bar chart, Wilson CI display, metric comparison table with Delta column, sample outputs side-by-side with judge scores + reasoning, item navigation.
4. Verify dashboard/pages/version_history.py renders: prompt selector, version timeline with status badges + production marker, template diff between versions, promotion history table.
5. Verify dashboard/pages/metric_trends.py renders: prompt + metric selectors, Plotly line chart with per-version traces over time.
6. Verify dashboard/components/charts.py has win_rate_bar_chart and metric_trend_line_chart using Plotly.
7. Verify dashboard/queries.py has all query functions needed by the pages.
8. Verify docker-compose.yml has the dashboard service (Section 11.1).
9. Run `ruff check dashboard/` and the FULL test suite: `pytest -v`. Report any failures.
10. Check file ownership: dashboard agent should only touch dashboard/. core should only touch docker-compose.yml.
Report PASS or FAIL with specific issues.
```

#### Commit

```bash
git add -A && git commit -m "Phase 5: Streamlit dashboard — overview, A/B test results, version history, metric trends"
```

---

## 5. Coordination Rules

### File Ownership Boundaries

| Directory/File | Owner | Agents that may read | Agents that may write |
|----------------|-------|---------------------|----------------------|
| `src/db/` | core | api, dashboard, reviewer | core only |
| `src/templating/` | core | api, reviewer | core only |
| `src/llm/` | core | api, reviewer | core only |
| `src/testing/` | core | api, reviewer | core only |
| `src/api/` | api | core, reviewer | api only |
| `src/sdk/` | api | core, reviewer | api only |
| `src/main.py` | api | core, reviewer | api only |
| `src/config.py` | api | core, reviewer | api only |
| `dashboard/` | dashboard | reviewer | dashboard only |
| `scripts/` | core | api, reviewer | core only |
| `docker/` | core | reviewer | core only |
| `docker-compose.yml` | core | reviewer | core only |
| `pyproject.toml` | core | all | core only |
| `tests/` | core (test_stats, test_templating, test_prompt_repo, test_ab_runner, test_judge, test_promotion) + api (test_api_*, test_sdk_*) | reviewer | core + api (per their test files) |

### Conflict Avoidance Rules

1. **No agent writes to another agent's owned directory.** If a cross-agent dependency is needed (e.g., api needs a new repo function from core), request it via hub messaging — do NOT edit the other agent's files.
2. **Shared `tests/conftest.py`** is owned by core. Api agents who need fixtures should request additions from core.
3. **`src/api/models.py`** is owned by api. If core needs a Pydantic model for a repo function return type, core should use plain dicts/dataclasses instead.
4. **Import contracts**: When api imports from core's modules, api writes imports assuming the function signatures documented in the phase prompts. Core is responsible for matching those signatures. Mismatches are resolved in the sequential integration step.

### Parallel vs Sequential

- **Parallel**: Phases 1, 2, 4 have core and api working on non-overlapping file sets simultaneously.
- **Sequential within a phase**: After parallel work completes, a short integration step runs (core verifies signatures match api's imports, runs scripts).
- **Phase 3**: Mostly core-only (stats engine). Api does a minor update to ab_tests route response format in parallel.
- **Phase 5**: Mostly dashboard-only. Core does a minor docker-compose verification in parallel.
- **Review always runs sequentially** after all parallel + integration work in a phase is complete.

### Blocked Agent Protocol

1. If an agent is blocked waiting for another agent's file to exist (e.g., api needs prompt_repo.py from core), the agent should:
   - Write their code with the import and assumed signature
   - Add a comment: `# TODO: verify import after core agent completes`
   - Continue with other work that doesn't depend on the missing file
   - Do NOT block — move to the next task in the phase
2. If an agent discovers a signature mismatch during integration, fix it in YOUR OWN files (adjust your imports/calls). Do NOT edit the other agent's files — message them via hub to request a change.
3. If an agent hits a fundamental blocker (e.g., DB schema is wrong and all work depends on it), message the owning agent via hub with the specific issue and expected fix.

---

## 6. Quick Reference

### Herdr Commands

```bash
# Pane management
herdr pane split --direction vertical --cwd "$PWD" --no-focus
herdr pane split --direction horizontal --cwd "$PWD" --no-focus
herdr pane select <id>
herdr pane list

# Agent management
herdr agent start <name> --kind codex --pane <id>
herdr agent list
herdr agent stop <name>
herdr agent send <name> "message text"

# Monitoring
herdr agent logs <name>
herdr agent status <name>
```

### Project Verification Commands

```bash
# Start all services
docker compose up -d

# Health check
curl http://localhost:8000/health

# Initialize DB schema
python scripts/init_db.py

# Seed sample data
python scripts/seed_data.py

# Run sample A/B test end-to-end
python scripts/run_sample_test.py

# Run full test suite
pytest -v

# Lint
ruff check src/ tests/ dashboard/

# View dashboard
open http://localhost:8501
```

### Phase Summary

| Phase | Days | Agents (parallel) | Key Deliverable | Commit Message |
|-------|------|-------------------|-----------------|----------------|
| 1 | 1-2 | core + api | DB schema, prompt CRUD, Jinja2 templating | Phase 1: Foundation — DB schema, prompt store, Jinja2 templating, FastAPI prompt CRUD |
| 2 | 3-4 | core + api | A/B test runner, LLM-as-judge, eval/test API | Phase 2: A/B test runner, LLM-as-judge, eval dataset + ab-test API endpoints |
| 3 | 5 | core (+ api minor) | Wilson CI, z-test, power analysis, metric history | Phase 3: Statistical significance (Wilson CI, z-test, power analysis), metric history tracking |
| 4 | 6 | core + api | Promotion/rollback, SDK endpoint, PromptClient | Phase 4: Transactional promotion/rollback, SDK endpoint, Python SDK client with caching |
| 5 | 7 | dashboard (+ core minor) | Streamlit 4-page dashboard | Phase 5: Streamlit dashboard — overview, A/B test results, version history, metric trends |
