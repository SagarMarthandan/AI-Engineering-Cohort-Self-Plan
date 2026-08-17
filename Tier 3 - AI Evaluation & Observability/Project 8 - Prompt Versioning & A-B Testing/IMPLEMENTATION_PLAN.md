# Prompt Versioning & A/B Testing System — Detailed Implementation Plan

> **Elevator pitch:** A system that treats prompts as version-controlled engineering artifacts — not magic incantations. Register prompts with git-like versioning, A/B test variants against evaluation datasets, compute statistical significance on win-rates, track accuracy/latency/cost metrics over time, visualize results in a Streamlit dashboard, and promote or roll back production prompts with a single API call. **Prompt engineering is the new SQL — make it systematic, version-controlled, and measurable.**

> **The parallel:** You already A/B test dashboard changes, dbt model variants, and feature flags. A prompt is just another query against data — except the "query" is natural language and the "result" is an LLM output. This system brings the same rigor: version control (git for prompts), hypothesis testing (A/B test runner), metric tracking (dbt-style incremental metrics), and promotion gates (CI/CD for prompts). The dashboard you build for prompt A/B tests is the same muscle as the experiment analysis dashboard you've already built — win-rate, confidence intervals, sample size, significance threshold.

---

## Table of Contents

1. [Project Goals & Non-Goals](#1-project-goals--non-goals)
2. [Architecture Overview](#2-architecture-overview)
3. [Tech Stack](#3-tech-stack)
4. [Database Schema](#4-database-schema)
5. [Project Structure](#5-project-structure)
6. [Implementation Phases](#6-implementation-phases)
7. [Component Specifications](#7-component-specifications)
8. [Security & Safety Considerations](#8-security--safety-considerations)
9. [API Specification](#9-api-specification)
10. [Testing Strategy](#10-testing-strategy)
11. [Deployment](#11-deployment)
12. [Roadmap & Milestones](#12-roadmap--milestones)
13. [Risk Register](#13-risk-register)
14. [Appendix](#appendix)

---

## 1. Project Goals & Non-Goals

### Goals

| # | Goal | Success Metric |
|---|------|----------------|
| G1 | **Prompt versioning** — every prompt has a unique `prompt_id` + monotonically increasing `version`, with full template history, variables, metadata, and authorship | Any prompt's full version history is queryable; no version is ever lost or mutated |
| G2 | **Jinja2 templating** — prompts support system/user/assistant message types with typed variables | Templates render correctly with provided variables; missing variables raise clear errors |
| G3 | **A/B test runner** — given a `prompt_id` with 2+ versions, run both against an eval dataset and collect metrics (accuracy via LLM-as-judge, latency, token cost, output length) | A/B test produces per-variant metrics across the full dataset with reproducible results |
| G4 | **Statistical significance** — compute win-rate, confidence intervals, and sample size requirements; declare a winner only when statistically significant (p < 0.05) | Winner declaration is gated on significance; inconclusive tests are labeled as such |
| G5 | **Metric tracking over time** — store all results in DB, track metrics per prompt version across multiple test runs | Dashboard shows metric trends per version over time (line charts) |
| G6 | **Streamlit dashboard** — visualize A/B test results (win-rate chart, metric comparison table, sample outputs side-by-side), prompt version history, and metric trends | Dashboard loads from DB and renders all three views without manual data entry |
| G7 | **Rollback / promotion** — promote a prompt version to `production` or roll back to a previous version via API | At most one version per `prompt_id` is `production` at any time; rollback is atomic |
| G8 | **Python SDK** — load the current production prompt in any application with a single function call | `PromptClient.get_production_prompt("summarizer")` returns the rendered template + variables |
| G9 | **Production-deployable** — containerized, documented, testable end-to-end | `docker compose up` starts API + DB + dashboard; health check passes |

### Non-Goals (explicitly out of scope)

- Multi-tenant isolation (v1 is single-team; multi-tenant is a v2 concern)
- Real-time online A/B testing (v1 is offline batch evaluation only)
- Prompt optimization via automatic search (DSPy, OPRO) — this is a tracking and testing system, not an optimizer
- Fine-tuning model weights (prompts are the artifact, not model parameters)
- Streaming LLM responses (v1 collects complete outputs for metric computation)
- User-facing chat UI (the dashboard is for engineers/analysts, not end users)
- Git repository integration (v1 uses DB-backed versioning; git sync is a v2 concern)
- Multi-model comparison (v1 compares prompt variants on a single model; model comparison is a v2 concern)

---

## 2. Architecture Overview

```
                              ┌──────────────────────────────────────────────────┐
                              │              Engineer / Analyst                   │
                              │  (creates prompts, runs A/B tests, views dashboard)│
                              └──────┬───────────────────────────┬───────────────┘
                                     │ REST API                    │ Browser
                              ┌──────▼───────────┐          ┌─────▼──────────────┐
                              │   FastAPI App    │          │  Streamlit         │
                              │  (Prompt Mgmt    │          │  Dashboard         │
                              │   + A/B Runner   │          │  (reads from DB)   │
                              │   + SDK endpoint)│          │                    │
                              └──┬──────┬────┬───┘          └──────┬──────────────┘
                                 │      │    │                     │
                  ┌──────────────┘      │    └─────────────────────┘
                  │              ┌──────┘                         │
                  │              │                                │
           ┌──────▼──────┐  ┌────▼─────────────┐          ┌───────▼──────────┐
           │  Prompt     │  │  A/B Test Runner  │          │   PostgreSQL     │
           │  Store      │  │                   │          │                  │
           │  (CRUD +    │  │  1. Load variants │          │  Tables:         │
           │  versioning)│  │  2. Render w/vars │          │  - prompts       │
           │             │  │  3. Call LLM      │          │  - prompt_versions│
           └──────┬──────┘  │  4. LLM-as-judge  │          │  - eval_datasets  │
                  │         │  5. Collect metrics│         │  - eval_items     │
                  │         │  6. Stats analysis │          │  - ab_tests       │
                  │         └────┬───────────────┘          │  - test_results   │
                  │              │                          │  - metric_history │
                  │              │                          │  - promotions     │
                  └──────────────┼──────────────────────────┤
                                 │                          │
                          ┌──────▼──────────┐      ┌────────▼─────────┐
                          │  LLM Provider   │      │  (shared DB)     │
                          │  (OpenAI /      │      │                  │
                          │   Ollama)       │      └──────────────────┘
                          └─────────────────┘

                    ┌──────────────────────────────────────────┐
                    │           Python SDK (pip installable)    │
                    │  PromptClient.get_production_prompt(id)   │
                    │  → fetches production version from API     │
                    │  → renders Jinja2 template with variables  │
                    │  → returns ready-to-use messages           │
                    └──────────────────────────────────────────┘
```

### Request Flow: Create Prompt Version

```
1. Engineer calls POST /prompts/{prompt_id}/versions
   with: {template, variables, message_type, metadata, changelog}
2. API validates Jinja2 template syntax (compile check)
3. API validates variable definitions (name, type, required, default)
4. API inserts new row in prompt_versions with auto-incremented version number
5. API returns {prompt_id, version, status: "draft", created_at}
```

### Request Flow: Run A/B Test

```
1. Engineer calls POST /ab-tests
   with: {prompt_id, version_a, version_b, eval_dataset_id, judge_model, sample_size}
2. A/B Test Runner:
   a. Load both prompt versions from prompt_versions
   b. Load eval dataset (eval_items: input variables + expected output)
   c. For each eval item, for each variant:
      - Render template with input variables (Jinja2)
      - Call LLM → raw output
      - Measure latency_ms, token_count, output_length
      - Call LLM-as-judge: compare output to expected output → score (0-1)
   d. Compute per-variant aggregate metrics:
      - mean_accuracy, mean_latency, mean_cost, mean_output_length
   e. Compute statistical significance:
      - win_rate (variant wins on accuracy per item)
      - Wilson confidence interval on win-rate
      - Two-proportion z-test p-value
      - Required sample size for desired power
   f. Declare winner ONLY if p < 0.05 AND sample size adequate
   g. Store all results in test_results + metric_history
3. API returns {test_id, status: "completed", winner: version_a|version_b|null, metrics, stats}
```

### Request Flow: Promote / Rollback

```
1. Engineer calls POST /prompts/{prompt_id}/promote
   with: {version, reason}
2. API begins DB transaction:
   a. UPDATE prompt_versions SET status='archived' WHERE prompt_id=? AND status='production'
   b. UPDATE prompt_versions SET status='production' WHERE prompt_id=? AND version=?
   c. INSERT INTO promotions (prompt_id, from_version, to_version, reason, promoted_by)
   d. COMMIT
3. SDK calls now resolve to the newly promoted version
4. Dashboard reflects the change in version history
```

---

## 3. Tech Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **Language** | Python 3.11+ | Ecosystem for AI/ML, type hints, async support |
| **Web Framework** | FastAPI | Async, auto OpenAPI docs, Pydantic validation — same as the analytics engineer's existing API experience |
| **Database** | PostgreSQL 16 | Relational store for prompts, versions, test results, metrics; JSONB for flexible metadata; the analytics engineer already knows SQL deeply |
| **Templating** | Jinja2 | Industry standard for Python templating; supports variables, conditionals, loops; safe sandboxed rendering |
| **Dashboard** | Streamlit | Python-native, fast to build, interactive charts (Plotly); the analytics engineer already thinks in dashboards |
| **LLM Client** | openai (Python SDK) | Supports OpenAI API; configurable base_url for Ollama/local LLMs |
| **Statistics** | scipy + statsmodels | Z-tests, confidence intervals, power analysis; numpy for array math |
| **Charts** | Plotly (via Streamlit) | Interactive, hoverable, exportable — same tool the analytics engineer uses for experiment dashboards |
| **SDK** | httpx + Jinja2 | Async HTTP client for API calls; Jinja2 for client-side rendering (or server-side render option) |
| **Containerization** | Docker + Docker Compose | Reproducible dev + prod; three services (API, DB, dashboard) |
| **Testing** | pytest + pytest-asyncio + httpx | Async test support, FastAPI-native test client |
| **Linting** | ruff | Fast, replaces flake8 + isort + black |
| **Migrations** | Alembic (optional) or raw SQL | Schema versioning; raw SQL is simpler for v1 |

### Why PostgreSQL (not a dedicated prompt store like LangSmith/Promptflow)?

**The analytics engineer already operates PostgreSQL.** Prompt versioning is a relational problem: prompts have versions, versions have tests, tests have results, results have metrics. This is a classic slowly-changing-dimension pattern — the same pattern used in dbt for tracking dimension changes over time. Using Postgres means:

1. **SQL-native metric tracking** — `SELECT version, avg(accuracy) FROM test_results GROUP BY version` is the same query you'd write in any analytics warehouse
2. **JSONB for flexible metadata** — prompt metadata (tags, description, changelog) varies; JSONB handles this without schema migrations
3. **Transactional promotion** — `BEGIN; UPDATE ... SET status='archived'; UPDATE ... SET status='production'; INSERT INTO promotions ...; COMMIT;` guarantees atomic rollback
4. **No new infrastructure** — one Postgres instance serves prompts, tests, results, and metrics

### Why Jinja2 (not f-strings or custom templating)?

Jinja2 provides:
- **Sandboxed execution** — `SandboxedEnvironment` prevents template injection from executing arbitrary code
- **Variable validation** — `StrictUndefined` raises on missing variables (no silent `None` substitution)
- **Conditionals and loops** — prompts can branch on variable values (`{% if context %}...{% endif %}`)
- **Familiarity** — widely used in Python web frameworks; the analytics engineer likely already knows it from Airflow/DAG templating

---

## 4. Database Schema

### 4.1 Extensions

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
```

### 4.2 Tables

```sql
-- =====================================================
-- PROMPTS: logical prompt identity (one row per named prompt)
-- Think of this as the "prompt name" — e.g. "summarizer", "qa-extractor"
-- Versions hang off this via prompt_versions
-- =====================================================
CREATE TABLE prompts (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    prompt_key      TEXT NOT NULL UNIQUE,              -- e.g. 'summarizer', 'qa-extractor'
    name            TEXT NOT NULL,                     -- human-readable name
    description     TEXT,
    tags            TEXT[] NOT NULL DEFAULT '{}',      -- e.g. {'summarization', 'customer-support'}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_prompts_tags ON prompts USING GIN(tags);
CREATE INDEX idx_prompts_key ON prompts(prompt_key);

-- =====================================================
-- PROMPT_VERSIONS: each version of a prompt (git-like)
-- This is the core table — every change to a prompt
-- creates a NEW row, never mutates an existing one.
-- Status lifecycle: draft → staging → production → archived
-- At most ONE version per prompt_id can be 'production'
-- =====================================================
CREATE TABLE prompt_versions (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    prompt_id       UUID NOT NULL REFERENCES prompts(id) ON DELETE CASCADE,
    version         INT NOT NULL,                      -- monotonically increasing per prompt_id
    template        TEXT NOT NULL,                     -- Jinja2 template string
    variables       JSONB NOT NULL DEFAULT '[]',       -- [{name, type, required, default, description}]
    message_type    TEXT NOT NULL DEFAULT 'user',      -- 'system', 'user', 'assistant'
    model_hint      TEXT,                              -- recommended model (e.g. 'gpt-4o-mini')
    temperature     REAL DEFAULT 0.0,                  -- recommended temperature
    max_tokens      INT DEFAULT 1024,                  -- recommended max output tokens
    metadata        JSONB NOT NULL DEFAULT '{}',       -- changelog, author notes, tags
    status          TEXT NOT NULL DEFAULT 'draft',     -- draft, staging, production, archived
    changelog       TEXT,                              -- what changed from previous version
    created_by      TEXT NOT NULL,                     -- author email/username
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    UNIQUE(prompt_id, version),
    CONSTRAINT valid_status CHECK (status IN ('draft', 'staging', 'production', 'archived')),
    CONSTRAINT valid_message_type CHECK (message_type IN ('system', 'user', 'assistant'))
);

CREATE INDEX idx_pv_prompt_id ON prompt_versions(prompt_id);
CREATE INDEX idx_pv_status ON prompt_versions(status);
CREATE INDEX idx_pv_prompt_status ON prompt_versions(prompt_id, status);

-- Enforce: at most one 'production' version per prompt_id
CREATE UNIQUE INDEX idx_pv_one_production
    ON prompt_versions(prompt_id)
    WHERE status = 'production';

-- =====================================================
-- EVAL_DATASETS: named evaluation datasets
-- A dataset is a collection of test cases (eval_items)
-- Each item has input variables + an expected output
-- =====================================================
CREATE TABLE eval_datasets (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name            TEXT NOT NULL UNIQUE,              -- e.g. 'summarizer-golden-v1'
    description     TEXT,
    item_count      INT NOT NULL DEFAULT 0,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_by      TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- =====================================================
-- EVAL_ITEMS: individual test cases within a dataset
-- input_variables: the variables to render into the prompt template
-- expected_output: the reference answer for LLM-as-judge comparison
-- =====================================================
CREATE TABLE eval_items (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    dataset_id      UUID NOT NULL REFERENCES eval_datasets(id) ON DELETE CASCADE,
    item_idx        INT NOT NULL,                      -- position within dataset
    input_variables JSONB NOT NULL,                    -- {"article": "...", "max_words": 50}
    expected_output TEXT,                              -- reference answer (for judge comparison)
    metadata        JSONB NOT NULL DEFAULT '{}',       -- tags, difficulty, category

    UNIQUE(dataset_id, item_idx)
);

CREATE INDEX idx_eval_items_dataset ON eval_items(dataset_id);

-- =====================================================
-- AB_TESTS: metadata for each A/B test run
-- A test compares two versions of a prompt on a dataset
-- =====================================================
CREATE TABLE ab_tests (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    prompt_id       UUID NOT NULL REFERENCES prompts(id) ON DELETE CASCADE,
    version_a       INT NOT NULL,                      -- version number (not version_id)
    version_b       INT NOT NULL,
    dataset_id      UUID NOT NULL REFERENCES eval_datasets(id),
    judge_model     TEXT NOT NULL DEFAULT 'gpt-4o-mini', -- model for LLM-as-judge
    target_model    TEXT NOT NULL DEFAULT 'gpt-4o-mini', -- model being tested
    sample_size     INT,                               -- null = use all items in dataset
    status          TEXT NOT NULL DEFAULT 'pending',   -- pending, running, completed, failed
    winner          TEXT,                              -- 'A', 'B', 'inconclusive', null (if not completed)
    p_value         REAL,                              -- statistical significance
    confidence_level REAL DEFAULT 0.95,                -- e.g. 0.95 for 95% CI
    notes           TEXT,
    created_by      TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at    TIMESTAMPTZ,

    CONSTRAINT valid_status CHECK (status IN ('pending', 'running', 'completed', 'failed')),
    CONSTRAINT valid_winner CHECK (winner IN ('A', 'B', 'inconclusive') OR winner IS NULL),
    CONSTRAINT different_versions CHECK (version_a <> version_b)
);

CREATE INDEX idx_abt_prompt ON ab_tests(prompt_id);
CREATE INDEX idx_abt_status ON ab_tests(status);

-- =====================================================
-- TEST_RESULTS: per-item results for each A/B test
-- One row per (test_id, eval_item_id, variant)
-- This is the fact table — every metric is computed from these rows
-- =====================================================
CREATE TABLE test_results (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    test_id         UUID NOT NULL REFERENCES ab_tests(id) ON DELETE CASCADE,
    eval_item_id    UUID NOT NULL REFERENCES eval_items(id) ON DELETE CASCADE,
    variant         TEXT NOT NULL,                     -- 'A' or 'B'
    version_number  INT NOT NULL,                      -- which prompt version

    -- LLM output
    rendered_prompt TEXT NOT NULL,                     -- the fully rendered prompt sent to LLM
    raw_output      TEXT NOT NULL,                     -- LLM response
    output_length   INT NOT NULL,                      -- character count of output

    -- Metrics
    accuracy_score  REAL,                              -- LLM-as-judge score (0.0 - 1.0)
    latency_ms      INT NOT NULL,                      -- time to LLM response
    prompt_tokens   INT,                               -- input token count
    completion_tokens INT,                              -- output token count
    total_tokens    INT,                               -- prompt + completion
    cost_usd        REAL,                              -- estimated cost in USD

    -- Judge details
    judge_reasoning TEXT,                              -- LLM-as-judge explanation
    judge_raw       TEXT,                              -- raw judge response

    error           TEXT,                              -- null if success; error message if failed
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT valid_variant CHECK (variant IN ('A', 'B'))
);

CREATE INDEX idx_tr_test ON test_results(test_id);
CREATE INDEX idx_tr_test_variant ON test_results(test_id, variant);
CREATE INDEX idx_tr_item ON test_results(eval_item_id);

-- =====================================================
-- METRIC_HISTORY: aggregated metrics per prompt version per test run
-- Pre-computed for dashboard performance (avoid re-aggregating on every page load)
-- One row per (test_id, version_number)
-- =====================================================
CREATE TABLE metric_history (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    prompt_id           UUID NOT NULL REFERENCES prompts(id) ON DELETE CASCADE,
    version_number      INT NOT NULL,
    test_id             UUID NOT NULL REFERENCES ab_tests(id) ON DELETE CASCADE,
    test_date           TIMESTAMPTZ NOT NULL,           -- denormalized from ab_tests.created_at

    -- Aggregate metrics
    mean_accuracy       REAL,
    median_accuracy     REAL,
    std_accuracy        REAL,
    mean_latency_ms     REAL,
    median_latency_ms   REAL,
    mean_cost_usd       REAL,
    total_cost_usd      REAL,
    mean_output_length  REAL,
    mean_total_tokens   REAL,

    -- A/B comparison metrics (relative to the other variant in same test)
    win_rate            REAL,                           -- fraction of items where this variant won
    wins                INT,
    losses              INT,
    ties                INT,
    ci_lower            REAL,                           -- Wilson CI lower bound on win-rate
    ci_upper            REAL,                           -- Wilson CI upper bound on win-rate

    sample_size         INT NOT NULL,

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    UNIQUE(prompt_id, version_number, test_id)
);

CREATE INDEX idx_mh_prompt_version ON metric_history(prompt_id, version_number);
CREATE INDEX idx_mh_prompt_date ON metric_history(prompt_id, test_date DESC);

-- =====================================================
-- PROMOTIONS: audit log of all promotion/rollback events
-- Append-only — tracks who promoted what, when, and why
-- =====================================================
CREATE TABLE promotions (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    prompt_id       UUID NOT NULL REFERENCES prompts(id) ON DELETE CASCADE,
    from_version    INT,                               -- null if first promotion
    to_version      INT NOT NULL,
    action          TEXT NOT NULL,                     -- 'promote', 'rollback'
    reason          TEXT,
    promoted_by     TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT valid_action CHECK (action IN ('promote', 'rollback'))
);

CREATE INDEX idx_promo_prompt ON promotions(prompt_id);
CREATE INDEX idx_promo_date ON promotions(created_at DESC);

-- Trigger: prevent UPDATE and DELETE on promotions (append-only)
CREATE OR REPLACE FUNCTION prevent_promotion_modification()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'promotions is append-only; UPDATE and DELETE are forbidden';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER no_promo_update
    BEFORE UPDATE ON promotions
    FOR EACH ROW EXECUTE FUNCTION prevent_promotion_modification();

CREATE TRIGGER no_promo_delete
    BEFORE DELETE ON promotions
    FOR EACH ROW EXECUTE FUNCTION prevent_promotion_modification();
```

### 4.3 Schema Diagram (Entity Relationships)

```
prompts (1) ──── (N) prompt_versions
    │                        │
    │                        │ (version number referenced)
    │                        │
    │ (N) ──── (N) ab_tests──┘
    │              │
    │              │ (1) ──── (N) test_results ──── (1) eval_items
    │              │                                       │
    │              │                                  (N) ─┘ (1) eval_datasets
    │              │
    │              │ (1) ──── (N) metric_history
    │
    │ (1) ──── (N) promotions
    │
    └── prompt_key is the stable identifier used by the SDK
```

---

## 5. Project Structure

```
prompt-AB-testing/
├── docker-compose.yml
├── .env.example
├── pyproject.toml
├── README.md
├── IMPLEMENTATION_PLAN.md          ← this file
├── agent.md                      # Herdr multi-agent orchestration guide
│
├── src/
│   ├── __init__.py
│   ├── main.py                     # FastAPI app factory + lifespan
│   ├── config.py                   # Pydantic Settings (env vars)
│   │
│   ├── db/
│   │   ├── __init__.py
│   │   ├── connection.py           # asyncpg pool
│   │   ├── schema.sql              # full DDL (tables, indexes, triggers)
│   │   └── repositories/
│   │       ├── __init__.py
│   │       ├── prompt_repo.py      # CRUD for prompts + prompt_versions
│   │       ├── eval_repo.py        # CRUD for eval_datasets + eval_items
│   │       ├── test_repo.py        # CRUD for ab_tests + test_results
│   │       ├── metric_repo.py      # CRUD for metric_history + aggregations
│   │       └── promotion_repo.py   # promotion/rollback logic (transactional)
│   │
│   ├── templating/
│   │   ├── __init__.py
│   │   ├── renderer.py             # Jinja2 SandboxedEnvironment + StrictUndefined
│   │   └── validator.py            # template compile check + variable validation
│   │
│   ├── llm/
│   │   ├── __init__.py
│   │   ├── client.py               # OpenAI-compatible client (supports Ollama via base_url)
│   │   ├── judge.py                # LLM-as-judge: compare output to expected
│   │   └── cost.py                 # token cost estimation per model
│   │
│   ├── testing/
│   │   ├── __init__.py
│   │   ├── runner.py               # A/B test orchestrator (the core engine)
│   │   ├── stats.py                # statistical significance: z-test, CI, power
│   │   └── metrics.py              # aggregate per-variant metrics from test_results
│   │
│   ├── api/
│   │   ├── __init__.py
│   │   ├── routes/
│   │   │   ├── __init__.py
│   │   │   ├── prompts.py          # CRUD + version management + promote/rollback
│   │   │   ├── eval_datasets.py    # CRUD for eval datasets + items
│   │   │   ├── ab_tests.py         # create, list, get results
│   │   │   └── sdk.py              # SDK endpoint: GET /sdk/prompts/{key}/production
│   │   ├── models.py               # Pydantic request/response models
│   │   └── dependencies.py         # FastAPI dependency injection (get db pool)
│   │
│   └── sdk/
│       ├── __init__.py
│       └── client.py               # PromptClient — pip-installable SDK
│
├── dashboard/
│   ├── __init__.py
│   ├── app.py                      # Streamlit entry point
│   ├── pages/
│   │   ├── __init__.py
│   │   ├── overview.py             # all prompts + their production versions
│   │   ├── ab_test_results.py      # win-rate chart, metric table, sample outputs
│   │   ├── version_history.py      # prompt version timeline + diffs
│   │   └── metric_trends.py        # metric trends over time per version
│   ├── components/
│   │   ├── __init__.py
│   │   ├── charts.py               # Plotly chart builders
│   │   └── tables.py               # styled dataframe renderers
│   └── queries.py                  # SQL queries for dashboard data
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py                 # pytest fixtures: test DB, test client, seed data
│   ├── test_templating.py          # Jinja2 rendering, variable validation, sandbox
│   ├── test_prompt_repo.py         # prompt CRUD, versioning, immutability
│   ├── test_promotion.py           # promote/rollback atomicity, one-production constraint
│   ├── test_ab_runner.py           # A/B test execution, metric collection
│   ├── test_stats.py               # statistical significance, CI, power analysis
│   ├── test_judge.py               # LLM-as-judge scoring (mocked LLM)
│   ├── test_api_prompts.py         # API endpoints for prompt management
│   ├── test_api_ab_tests.py        # API endpoints for A/B tests
│   ├── test_api_sdk.py             # SDK endpoint returns production prompt
│   └── test_sdk_client.py          # SDK client integration
│
├── scripts/
│   ├── init_db.py                  # Run schema.sql against fresh database
│   ├── seed_data.py                # Insert sample prompts, versions, eval datasets
│   └── run_sample_test.py          # End-to-end: create prompt → A/B test → view results
│
└── docker/
    ├── Dockerfile.api              # FastAPI app image
    ├── Dockerfile.dashboard        # Streamlit dashboard image
    └── postgres/
        └── init.sql                # extensions on first boot
```

---

## 6. Implementation Phases

### Phase 1: Foundation — DB + Prompt Store + Templating (Days 1-2)

**Goal:** PostgreSQL running, schema deployed, prompt CRUD API works, Jinja2 templating validated.

| Step | Task | Deliverable |
|------|------|-------------|
| 1.1 | Write `docker-compose.yml` with Postgres + API service | `docker compose up` starts both |
| 1.2 | Write `docker/postgres/init.sql` (extensions) | DB initializes on first boot |
| 1.3 | Write `src/db/schema.sql` (all 7 tables, indexes, triggers) | Schema deploys via `scripts/init_db.py` |
| 1.4 | Write `src/config.py` (Pydantic Settings) | Env-driven config |
| 1.5 | Write `src/db/connection.py` (asyncpg pool) | DB connection pool works |
| 1.6 | Write `src/templating/renderer.py` (SandboxedEnvironment + StrictUndefined) | Templates render with variables |
| 1.7 | Write `src/templating/validator.py` (compile check + variable type validation) | Invalid templates rejected with clear errors |
| 1.8 | Write `src/db/repositories/prompt_repo.py` (create prompt, create version, list versions, get by key+version) | Full prompt versioning CRUD |
| 1.9 | Write `src/api/routes/prompts.py` (REST endpoints) | API creates/reads prompts and versions |
| 1.10 | Write `src/main.py` (FastAPI app factory) | App starts, health check responds |
| 1.11 | Write `tests/test_templating.py` + `tests/test_prompt_repo.py` | Templating + repo tests pass |
| 1.12 | Write `scripts/seed_data.py` (sample prompts + eval dataset) | Seed data loads |

**Verification:** `docker compose up` → `POST /prompts` creates a prompt → `POST /prompts/{id}/versions` creates v1 → `GET /prompts/{id}/versions` returns version history. Template with `{{ article }}` renders correctly with `{"article": "..."}`. Missing variable raises `StrictUndefined` error.

---

### Phase 2: A/B Test Runner + LLM-as-Judge (Days 3-4)

**Goal:** Run an A/B test comparing two prompt versions on an eval dataset, collect per-item metrics.

| Step | Task | Deliverable |
|------|------|-------------|
| 2.1 | Write `src/llm/client.py` (OpenAI-compatible, configurable base_url for Ollama) | LLM calls work (test with gpt-4o-mini) |
| 2.2 | Write `src/llm/cost.py` (per-model token pricing table) | Cost estimation per call |
| 2.3 | Write `src/llm/judge.py` (LLM-as-judge: compare output to expected, return 0-1 score + reasoning) | Judge produces scores |
| 2.4 | Write `src/db/repositories/eval_repo.py` (dataset + item CRUD) | Eval datasets loadable via API |
| 2.5 | Write `src/db/repositories/test_repo.py` (create test, insert results) | Test + results persist to DB |
| 2.6 | Write `src/testing/runner.py` (orchestrator: load variants → render → LLM → judge → metrics → store) | Full A/B test runs end-to-end |
| 2.7 | Write `src/testing/metrics.py` (aggregate per-variant: mean accuracy, latency, cost, output length) | Aggregate metrics computed |
| 2.8 | Write `src/api/routes/eval_datasets.py` + `src/api/routes/ab_tests.py` | API triggers A/B tests |
| 2.9 | Write `tests/test_ab_runner.py` + `tests/test_judge.py` (mocked LLM) | Runner + judge tests pass |
| 2.10 | Write `scripts/run_sample_test.py` (end-to-end demo) | Sample test runs and produces results |

**Verification:** Seed two prompt versions (v1 and v2 of "summarizer") → load a 10-item eval dataset → `POST /ab-tests` → test completes → `GET /ab-tests/{id}/results` returns per-item metrics for both variants. Accuracy scores are between 0 and 1. Latency and token counts are recorded.

---

### Phase 3: Statistical Significance + Metric Tracking (Day 5)

**Goal:** Compute win-rate, confidence intervals, p-values, and sample size requirements. Store aggregated metrics for trend tracking.

| Step | Task | Deliverable |
|------|------|-------------|
| 3.1 | Write `src/testing/stats.py` (Wilson CI, two-proportion z-test, power analysis) | Stats computed correctly |
| 3.2 | Wire stats into `runner.py` — after collecting per-item results, compute significance | Winner declared only if p < 0.05 |
| 3.3 | Write `src/db/repositories/metric_repo.py` (insert aggregated metrics into metric_history) | Metrics persisted per test run |
| 3.4 | Update `ab_tests` row with `winner`, `p_value`, `completed_at` after test finishes | Test status reflects outcome |
| 3.5 | Write `tests/test_stats.py` (known-answer tests for z-test, CI, power) | Stats tests pass with known values |
| 3.6 | Run `scripts/run_sample_test.py` with larger dataset (50+ items) → verify significance | Significance threshold works |

**Verification:** Run A/B test with 50 items where variant A is clearly better → `winner = 'A'`, `p_value < 0.05`. Run A/B test with 5 items where results are mixed → `winner = 'inconclusive'`. `metric_history` table has rows with `win_rate`, `ci_lower`, `ci_upper` for each variant.

---

### Phase 4: Promotion / Rollback + Python SDK (Day 6)

**Goal:** Promote prompt versions to production, roll back, and load production prompts via SDK.

| Step | Task | Deliverable |
|------|------|-------------|
| 4.1 | Write `src/db/repositories/promotion_repo.py` (transactional promote/rollback) | Atomic promotion with one-production constraint |
| 4.2 | Add `POST /prompts/{id}/promote` and `POST /prompts/{id}/rollback` to API | Promotion works via API |
| 4.3 | Write `src/api/routes/sdk.py` (`GET /sdk/prompts/{key}/production` → returns production version + template + variables) | SDK endpoint returns production prompt |
| 4.4 | Write `src/sdk/client.py` (`PromptClient` class: `get_production_prompt(key)`, `get_version(key, version)`, `render(key, variables)`) | SDK loads + renders prompts |
| 4.5 | Write `tests/test_promotion.py` (atomicity, one-production constraint, rollback) | Promotion tests pass |
| 4.6 | Write `tests/test_api_sdk.py` + `tests/test_sdk_client.py` | SDK endpoint + client tests pass |

**Verification:** Promote v2 of "summarizer" to production → `GET /sdk/prompts/summarizer/production` returns v2. Roll back to v1 → SDK now returns v1. Attempt to promote v2 when v3 is already production → v3 archived, v2 promoted (atomic). `promotions` table has audit trail.

---

### Phase 5: Streamlit Dashboard (Day 7)

**Goal:** Interactive dashboard showing A/B test results, version history, and metric trends.

| Step | Task | Deliverable |
|------|------|-------------|
| 5.1 | Write `dashboard/app.py` (Streamlit entry, sidebar navigation) | Dashboard loads |
| 5.2 | Write `dashboard/queries.py` (SQL queries for all dashboard data) | Data loads from DB |
| 5.3 | Write `dashboard/pages/overview.py` (all prompts + production versions + test count) | Overview page renders |
| 5.4 | Write `dashboard/pages/ab_test_results.py` (win-rate chart, metric comparison table, sample outputs side-by-side) | A/B results page renders |
| 5.5 | Write `dashboard/pages/version_history.py` (version timeline + template diff) | Version history renders |
| 5.6 | Write `dashboard/pages/metric_trends.py` (metric line charts over time per version) | Trends page renders |
| 5.7 | Write `dashboard/components/charts.py` (Plotly chart builders) | Charts are interactive |
| 5.8 | Add dashboard service to `docker-compose.yml` | Dashboard runs in Docker |
| 5.9 | End-to-end smoke test: seed → run test → promote → view dashboard | Full flow visible in dashboard |

**Verification:** `docker compose up` → open `localhost:8501` → overview shows seeded prompts → select an A/B test → win-rate chart renders → metric comparison table shows both variants → sample outputs display side-by-side → version history shows promotion events → metric trends show accuracy/latency/cost over time.

---

## 7. Component Specifications

### 7.1 Jinja2 Templating Engine (`src/templating/renderer.py`)

```python
"""
Sandboxed Jinja2 rendering for prompt templates.

Key decisions:
- SandboxedEnvironment: prevents template injection from executing
  arbitrary Python code (e.g. {{ config.__class__.__init__.__globals__ }})
- StrictUndefined: raises UndefinedError on missing variables — no silent
  None substitution. A prompt with a missing variable is a bug, not a feature.
- Autoescape disabled: we're generating LLM prompts, not HTML. Escaping
  would corrupt the prompt text.
"""

from jinja2 import Environment, StrictUndefined
from jinja2.sandbox import SandboxedEnvironment


def create_env() -> Environment:
    return SandboxedEnvironment(
        undefined=StrictUndefined,
        autoescape=False,
        trim_blocks=True,
        lstrip_blocks=True,
    )


def render_template(template_str: str, variables: dict[str, any]) -> str:
    """Render a Jinja2 template string with the given variables.

    Raises:
        TemplateSyntaxError: if template has syntax errors
        UndefinedError: if a required variable is missing
    """
    env = create_env()
    template = env.from_string(template_str)
    return template.render(**variables)


def validate_template(template_str: str, variable_defs: list[dict]) -> list[str]:
    """Compile-check a template and verify all referenced variables are declared.

    Returns list of error messages (empty = valid).
    """
    errors = []
    env = create_env()

    # 1. Compile check — catches syntax errors
    try:
        env.from_string(template_str)
    except Exception as e:
        errors.append(f"Template syntax error: {e}")
        return errors  # can't check variables if it doesn't compile

    # 2. Extract variable names from template (ast analysis)
    from jinja2 import meta
    ast = env.parse(template_str)
    referenced_vars = meta.find_undeclared_variables(ast)

    # 3. Check all referenced variables are declared
    declared_names = {v["name"] for v in variable_defs}
    for var_name in referenced_vars:
        if var_name not in declared_names:
            errors.append(
                f"Template references '{var_name}' but it is not declared "
                f"in variables. Declared: {declared_names}"
            )

    return errors
```

### 7.2 A/B Test Runner (`src/testing/runner.py`)

```python
"""
The A/B test orchestrator — the core engine of the system.

Given a prompt_id with two versions and an eval dataset:
  1. Load both prompt versions (templates + variables)
  2. Load eval items (input_variables + expected_output)
  3. For each eval item, for each variant:
     a. Render the template with input variables
     b. Call the target LLM → raw output
     c. Measure latency, token count, cost
     d. Call LLM-as-judge → accuracy score (0-1)
  4. Compute per-variant aggregate metrics
  5. Compute statistical significance (win-rate, CI, p-value)
  6. Declare winner (only if significant)
  7. Store all results in DB

This is the "experiment runner" — same concept as an A/B test
runner for dashboard changes, except the "treatment" is a prompt
variant and the "metric" is LLM output quality.
"""

import asyncio
import time
from dataclasses import dataclass


@dataclass
class ItemResult:
    eval_item_id: str
    variant: str          # 'A' or 'B'
    version_number: int
    rendered_prompt: str
    raw_output: str
    output_length: int
    accuracy_score: float | None
    latency_ms: int
    prompt_tokens: int
    completion_tokens: int
    total_tokens: int
    cost_usd: float
    judge_reasoning: str
    error: str | None


async def run_ab_test(
    test_id: str,
    prompt_id: str,
    version_a: int,
    version_b: int,
    dataset_id: str,
    target_model: str,
    judge_model: str,
    sample_size: int | None,
) -> None:
    """Execute a full A/B test. Updates ab_tests row + inserts test_results."""

    # 1. Load prompt versions
    version_a_row = await prompt_repo.get_version(prompt_id, version_a)
    version_b_row = await prompt_repo.get_version(prompt_id, version_b)

    # 2. Load eval items
    eval_items = await eval_repo.get_items(dataset_id)
    if sample_size:
        eval_items = eval_items[:sample_size]

    # 3. Mark test as running
    await test_repo.update_status(test_id, "running")

    # 4. Run both variants on all items (concurrent within each variant)
    #    Process items concurrently with a semaphore to avoid rate limits
    semaphore = asyncio.Semaphore(5)  # max 5 concurrent LLM calls

    async def run_single(
        item, version_row, variant_label
    ) -> ItemResult:
        async with semaphore:
            return await _run_single_item(
                item, version_row, variant_label,
                target_model, judge_model
            )

    tasks_a = [run_single(item, version_a_row, "A") for item in eval_items]
    tasks_b = [run_single(item, version_b_row, "B") for item in eval_items]

    results_a = await asyncio.gather(*tasks_a)
    results_b = await asyncio.gather(*tasks_b)

    all_results = results_a + results_b

    # 5. Store per-item results
    for result in all_results:
        await test_repo.insert_result(test_id, result)

    # 6. Compute aggregate metrics per variant
    metrics_a = compute_aggregate_metrics(results_a)
    metrics_b = compute_aggregate_metrics(results_b)

    # 7. Compute statistical significance
    stats = compute_significance(
        results_a, results_b, confidence_level=0.95
    )

    # 8. Declare winner
    winner = _declare_winner(stats)

    # 9. Store aggregated metrics in metric_history
    await metric_repo.insert_history(
        prompt_id, version_a, test_id, metrics_a, stats, "A"
    )
    await metric_repo.insert_history(
        prompt_id, version_b, test_id, metrics_b, stats, "B"
    )

    # 10. Update ab_tests row
    await test_repo.complete_test(
        test_id, winner=winner, p_value=stats.p_value
    )


async def _run_single_item(
    item, version_row, variant_label,
    target_model, judge_model,
) -> ItemResult:
    """Run a single eval item through one prompt variant."""
    try:
        # a. Render template with input variables
        rendered = render_template(
            version_row.template,
            item.input_variables
        )

        # b. Call target LLM
        start = time.monotonic()
        response = await llm_client.generate(
            model=target_model,
            messages=[{
                "role": version_row.message_type,
                "content": rendered,
            }],
            temperature=version_row.temperature,
            max_tokens=version_row.max_tokens,
        )
        latency_ms = int((time.monotonic() - start) * 1000)

        raw_output = response.content
        output_length = len(raw_output)

        # c. Estimate cost
        cost = estimate_cost(
            target_model,
            response.prompt_tokens,
            response.completion_tokens
        )

        # d. LLM-as-judge: compare output to expected
        if item.expected_output:
            judge_result = await judge.score(
                judge_model=judge_model,
                prompt=rendered,
                output=raw_output,
                expected=item.expected_output,
            )
            accuracy_score = judge_result.score
            judge_reasoning = judge_result.reasoning
        else:
            accuracy_score = None
            judge_reasoning = "No expected output provided"

        return ItemResult(
            eval_item_id=item.id,
            variant=variant_label,
            version_number=version_row.version,
            rendered_prompt=rendered,
            raw_output=raw_output,
            output_length=output_length,
            accuracy_score=accuracy_score,
            latency_ms=latency_ms,
            prompt_tokens=response.prompt_tokens,
            completion_tokens=response.completion_tokens,
            total_tokens=response.total_tokens,
            cost_usd=cost,
            judge_reasoning=judge_reasoning,
            error=None,
        )

    except Exception as e:
        return ItemResult(
            eval_item_id=item.id,
            variant=variant_label,
            version_number=version_row.version,
            rendered_prompt="",
            raw_output="",
            output_length=0,
            accuracy_score=None,
            latency_ms=0,
            prompt_tokens=0,
            completion_tokens=0,
            total_tokens=0,
            cost_usd=0.0,
            judge_reasoning="",
            error=str(e),
        )


def _declare_winner(stats) -> str | None:
    """Declare winner only if statistically significant."""
    if stats.p_value < 0.05 and stats.sample_size >= stats.required_sample_size:
        return "A" if stats.win_rate_a > 0.5 else "B"
    return "inconclusive"
```

### 7.3 Statistical Significance (`src/testing/stats.py`)

```python
"""
Statistical significance computation for A/B prompt testing.

Concepts (mapped from the analytics engineer's A/B testing background):

  - Win-rate: fraction of eval items where variant A's accuracy score
    is higher than variant B's. This is the "conversion rate" equivalent.
    In dashboard A/B testing, you compare click-through rates; here you
    compare per-item accuracy scores.

  - Wilson confidence interval: a tighter CI than the normal approximation
    for proportions, especially with small sample sizes. Gives the range
    within which the true win-rate lies with 95% confidence.

  - Two-proportion z-test: tests whether the win-rates of A and B are
    significantly different. The null hypothesis is that both variants
    are equally good (win-rate = 0.5). p < 0.05 → reject the null.

  - Power analysis / sample size: how many eval items do we need to
    detect a difference of size δ with 80% power at α=0.05? This prevents
    declaring "inconclusive" just because the sample was too small.

These are the exact same calculations you'd do in an experiment analysis
notebook — just applied to prompt quality instead of click-through rate.
"""

import math
from dataclasses import dataclass
from scipy import stats as scipy_stats


@dataclass
class SignificanceResult:
    win_rate_a: float          # fraction of items where A beat B
    win_rate_b: float          # fraction of items where B beat A
    tie_rate: float            # fraction of items where A == B
    wins_a: int
    wins_b: int
    ties: int
    sample_size: int
    ci_lower: float            # Wilson CI lower bound on win_rate_a
    ci_upper: float            # Wilson CI upper bound on win_rate_a
    p_value: float             # two-proportion z-test p-value
    z_score: float
    required_sample_size: int  # min items for 80% power at α=0.05
    is_significant: bool       # p < 0.05 AND sample_size >= required
    effect_size: float         # |win_rate_a - 0.5| — how far from 50/50


def compute_significance(
    results_a: list,
    results_b: list,
    confidence_level: float = 0.95,
) -> SignificanceResult:
    """Compute statistical significance from per-item results.

    Args:
        results_a: list of ItemResult for variant A
        results_b: list of ItemResult for variant B
        confidence_level: e.g. 0.95 for 95% CI

    Returns:
        SignificanceResult with all computed statistics
    """
    # 1. Pair up results by eval_item_id
    pairs = _pair_by_item(results_a, results_b)

    # 2. Count wins, losses, ties (based on accuracy_score)
    wins_a, wins_b, ties = 0, 0, 0
    for item_id, (ra, rb) in pairs.items():
        if ra.accuracy_score is None or rb.accuracy_score is None:
            ties += 1  # can't compare if no score
            continue
        if ra.accuracy_score > rb.accuracy_score:
            wins_a += 1
        elif rb.accuracy_score > ra.accuracy_score:
            wins_b += 1
        else:
            ties += 1

    n = len(pairs)
    if n == 0:
        return SignificanceResult(
            win_rate_a=0.5, win_rate_b=0.5, tie_rate=1.0,
            wins_a=0, wins_b=0, ties=0, sample_size=0,
            ci_lower=0.0, ci_upper=1.0, p_value=1.0, z_score=0.0,
            required_sample_size=0, is_significant=False, effect_size=0.0
        )

    # 3. Win-rates (exclude ties from the proportion, or include as 0.5)
    #    Standard approach: treat ties as half-wins for each
    effective_wins_a = wins_a + ties * 0.5
    win_rate_a = effective_wins_a / n
    win_rate_b = 1.0 - win_rate_a
    tie_rate = ties / n

    # 4. Wilson confidence interval on win_rate_a
    #    Wilson is more accurate than normal approximation for small n
    z = scipy_stats.norm.ppf((1 + confidence_level) / 2)  # 1.96 for 95%
    ci_lower, ci_upper = wilson_confidence_interval(
        effective_wins_a, n, z
    )

    # 5. Two-proportion z-test
    #    H0: win_rate_a = 0.5 (no difference between variants)
    #    H1: win_rate_a ≠ 0.5
    #    This is equivalent to a sign test / binomial test
    p_value, z_score = binomial_z_test(effective_wins_a, n, p0=0.5)

    # 6. Required sample size (power analysis)
    #    For a two-sided test at α=0.05, power=0.80, detecting
    #    a win-rate of 0.60 (i.e., 10% above chance):
    effect_size = abs(win_rate_a - 0.5)
    required_sample_size = required_sample_size_for_power(
        effect_size=effect_size if effect_size > 0 else 0.1,
        alpha=1 - confidence_level,
        power=0.80,
    )

    is_significant = (
        p_value < (1 - confidence_level)
        and n >= required_sample_size
    )

    return SignificanceResult(
        win_rate_a=win_rate_a,
        win_rate_b=win_rate_b,
        tie_rate=tie_rate,
        wins_a=wins_a,
        wins_b=wins_b,
        ties=ties,
        sample_size=n,
        ci_lower=ci_lower,
        ci_upper=ci_upper,
        p_value=p_value,
        z_score=z_score,
        required_sample_size=required_sample_size,
        is_significant=is_significant,
        effect_size=effect_size,
    )


def wilson_confidence_interval(
    successes: float, n: int, z: float
) -> tuple[float, float]:
    """Wilson score interval for a proportion.

    More accurate than the normal approximation, especially for
    small sample sizes or proportions near 0 or 1.

    Formula:
      center = (p + z²/(2n)) / (1 + z²/n)
      spread = (z / (1 + z²/n)) * sqrt(p(1-p)/n + z²/(4n²))
      CI = (center - spread, center + spread)
    """
    if n == 0:
        return 0.0, 1.0
    p = successes / n
    z2 = z * z
    denom = 1 + z2 / n
    center = (p + z2 / (2 * n)) / denom
    spread = (z / denom) * math.sqrt(
        p * (1 - p) / n + z2 / (4 * n * n)
    )
    return max(0.0, center - spread), min(1.0, center + spread)


def binomial_z_test(
    successes: float, n: int, p0: float = 0.5
) -> tuple[float, float]:
    """Two-sided z-test for a proportion.

    H0: true proportion = p0 (e.g., 0.5 = no difference)
    Returns (p_value, z_score).
    """
    if n == 0:
        return 1.0, 0.0
    p_hat = successes / n
    se = math.sqrt(p0 * (1 - p0) / n)
    if se == 0:
        return 1.0, 0.0
    z_score = (p_hat - p0) / se
    p_value = 2 * (1 - scipy_stats.norm.cdf(abs(z_score)))
    return p_value, z_score


def required_sample_size_for_power(
    effect_size: float,
    alpha: float = 0.05,
    power: float = 0.80,
) -> int:
    """Compute required sample size to detect an effect of given size.

    For a one-sample proportion test (H0: p=0.5):
      n = (z_{α/2} + z_{power})² * p0*(1-p0) / (p1 - p0)²

    where p0 = 0.5 (null), p1 = 0.5 + effect_size (alternative).

    This tells you: "if the true win-rate is 0.5 + effect_size,
    how many eval items do you need to detect it with 80% power
    at a 5% significance level?"
    """
    if effect_size <= 0:
        return float("inf")

    z_alpha = scipy_stats.norm.ppf(1 - alpha / 2)  # 1.96 for α=0.05
    z_power = scipy_stats.norm.ppf(power)           # 0.84 for 80% power
    p0 = 0.5
    p1 = 0.5 + effect_size

    n = ((z_alpha + z_power) ** 2 * p0 * (1 - p0)) / (p1 - p0) ** 2
    return math.ceil(n)


def _pair_by_item(results_a, results_b) -> dict:
    """Pair results by eval_item_id for per-item comparison."""
    map_a = {r.eval_item_id: r for r in results_a}
    map_b = {r.eval_item_id: r for r in results_b}
    return {
        item_id: (map_a[item_id], map_b[item_id])
        for item_id in map_a.keys() & map_b.keys()
    }
```

### 7.4 LLM-as-Judge (`src/llm/judge.py`)

```python
"""
LLM-as-judge: use a (potentially different) LLM to score the quality
of a target LLM's output against an expected output.

This is the "accuracy metric" for prompt A/B testing. Just as you'd
measure click-through rate for a dashboard A/B test, here we measure
output quality via an LLM judge.

The judge receives:
  - The original prompt (rendered)
  - The LLM's output
  - The expected/reference output

And returns:
  - A score from 0.0 to 1.0
  - A reasoning string explaining the score

Design decisions:
  - Judge uses a different prompt than the target (meta-prompt)
  - Judge model can differ from target model (e.g. judge with GPT-4o,
    target with GPT-4o-mini) to avoid self-preference bias
  - Score is continuous (0.0-1.0), not binary, to capture partial credit
  - Judge reasoning is stored for auditability and dashboard display
"""

from dataclasses import dataclass


JUDGE_META_PROMPT = """\
You are an expert evaluator. Your task is to score the quality of an AI's \
response against a reference answer.

## Input
- Prompt given to the AI: {prompt}
- AI's response: {output}
- Reference (expected) answer: {expected}

## Scoring Criteria
Score from 0.0 to 1.0:
- 1.0: The response matches or exceeds the reference in correctness, \
completeness, and clarity.
- 0.7-0.9: The response is mostly correct with minor gaps or imprecisions.
- 0.4-0.6: The response is partially correct but has significant gaps.
- 0.1-0.3: The response has major errors or is mostly incomplete.
- 0.0: The response is completely wrong or irrelevant.

## Output Format
Respond in EXACTLY this format:
SCORE: <float between 0.0 and 1.0>
REASONING: <one or two sentences explaining your score>
"""


@dataclass
class JudgeResult:
    score: float
    reasoning: str
    raw: str


async def score(
    judge_model: str,
    prompt: str,
    output: str,
    expected: str,
) -> JudgeResult:
    """Score an LLM output against an expected answer."""
    rendered = JUDGE_META_PROMPT.format(
        prompt=prompt[:500],      # truncate to avoid token limits
        output=output,
        expected=expected,
    )

    response = await llm_client.generate(
        model=judge_model,
        messages=[{"role": "user", "content": rendered}],
        temperature=0.0,          # deterministic judging
        max_tokens=256,
    )

    return _parse_judge_response(response.content)


def _parse_judge_response(raw: str) -> JudgeResult:
    """Parse 'SCORE: 0.85\\nREASONING: ...' into JudgeResult."""
    score = 0.5  # default if parsing fails
    reasoning = raw

    for line in raw.strip().split("\n"):
        if line.upper().startswith("SCORE:"):
            try:
                score = float(line.split(":", 1)[1].strip())
                score = max(0.0, min(1.0, score))  # clamp
            except ValueError:
                pass
        elif line.upper().startswith("REASONING:"):
            reasoning = line.split(":", 1)[1].strip()

    return JudgeResult(score=score, reasoning=reasoning, raw=raw)
```

### 7.5 Promotion / Rollback (`src/db/repositories/promotion_repo.py`)

```python
"""
Transactional promotion and rollback of prompt versions.

The one-production invariant is enforced at TWO levels:
  1. Database: a partial unique index ensures at most one row per
     prompt_id has status='production'
  2. Application: this function uses a transaction to atomically
     archive the old production version and promote the new one

If either step fails, the transaction rolls back — no prompt is
left without a production version (unless it never had one).

This is the "deploy" step — equivalent to merging a PR to main
or promoting a dbt model from staging to production.
"""

from asyncpg import Connection


async def promote(
    conn: Connection,
    prompt_id: str,
    to_version: int,
    promoted_by: str,
    reason: str,
) -> dict:
    """Promote a prompt version to production. Atomic.

    Returns: {from_version, to_version, action: 'promote'|'rollback'}
    """
    async with conn.transaction():
        # 1. Find current production version (if any)
        current = await conn.fetchrow(
            """SELECT version FROM prompt_versions
               WHERE prompt_id = $1 AND status = 'production'""",
            prompt_id
        )
        from_version = current["version"] if current else None

        # 2. If already production, no-op
        if from_version == to_version:
            return {
                "from_version": from_version,
                "to_version": to_version,
                "action": "no-op",
            }

        # 3. Archive current production version
        if from_version is not None:
            await conn.execute(
                """UPDATE prompt_versions
                   SET status = 'archived'
                   WHERE prompt_id = $1 AND status = 'production'""",
                prompt_id
            )

        # 4. Promote target version
        await conn.execute(
            """UPDATE prompt_versions
               SET status = 'production'
               WHERE prompt_id = $1 AND version = $2""",
            prompt_id, to_version
        )

        # 5. Determine action type
        action = "rollback" if (
            from_version is not None and to_version < from_version
        ) else "promote"

        # 6. Record in promotions (append-only)
        await conn.execute(
            """INSERT INTO promotions
               (prompt_id, from_version, to_version, action, reason, promoted_by)
               VALUES ($1, $2, $3, $4, $5, $6)""",
            prompt_id, from_version, to_version, action, reason, promoted_by
        )

        return {
            "from_version": from_version,
            "to_version": to_version,
            "action": action,
        }
```

### 7.6 Python SDK (`src/sdk/client.py`)

```python
"""
PromptClient — the Python SDK for loading production prompts.

Usage in any application:

    from prompt_sdk import PromptClient

    client = PromptClient(base_url="http://localhost:8000")

    # Get the current production prompt
    prompt = client.get_production_prompt("summarizer")

    # prompt.template  → Jinja2 template string
    # prompt.variables  → list of variable definitions
    # prompt.version    → version number
    # prompt.render({"article": "..."})  → rendered string
    # prompt.to_messages() → [{"role": "user", "content": "..."}]

    # Or render server-side (API does the Jinja2 rendering)
    rendered = client.render("summarizer", {"article": "..."})

Design decisions:
  - Client-side rendering by default (reduces API calls; Jinja2 is fast)
  - Optional caching with TTL (avoid hitting API on every call)
  - Fallback: if API is down, use last-known-good cached prompt
  - Type-safe: variable definitions include type info for validation
"""

import httpx
from dataclasses import dataclass
from jinja2 import Environment, StrictUndefined
from jinja2.sandbox import SandboxedEnvironment
import time


@dataclass
class PromptVersion:
    prompt_key: str
    version: int
    template: str
    variables: list[dict]
    message_type: str
    model_hint: str | None
    temperature: float
    max_tokens: int

    def render(self, variables: dict) -> str:
        """Render the template with the given variables (client-side)."""
        env = SandboxedEnvironment(
            undefined=StrictUndefined,
            autoescape=False,
            trim_blocks=True,
            lstrip_blocks=True,
        )
        return env.from_string(self.template).render(**variables)

    def to_messages(self, variables: dict) -> list[dict]:
        """Render and return as OpenAI-format messages list."""
        content = self.render(variables)
        return [{"role": self.message_type, "content": content}]


class PromptClient:
    """SDK client for the Prompt Versioning & A/B Testing system."""

    def __init__(
        self,
        base_url: str = "http://localhost:8000",
        cache_ttl_seconds: int = 300,  # 5-minute cache
        timeout: float = 10.0,
    ):
        self._base_url = base_url.rstrip("/")
        self._client = httpx.Client(timeout=timeout)
        self._cache: dict[str, tuple[PromptVersion, float]] = {}
        self._cache_ttl = cache_ttl_seconds

    def get_production_prompt(self, prompt_key: str) -> PromptVersion:
        """Get the current production version of a prompt.

        Uses in-memory cache with TTL. If the API is unreachable
        and a cached version exists, returns the cached version.
        """
        # Check cache
        cached = self._cache.get(prompt_key)
        if cached and (time.time() - cached[1]) < self._cache_ttl:
            return cached[0]

        # Fetch from API
        try:
            resp = self._client.get(
                f"{self._base_url}/sdk/prompts/{prompt_key}/production"
            )
            resp.raise_for_status()
            data = resp.json()
            prompt = PromptVersion(
                prompt_key=data["prompt_key"],
                version=data["version"],
                template=data["template"],
                variables=data["variables"],
                message_type=data["message_type"],
                model_hint=data.get("model_hint"),
                temperature=data.get("temperature", 0.0),
                max_tokens=data.get("max_tokens", 1024),
            )
            self._cache[prompt_key] = (prompt, time.time())
            return prompt
        except httpx.HTTPError:
            # Fallback to stale cache if available
            if cached:
                return cached[0]
            raise

    def get_version(
        self, prompt_key: str, version: int
    ) -> PromptVersion:
        """Get a specific version of a prompt (not necessarily production)."""
        resp = self._client.get(
            f"{self._base_url}/sdk/prompts/{prompt_key}/versions/{version}"
        )
        resp.raise_for_status()
        data = resp.json()
        return PromptVersion(
            prompt_key=data["prompt_key"],
            version=data["version"],
            template=data["template"],
            variables=data["variables"],
            message_type=data["message_type"],
            model_hint=data.get("model_hint"),
            temperature=data.get("temperature", 0.0),
            max_tokens=data.get("max_tokens", 1024),
        )

    def render(
        self, prompt_key: str, variables: dict
    ) -> str:
        """Get the production prompt and render it with variables."""
        prompt = self.get_production_prompt(prompt_key)
        return prompt.render(variables)

    def to_messages(
        self, prompt_key: str, variables: dict
    ) -> list[dict]:
        """Get production prompt, render, and return as messages list."""
        prompt = self.get_production_prompt(prompt_key)
        return prompt.to_messages(variables)

    def clear_cache(self) -> None:
        """Clear the in-memory prompt cache."""
        self._cache.clear()
```

### 7.7 Streamlit Dashboard Layout (`dashboard/app.py`)

```python
"""
Streamlit dashboard for the Prompt Versioning & A/B Testing system.

Layout:
  ┌─────────────────────────────────────────────────────────┐
  │  [Sidebar]                    [Main Content Area]        │
  │                               (changes per page)         │
  │  Navigation:                                              │
  │  ○ Overview                                               │
  │  ○ A/B Test Results                                       │
  │  ○ Version History                                        │
  │  ○ Metric Trends                                          │
  │                                                           │
  │  DB Connection:                                           │
  │  host: localhost:5432                                     │
  │  status: ● connected                                      │
  └─────────────────────────────────────────────────────────┘

Page: Overview
  ┌─────────────────────────────────────────────────────────┐
  │  Prompt Registry Overview                                │
  │                                                          │
  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐    │
  │  │ Total Prompts │ │ Active Tests  │ │ Prod Prompts │    │
  │  │      12       │ │      3        │ │      8       │    │
  │  └──────────────┘ └──────────────┘ └──────────────┘    │
  │                                                          │
  │  Prompt Table:                                           │
  │  ┌─────────────┬──────────┬──────────┬────────┬───────┐ │
  │  │ Prompt Key  │ Versions │ Prod Ver │ Tests  │ Tags  │ │
  │  │ summarizer  │    5     │    v3    │   4    │ sum.. │ │
  │  │ qa-extract  │    3     │    v2    │   2    │ qa    │ │
  │  │ classifier  │    7     │    v5    │   6    │ cls   │ │
  │  └─────────────┴──────────┴──────────┴────────┴───────┘ │
  └─────────────────────────────────────────────────────────┘

Page: A/B Test Results
  ┌─────────────────────────────────────────────────────────┐
  │  [Test Selector Dropdown: ab_test_id]                    │
  │                                                          │
  │  Test: summarizer v2 vs v3                               │
  │  Status: completed    Winner: v3 (B)    p-value: 0.012   │
  │                                                          │
  │  Win-Rate Chart (Plotly bar):                            │
  │  ┌────────────────────────────────────────┐             │
  │  │  ██████████████████████  72%  (B) v3   │             │
  │  │  ████████████            28%  (A) v2   │             │
  │  │  ██                        8%  ties    │             │
  │  └────────────────────────────────────────┘             │
  │  Wilson 95% CI: [0.59, 0.82]                             │
  │                                                          │
  │  Metric Comparison Table:                                │
  │  ┌──────────────┬──────────┬──────────┬────────┐        │
  │  │ Metric       │ v2 (A)   │ v3 (B)   │ Delta  │        │
  │  │ Accuracy     │ 0.74     │ 0.86     │ +0.12  │        │
  │  │ Latency (ms) │ 842      │ 910      │ +68    │        │
  │  │ Cost ($/call)│ 0.0021   │ 0.0023   │ +0.0002│        │
  │  │ Output Len   │ 145      │ 162      │ +17    │        │
  │  │ Total Tokens │ 312      │ 340      │ +28    │        │
  │  └──────────────┴──────────┴──────────┴────────┘        │
  │                                                          │
  │  Sample Outputs (side-by-side):                          │
  │  ┌─────────────────────┬─────────────────────┐          │
  │  │ Item #3             │ Item #3             │          │
  │  │ Variant A (v2)      │ Variant B (v3)      │          │
  │  │ ┌─────────────────┐ │ ┌─────────────────┐ │          │
  │  │ │ "The article    │ │ │ "The article    │ │          │
  │  │ │  discusses..."  │ │ │  discusses the  │ │          │
  │  │ │                 │ │ │  impact of..."  │ │          │
  │  │ └─────────────────┘ │ └─────────────────┘ │          │
  │  │ Judge: 0.6          │ Judge: 0.9          │          │
  │  │ "Misses key point"  │ "Covers all points" │          │
  │  └─────────────────────┴─────────────────────┘          │
  │  [◀ Prev Item]  Item 3 of 50  [Next Item ▶]             │
  └─────────────────────────────────────────────────────────┘

Page: Version History
  ┌─────────────────────────────────────────────────────────┐
  │  [Prompt Selector: summarizer]                           │
  │                                                          │
  │  Version Timeline:                                       │
  │  v5 [draft]     2025-08-10  "Added few-shot examples"   │
  │  v4 [archived]  2025-08-08  "Tried chain-of-thought"    │
  │  v3 [production] 2025-08-05 "Improved structure" ◀ prod │
  │  v2 [archived]  2025-08-01  "Added output format"       │
  │  v1 [archived]  2025-07-28  "Initial version"           │
  │                                                          │
  │  Template Diff (v2 → v3):                                │
  │  ┌────────────────────────────────────────┐             │
  │  │ - Summarize the article in {max_words}.│             │
  │  │ + Summarize the article in {max_words}.│             │
  │  │ + Focus on the main argument and key   │             │
  │  │ + supporting evidence.                 │             │
  │  └────────────────────────────────────────┘             │
  │                                                          │
  │  Promotion History:                                      │
  │  2025-08-05  v2 → v3  promote  "Won A/B test (p=0.012)"│
  │  2025-08-01  v1 → v2  promote  "Initial deployment"    │
  └─────────────────────────────────────────────────────────┘

Page: Metric Trends
  ┌─────────────────────────────────────────────────────────┐
  │  [Prompt Selector: summarizer]                           │
  │  [Metric Selector: Accuracy / Latency / Cost]            │
  │                                                          │
  │  Accuracy Over Time (Plotly line chart):                 │
  │  ┌────────────────────────────────────────┐             │
  │  │ 1.0│                                    │             │
  │  │    │              ●─────● v3            │             │
  │  │ 0.8│         ●─────●                    │             │
  │  │    │    ●─────● v2                      │             │
  │  │ 0.6│●─────● v1                         │             │
  │  │    │                                    │             │
  │  │ 0.4└──────────────────────────────      │             │
  │  │     Jul28  Aug01  Aug05  Aug08  Aug10   │             │
  │  └────────────────────────────────────────┘             │
  │  Each point = one A/B test run                           │
  │  Line color = prompt version                             │
  └─────────────────────────────────────────────────────────┘
"""

import streamlit as st
import plotly.graph_objects as go
import pandas as pd
import asyncpg

# Page configuration
st.set_page_config(
    page_title="Prompt A/B Testing Dashboard",
    page_icon="🧪",
    layout="wide",
)

# Sidebar navigation
st.sidebar.title("🧪 Prompt A/B Testing")
page = st.sidebar.radio(
    "Navigation",
    ["Overview", "A/B Test Results", "Version History", "Metric Trends"]
)

# DB connection status
st.sidebar.divider()
st.sidebar.write("**Database**")
st.sidebar.write("● Connected")

# Route to page
if page == "Overview":
    from pages import overview
    overview.render()
elif page == "A/B Test Results":
    from pages import ab_test_results
    ab_test_results.render()
elif page == "Version History":
    from pages import version_history
    version_history.render()
elif page == "Metric Trends":
    from pages import metric_trends
    metric_trends.render()
```

---

## 8. Security & Safety Considerations

### 8.1 Prompt Injection in Templates

| Threat | Mitigation | Layer |
|--------|-----------|-------|
| **Template injection** (malicious Jinja2 in template string executes code) | `SandboxedEnvironment` blocks access to Python internals; templates are authored by trusted engineers, not end users | App |
| **Variable injection** (user input in template variables contains Jinja2 syntax) | Variables are inserted as values, not parsed as template code — Jinja2's `render(**variables)` treats them as data, not code | App |
| **LLM-as-judge bias** (judge prefers longer outputs, or prefers its own style) | Judge uses a different model than target; structured scoring rubric; judge reasoning stored for audit | App |
| **Prompt content leakage** (production prompt template exposed via SDK) | SDK returns template + variables to trusted internal applications; API is internal-only in v1; no external exposure | Network |
| **Cost runaway** (A/B test calls LLM 1000s of times, racking up cost) | `sample_size` parameter limits items per test; cost estimation displayed before test runs; total cost tracked per test | App |
| **Promotion race condition** (two engineers promote different versions simultaneously) | Transactional promotion with row-level lock; partial unique index on `status='production'` | DB |
| **Audit log tampering** (someone edits promotion history) | `promotions` table has triggers blocking UPDATE/DELETE — append-only | DB |

### 8.2 Safe Defaults

```
LLM Calls:
  - temperature=0.0 for judge (deterministic scoring)
  - max_tokens=1024 for target (prevent runaway outputs)
  - Semaphore limits concurrent calls (default: 5)
  - Retry with exponential backoff (3 attempts, 1s/2s/4s)

Templates:
  - SandboxedEnvironment (no Python code execution)
  - StrictUndefined (missing variables → error, not silent None)
  - Compile check on creation (reject invalid templates at API layer)

Promotion:
  - Transactional (atomic archive + promote)
  - One-production constraint (DB-level partial unique index)
  - Audit trail (append-only promotions table)

SDK:
  - Cache with TTL (5 min default) to reduce API load
  - Stale-cache fallback (if API down, use last-known-good prompt)
  - Timeout on API calls (10s default)
```

### 8.3 Cost Safety

```python
# Before running an A/B test, estimate and display cost:
def estimate_test_cost(
    n_items: int,
    n_variants: int,
    target_model: str,
    judge_model: str,
    avg_prompt_tokens: int = 500,
    avg_output_tokens: int = 200,
) -> float:
    """Estimate total cost before running a test."""
    target_cost_per_call = get_cost(target_model, avg_prompt_tokens, avg_output_tokens)
    judge_cost_per_call = get_cost(judge_model, 600, 100)  # judge meta-prompt + response

    total_calls = n_items * n_variants  # target LLM calls
    judge_calls = n_items * n_variants  # one judge call per target call

    return (total_calls * target_cost_per_call) + (judge_calls * judge_cost_per_call)

# Example: 50 items × 2 variants × (gpt-4o-mini @ $0.00015/call + judge @ $0.00015)
# = 100 × $0.0003 = $0.03 — negligible
# But with gpt-4o as target: 100 × $0.006 = $0.60 per test
```

---

## 9. API Specification

### 9.1 `POST /prompts`

Create a new prompt (logical identity).

```
Body:
{
  "prompt_key": "summarizer",
  "name": "Article Summarizer",
  "description": "Summarizes articles into concise summaries",
  "tags": ["summarization", "customer-support"]
}

Response 201:
{
  "id": "uuid",
  "prompt_key": "summarizer",
  "name": "Article Summarizer",
  "created_at": "2025-08-10T10:00:00Z"
}

Response 409: prompt_key already exists
```

### 9.2 `POST /prompts/{prompt_id}/versions`

Create a new version of a prompt. Version number auto-increments.

```
Body:
{
  "template": "Summarize the following article in {{ max_words }} words:\n\n{{ article }}",
  "variables": [
    {"name": "article", "type": "string", "required": true, "description": "The article text"},
    {"name": "max_words", "type": "integer", "required": false, "default": 100, "description": "Max summary length"}
  ],
  "message_type": "user",
  "model_hint": "gpt-4o-mini",
  "temperature": 0.3,
  "max_tokens": 256,
  "changelog": "Added max_words variable for length control",
  "metadata": {"author": "sagar", "ticket": "PROMPT-42"}
}

Response 201:
{
  "id": "uuid",
  "prompt_id": "uuid",
  "version": 3,
  "status": "draft",
  "created_at": "2025-08-10T10:05:00Z"
}

Response 400: Template syntax error / undeclared variable
```

### 9.3 `GET /prompts/{prompt_id}/versions`

List all versions of a prompt.

```
Response 200:
{
  "prompt_id": "uuid",
  "versions": [
    {
      "version": 3,
      "status": "production",
      "changelog": "Added max_words variable",
      "created_by": "sagar",
      "created_at": "2025-08-10T10:05:00Z"
    },
    {
      "version": 2,
      "status": "archived",
      "changelog": "Improved structure",
      "created_by": "sagar",
      "created_at": "2025-08-08T14:20:00Z"
    },
    {
      "version": 1,
      "status": "archived",
      "changelog": "Initial version",
      "created_by": "sagar",
      "created_at": "2025-07-28T09:00:00Z"
    }
  ]
}
```

### 9.4 `POST /prompts/{prompt_id}/promote`

Promote a version to production. Atomic transaction.

```
Body:
{
  "version": 3,
  "reason": "Won A/B test against v2 (p=0.012, win-rate 72%)"
}

Response 200:
{
  "prompt_id": "uuid",
  "from_version": 2,
  "to_version": 3,
  "action": "promote",
  "promoted_at": "2025-08-10T11:00:00Z"
}
```

### 9.5 `POST /prompts/{prompt_id}/rollback`

Roll back to a previous version. Same as promote but to a lower version number.

```
Body:
{
  "version": 2,
  "reason": "v3 showing regression in production; rolling back to v2"
}

Response 200:
{
  "prompt_id": "uuid",
  "from_version": 3,
  "to_version": 2,
  "action": "rollback",
  "promoted_at": "2025-08-10T12:00:00Z"
}
```

### 9.6 `POST /eval-datasets`

Create an evaluation dataset.

```
Body:
{
  "name": "summarizer-golden-v1",
  "description": "50 articles with reference summaries",
  "items": [
    {
      "input_variables": {"article": "The article text...", "max_words": 100},
      "expected_output": "A concise summary of the article...",
      "metadata": {"category": "tech", "difficulty": "medium"}
    }
  ]
}

Response 201:
{
  "id": "uuid",
  "name": "summarizer-golden-v1",
  "item_count": 50,
  "created_at": "2025-08-10T10:10:00Z"
}
```

### 9.7 `POST /ab-tests`

Create and start an A/B test. Returns immediately with test_id; test runs asynchronously.

```
Body:
{
  "prompt_id": "uuid",
  "version_a": 2,
  "version_b": 3,
  "dataset_id": "uuid",
  "target_model": "gpt-4o-mini",
  "judge_model": "gpt-4o-mini",
  "sample_size": 50,
  "confidence_level": 0.95
}

Response 202:
{
  "test_id": "uuid",
  "status": "pending",
  "estimated_cost_usd": 0.03,
  "estimated_items": 50
}
```

### 9.8 `GET /ab-tests/{test_id}`

Get A/B test status and results.

```
Response 200 (completed):
{
  "test_id": "uuid",
  "status": "completed",
  "winner": "B",
  "p_value": 0.012,
  "confidence_level": 0.95,
  "version_a": 2,
  "version_b": 3,
  "metrics": {
    "A": {
      "mean_accuracy": 0.74,
      "mean_latency_ms": 842,
      "mean_cost_usd": 0.0021,
      "mean_output_length": 145,
      "win_rate": 0.28,
      "ci_lower": 0.18,
      "ci_upper": 0.41
    },
    "B": {
      "mean_accuracy": 0.86,
      "mean_latency_ms": 910,
      "mean_cost_usd": 0.0023,
      "mean_output_length": 162,
      "win_rate": 0.72,
      "ci_lower": 0.59,
      "ci_upper": 0.82
    }
  },
  "sample_size": 50,
  "required_sample_size": 39,
  "completed_at": "2025-08-10T10:30:00Z"
}

Response 200 (running):
{
  "test_id": "uuid",
  "status": "running",
  "progress": 0.6,
  "completed_items": 30,
  "total_items": 50
}
```

### 9.9 `GET /ab-tests/{test_id}/items/{item_idx}`

Get per-item results for side-by-side comparison (used by dashboard).

```
Response 200:
{
  "item_idx": 3,
  "input_variables": {"article": "The article text..."},
  "expected_output": "A concise summary...",
  "results": {
    "A": {
      "rendered_prompt": "Summarize the following article...",
      "raw_output": "The article discusses...",
      "accuracy_score": 0.6,
      "latency_ms": 820,
      "judge_reasoning": "Misses key point about..."
    },
    "B": {
      "rendered_prompt": "Summarize the following article...",
      "raw_output": "The article discusses the impact of...",
      "accuracy_score": 0.9,
      "latency_ms": 890,
      "judge_reasoning": "Covers all key points with..."
    }
  }
}
```

### 9.10 `GET /sdk/prompts/{prompt_key}/production`

SDK endpoint: get the current production prompt version. Used by `PromptClient`.

```
Response 200:
{
  "prompt_key": "summarizer",
  "version": 3,
  "template": "Summarize the following article in {{ max_words }} words:\n\n{{ article }}",
  "variables": [
    {"name": "article", "type": "string", "required": true},
    {"name": "max_words", "type": "integer", "required": false, "default": 100}
  ],
  "message_type": "user",
  "model_hint": "gpt-4o-mini",
  "temperature": 0.3,
  "max_tokens": 256
}

Response 404: No production version found for this prompt_key
```

### 9.11 `GET /health`

```
Response 200:
{
  "status": "healthy",
  "database": "connected",
  "version": "1.0.0"
}
```

---

## 10. Testing Strategy

### 10.1 Test Categories

| Category | What It Proves | Priority |
|----------|---------------|----------|
| **Templating tests** | Jinja2 rendering, sandbox safety, variable validation | P0 |
| **Versioning tests** | Versions are immutable, auto-increment, never lost | P0 |
| **Promotion tests** | One-production invariant, atomic rollback, audit trail | P0 |
| **Stats tests** | Z-test, Wilson CI, power analysis produce correct values | P0 |
| **A/B runner tests** | Runner collects correct metrics per item per variant | P1 |
| **Judge tests** | LLM-as-judge parses scores correctly (mocked LLM) | P1 |
| **API tests** | Endpoints return correct status codes + payloads | P1 |
| **SDK tests** | Client loads production prompt, renders, caches | P1 |

### 10.2 Critical Tests (`tests/test_stats.py`)

```python
"""
Statistical significance tests with known-answer verification.
These are the mathematical backbone — if these are wrong, every
A/B test conclusion is unreliable.
"""

import pytest
from src.testing.stats import (
    compute_significance,
    wilson_confidence_interval,
    binomial_z_test,
    required_sample_size_for_power,
)


class TestWilsonConfidenceInterval:
    def test_50_percent_100_samples(self):
        """50 successes out of 100 → CI should contain 0.5."""
        lower, upper = wilson_confidence_interval(50, 100, z=1.96)
        assert lower < 0.5 < upper
        assert 0.35 < lower < 0.45   # approximately [0.40, 0.60]
        assert 0.55 < upper < 0.65

    def test_100_percent_small_sample(self):
        """10/10 successes → CI should NOT include 1.0 (Wilson shrinks)."""
        lower, upper = wilson_confidence_interval(10, 10, z=1.96)
        assert upper < 1.0
        assert lower > 0.5

    def test_zero_successes(self):
        """0/10 → CI should NOT include 0.0."""
        lower, upper = wilson_confidence_interval(0, 10, z=1.96)
        assert lower > 0.0
        assert upper < 0.5


class TestBinomialZTest:
    def test_equal_to_null(self):
        """50/100 → p-value should be 1.0 (no difference from 0.5)."""
        p_value, z = binomial_z_test(50, 100, p0=0.5)
        assert abs(z) < 0.01
        assert p_value > 0.99

    def test_highly_significant(self):
        """90/100 → should be highly significant (p < 0.001)."""
        p_value, z = binomial_z_test(90, 100, p0=0.5)
        assert z > 7.0
        assert p_value < 0.001

    def test_not_significant_small_sample(self):
        """6/10 → should NOT be significant (p > 0.05)."""
        p_value, z = binomial_z_test(6, 10, p0=0.5)
        assert p_value > 0.05


class TestRequiredSampleSize:
    def test_large_effect_needs_few_samples(self):
        """Detecting win-rate of 0.80 (effect=0.30) needs ~30-40 samples."""
        n = required_sample_size_for_power(effect_size=0.30, alpha=0.05, power=0.80)
        assert 20 < n < 50

    def test_small_effect_needs_many_samples(self):
        """Detecting win-rate of 0.55 (effect=0.05) needs ~1000+ samples."""
        n = required_sample_size_for_power(effect_size=0.05, alpha=0.05, power=0.80)
        assert n > 500

    def test_zero_effect_needs_infinite_samples(self):
        """No effect → can't detect → infinite sample size."""
        n = required_sample_size_for_power(effect_size=0.0)
        assert n == float('inf')


class TestComputeSignificance:
    def test_clear_winner_declared(self):
        """Variant A wins 80% of items → A declared winner, p < 0.05."""
        # Create mock results: A wins 40/50, B wins 10/50
        results_a, results_b = _make_mock_results(
            n=50, a_wins=40, b_wins=10
        )
        stats = compute_significance(results_a, results_b)
        assert stats.is_significant
        assert stats.p_value < 0.05
        assert stats.win_rate_a > 0.7

    def test_inconclusive_with_small_sample(self):
        """Variant A wins 7/10 → NOT significant (sample too small)."""
        results_a, results_b = _make_mock_results(
            n=10, a_wins=7, b_wins=3
        )
        stats = compute_significance(results_a, results_b)
        # 7/10 is not significant at α=0.05
        assert not stats.is_significant or stats.p_value > 0.05

    def test_ties_handled_correctly(self):
        """All ties → win_rate = 0.5, not significant."""
        results_a, results_b = _make_mock_results(
            n=20, a_wins=0, b_wins=0, ties=20
        )
        stats = compute_significance(results_a, results_b)
        assert stats.win_rate_a == 0.5
        assert not stats.is_significant
```

### 10.3 Critical Tests (`tests/test_promotion.py`)

```python
"""
Promotion/rollback tests — the one-production invariant is critical.
If two versions are ever 'production' simultaneously, the SDK will
non-deterministically return different prompts to different callers.
"""

import pytest


async def test_promote_sets_production_status():
    """Promoting v2 → v2 has status='production'."""
    prompt_id = await create_prompt("summarizer")
    await create_version(prompt_id, template="v1 template")
    await create_version(prompt_id, template="v2 template")

    await promote(prompt_id, to_version=2, reason="test")

    v2 = await get_version(prompt_id, 2)
    assert v2["status"] == "production"


async def test_promote_archives_previous_production():
    """Promoting v2 when v1 is production → v1 archived, v2 production."""
    prompt_id = await create_prompt("summarizer")
    await create_version(prompt_id, template="v1")
    await create_version(prompt_id, template="v2")

    await promote(prompt_id, to_version=1, reason="first deploy")
    await promote(prompt_id, to_version=2, reason="second deploy")

    v1 = await get_version(prompt_id, 1)
    v2 = await get_version(prompt_id, 2)
    assert v1["status"] == "archived"
    assert v2["status"] == "production"


async def test_rollback_to_previous_version():
    """Rollback from v3 to v2 → v2 production, v3 archived."""
    prompt_id = await create_prompt("summarizer")
    for i in range(1, 4):
        await create_version(prompt_id, template=f"v{i}")

    await promote(prompt_id, to_version=3, reason="deploy v3")
    await promote(prompt_id, to_version=2, reason="rollback: v3 regression")

    v2 = await get_version(prompt_id, 2)
    v3 = await get_version(prompt_id, 3)
    assert v2["status"] == "production"
    assert v3["status"] == "archived"


async def test_one_production_invariant_at_db_level():
    """DB partial unique index prevents two production versions."""
    prompt_id = await create_prompt("summarizer")
    await create_version(prompt_id, template="v1")
    await create_version(prompt_id, template="v2")

    await promote(prompt_id, to_version=1, reason="deploy")

    # Attempt to directly set v2 to production without archiving v1
    # should fail due to partial unique index
    with pytest.raises(Exception):
        await db.execute(
            "UPDATE prompt_versions SET status='production' "
            "WHERE prompt_id=$1 AND version=2",
            prompt_id
        )


async def test_promotion_audit_trail():
    """Promotions table records all promote/rollback events."""
    prompt_id = await create_prompt("summarizer")
    await create_version(prompt_id, template="v1")
    await create_version(prompt_id, template="v2")

    await promote(prompt_id, to_version=1, reason="initial")
    await promote(prompt_id, to_version=2, reason="upgrade")
    await promote(prompt_id, to_version=1, reason="rollback")

    promos = await db.fetch(
        "SELECT * FROM promotions WHERE prompt_id=$1 ORDER BY created_at",
        prompt_id
    )
    assert len(promos) == 3
    assert promos[0]["action"] == "promote"
    assert promos[1]["action"] == "promote"
    assert promos[2]["action"] == "rollback"


async def test_promotions_table_is_immutable():
    """Attempt to UPDATE or DELETE promotions → must raise."""
    with pytest.raises(Exception):
        await db.execute("UPDATE promotions SET reason='tampered'")
    with pytest.raises(Exception):
        await db.execute("DELETE FROM promotions")
```

### 10.4 Critical Tests (`tests/test_templating.py`)

```python
"""
Templating tests — sandbox safety and variable validation.
A template injection vulnerability would allow arbitrary code execution.
"""

import pytest
from src.templating.renderer import render_template, validate_template


class TestRendering:
    def test_simple_variable(self):
        result = render_template("Hello {{ name }}", {"name": "World"})
        assert result == "Hello World"

    def test_conditional(self):
        template = "{% if verbose %}Detailed: {% endif %}{{ content }}"
        assert render_template(template, {"verbose": True, "content": "x"}) == "Detailed: x"
        assert render_template(template, {"verbose": False, "content": "x"}) == "x"

    def test_missing_variable_raises(self):
        with pytest.raises(Exception):  # UndefinedError
            render_template("Hello {{ name }}", {})

    def test_sandbox_blocks_code_execution(self):
        """Template injection attempt must be blocked."""
        malicious = "{{ ''.__class__.__mro__[1].__subclasses__() }}"
        with pytest.raises(Exception):  # SecurityError
            render_template(malicious, {})


class TestValidation:
    def test_valid_template_passes(self):
        errors = validate_template(
            "Summarize: {{ article }}",
            [{"name": "article", "type": "string", "required": True}]
        )
        assert errors == []

    def test_undeclared_variable_caught(self):
        errors = validate_template(
            "Summarize: {{ article }} and {{ max_words }}",
            [{"name": "article", "type": "string", "required": True}]
        )
        assert len(errors) == 1
        assert "max_words" in errors[0]

    def test_syntax_error_caught(self):
        errors = validate_template(
            "Summarize: {{ article ",
            [{"name": "article", "type": "string", "required": True}]
        )
        assert len(errors) == 1
        assert "syntax" in errors[0].lower()
```

### 10.5 Test Fixtures (`tests/conftest.py`)

```python
"""
Test setup:
  - Spin up a test Postgres (docker or testcontainers)
  - Run schema.sql
  - Provide asyncpg connection pool
  - Provide FastAPI TestClient with overridden DB
  - Provide seed data: 2 prompts, 3 versions each, 1 eval dataset with 10 items
"""

import pytest
import asyncpg


@pytest.fixture
async def db():
    """Fresh database for each test module."""
    conn = await asyncpg.connect(TEST_DB_URL)
    await conn.execute(open("src/db/schema.sql").read())
    yield conn
    await conn.execute(
        "DROP TABLE IF EXISTS promotions, metric_history, test_results, "
        "ab_tests, eval_items, eval_datasets, prompt_versions, prompts CASCADE;"
    )
    await conn.close()


@pytest.fixture
async def client(db):
    """FastAPI TestClient with overridden DB pool."""
    from src.main import create_app
    app = create_app(db_pool=db)
    async with httpx.AsyncClient(app=app, base_url="http://test") as c:
        yield c


@pytest.fixture
async def seeded_data(db):
    """Seed: 2 prompts, 3 versions each, 1 eval dataset with 10 items."""
    # Create prompts
    p1 = await create_prompt(db, "summarizer", "Article Summarizer")
    p2 = await create_prompt(db, "qa-extractor", "QA Extractor")

    # Create versions
    for prompt_id in [p1, p2]:
        await create_version(db, prompt_id, template="v1: {{ input }}")
        await create_version(db, prompt_id, template="v2: {{ input }} improved")
        await create_version(db, prompt_id, template="v3: {{ input }} best version")

    # Create eval dataset
    ds = await create_dataset(db, "test-dataset", n_items=10)

    return {"prompt_ids": [p1, p2], "dataset_id": ds}
```

---

## 11. Deployment

### 11.1 `docker-compose.yml`

```yaml
version: "3.9"

services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: prompt_ab
      POSTGRES_USER: prompt_user
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-changeme}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./docker/postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U prompt_user -d prompt_ab"]
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
      DATABASE_URL: postgres://prompt_user:${POSTGRES_PASSWORD:-changeme}@postgres:5432/prompt_ab
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      OPENAI_BASE_URL: ${OPENAI_BASE_URL:-https://api.openai.com/v1}
      DEFAULT_TARGET_MODEL: ${DEFAULT_TARGET_MODEL:-gpt-4o-mini}
      DEFAULT_JUDGE_MODEL: ${DEFAULT_JUDGE_MODEL:-gpt-4o-mini}
      LOG_LEVEL: ${LOG_LEVEL:-info}
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 10s
      timeout: 5s
      retries: 3

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

volumes:
  postgres_data:
```

### 11.2 `docker/Dockerfile.api`

```dockerfile
FROM python:3.12-slim

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential libpq-dev curl \
    && rm -rf /var/lib/apt/lists/*

COPY pyproject.toml .
RUN pip install --no-cache-dir -e .

COPY src/ ./src/
COPY scripts/ ./scripts/

EXPOSE 8000

CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 11.3 `docker/Dockerfile.dashboard`

```dockerfile
FROM python:3.12-slim

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential libpq-dev \
    && rm -rf /var/lib/apt/lists/*

COPY pyproject.toml .
RUN pip install --no-cache-dir -e .

COPY src/ ./src/
COPY dashboard/ ./dashboard/

EXPOSE 8501

CMD ["streamlit", "run", "dashboard/app.py", \
     "--server.port=8501", "--server.address=0.0.0.0", \
     "--browser.gatherUsageStats=false"]
```

### 11.4 `docker/postgres/init.sql`

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
-- Schema tables are created by scripts/init_db.py on first API startup
```

### 11.5 `.env.example`

```env
# Database
POSTGRES_PASSWORD=changeme-in-production

# OpenAI (or compatible: Ollama, vLLM, etc.)
OPENAI_API_KEY=sk-...
OPENAI_BASE_URL=https://api.openai.com/v1

# For local LLM (Ollama):
# OPENAI_API_KEY=ollama
# OPENAI_BASE_URL=http://localhost:11434/v1

# Models
DEFAULT_TARGET_MODEL=gpt-4o-mini
DEFAULT_JUDGE_MODEL=gpt-4o-mini

# App
LOG_LEVEL=info

# Dashboard
STREAMLAP_HEADLESS=true
```

### 11.6 Production Considerations

| Concern | Recommendation |
|---------|---------------|
| **TLS** | Terminate TLS at reverse proxy (nginx/traefik); API + dashboard listen on HTTP |
| **DB backups** | `pg_dump` cron job; prompt versions and test results are valuable IP |
| **Secrets** | Use Docker secrets or vault; never bake `OPENAI_API_KEY` into images |
| **API rate limiting** | LLM calls are expensive — rate-limit A/B test creation (e.g. 5 concurrent tests) |
| **Cost monitoring** | Track `total_cost_usd` per test; alert if monthly spend exceeds budget |
| **Dashboard auth** | v1 is internal-only; add OIDC/SAML for external access in v2 |
| **DB partitioning** | `test_results` grows fast (N items × 2 variants per test); partition by `test_id` range if >100K rows |
| **Prompt backup** | Export prompts + versions to JSON periodically (disaster recovery) |

---

## 12. Roadmap & Milestones

```
Week 1 (Days 1-5)
├── Phase 1: Foundation (DB + Prompt Store + Templating)  [█░░░░░░] Days 1-2
├── Phase 2: A/B Test Runner + LLM-as-Judge               [░█░░░░░] Days 3-4
└── Phase 3: Statistical Significance + Metric Tracking   [░░█░░░░] Day 5

Week 1.5 (Days 6-7)
├── Phase 4: Promotion/Rollback + Python SDK              [░░░█░░░] Day 6
└── Phase 5: Streamlit Dashboard                           [░░░░█░░] Day 7

Future (Post-v1)
├── Online A/B testing (route live traffic to prompt variants)
├── Multi-model comparison (same prompt, different models)
├── Prompt optimization (DSPy/OPRO integration for auto-improvement)
├── Git repository sync (bi-directional: DB ↔ git repo for prompts)
├── Multi-tenant support (team isolation, per-team prompt registries)
├── Prompt lineage tracking (which prompt produced which output in prod)
├── Automated regression testing (re-run eval suite on every new version)
├── CI/CD integration (block prompt merge if eval score drops)
├── Human evaluation interface (crowdsourced preference labeling)
└── Model cost optimization (recommend cheapest model that passes eval)
```

### Milestone Summary


| Milestone | Day | Deliverable |
|-----------|-----|-------------|
| M1: Prompt store + templating | Day 2 | Docker Compose up, prompt CRUD API works, Jinja2 templates render + validate |
| M2: A/B test runner | Day 4 | Run A/B test on eval dataset, collect per-item metrics (accuracy, latency, cost, tokens) |
| M3: Statistical significance | Day 5 | Win-rate, Wilson CI, z-test p-value, power analysis; winner declared only if significant |
| M4: Promotion + SDK | Day 6 | Promote/rollback via API, one-production invariant enforced, SDK loads production prompt |
| M5: Dashboard | Day 7 | Streamlit dashboard: overview, A/B results (win-rate chart + metric table + sample outputs), version history, metric trends |
| M6: Production-ready | Day 7 | Full test suite green, documented, `docker compose up` starts API + DB + dashboard |

---

## 13. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| **LLM-as-judge is unreliable** (scores are noisy, biased, or inconsistent) | High | High | Use temperature=0 for judge; use a different model than target to avoid self-preference; store judge reasoning for human review; run judge multiple times and average if budget allows; treat judge scores as one signal, not ground truth |
| **Sample size too small for significance** (common with expensive LLM calls) | High | Medium | Compute required sample size before running test; display it in dashboard; if test is inconclusive due to small n, recommend collecting more data rather than guessing |
| **Cost runaway** (large eval datasets × expensive models = high cost) | Medium | High | Estimate cost before test and display it; `sample_size` parameter limits items; track `total_cost_usd` per test; use cheaper models (gpt-4o-mini) for testing, expensive models (gpt-4o) only for final validation |
| **Prompt template injection** (malicious Jinja2 in template executes code) | Low | Critical | `SandboxedEnvironment` blocks Python internals; templates authored by trusted engineers only; compile-check + variable validation at API layer |
| **Promotion race condition** (two engineers promote simultaneously) | Low | High | Transactional promotion with DB-level partial unique index on `status='production'`; second promotion waits for first to commit |
| **LLM API downtime during test** (OpenAI/Ollama unavailable mid-test) | Medium | Medium | Retry with exponential backoff (3 attempts); mark failed items with `error` field; test can complete with partial results (significance computed on successful items only); dashboard shows completion rate |
| **Metric history grows unbounded** (every test adds rows to metric_history + test_results) | High | Low | Partition `test_results` by test_id range; archive old test results to cold storage; keep `metric_history` (aggregated) indefinitely — it's small |
| **Dashboard query performance degrades** (large DB → slow page loads) | Medium | Medium | Pre-aggregate metrics in `metric_history` (avoid re-aggregating `test_results` on every page load); add indexes on `prompt_id + test_date`; limit dashboard to recent N tests by default |
| **SDK cache returns stale prompt** (production prompt changed but SDK cache hasn't expired) | Medium | Medium | Configurable TTL (default 5 min); `clear_cache()` method for manual invalidation; dashboard shows current production version for cross-checking; consider webhook-based cache invalidation in v2 |
| **Judge prompt itself needs versioning** (the meta-prompt used for judging is also a prompt) | Medium | Low | v1: judge meta-prompt is hardcoded in `judge.py`; v2: store judge prompt in the same prompt store and version it (meta-prompt A/B testing) |

---

## Appendix

### A. Quick Start

```bash
# 1. Clone
git clone <repo-url> prompt-ab-testing
cd prompt-ab-testing

# 2. Configure
cp .env.example .env
# Edit .env: set OPENAI_API_KEY, POSTGRES_PASSWORD

# 3. Start all services
docker compose up -d

# 4. Wait for health
curl http://localhost:8000/health

# 5. Initialize database schema
python scripts/init_db.py

# 6. Seed sample data
python scripts/seed_data.py

# 7. Create a prompt
curl -X POST http://localhost:8000/prompts \
  -H "Content-Type: application/json" \
  -d '{"prompt_key": "summarizer", "name": "Article Summarizer", "tags": ["summarization"]}'

# 8. Create version 1
curl -X POST http://localhost:8000/prompts/<prompt_id>/versions \
  -H "Content-Type: application/json" \
  -d '{"template": "Summarize: {{ article }}", "variables": [{"name": "article", "type": "string", "required": true}], "message_type": "user", "changelog": "Initial version"}'

# 9. Create version 2 (improved)
curl -X POST http://localhost:8000/prompts/<prompt_id>/versions \
  -H "Content-Type: application/json" \
  -d '{"template": "Summarize in {{ max_words }} words: {{ article }}", "variables": [{"name": "article", "type": "string", "required": true}, {"name": "max_words", "type": "integer", "required": false, "default": 100}], "message_type": "user", "changelog": "Added word limit"}'

# 10. Run A/B test
curl -X POST http://localhost:8000/ab-tests \
  -H "Content-Type: application/json" \
  -d '{"prompt_id": "<prompt_id>", "version_a": 1, "version_b": 2, "dataset_id": "<dataset_id>", "sample_size": 50}'

# 11. Check results
curl http://localhost:8000/ab-tests/<test_id>

# 12. Promote winner
curl -X POST http://localhost:8000/prompts/<prompt_id>/promote \
  -H "Content-Type: application/json" \
  -d '{"version": 2, "reason": "Won A/B test"}'

# 13. Use SDK in your application
python -c "
from src.sdk.client import PromptClient
client = PromptClient('http://localhost:8000')
prompt = client.get_production_prompt('summarizer')
print(prompt.to_messages({'article': 'Long article text...', 'max_words': 50}))
"

# 14. View dashboard
open http://localhost:8501
```

### B. Key Design Decisions

| Decision | Chosen | Rejected Alternative | Why |
|----------|--------|----------------------|-----|
| Prompt store | PostgreSQL with versioning tables | Git repo + file-based prompts | DB enables SQL metric tracking, transactional promotion, and dashboard queries — the analytics engineer's native tool. Git sync is a v2 concern. |
| Versioning model | Append-only rows with auto-increment version per prompt_id | Full git-like DAG with branches | Simpler; covers 95% of use cases (linear version history with promotion). Branches/merges add complexity without clear value for prompts. |
| Templating | Jinja2 SandboxedEnvironment | f-strings, custom DSL, Mustache | Sandboxed for safety; StrictUndefined catches missing variables; supports conditionals/loops; widely known from Airflow/DAG templating. |
| Accuracy metric | LLM-as-judge (continuous 0-1 score) | Exact match, BLEU, ROUGE, BERTScore | LLM-as-judge captures semantic quality (not just token overlap); continuous score enables win-rate comparison; judge reasoning provides auditability. Exact match is too strict; BLEU/ROUGE are too narrow. |
| Significance test | Binomial z-test on win-rate + Wilson CI | t-test on raw scores, bootstrap, Bayesian | Win-rate is a proportion (binomial); z-test is the standard A/B testing approach the analytics engineer already knows; Wilson CI is more accurate than normal approximation for small samples. Bootstrap is computationally expensive; Bayesian adds complexity. |
| Dashboard | Streamlit + Plotly | Grafana, Superset, custom React | Python-native (same language as the system); fast to build; interactive charts; the analytics engineer already thinks in Streamlit/Plotly dashboards. |
| SDK rendering | Client-side Jinja2 with server-side option | Server-side only | Client-side reduces API calls (cache + render locally); Jinja2 is fast; server-side option available for non-Python clients via API. |
| One-production enforcement | DB partial unique index + app transaction | App-only check | DB index is a hard constraint — even if app has a bug, two versions can't be production simultaneously. Defense in depth. |
| Metric storage | Pre-aggregated in `metric_history` + raw in `test_results` | Compute on demand from `test_results` | Pre-aggregation makes dashboard queries fast (no re-aggregation on page load); raw results preserved for drill-down (sample outputs side-by-side). Same pattern as materialized views in dbt. |
| Concurrency in test runner | asyncio with semaphore (5 concurrent LLM calls) | Sequential, multiprocessing | asyncio is efficient for I/O-bound LLM calls; semaphore prevents rate-limit errors; sequential is too slow for 50+ items; multiprocessing is overkill for I/O-bound work. |

### C. The Analytics Engineer's Mental Model

| Concept (Analytics Engineering) | Equivalent (Prompt A/B Testing) |
|----------------------------------|----------------------------------|
| dbt model version | Prompt version (`prompt_versions` row) |
| dbt model promotion (staging → prod) | Prompt promotion (`status: draft → production`) |
| Dashboard A/B test (CTR variant A vs B) | Prompt A/B test (accuracy variant A vs B) |
| Conversion rate | Win-rate (fraction of items where variant wins) |
| Confidence interval on CTR | Wilson CI on win-rate |
| Sample size / power analysis | Required sample size for detecting effect size δ |
| p-value < 0.05 → ship the winner | p-value < 0.05 → promote the winning prompt |
| Experiment table (one row per experiment) | `ab_tests` table (one row per test) |
| Event table (one row per user event) | `test_results` table (one row per eval item × variant) |
| Materialized view (pre-aggregated metrics) | `metric_history` table (pre-aggregated per test × version) |
| Metric trend dashboard (CTR over time) | Metric trends page (accuracy over time per version) |
| Feature flag (roll out to X% of users) | Production prompt (served to 100% of SDK callers) |
| Rollback a deploy | Rollback a prompt promotion |
| Data quality tests (dbt tests) | Prompt validation (template compile check + variable validation) |
| Audit log (who ran what query) | `promotions` table (who promoted what version, when, why) |
