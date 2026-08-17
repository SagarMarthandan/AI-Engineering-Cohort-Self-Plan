# LLM Evaluation Harness — Detailed Implementation Plan

> **Elevator pitch:** A systematic evaluation framework for RAG/LLM outputs — the AI equivalent of dbt tests. Everyone can call the OpenAI API; almost nobody builds proper evaluation. This is where the data engineer's quality-testing mindset (dbt tests, Soda checks, Great Expectations suites) directly transfers to AI engineering. `dbt test = assert_trip_fare_is_positive` → `AI eval = assert_answer_is_grounded_in_context`.

---

## Table of Contents

1. [Project Goals & Non-Goals](#1-project-goals--non-goals)
2. [Architecture Overview](#2-architecture-overview)
3. [Tech Stack](#3-tech-stack)
4. [Database Schema](#4-database-schema)
5. [Eval Dataset Format](#5-eval-dataset-format)
6. [Project Structure](#6-project-structure)
7. [Implementation Phases](#7-implementation-phases)
8. [Component Specifications](#8-component-specifications)
9. [Metric Definitions](#9-metric-definitions)
10. [LLM-as-Judge](#10-llm-as-judge)
11. [Human Feedback Interface](#11-human-feedback-interface)
12. [Evaluation Dashboard](#12-evaluation-dashboard)
13. [CI Integration](#13-ci-integration)
14. [Security & Safety Considerations](#14-security--safety-considerations)
15. [API Specification](#15-api-specification)
16. [Testing Strategy](#16-testing-strategy)
17. [Deployment](#17-deployment)
18. [Roadmap & Milestones](#18-roadmap--milestones)
19. [Risk Register](#19-risk-register)
20. [Appendix](#20-appendix)

---

## 1. Project Goals & Non-Goals

### Goals

| # | Goal | Success Metric |
|---|------|----------------|
| G1 | **Systematic metric coverage** — implement faithfulness, answer relevance, context precision, context recall, citation accuracy, latency, cost | 7 metrics implemented, each unit-tested with a known-good and known-bad fixture |
| G2 | **LLM-as-judge with rubrics** — GPT-4 evaluates GPT-4o-mini outputs using structured, versioned rubrics | Judge outputs are structured JSON (score + reasoning); inter-rater agreement κ ≥ 0.6 vs. human gold labels on a 50-case calibration set |
| G3 | **Version-controlled eval datasets** — test cases stored in YAML, git-tracked, reviewed via PR | Every metric regression is traceable to a commit; dataset diff visible in PR |
| G4 | **Human feedback loop** — Streamlit dashboard for thumbs up/down + free-text on sampled Q&A pairs | Feedback persisted to PostgreSQL; ≥ 30% of sampled pairs annotated within first week |
| G5 | **Regression test suite in CI** — pytest integration that runs the eval dataset through a RAG system and asserts metric thresholds | GitHub Actions workflow runs on every PR; fails the build if any metric drops below threshold |
| G6 | **Evaluation dashboard** — Streamlit showing metric trends over time, per-prompt-version comparisons, failing test cases | Dashboard renders trend charts from `eval_runs` table; filterable by prompt version + dataset |
| G8 | **Context engineering evaluation** — test how context window management, compression, and ordering affect RAG quality | Context engineering metrics (context utilization, compression ratio, position bias) implemented; eval dataset includes long-context test cases |

### Non-Goals (explicitly out of scope)

- Building a RAG system from scratch (this harness *evaluates* an existing RAG system via a pluggable adapter)
- Online/production monitoring of live traffic (v1 is offline batch eval + CI; online observability is a follow-on project)
- Fine-tuning models to improve metrics (the harness measures; fixing is the RAG team's job)
- Multi-model benchmarking platforms (v1 judges one system-under-test at a time; A/B across models is a future extension)
- Auto-generating eval datasets from production logs (v1 datasets are human-authored; log mining is future work)
- Real-time eval during user conversations (batch only)

### The dbt → AI Eval Parallel

This project is deliberately framed for a data engineer audience. The mental model transfer is:

| Data Engineering (dbt / Soda / GX) | AI Engineering (this harness) |
|---|---|
| `assert_trip_fare_is_positive` | `assert_answer_is_grounded_in_context` |
| `assert_no_nulls(passenger_id)` | `assert_citations_match_source_chunks` |
| `assert_row_count_change < 20%` | `assert_faithfulness_score >= 0.85` |
| Freshness check on source tables | Latency p95 ≤ threshold |
| Soda scan in Airflow DAG | Eval suite in GitHub Actions PR check |
| `dbt test --select tag:critical` | `pytest -m critical_eval` |
| Test results in dbt artifacts JSON | Eval results in `eval_runs` + `eval_case_results` tables |
| Data quality dashboard in Metabase | Eval dashboard in Streamlit |

The harness is a **test framework**, not a monitoring tool. You write test cases (like dbt tests), you run them in CI (like `dbt test`), and you fail the build when quality regresses (like a failing dbt test blocks merge).

---

## 2. Architecture Overview

```
                         ┌──────────────────────────────────────────────────────────┐
                         │                    Developer / CI                         │
                         │  git push → PR → GitHub Actions → pytest eval suite       │
                         └────────────────────────────┬─────────────────────────────┘
                                                      │
                         ┌────────────────────────────▼─────────────────────────────┐
                         │                  Evaluation Harness (Python)              │
                         │                                                          │
                         │  ┌──────────────┐   ┌───────────────   ┌──────────────┐  │
                         │  │  Eval Dataset │   │  System Under   │   │  Metrics     │  │
                         │  │  Loader       │   │  Test (SUT)     │   │  Engine      │  │
                         │  │  (YAML/JSON)  │   │  Adapter        │   │              │  │
                         │  └──────┬───────┘   └───────┬────────   └──────┬───────┘  │
                         │         │                   │                  │          │
                         │         │  test cases       │  query + ctx     │          │
                         │         ▼                   ▼                  │          │
                         │  ┌──────────────────────────────────┐          │          │
                         │  │        Eval Runner (Orchestrator) │◄─────────┘          │
                         │  │  for each case: run SUT → score   │                     │
                         │  └──────────────┬───────────────────┘                     │
                         │                 │ results                                  │
                         │     ┌───────────┼───────────────┐                         │
                         │     ▼           ▼               ▼                         │
                         │  ┌──────┐  ┌─────────┐    ┌──────────────┐                │
                         │  │ LLM- │  │ Result  │    │  Threshold   │                │
                         │  │ Judge│  │ Writer  │    │  Assertor    │                │
                         │  │(GPT4)│  │ (→ DB)  │    │ (pytest)     │                │
                         │  └──┬───┘  └────┬────┘    └──────┬───────┘                │
                         └─────┼───────────┼─────────────────┼──────────────────────┘
                               │           │                 │
                    ┌──────────▼──┐  ┌─────▼──────┐  ┌───────▼────────┐
                    │ OpenAI API  │  │ PostgreSQL │  │ pytest report  │
                    │ GPT-4 judge │  │ eval_runs  │  │ (CI pass/fail) │
                    │ GPT-4o-mini │  │ eval_case_ │  └────────────────┘
                    │   (SUT)     │  │  results   │
                    └─────────────┘  │ human_     │
                                     │ feedback   │
                                     └─────┬──────┘
                                           │
                          ┌────────────────┼────────────────┐
                          ▼                                 ▼
                 ┌─────────────────┐              ┌──────────────────┐
                 │  Streamlit:     │              │  Streamlit:      │
                 │  Human Feedback │              │  Eval Dashboard  │
                 │  (thumbs, text) │              │  (trends, fails) │
                 └─────────────────┘              └──────────────────┘
```

### Eval Run Flow

```
1. Load eval dataset (YAML) → list[EvalCase]
2. For each EvalCase:
   a. Call SUT adapter: sut.answer(question) → {answer, retrieved_contexts, citations, latency_ms, tokens}
   b. Compute deterministic metrics (latency, cost, context precision/recall, citation accuracy)
   c. Compute LLM-as-judge metrics (faithfulness, answer relevance) via GPT-4
   d. Assemble CaseResult {case_id, metrics{}, judge_reasoning, raw_output}
3. Aggregate: EvalRunResult {run_id, prompt_version, dataset_version, mean metrics, pass/fail per case}
4. Persist: INSERT into eval_runs + eval_case_results
5. Assert thresholds: if any metric < threshold → pytest.fail() (in CI context)
6. Return structured report (JSON + DB rows)
```

---

## 3. Tech Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **Language** | Python 3.11+ | Ecosystem maturity, TypedDict/Pydantic v2, asyncio |
| **Validation** | Pydantic v2 | Strict schema for eval cases, metric results, judge outputs; `model_validate` with `extra='forbid'` catches dataset typos |
| **LLM Judge** | OpenAI GPT-4 (or GPT-4o) | Strongest reasoning for rubric-based grading; structured output via `response_format={"type": "json_object"}` |
| **SUT LLM** | OpenAI GPT-4o-mini | The system under test — cheap, fast; the harness evaluates its outputs |
| **Eval Dataset** | YAML (PyYAML) | Human-readable, git-diffable, reviewable in PRs; JSON schema validation via Pydantic |
| **Results Store** | PostgreSQL 16 | Relational store for eval runs, per-case results, human feedback; time-series queries for trend charts |
| **Dashboard** | Streamlit 1.30+ | Fastest path to internal data dashboard; pandas + plotly charts; Python-native, no JS |
| **Test Framework** | pytest 8 + pytest-asyncio | Industry standard; custom markers (`@pytest.mark.eval`), fixtures for SUT adapter, parametrize over eval dataset |
| **CI** | GitHub Actions | Native to the repo; `pull_request` trigger; cache pip deps; store eval report as artifact |
| **Containerization** | Docker + Docker Compose | Reproducible eval environment; Postgres + harness + dashboards in one `docker compose up` |
| **Charting** | Plotly | Interactive trend lines, per-version bar charts; Streamlit-native |
| **Linting** | ruff | Replaces flake8 + isort + black; fast CI |
| **DB Migrations** | Alembic | Versioned schema for eval tables; reproducible across dev/CI/prod |

### Why GPT-4 as judge and GPT-4o-mini as SUT?

The **judge must be stronger than the system under test**. If the judge is the same model as the SUT, it shares the same blind spots and biases — a model won't reliably catch its own hallucinations. GPT-4 (or GPT-4o) as judge over GPT-4o-mini outputs gives a genuine quality signal. This is the "stronger model grades weaker model" pattern from the LLM-as-judge literature (Zheng et al., 2023; RAGAS). When the SUT is upgraded to GPT-4 itself, the judge should be upgraded to GPT-4o or a frontier model — the judge is always ≥ one tier above the SUT.

### Why PostgreSQL and not just JSON files?

Eval results are **time-series**. To answer "did faithfulness drop after the prompt change on Tuesday?" you need to query across runs, join with prompt versions, and aggregate. JSON files per run work for a week; at 50+ runs they become unqueryable. PostgreSQL gives us indexed `eval_runs(timestamp, prompt_version)` and `eval_case_results(run_id, metric_name, score)` — the same instinct that makes a data engineer choose a warehouse over CSV files.

---

## 4. Database Schema

### 4.1 Extensions

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
```

### 4.2 Tables

```sql
-- =====================================================
-- PROMPT_VERSIONS: tracks the prompt template under test
-- =====================================================
CREATE TABLE prompt_versions (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    version_tag     TEXT NOT NULL UNIQUE,        -- e.g. 'v1.2', 'prompt-cite-v3'
    system_prompt   TEXT NOT NULL,
    user_prompt_tpl TEXT NOT NULL,               -- template with {question} {context}
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by      TEXT
);

-- =====================================================
-- EVAL_DATASETS: metadata for version-controlled test sets
-- =====================================================
CREATE TABLE eval_datasets (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name            TEXT NOT NULL,               -- e.g. 'customer-support-faq'
    version         TEXT NOT NULL,               -- git short SHA or semver
    case_count      INT NOT NULL,
    file_path       TEXT NOT NULL,               -- path in repo, e.g. 'evals/datasets/cs_faq.yaml'
    file_sha        TEXT NOT NULL,               -- git blob SHA for provenance
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(name, version)
);

-- =====================================================
-- EVAL_RUNS: one row per execution of the harness
-- =====================================================
CREATE TABLE eval_runs (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    started_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    finished_at         TIMESTAMPTZ,
    prompt_version_id   UUID REFERENCES prompt_versions(id),
    eval_dataset_id     UUID REFERENCES eval_datasets(id),
    git_sha             TEXT,                    -- code version under test
    environment         TEXT NOT NULL,           -- 'ci', 'local', 'staging'
    status              TEXT NOT NULL,           -- 'running', 'completed', 'failed', 'aborted'

    -- Aggregate metrics (mean across cases)
    mean_faithfulness       DOUBLE PRECISION,
    mean_answer_relevance   DOUBLE PRECISION,
    mean_context_precision  DOUBLE PRECISION,
    mean_context_recall     DOUBLE PRECISION,
    mean_citation_accuracy  DOUBLE PRECISION,
    p50_latency_ms          INT,
    p95_latency_ms          INT,
    total_tokens            INT,
    total_cost_usd          NUMERIC(10,4),
    pass_count              INT,
    fail_count              INT,

    CONSTRAINT valid_status CHECK (status IN ('running','completed','failed','aborted'))
);

CREATE INDEX idx_eval_runs_time ON eval_runs(started_at DESC);
CREATE INDEX idx_eval_runs_prompt ON eval_runs(prompt_version_id, started_at DESC);

-- =====================================================
-- EVAL_CASE_RESULTS: per-test-case metric scores
-- =====================================================
CREATE TABLE eval_case_results (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    run_id              UUID NOT NULL REFERENCES eval_runs(id) ON DELETE CASCADE,
    case_id             TEXT NOT NULL,           -- matches EvalCase.id in YAML
    question            TEXT NOT NULL,
    expected_answer     TEXT,
    generated_answer    TEXT,
    retrieved_contexts  JSONB,                   -- array of {chunk_id, content, score}
    expected_contexts   JSONB,
    citations           JSONB,                   -- array of {chunk_id, snippet}

    -- Per-case metric scores (0.0–1.0 unless noted)
    faithfulness        DOUBLE PRECISION,
    answer_relevance    DOUBLE PRECISION,        -- 1–5 scale, normalized
    context_precision   DOUBLE PRECISION,
    context_recall      DOUBLE PRECISION,
    citation_accuracy   DOUBLE PRECISION,
    latency_ms          INT,
    tokens_in           INT,
    tokens_out          INT,
    cost_usd            NUMERIC(10,6),

    -- Context engineering metrics (§9.8)
    context_utilization     DOUBLE PRECISION,    -- fraction of retrieved context actually used
    compression_ratio       DOUBLE PRECISION,    -- tokens after compression / before (1.0 = no compression)
    position_bias           DOUBLE PRECISION,    -- 1.0 = no bias; < 0.7 = significant "lost in middle" effect
    context_overflow        BOOLEAN,             -- true if retrieved context exceeded model window

    -- LLM-as-judge reasoning (for debugging / dashboard)
    judge_reasoning     TEXT,
    judge_raw           JSONB,                   -- full structured judge response

    passed              BOOLEAN NOT NULL,        -- all metrics above threshold?
    failure_reasons     TEXT[],                  -- e.g. ['faithfulness: 0.60 < 0.85']

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_case_results_run ON eval_case_results(run_id);
CREATE INDEX idx_case_results_case ON eval_case_results(case_id);
CREATE INDEX idx_case_results_pass ON eval_case_results(run_id, passed);

-- =====================================================
-- HUMAN_FEEDBACK: thumbs up/down + free-text from dashboard
-- =====================================================
CREATE TABLE human_feedback (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    case_result_id  UUID NOT NULL REFERENCES eval_case_results(id) ON DELETE CASCADE,
    run_id          UUID NOT NULL REFERENCES eval_runs(id) ON DELETE CASCADE,
    annotator       TEXT NOT NULL,               -- user email / id
    rating          TEXT NOT NULL,               -- 'up', 'down'
    comment         TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT valid_rating CHECK (rating IN ('up','down'))
);

CREATE INDEX idx_feedback_case ON human_feedback(case_result_id);
CREATE INDEX idx_feedback_run ON human_feedback(run_id);
```

### 4.3 Key Queries

```sql
-- Metric trend over time for a given prompt version lineage
SELECT started_at, mean_faithfulness, mean_answer_relevance
FROM eval_runs
WHERE prompt_version_id = $1 AND status = 'completed'
ORDER BY started_at;

-- Failing test cases in the latest run
SELECT case_id, question, failure_reasons, judge_reasoning
FROM eval_case_results
WHERE run_id = $1 AND passed = FALSE
ORDER BY faithfulness ASC;

-- Compare two prompt versions side-by-side
SELECT metric, a.value AS version_a, b.value AS version_b, (b.value - a.value) AS delta
FROM (...) -- unpivot mean metrics for two run IDs
```

---

## 5. Eval Dataset Format

Eval datasets are YAML files, version-controlled in the repo under `evals/datasets/`. Each file is a list of test cases. The schema is enforced by a Pydantic model at load time — a typo or missing field fails fast, just like a malformed dbt YAML.

### 5.1 YAML Format

```yaml
# evals/datasets/customer_support_faq.yaml
metadata:
  name: customer-support-faq
  description: >-
    Core FAQ test cases for the customer support RAG. Covers billing,
    returns, and account access. Each case has a gold answer and the
    context chunks that SHOULD be retrieved.
  version: "1.0.0"
  author: sagar

# Thresholds applied to ALL cases in this dataset unless overridden per-case
thresholds:
  faithfulness: 0.85
  answer_relevance: 4.0        # out of 5
  context_precision: 0.75
  context_recall: 0.80
  citation_accuracy: 0.90
  p95_latency_ms: 3000

cases:
  - id: cs-billing-001
    question: "How do I get a refund for a duplicate charge?"
    expected_answer: >-
      To get a refund for a duplicate charge, log into your account,
      go to Billing > Transactions, select the duplicate charge, and
      click 'Request Refund'. Refunds are processed within 5–7 business
      days to the original payment method.
    expected_contexts:
      - chunk_id: "billing-faq#chunk-04"
        content: "Duplicate charges can be refunded from Billing > Transactions. Select the charge and click 'Request Refund'. Processing time is 5–7 business days."
      - chunk_id: "billing-faq#chunk-05"
        content: "Refunds are always returned to the original payment method."
    expected_citations:
      - chunk_id: "billing-faq#chunk-04"
    tags: [billing, refund, critical]

  - id: cs-returns-002
    question: "What is the return window for opened electronics?"
    expected_answer: >-
      Opened electronics can be returned within 14 days of purchase
      with the original packaging. A 15% restocking fee applies.
    expected_contexts:
      - chunk_id: "returns-policy#chunk-02"
        content: "Electronics: 14-day return window. Opened items require original packaging. 15% restocking fee for opened electronics."
    expected_citations:
      - chunk_id: "returns-policy#chunk-02"
    tags: [returns, electronics]
    # Per-case threshold override
    thresholds:
      faithfulness: 0.90   # stricter for this critical case

  - id: cs-account-003
    question: "How do I enable two-factor authentication?"
    expected_answer: >-
      Enable 2FA in Account > Security > Two-Factor Authentication.
      Scan the QR code with an authenticator app and enter the 6-digit code.
    expected_contexts:
      - chunk_id: "account-security#chunk-01"
        content: "Two-factor authentication: Account > Security > 2FA. Scan QR, enter 6-digit code from authenticator app."
    expected_citations:
      - chunk_id: "account-security#chunk-01"
    tags: [account, security, critical]
```

### 5.2 Pydantic Schema (enforced at load)

```python
class ExpectedContext(BaseModel):
    chunk_id: str
    content: str
    model_config = ConfigDict(extra="forbid")

class EvalCase(BaseModel):
    id: str
    question: str
    expected_answer: str
    expected_contexts: list[ExpectedContext]
    expected_citations: list[ExpectedContext] = []   # subset of expected_contexts
    tags: list[str] = []
    thresholds: dict[str, float] | None = None        # per-case overrides

class EvalDatasetMetadata(BaseModel):
    name: str
    description: str = ""
    version: str
    author: str = ""

class Thresholds(BaseModel):
    faithfulness: float = 0.85
    answer_relevance: float = 4.0
    context_precision: float = 0.75
    context_recall: float = 0.80
    citation_accuracy: float = 0.90
    p95_latency_ms: int = 3000

class EvalDataset(BaseModel):
    metadata: EvalDatasetMetadata
    thresholds: Thresholds
    cases: list[EvalCase]
    model_config = ConfigDict(extra="forbid")
```

The `extra="forbid"` on every model means a misspelled key in the YAML (e.g. `threshlds:`) raises a `ValidationError` at load — **fail fast, fail loud**, the same principle as dbt's schema enforcement.

---

## 6. Project Structure

```
llm-eval-harness/
├── evals/                              # Version-controlled eval assets
│   ├── datasets/
│   │   ├── customer_support_faq.yaml
│   │   ├── product_docs.yaml
│   │   └── edge_cases.yaml
│   ├── rubrics/
│   │   ├── faithfulness.yaml            # LLM-as-judge rubric definitions
│   │   └── answer_relevance.yaml
│   └── thresholds/
│       └── default.yaml
│
├── src/
│   └── eval_harness/
│       ├── __init__.py
│       ├── models.py                    # Pydantic: EvalCase, CaseResult, EvalRunResult
│       ├── dataset.py                   # YAML loader + schema validation
│       ├── runner.py                    # Orchestrator: load → run SUT → score → persist
│       ├── sut/
│       │   ├── __init__.py
│       │   ├── base.py                  # SUTAdapter ABC
│       │   ├── rag_adapter.py           # Adapter for the RAG system under test
│       │   └── mock_adapter.py          # Deterministic adapter for unit tests
│       ├── metrics/
│       │   ├── __init__.py
│       │   ├── base.py                  # Metric ABC: name, compute(case, sut_output) -> score
│       │   ├── faithfulness.py          # LLM-as-judge
│       │   ├── answer_relevance.py      # LLM-as-judge
│       │   ├── context_precision.py     # chunk overlap (deterministic)
│       │   ├── context_recall.py        # chunk overlap (deterministic)
│       │   ├── citation_accuracy.py     # citation ↔ source matching
│       │   ├── latency.py               # passthrough from SUT timing
│       │   └── cost.py                  # tokens × price
│       ├── judge/
│       │   ├── __init__.py
│       │   ├── llm_judge.py             # GPT-4 judge client, structured output
│       │   └── prompts.py               # Rubric prompt templates (versioned)
│       ├── storage/
│       │   ├── __init__.py
│       │   ├── db.py                    # AsyncPG connection pool
│       │   ├── repository.py            # INSERT/SELECT eval_runs, case_results, feedback
│       │   └── migrations/              # Alembic migrations
│       │       ├── env.py
│       │       └── versions/
│       ├── assertor.py                  # Threshold checks → pass/fail per case
│       ├── report.py                    # JSON + Markdown report generation
│       └── config.py                    # Settings (pydantic-settings, env-driven)
│
├── dashboards/
│   ├── eval_dashboard.py                # Streamlit: metric trends, version compare, failures
│   └── human_feedback.py                # Streamlit: sample Q&A, thumbs up/down, comments
│
├── tests/
│   ├── unit/
│   │   ├── test_dataset_loader.py
│   │   ├── test_metrics.py              # Each metric with known-good/bad fixtures
│   │   ├── test_judge.py                # Mock judge client
│   │   └── test_assertor.py
│   ├── integration/
│   │   ├── test_runner_e2e.py           # Full run with mock SUT + mock judge
│   │   └── test_storage.py              # DB round-trip
│   └── eval/
│       └── test_regression.py           # pytest entry point: runs eval dataset, asserts thresholds
│
├── .github/
│   └── workflows/
│       └── eval-suite.yml               # PR-triggered eval run
│
├── alembic.ini
├── docker-compose.yml
├── Dockerfile
├── pyproject.toml
├── .env.example
├── README.md
├── agent.md                      # Herdr multi-agent orchestration guide
```

---

## 7. Implementation Phases

**Duration: 10 working days (~2 weeks).** Each day has a deliverable and a verification step.

### Phase 1: Foundation (Days 1–3)

#### Day 1 — Project scaffold, schema, dataset loader

**Deliverables:**
- `pyproject.toml` with deps: pydantic v2, pydantic-settings, pyyaml, asyncpg, openai, streamlit, plotly, pytest, pytest-asyncio, ruff, alembic
- `src/eval_harness/models.py` — all Pydantic models (`EvalCase`, `CaseResult`, `EvalRunResult`, `Thresholds`)
- `src/eval_harness/dataset.py` — YAML loader with schema validation
- `evals/datasets/customer_support_faq.yaml` — seed dataset with 10 cases
- `src/eval_harness/config.py` — settings from env (DB URL, OpenAI key, judge model, SUT model)

**Verification:**
```bash
python -c "from eval_harness.dataset import load_dataset; ds = load_dataset('evals/datasets/customer_support_faq.yaml'); print(len(ds.cases))"
# → 10
# Malformed YAML (rename 'thresholds' to 'threshlds') → ValidationError raised
```

#### Day 2 — Database schema, migrations, storage layer

**Deliverables:**
- Alembic migration creating `prompt_versions`, `eval_datasets`, `eval_runs`, `eval_case_results`, `human_feedback`
- `src/eval_harness/storage/db.py` — asyncpg pool
- `src/eval_harness/storage/repository.py` — `insert_run()`, `insert_case_result()`, `insert_feedback()`, `get_runs()`, `get_failing_cases()`
- `docker-compose.yml` with Postgres 16

**Verification:**
```bash
docker compose up -d postgres
alembic upgrade head
python -c "import asyncio; from eval_harness.storage.repository import insert_run; asyncio.run(insert_run({...}))"
# → row in eval_runs
```

#### Day 3 — SUT adapter, deterministic metrics (context precision/recall, citation accuracy, latency, cost)

**Deliverables:**
- `src/eval_harness/sut/base.py` — `SUTAdapter` ABC: `async answer(question: str) -> SUTOutput`
- `src/eval_harness/sut/mock_adapter.py` — deterministic adapter returning canned answers (for unit tests, no API calls)
- `src/eval_harness/sut/rag_adapter.py` — real adapter calling the RAG system (HTTP or direct)
- `metrics/context_precision.py`, `context_recall.py`, `citation_accuracy.py`, `latency.py`, `cost.py`

**Verification:**
```bash
pytest tests/unit/test_metrics.py -v
# → all metric tests pass with known-good (score=1.0) and known-bad (score=0.0) fixtures
```

### Phase 2: LLM-as-Judge & Runner (Days 4–6)

#### Day 4 — LLM-as-judge: faithfulness metric

**Deliverables:**
- `src/eval_harness/judge/llm_judge.py` — GPT-4 client with structured JSON output
- `src/eval_harness/judge/prompts.py` — faithfulness rubric prompt (see §10)
- `metrics/faithfulness.py` — decompose answer into claims, verify each against context

**Verification:**
```bash
pytest tests/unit/test_metrics.py::test_faithfulness_grounded -v
pytest tests/unit/test_metrics.py::test_faithfulness_hallucinated -v
# grounded answer → faithfulness ≈ 1.0; hallucinated claim → faithfulness < 0.5
```

#### Day 5 — LLM-as-judge: answer relevance metric + judge calibration

**Deliverables:**
- `metrics/answer_relevance.py` — 1–5 rubric scoring
- `evals/rubrics/answer_relevance.yaml` — rubric definition
- Calibration script: run judge on 50-case gold set, compute Cohen's κ vs. human labels

**Verification:**
```bash
python -m eval_harness.judge.calibrate --gold evals/gold_labels.csv
# → Cohen's κ ≥ 0.6 (target); if below, iterate rubric
```

#### Day 6 — Eval runner orchestrator + assertor + report

**Deliverables:**
- `src/eval_harness/runner.py` — `async run_eval(dataset, sut, judge, thresholds) -> EvalRunResult`
- `src/eval_harness/assertor.py` — per-case threshold checks → `passed: bool`, `failure_reasons: list[str]`
- `src/eval_harness/report.py` — JSON + Markdown report
- `tests/integration/test_runner_e2e.py` — full run with mock SUT + mock judge, assert results persisted

**Verification:**
```bash
python -m eval_harness.runner --dataset evals/datasets/customer_support_faq.yaml --sut mock
# → EvalRunResult JSON printed, rows in eval_runs + eval_case_results, Markdown report written
pytest tests/integration/test_runner_e2e.py -v
```

### Phase 3: CI, Dashboards, Human Feedback (Days 7–10)

#### Day 7 — pytest integration + CI workflow

**Deliverables:**
- `tests/eval/test_regression.py` — pytest parametrized over eval dataset, asserts thresholds (see §13)
- `.github/workflows/eval-suite.yml` — PR-triggered, runs eval suite, uploads report artifact, fails on regression (see §13)
- `@pytest.mark.eval` marker; `@pytest.mark.critical_eval` for high-priority cases

**Verification:**
```bash
pytest tests/eval/test_regression.py -v
# → N passed (one per case), or FAILED with metric name + score + threshold
# Push a PR → GitHub Actions runs eval-suite.yml → check passes/fails
```

#### Day 8 — Evaluation dashboard (Streamlit)

**Deliverables:**
- `dashboards/eval_dashboard.py` — metric trend charts (plotly), per-prompt-version comparison, failing cases table, cost/latency trends (see §12)

**Verification:**
```bash
streamlit run dashboards/eval_dashboard.py
# → dashboard loads, shows trend line for faithfulness over last 10 runs,
#   failing cases table populated from latest run
```

#### Day 9 — Human feedback interface (Streamlit)

**Deliverables:**
- `dashboards/human_feedback.py` — sample Q&A pairs, thumbs up/down, free-text comment, annotator name, persists to `human_feedback` table (see §11)

**Verification:**
```bash
streamlit run dashboards/human_feedback.py
# → click thumbs down on a case, enter comment → row in human_feedback table
```

#### Day 10 — Dockerization, docs, end-to-end smoke test, polish

**Deliverables:**
- `Dockerfile` (harness + dashboards)
- `docker-compose.yml` updated: postgres + harness + eval-dashboard + human-feedback
- `.env.example` with all env vars documented
- `README.md` quick start
- End-to-end smoke: `docker compose up` → run eval → open dashboards → verify data flows

**Verification:**
```bash
docker compose up -d
docker compose run harness python -m eval_harness.runner --dataset evals/datasets/customer_support_faq.yaml --sut mock
# → eval runs, results in Postgres, dashboards show data
```

---

## 8. Component Specifications

### 8.1 SUT Adapter (ABC)

The System Under Test is pluggable. The harness never assumes the RAG system's internals — it only needs an async function that takes a question and returns an answer + retrieved contexts + citations + timing.

```python
# src/eval_harness/sut/base.py
from abc import ABC, abstractmethod
from eval_harness.models import SUTOutput

class SUTAdapter(ABC):
    """Adapter for the system under test (RAG / LLM pipeline)."""

    @abstractmethod
    async def answer(self, question: str) -> SUTOutput:
        """
        Run the SUT on a single question.
        Returns: SUTOutput(answer, retrieved_contexts, citations, latency_ms, tokens_in, tokens_out)
        """
        ...

    @property
    @abstractmethod
    def name(self) -> str:
        """Identifier for the SUT (e.g. 'rag-gpt4o-mini-v2')."""
        ...
```

```python
# src/eval_harness/models.py
class RetrievedContext(BaseModel):
    chunk_id: str
    content: str
    score: float = 0.0          # retrieval similarity score

class Citation(BaseModel):
    chunk_id: str
    snippet: str = ""

class SUTOutput(BaseModel):
    answer: str
    retrieved_contexts: list[RetrievedContext]
    citations: list[Citation] = []
    latency_ms: int
    tokens_in: int = 0
    tokens_out: int = 0
```

### 8.2 Mock Adapter (deterministic, for unit tests)

```python
# src/eval_harness/sut/mock_adapter.py
class MockSUTAdapter(SUTAdapter):
    """Returns canned answers keyed by question prefix. No API calls."""

    def __init__(self, responses: dict[str, SUTOutput]):
        self._responses = responses

    @property
    def name(self) -> str:
        return "mock-sut"

    async def answer(self, question: str) -> SUTOutput:
        for key, output in self._responses.items():
            if question.startswith(key):
                return output
        # Default: return a generic grounded answer
        return SUTOutput(
            answer="I don't have information about that.",
            retrieved_contexts=[],
            citations=[],
            latency_ms=50,
        )
```

### 8.3 Metric Base Class

```python
# src/eval_harness/metrics/base.py
from abc import ABC, abstractmethod
from eval_harness.models import EvalCase, SUTOutput

class Metric(ABC):
    """Base class for all evaluation metrics."""

    name: str

    @abstractmethod
    async def compute(self, case: EvalCase, sut_output: SUTOutput) -> "MetricResult":
        ...

class MetricResult(BaseModel):
    name: str
    score: float                 # 0.0–1.0 (or raw for latency)
    details: dict = {}           # e.g. per-claim faithfulness breakdown
    reasoning: str | None = None # LLM-as-judge explanation
```

### 8.4 Eval Runner (Orchestrator)

```python
# src/eval_harness/runner.py
async def run_eval(
    dataset: EvalDataset,
    sut: SUTAdapter,
    judge: LLMJudge,
    metrics: list[Metric],
    thresholds: Thresholds,
    repo: EvalRepository,
    prompt_version_id: UUID | None = None,
    git_sha: str | None = None,
    environment: str = "local",
) -> EvalRunResult:

    run_id = await repo.insert_run(
        prompt_version_id=prompt_version_id,
        eval_dataset_id=...,  # resolved from dataset metadata
        git_sha=git_sha,
        environment=environment,
        status="running",
    )

    case_results: list[CaseResult] = []
    for case in dataset.cases:
        # 1. Run the SUT
        sut_output = await sut.answer(case.question)

        # 2. Score every metric
        metric_scores: dict[str, MetricResult] = {}
        for metric in metrics:
            result = await metric.compute(case, sut_output)
            metric_scores[metric.name] = result

        # 3. Assert thresholds (per-case override → dataset default)
        case_thresholds = case.thresholds or thresholds
        passed, failures = assertor.check(metric_scores, case_thresholds)

        # 4. Assemble + persist
        cr = CaseResult(
            run_id=run_id, case_id=case.id, question=case.question,
            expected_answer=case.expected_answer,
            generated_answer=sut_output.answer,
            retrieved_contexts=[c.model_dump() for c in sut_output.retrieved_contexts],
            expected_contexts=[c.model_dump() for c in case.expected_contexts],
            citations=[c.model_dump() for c in sut_output.citations],
            faithfulness=metric_scores["faithfulness"].score,
            answer_relevance=metric_scores["answer_relevance"].score,
            context_precision=metric_scores["context_precision"].score,
            context_recall=metric_scores["context_recall"].score,
            citation_accuracy=metric_scores["citation_accuracy"].score,
            latency_ms=sut_output.latency_ms,
            tokens_in=sut_output.tokens_in,
            tokens_out=sut_output.tokens_out,
            cost_usd=cost_metric.compute(sut_output),
            judge_reasoning=metric_scores["faithfulness"].reasoning,
            judge_raw=metric_scores["faithfulness"].details,
            passed=passed,
            failure_reasons=failures,
        )
        await repo.insert_case_result(cr)
        case_results.append(cr)

    # 5. Aggregate
    run_result = EvalRunResult.from_case_results(run_id, case_results)
    await repo.update_run_aggregates(run_id, run_result)
    await repo.update_run_status(run_id, "completed")

    return run_result
```

### 8.5 Assertor

```python
# src/eval_harness/assertor.py
def check(
    metric_scores: dict[str, MetricResult],
    thresholds: Thresholds,
) -> tuple[bool, list[str]]:
    failures: list[str] = []
    checks = [
        ("faithfulness",       thresholds.faithfulness,      ">="),
        ("answer_relevance",   thresholds.answer_relevance,  ">="),
        ("context_precision",  thresholds.context_precision, ">="),
        ("context_recall",     thresholds.context_recall,    ">="),
        ("citation_accuracy",  thresholds.citation_accuracy, ">="),
        ("p95_latency_ms",     thresholds.p95_latency_ms,    "<="),
    ]
    for name, threshold, op in checks:
        score = metric_scores[name].score
        if op == ">=" and score < threshold:
            failures.append(f"{name}: {score:.2f} < {threshold}")
        elif op == "<=" and score > threshold:
            failures.append(f"{name}: {score} > {threshold}")
    return (len(failures) == 0, failures)
```

---

## 9. Metric Definitions

Each metric is defined with its **formula/pseudocode**, **range**, and **interpretation**. Deterministic metrics require no LLM; LLM-as-judge metrics call GPT-4.

### 9.1 Faithfulness (LLM-as-judge)

**Question:** Is every claim in the answer grounded in the provided context?

**Range:** 0.0 – 1.0

**Formula:**
$$\text{faithfulness} = \frac{\text{number of claims supported by context}}{\text{total number of claims in answer}}$$

**Pseudocode:**
```python
async def compute(self, case, sut_output) -> MetricResult:
    # Step 1: Decompose the answer into atomic claims
    claims = await self.judge.decompose_claims(sut_output.answer)
    #   → ["The refund window is 14 days", "A 15% restocking fee applies", ...]

    # Step 2: For each claim, ask judge: is this claim supported by the context?
    context_text = "\n".join(c.content for c in sut_output.retrieved_contexts)
    supported = 0
    per_claim = []
    for claim in claims:
        verdict = await self.judge.verify_claim(claim, context_text)
        #   → {"supported": true/false, "reasoning": "..."}
        per_claim.append(verdict)
        if verdict["supported"]:
            supported += 1

    score = supported / len(claims) if claims else 1.0
    return MetricResult(
        name="faithfulness",
        score=score,
        details={"claims": per_claim},
        reasoning=f"{supported}/{len(claims)} claims supported by context",
    )
```

**dbt parallel:** `assert_answer_is_grounded_in_context` — every "fact" in the answer must trace to a source row, just like every metric in a dbt model must trace to a source column.

### 9.2 Answer Relevance (LLM-as-judge)

**Question:** Does the answer actually address the question asked?

**Range:** 1 – 5 (integer rubric), normalized to 0.0–1.0 as `score / 5.0` for aggregation

**Rubric:**
| Score | Definition |
|-------|-----------|
| 5 | Directly and completely answers the question; no irrelevant information |
| 4 | Answers the question but includes minor irrelevant tangents |
| 3 | Partially answers; misses an aspect of the question |
| 2 | Mostly irrelevant; touches the topic but doesn't answer |
| 1 | Completely irrelevant or wrong topic |

**Pseudocode:**
```python
async def compute(self, case, sut_output) -> MetricResult:
    score = await self.judge.score_relevance(
        question=case.question,
        answer=sut_output.answer,
    )
    # → {"score": 4, "reasoning": "Answers the refund question but adds unnecessary detail about shipping."}
    return MetricResult(
        name="answer_relevance",
        score=score["score"] / 5.0,   # normalized for aggregation
        details={"raw_score": score["score"]},
        reasoning=score["reasoning"],
    )
```

### 9.3 Context Precision (deterministic)

**Question:** Were the retrieved chunks actually relevant? (Precision = did we retrieve junk?)

**Range:** 0.0 – 1.0

**Formula:**
$$\text{context\_precision} = \frac{|\text{retrieved\_chunks} \cap \text{expected\_relevant\_contexts}|}{|\text{retrieved\_chunks}|}$$

**Pseudocode:**
```python
def compute(self, case, sut_output) -> MetricResult:
    retrieved_ids = {c.chunk_id for c in sut_output.retrieved_contexts}
    expected_ids  = {c.chunk_id for c in case.expected_contexts}

    if not retrieved_ids:
        # Retrieved nothing — precision undefined; if we also expected nothing, 1.0; else 0.0
        score = 1.0 if not expected_ids else 0.0
    else:
        relevant_retrieved = retrieved_ids & expected_ids
        score = len(relevant_retrieved) / len(retrieved_ids)

    return MetricResult(
        name="context_precision",
        score=score,
        details={
            "retrieved": list(retrieved_ids),
            "expected": list(expected_ids),
            "relevant_retrieved": list(retrieved_ids & expected_ids),
        },
    )
```

**Interpretation:** Low precision = the retriever is returning irrelevant chunks (noise), which can mislead the LLM. High precision = most retrieved chunks are useful.

### 9.4 Context Recall (deterministic)

**Question:** Did we retrieve all the context needed to answer? (Recall = did we miss anything?)

**Range:** 0.0 – 1.0

**Formula:**
$$\text{context\_recall} = \frac{|\text{retrieved\_chunks} \cap \text{expected\_relevant\_contexts}|}{|\text{expected\_relevant\_contexts}|}$$

**Pseudocode:**
```python
def compute(self, case, sut_output) -> MetricResult:
    retrieved_ids = {c.chunk_id for c in sut_output.retrieved_contexts}
    expected_ids  = {c.chunk_id for c in case.expected_contexts}

    if not expected_ids:
        score = 1.0   # nothing expected → perfect recall
    else:
        retrieved_expected = retrieved_ids & expected_ids
        score = len(retrieved_expected) / len(expected_ids)

    return MetricResult(
        name="context_recall",
        score=score,
        details={
            "missing": list(expected_ids - retrieved_ids),  # chunks we failed to retrieve
        },
    )
```

**Interpretation:** Low recall = the retriever missed chunks needed for the gold answer → the LLM can't be faithful even if it tries. This is the most actionable metric for retrieval tuning.

**Precision vs. Recall — the data engineer's instinct:** This is identical to the precision/recall you already know from classification. High precision + low recall = conservative retriever (misses things). Low precision + high recall = noisy retriever (returns junk). You tune the tradeoff with `top_k` and similarity thresholds, exactly like tuning a classifier's decision boundary.

### 9.5 Citation Accuracy (deterministic)

**Question:** Do the citations in the answer match actual source chunks (and are they the *right* chunks)?

**Range:** 0.0 – 1.0

**Formula:**
$$\text{citation\_accuracy} = \frac{\text{correct citations}}{\text{total citations in answer}}$$

Where a citation is "correct" if: (a) its `chunk_id` exists in `retrieved_contexts` (not fabricated), AND (b) its `chunk_id` is in `expected_citations` (it's the right source).

**Pseudocode:**
```python
def compute(self, case, sut_output) -> MetricResult:
    retrieved_ids = {c.chunk_id for c in sut_output.retrieved_contexts}
    expected_cite_ids = {c.chunk_id for c in case.expected_citations}
    cited_ids = {c.chunk_id for c in sut_output.citations}

    if not cited_ids:
        # No citations at all — if we expected some, that's 0; if none expected, 1.0
        score = 1.0 if not expected_cite_ids else 0.0
    else:
        correct = 0
        per_citation = []
        for cite_id in cited_ids:
            exists = cite_id in retrieved_ids      # not fabricated
            is_expected = cite_id in expected_cite_ids
            ok = exists and is_expected
            per_citation.append({"chunk_id": cite_id, "exists": exists, "expected": is_expected, "correct": ok})
            if ok:
                correct += 1
        score = correct / len(cited_ids)

    return MetricResult(
        name="citation_accuracy",
        score=score,
        details={"per_citation": per_citation},
    )
```

**dbt parallel:** `assert_citations_match_source_chunks` — every citation is a foreign key; a fabricated citation is a referential integrity violation.

### 9.6 Latency (passthrough)

**Question:** How long did the SUT take to generate the answer?

**Range:** milliseconds (lower is better)

**Pseudocode:** Latency is measured by the SUT adapter (wraps the call with `time.perf_counter()`). The metric is a passthrough — it just reads `sut_output.latency_ms`. Threshold is on p95 across all cases in a run, not per-case.

```python
def compute(self, case, sut_output) -> MetricResult:
    return MetricResult(name="latency_ms", score=float(sut_output.latency_ms))
```

### 9.7 Cost (deterministic)

**Question:** What did this answer cost in API charges?

**Range:** USD (lower is better)

**Formula:**
$$\text{cost} = (\text{tokens\_in} \times \text{price\_in}) + (\text{tokens\_out} \times \text{price\_out})$$

**Pseudocode:**
```python
# Price table per 1M tokens (config-driven)
PRICING = {
    "gpt-4o-mini": {"in": 0.150, "out": 0.600},
    "gpt-4o":      {"in": 2.50,  "out": 10.00},
    "gpt-4":       {"in": 30.00, "out": 60.00},
}

def compute(self, sut_output, model: str) -> float:
    p = PRICING[model]
    return (sut_output.tokens_in * p["in"] / 1_000_000) + \
           (sut_output.tokens_out * p["out"] / 1_000_000)
```

**Interpretation:** Cost is tracked per case and aggregated per run. The dashboard shows cost trend per prompt version — a prompt change that doubles token usage is a cost regression, caught in CI just like a quality regression.

### 9.8 Context Engineering Metrics (deterministic + LLM-as-judge)

Context engineering — the discipline of managing what goes into the LLM's context window and how — is a first-class eval concern. A RAG system can have perfect retrieval but fail because the context is too long, poorly ordered, or missing critical information due to compression. These metrics evaluate the *context construction* layer, not just the final answer.

#### Context Utilization (deterministic)

**Question:** What fraction of the retrieved context was actually used in the answer?

**Range:** 0.0 – 1.0

**Formula:**
$$\text{context\_utilization} = \frac{\text{tokens in context chunks cited or referenced in answer}}{\text{total tokens in all retrieved context chunks}}$$

**Pseudocode:**
```python
def compute_context_utilization(sut_output: SUTOutput) -> float:
    cited_chunk_ids = {c.chunk_id for c in sut_output.citations}
    total_context_tokens = sum(estimate_tokens(c.content) for c in sut_output.retrieved_contexts)
    used_context_tokens = sum(
        estimate_tokens(c.content) for c in sut_output.retrieved_contexts
        if c.chunk_id in cited_chunk_ids
    )
    return used_context_tokens / total_context_tokens if total_context_tokens > 0 else 0.0
```

**Interpretation:** Low utilization (< 0.3) means the retriever is fetching too many irrelevant chunks — the context window is polluted. High utilization (> 0.8) means the retriever is precise or the LLM is heavily relying on all fetched context. Track this alongside context_precision to distinguish "retrieved relevant but didn't use" from "retrieved irrelevant."

#### Context Compression Ratio (deterministic)

**Question:** How much was the original context compressed before being sent to the LLM?

**Range:** 0.0 – 1.0 (1.0 = no compression; 0.0 = fully compressed away)

**Formula:**
$$\text{compression\_ratio} = \frac{\text{tokens after compression}}{\text{tokens before compression}}$$

**Pseudocode:**
```python
def compute_compression_ratio(
    original_context_tokens: int,
    compressed_context_tokens: int,
) -> float:
    return compressed_context_tokens / original_context_tokens if original_context_tokens > 0 else 1.0
```

**Interpretation:** If the SUT adapter applies context compression (e.g., LLMLingua, summary-based compression, or chunk truncation), this ratio tracks how aggressive the compression is. Pair with faithfulness: if compression_ratio drops below 0.5 and faithfulness drops, the compression is destroying information the LLM needs. The eval harness should test the same cases at multiple compression levels to find the knee in the curve.

#### Position Bias (LLM-as-judge)

**Question:** Does the LLM favor information at the beginning/end of the context over the middle ("lost in the middle" effect)?

**Range:** 0.0 – 1.0 (1.0 = no position bias; < 0.7 = significant bias)

**Pseudocode:**
```python
async def compute_position_bias(case, sut_output, judge) -> MetricResult:
    # Run the same query with contexts in 3 different orderings:
    #   1. Original order
    #   2. Reversed (relevant chunk moved from middle to end)
    #   3. Shuffled (relevant chunk moved from middle to start)
    # Compare answers: if faithfulness varies significantly across orderings,
    # the system has position bias.
    orderings = ["original", "reversed", "shuffled"]
    scores = []
    for ordering in orderings:
        reordered = reorder_contexts(sut_output.retrieved_contexts, ordering)
        answer = await rerun_sut(case.question, reordered)
        faith = await faithfulness_metric.compute(case, answer)
        scores.append(faith.score)

    # Variance across orderings → position bias
    import statistics
    variance = statistics.pvariance(scores)
    score = 1.0 - min(variance * 4, 1.0)  # scale: 0.25 variance → 0 score
    return MetricResult(
        name="position_bias",
        score=score,
        details={"per_ordering_faithfulness": dict(zip(orderings, scores))},
        reasoning=f"Faithfulness variance across orderings: {variance:.4f}",
    )
```

**Interpretation:** The "lost in the middle" phenomenon (Liu et al., 2024) shows that LLMs disproportionately attend to the beginning and end of the context window. If position_bias is low, the system is vulnerable to context ordering — a fix is to place the most relevant chunks at the beginning (re-ranking already helps) or to use a smaller context window with only top-k chunks. This metric is expensive (3× SUT calls) and should be run on a subset of cases, not every eval run.

#### Context Window Overflow Rate (deterministic)

**Question:** How often does the retrieved context exceed the LLM's context window?

**Range:** 0.0 – 1.0 (fraction of eval cases where overflow occurred)

**Pseudocode:**
```python
def compute_overflow_rate(case_results: list[CaseResult], model_context_limit: int) -> float:
    overflow_count = sum(
        1 for cr in case_results
        if sum(estimate_tokens(c["content"]) for c in cr.retrieved_contexts)
           + estimate_tokens(cr.question)
           > model_context_limit
    )
    return overflow_count / len(case_results) if case_results else 0.0
```

**Interpretation:** If overflow_rate > 0, the system is silently truncating context (or failing). This is a configuration bug, not a quality issue — the retriever's top-k is too high for the model's window. Track this as a P0 metric: any overflow should fail CI immediately, not just lower a score.

#### Context Engineering Eval Dataset Design

The eval dataset should include cases specifically designed to test context engineering:

| Case Type | Description | What It Tests |
|-----------|-------------|---------------|
| **Long-context** | Question requires information from a chunk at position 15+ in the retrieved set | Position bias; context window utilization at scale |
| **Multi-hop** | Answer requires synthesizing information from 3+ separate chunks | Context utilization; faithfulness with distributed evidence |
| **Noise-injected** | Retrieved set includes 5+ irrelevant chunks alongside the relevant one | Context precision under noise; context_utilization metric |
| **Compression-sensitive** | Case where the key information is in a dense paragraph that compression might destroy | Compression ratio vs. faithfulness tradeoff |
| **Window-boundary** | Total context tokens are within 5% of the model's context limit | Overflow detection; truncation behavior |

These cases are tagged `context_engineering` in the YAML dataset and can be run as a subset via `pytest -m context_engineering`.

---

## 10. LLM-as-Judge

### 10.1 Design Principles

1. **Judge ≥ one tier above SUT.** GPT-4 judges GPT-4o-mini. Never let a model grade itself.
2. **Structured output.** Judge returns JSON (`response_format={"type": "json_object"}`), not free text. This makes scores machine-parseable and CI-assertable.
3. **Rubrics are versioned.** Rubric prompts live in `evals/rubrics/*.yaml`, git-tracked. A rubric change is a PR, reviewed like any code change.
4. **Reasoning is captured.** Every judge call returns `{score, reasoning}`. The reasoning is stored in `eval_case_results.judge_reasoning` and shown in the dashboard so humans can audit *why* the judge scored something low.
5. **Calibration against human labels.** Before trusting the judge, run it on a 50-case gold set annotated by humans. Compute Cohen's κ. Target κ ≥ 0.6. Below that, iterate the rubric.

### 10.2 Faithfulness Judge Prompt Template

```python
# src/eval_harness/judge/prompts.py

FAITHFULNESS_SYSTEM = """\
You are a strict evaluator for RAG system answers. Your job is to determine
whether each claim in an answer is fully supported by the provided context.

Rules:
- A claim is "supported" ONLY if the context contains information that
  directly entails the claim. If the claim requires outside knowledge
  not in the context, it is "not supported".
- A claim is "supported" if the context states it explicitly OR if it
  is a reasonable paraphrase of context content.
- Numeric values must match exactly (e.g. "14 days" vs "14-day" is fine,
  but "14 days" vs "30 days" is not supported).
- If the claim is vague and cannot be verified from context, mark "not supported".
- Be strict: when in doubt, mark "not supported".

Respond ONLY with valid JSON matching this schema:
{
  "claims": [
    {"claim": "<the atomic claim>", "supported": true|false, "reasoning": "<one sentence>"}
  ],
  "faithfulness_score": <float 0.0-1.0>,
  "summary": "<one sentence overall assessment>"
}
"""

FAITHFULNESS_USER = """\
Context (retrieved passages):
---
{context}
---

Answer to evaluate:
---
{answer}
---

Decompose the answer into atomic claims. For each claim, determine if it is
supported by the context above. Return the JSON.
"""
```

### 10.3 Answer Relevance Judge Prompt Template

```python
ANSWER_RELEVANCE_SYSTEM = """\
You are an evaluator scoring how well an answer addresses a question.

Score on a 1-5 scale:
5 = Directly and completely answers the question; no irrelevant information.
4 = Answers the question but includes minor irrelevant tangents.
3 = Partially answers; misses an aspect of the question.
2 = Mostly irrelevant; touches the topic but doesn't answer.
1 = Completely irrelevant or wrong topic.

Respond ONLY with valid JSON:
{
  "score": <integer 1-5>,
  "reasoning": "<one to two sentences explaining the score>",
  "missing_aspects": ["<aspects of the question not addressed, if any>"]
}
"""

ANSWER_RELEVANCE_USER = """\
Question:
{question}

Answer:
{answer}

Score the answer's relevance to the question on the 1-5 scale. Return JSON.
"""
```

### 10.4 Judge Client

```python
# src/eval_harness/judge/llm_judge.py
import json
from openai import AsyncOpenAI
from eval_harness.config import settings

class LLMJudge:
    """GPT-4-based judge with structured JSON output."""

    def __init__(self, model: str = "gpt-4o", client: AsyncOpenAI | None = None):
        self.model = model
        self.client = client or AsyncOpenAI(api_key=settings.openai_api_key)

    async def _judge(self, system: str, user: str) -> dict:
        resp = await self.client.chat.completions.create(
            model=self.model,
            response_format={"type": "json_object"},
            temperature=0.0,           # deterministic grading
            messages=[
                {"role": "system", "content": system},
                {"role": "user", "content": user},
            ],
        )
        return json.loads(resp.choices[0].message.content)

    async def verify_claims(self, answer: str, context: str) -> dict:
        user = FAITHFULNESS_USER.format(context=context, answer=answer)
        return await self._judge(FAITHFULNESS_SYSTEM, user)

    async def score_relevance(self, question: str, answer: str) -> dict:
        user = ANSWER_RELEVANCE_USER.format(question=question, answer=answer)
        return await self._judge(ANSWER_RELEVANCE_SYSTEM, user)
```

### 10.5 Judge Calibration

```python
# eval_harness/judge/calibrate.py
"""Run the judge on a human-annotated gold set, compute Cohen's kappa."""
import pandas as pd
from sklearn.metrics import cohen_kappa_score

async def calibrate(judge: LLMJudge, gold_csv: str) -> dict:
    gold = pd.read_csv(gold_csv)  # columns: question, answer, context, human_faithfulness, human_relevance
    judge_scores = []
    human_scores = []
    for _, row in gold.iterrows():
        result = await judge.verify_claims(row["answer"], row["context"])
        judge_scores.append(1 if result["faithfulness_score"] >= 0.85 else 0)
        human_scores.append(1 if row["human_faithfulness"] >= 0.85 else 0)

    kappa = cohen_kappa_score(human_scores, judge_scores)
    return {"cohen_kappa": kappa, "n_cases": len(gold), "passed": kappa >= 0.6}
```

---

## 11. Human Feedback Interface

A Streamlit dashboard for human annotators to review sampled Q&A pairs and provide thumbs up/down + free-text feedback. This closes the loop: the LLM-as-judge is calibrated against this human signal, and disagreements between judge and human become calibration cases.

### 11.1 Wireframe Description

```
┌─────────────────────────────────────────────────────────────────────┐
│  🧑‍⚖️  Human Feedback — Eval Harness          [Annotator: sagar ▾]   │
├─────────────────────────────────────────────────────────────────────┤
│  Run: 2024-03-15 #a3f2  |  Prompt: v1.2  |  Dataset: cs-faq  | 10/10│
│  [◀ Prev]  Case 3 of 10  [Next ▶]                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ❓ Question:                                                        │
│  How do I get a refund for a duplicate charge?                      │
│                                                                     │
│  ✅ Expected Answer:                                                 │
│  To get a refund for a duplicate charge, log into your account...   │
│                                                                     │
│  🤖 Generated Answer:                                               │
│  You can request a refund by contacting support within 30 days.     │
│  (Faithfulness: 0.40  |  Judge: "30 days not in context")           │
│                                                                     │
│  📎 Retrieved Contexts:                                              │
│  [billing-faq#chunk-04] Duplicate charges can be refunded from...   │
│  [billing-faq#chunk-12] Support contact: support@example.com        │
│                                                                     │
│  ───────────────────────────────────────────────────────────────    │
│  Your Rating:   👍 Good   👎 Bad                                     │
│                                                                     │
│  Comment:                                                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Answer says 30 days but policy is 5-7 business days.        │   │
│  │ Also missed the self-service billing path.                  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  [Submit Feedback]              [Skip]                              │
│                                                                     │
│  Judge vs. You: Judge said faithfulness=0.40 (bad). You agreed. ✅  │
└─────────────────────────────────────────────────────────────────────┘
```

### 11.2 Implementation Sketch

```python
# dashboards/human_feedback.py
import streamlit as st
from eval_harness.storage.repository import get_latest_run_cases, insert_feedback

st.set_page_config(page_title="Human Feedback", layout="wide")
st.title("🧑‍⚖️ Human Feedback — Eval Harness")

annotator = st.text_input("Annotator", value="sagar")
run_id = st.session_state.get("run_id")
cases = get_latest_run_cases(run_id) if run_id else []
idx = st.session_state.get("case_idx", 0)

if cases:
    case = cases[idx]
    st.subheader(f"Case {idx+1} of {len(cases)}")
    st.markdown(f"**❓ Question:** {case.question}")
    st.markdown(f"**✅ Expected:** {case.expected_answer}")
    st.markdown(f"**🤖 Generated:** {case.generated_answer}")
    st.caption(f"Faithfulness: {case.faithfulness:.2f} | Judge: {case.judge_reasoning}")

    with st.expander("📎 Retrieved Contexts"):
        for ctx in case.retrieved_contexts:
            st.text(f"[{ctx['chunk_id']}] {ctx['content'][:200]}...")

    col1, col2 = st.columns(2)
    rating = None
    with col1:
        if st.button("👍 Good", key="up"):
            rating = "up"
    with col2:
        if st.button("👎 Bad", key="down"):
            rating = "down"

    comment = st.text_area("Comment", height=100)

    if rating and st.button("Submit Feedback"):
        insert_feedback(case_result_id=case.id, run_id=run_id,
                        annotator=annotator, rating=rating, comment=comment)
        st.success("Feedback saved!")
        st.session_state.case_idx = min(idx + 1, len(cases) - 1)
        st.rerun()

    if st.button("Skip"):
        st.session_state.case_idx = min(idx + 1, len(cases) - 1)
        st.rerun()
```

---

## 12. Evaluation Dashboard

A Streamlit dashboard for engineers to monitor metric trends, compare prompt versions, and drill into failing cases. This is the "Metabase for AI quality" — the same role a data quality dashboard plays for pipelines.

### 12.1 Wireframe Description

```
┌──────────────────────────────────────────────────────────────────────────┐
│  📊 Eval Dashboard — LLM Evaluation Harness                               │
├──────────────────────────────────────────────────────────────────────────┤
│  Dataset: [customer-support-faq ▾]   Prompt: [All ▾]   Last: [30 ▾] runs │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  📈 Metric Trends (last 30 runs)                                         │
│  ┌──────────────────────────────────────────────────────────────┐       │
│  │  Faithfulness ─────────────────────╱────●  0.91              │       │
│  │                       ╱──────╱                               │       │
│  │  Threshold ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  0.85                │       │
│  │                                                              │       │
│  │  Answer Relevance ───────╱────────●  4.2/5                   │       │
│  │  Context Recall ─────╱────────●  0.88                        │       │
│  └──────────────────────────────────────────────────────────────┘       │
│  [Faithfulness] [Relevance] [Ctx Precision] [Ctx Recall] [Citations]    │
│                                                                          │
├──────────────────────────────────────────────────────────────────────────┤
│  🔀 Prompt Version Comparison                                            │
│  ┌────────────────┬──────────────┬──────────────┬──────────────┐        │
│  │ Metric         │ v1.1         │ v1.2         │ Δ            │        │
│  ├────────────────┼──────────────┼──────────────┼──────────────┤        │
│  │ Faithfulness   │ 0.88         │ 0.91         │ +0.03 ✅     │        │
│  │ Ctx Recall     │ 0.82         │ 0.88         │ +0.06 ✅     │        │
│  │ Cost / run     │ $0.014       │ $0.022       │ +$0.008 ⚠️   │        │
│  │ p95 Latency    │ 2100ms       │ 2800ms       │ +700ms ⚠️    │        │
│  └────────────────┴──────────────┴──────────────┴──────────────┘        │
│                                                                          │
├──────────────────────────────────────────────────────────────────────────┤
│  ❌ Failing Cases (latest run)                                           │
│  ┌──────────┬────────────────────────────┬──────────┬─────────────────┐ │
│  │ Case ID  │ Question                    │ Faithfl. │ Failure Reasons │ │
│  ├──────────┼────────────────────────────┼──────────┼─────────────────┤ │
│  │ cs-b-001 │ Refund for duplicate charge│ 0.40     │ faithfulness    │ │
│  │          │                            │          │ < 0.85          │ │
│  │ cs-r-005 │ Return window electronics  │ 0.72     │ citation_acc   │ │
│  │          │                            │          │ < 0.90          │ │
│  └──────────┴────────────────────────────┴──────────┴─────────────────┘ │
│  [Expand] → shows judge reasoning + retrieved contexts                   │
│                                                                          │
├──────────────────────────────────────────────────────────────────────────┤
│  💰 Cost & Latency Trends                                                │
│  ┌──────────────────────────────────────────────────────────────┐       │
│  │  Cost/run $ ────╱──────╱────────●  $0.022                    │       │
│  │  p95 ms   ────╱──────╱─────────●  2800                       │       │
│  └──────────────────────────────────────────────────────────────┘       │
└──────────────────────────────────────────────────────────────────────────┘
```

### 12.2 Implementation Sketch

```python
# dashboards/eval_dashboard.py
import streamlit as st
import pandas as pd
import plotly.graph_objects as go
from eval_harness.storage.repository import get_runs, get_failing_cases

st.set_page_config(page_title="Eval Dashboard", layout="wide")
st.title("📊 Eval Dashboard — LLM Evaluation Harness")

# Filters
col1, col2, col3 = st.columns(3)
dataset = col1.selectbox("Dataset", ["customer-support-faq", "product-docs"])
prompt_filter = col2.selectbox("Prompt Version", ["All", "v1.1", "v1.2"])
last_n = col3.slider("Last N runs", 5, 100, 30)

runs = get_runs(dataset=dataset, prompt=prompt_filter, limit=last_n)
df = pd.DataFrame([r.dict() for r in runs])

# Metric trend chart
st.subheader("📈 Metric Trends")
metrics = ["mean_faithfulness", "mean_answer_relevance", "mean_context_recall"]
fig = go.Figure()
for m in metrics:
    fig.add_trace(go.Scatter(x=df["started_at"], y=df[m], mode="lines+markers", name=m))
# Threshold line for faithfulness
fig.add_hline(y=0.85, line_dash="dash", line_color="red", annotation_text="faithfulness threshold")
st.plotly_chart(fig, use_container_width=True)

# Prompt version comparison
st.subheader("🔀 Prompt Version Comparison")
if len(df) > 0:
    latest = df.iloc[0]
    prev = df.iloc[1] if len(df) > 1 else None
    if prev is not None:
        comp = pd.DataFrame({
            "Metric": ["Faithfulness", "Ctx Recall", "Cost/run", "p95 Latency"],
            "Previous": [prev["mean_faithfulness"], prev["mean_context_recall"],
                         prev["total_cost_usd"], prev["p95_latency_ms"]],
            "Latest":   [latest["mean_faithfulness"], latest["mean_context_recall"],
                         latest["total_cost_usd"], latest["p95_latency_ms"]],
        })
        comp["Δ"] = comp["Latest"] - comp["Previous"]
        st.dataframe(comp, use_container_width=True)

# Failing cases
st.subheader("❌ Failing Cases (latest run)")
failing = get_failing_cases(latest["id"])
if failing:
    st.dataframe(pd.DataFrame([f.dict() for f in failing])[["case_id","question","faithfulness","failure_reasons"]],
                 use_container_width=True)
    with st.expander("🔍 Judge reasoning for first failure"):
        st.text(failing[0].judge_reasoning)
else:
    st.success("No failing cases in the latest run! 🎉")

# Cost & latency
st.subheader("💰 Cost & Latency Trends")
fig2 = go.Figure()
fig2.add_trace(go.Scatter(x=df["started_at"], y=df["total_cost_usd"], name="Cost ($)", yaxis="y"))
fig2.add_trace(go.Scatter(x=df["started_at"], y=df["p95_latency_ms"], name="p95 Latency (ms)", yaxis="y2"))
st.plotly_chart(fig2, use_container_width=True)
```

---

## 13. CI Integration

### 13.1 pytest Regression Suite

The eval suite is a pytest module. Each eval case becomes a parametrized test. If any metric drops below threshold, the test fails — and in CI, a failing test blocks the merge, exactly like a failing dbt test.

```python
# tests/eval/test_regression.py
"""
Regression test suite: runs the eval dataset through the RAG SUT and
asserts all metrics are above threshold. This is the AI equivalent of
`dbt test` — it blocks merge when quality regresses.

Run locally:
    pytest tests/eval/test_regression.py -v

In CI: triggered by .github/workflows/eval-suite.yml on every PR.
"""
import pytest
from eval_harness.dataset import load_dataset
from eval_harness.sut.rag_adapter import RAGSUTAdapter
from eval_harness.judge.llm_judge import LLMJudge
from eval_harness.metrics import (
    Faithfulness, AnswerRelevance, ContextPrecision,
    ContextRecall, CitationAccuracy, LatencyMetric, CostMetric,
)
from eval_harness.runner import run_eval
from eval_harness.assertor import check
from eval_harness.config import settings

pytestmark = pytest.mark.eval

# Load dataset once (session scope)
DATASET_PATH = settings.eval_dataset_path  # e.g. "evals/datasets/customer_support_faq.yaml"

@pytest.fixture(scope="session")
def eval_dataset():
    return load_dataset(DATASET_PATH)

@pytest.fixture(scope="session")
def sut():
    return RAGSUTAdapter(base_url=settings.sut_base_url, model=settings.sut_model)

@pytest.fixture(scope="session")
def judge():
    return LLMJudge(model=settings.judge_model)

@pytest.fixture(scope="session")
def metrics():
    return [
        Faithfulness(judge=judge()),
        AnswerRelevance(judge=judge()),
        ContextPrecision(),
        ContextRecall(),
        CitationAccuracy(),
        LatencyMetric(),
        CostMetric(model=settings.sut_model),
    ]

@pytest.fixture(scope="session")
def run_result(eval_dataset, sut, judge, metrics):
    """Run the full eval suite once, share results across all parametrized tests."""
    import asyncio
    result = asyncio.run(run_eval(
        dataset=eval_dataset,
        sut=sut,
        judge=judge,
        metrics=metrics,
        thresholds=eval_dataset.thresholds,
        environment="ci",
        git_sha=settings.git_sha,
    ))
    return result

def _case_result(run_result, case_id):
    return next(cr for cr in run_result.case_results if cr.case_id == case_id)

# Parametrize one test per eval case — each case is independently assertable
def _case_ids():
    return [c.id for c in load_dataset(DATASET_PATH).cases]

@pytest.mark.parametrize("case_id", _case_ids(), ids=_case_ids())
def test_case_passes_thresholds(run_result, case_id, eval_dataset):
    """Each eval case must pass all metric thresholds."""
    cr = _case_result(run_result, case_id)
    case = next(c for c in eval_dataset.cases if c.id == case_id)
    thresholds = case.thresholds or eval_dataset.thresholds

    metric_scores = {
        "faithfulness": cr.faithfulness,
        "answer_relevance": cr.answer_relevance,
        "context_precision": cr.context_precision,
        "context_recall": cr.context_recall,
        "citation_accuracy": cr.citation_accuracy,
    }
    passed, failures = check(metric_scores, thresholds)
    if not passed:
        msg = f"\nCase {case_id} FAILED:\n"
        msg += f"  Question: {case.question}\n"
        msg += f"  Answer: {cr.generated_answer[:200]}...\n"
        msg += f"  Failures: {failures}\n"
        msg += f"  Judge reasoning: {cr.judge_reasoning}\n"
        pytest.fail(msg)
    assert passed, f"Case {case_id} failed thresholds: {failures}"

def test_run_aggregate_passes(run_result, eval_dataset):
    """The run's aggregate (mean) metrics must also pass thresholds."""
    t = eval_dataset.thresholds
    assert run_result.mean_faithfulness >= t.faithfulness, \
        f"Aggregate faithfulness {run_result.mean_faithfulness:.2f} < {t.faithfulness}"
    assert run_result.mean_context_recall >= t.context_recall, \
        f"Aggregate context_recall {run_result.mean_context_recall:.2f} < {t.context_recall}"
    assert run_result.p95_latency_ms <= t.p95_latency_ms, \
        f"p95 latency {run_result.p95_latency_ms}ms > {t.p95_latency_ms}ms"

@pytest.mark.critical_eval
@pytest.mark.parametrize("case_id", [
    c.id for c in load_dataset(DATASET_PATH).cases if "critical" in c.tags
])
def test_critical_cases(run_result, case_id, eval_dataset):
    """Critical-tagged cases have a stricter bar — always run, never skip."""
    cr = _case_result(run_result, case_id)
    assert cr.passed, f"Critical case {case_id} failed: {cr.failure_reasons}"
```

### 13.2 GitHub Actions Workflow

```yaml
# .github/workflows/eval-suite.yml
name: LLM Eval Suite

on:
  pull_request:
    paths:
      - 'evals/**'           # eval datasets or rubrics changed
      - 'src/eval_harness/**'# harness code changed
      - 'tests/eval/**'      # eval tests changed
      - '.github/workflows/eval-suite.yml'
  push:
    branches: [main]         # run on merge to main for trend tracking

jobs:
  eval:
    runs-on: ubuntu-latest
    timeout-minutes: 30

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: eval
          POSTGRES_PASSWORD: eval
          POSTGRES_DB: eval_harness
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    env:
      DATABASE_URL: postgresql://eval:eval@localhost:5432/eval_harness
      OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
      JUDGE_MODEL: gpt-4o
      SUT_MODEL: gpt-4o-mini
      SUT_BASE_URL: ${{ secrets.SUT_BASE_URL }}
      EVAL_DATASET_PATH: evals/datasets/customer_support_faq.yaml
      GIT_SHA: ${{ github.sha }}
      ENVIRONMENT: ci

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: pip

      - name: Install dependencies
        run: |
          pip install --upgrade pip
          pip install -e ".[dev]"

      - name: Run database migrations
        run: alembic upgrade head

      - name: Run eval regression suite
        run: pytest tests/eval/test_regression.py -v --tb=short --junitxml=eval-report.xml

      - name: Generate Markdown report
        if: always()
        run: python -m eval_harness.report --latest-run --format markdown --output eval-report.md

      - name: Upload eval report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: eval-report
          path: |
            eval-report.xml
            eval-report.md

      - name: Comment PR with eval summary
        if: github.event_name == 'pull_request' && always()
        uses: marocchino/sticky-pull-request-comment@v2
        with:
          path: eval-report.md
```

### 13.3 What Blocks Merge?

| Failure | Blocks merge? | Why |
|---------|:---:|-----|
| Any eval case fails its thresholds | ✅ Yes | Quality regression — same as a failing dbt test |
| Aggregate metric below threshold | ✅ Yes | Systemic regression across the dataset |
| Critical-tagged case fails | ✅ Yes + required review | High-impact cases have a stricter bar |
| Cost/run increased > 20% | ⚠️ Warning (comment, not fail) | Cost regression is informational; team decides |
| p95 latency increased | ⚠️ Warning | Latency regression is informational unless over hard threshold |

The eval report is posted as a PR comment (via `sticky-pull-request-comment`) so reviewers see the metric deltas without leaving GitHub. The full XML + Markdown report is uploaded as an artifact for archival.

---

## 14. Security & Safety Considerations

### 14.1 API Key Management

- OpenAI API keys are **never** committed. They live in GitHub Actions secrets (`OPENAI_API_KEY`) and `.env` (gitignored, `.env.example` documents the vars).
- The judge key and SUT key can be separate secrets if the judge uses a different account/tier.
- Keys are loaded via `pydantic-settings` from env — no hardcoded credentials in code.

### 14.2 PII in Eval Datasets

- Eval datasets may contain realistic questions. **Do not include real customer PII** in test cases. Use synthetic or anonymized data.
- If production-like data is needed, run it through a redaction step before adding to the dataset. The PII Guardrails project (sibling) provides this.
- The `human_feedback` table stores annotator names — ensure these are internal emails only, not customer data.

### 14.3 Judge Bias & Safety

- **Position bias:** When the judge compares two answers, order matters. Mitigation: randomize order, or evaluate answers independently (not pairwise) as we do here.
- **Verbosity bias:** Longer answers tend to score higher. Mitigation: the rubric explicitly penalizes irrelevant tangents (relevance score 4 = "includes minor irrelevant tangents").
- **Self-preference bias:** A model judging its own outputs is biased. Mitigation: judge is always ≥ one tier above SUT (GPT-4 judges GPT-4o-mini).
- **Calibration drift:** The judge's behavior changes as OpenAI updates models. Mitigation: pin judge model version (e.g. `gpt-4o-2024-08-06`), re-run calibration quarterly.

### 14.4 Cost Safety

- LLM-as-judge calls are not free. A 100-case dataset with faithfulness (N claims × 1 judge call) + relevance (1 call) can be 200+ judge calls per run.
- **Cost guardrail:** the runner estimates cost before executing and aborts if it exceeds a configurable ceiling (`settings.max_eval_cost_usd`, default $5.00).
- CI runs use the cheapest sufficient judge model (`gpt-4o-mini` can judge `gpt-4o-mini` outputs for trivial metrics, but faithfulness needs `gpt-4o`).

### 14.5 Eval Dataset Integrity

- Datasets are YAML, git-tracked, PR-reviewed. A malicious or accidental dataset change (e.g. lowering thresholds to make a failing run pass) is visible in the PR diff.
- `extra="forbid"` on Pydantic models prevents silent field additions.
- Dataset `file_sha` is stored in `eval_datasets` for provenance — every run is traceable to the exact dataset version.

---

## 15. API Specification

The harness is primarily a library + CLI + pytest integration, not a long-running service. However, it exposes a minimal CLI and the dashboards read from the DB. The CLI is the primary interface.

### 15.1 CLI

```bash
# Run an eval suite
python -m eval_harness.runner \
  --dataset evals/datasets/customer_support_faq.yaml \
  --sut rag \                    # or 'mock'
  --sut-url http://localhost:8000 \
  --judge-model gpt-4o \
  --sut-model gpt-4o-mini \
  --environment ci \
  --output eval-report.json

# Generate a Markdown report from the latest run
python -m eval_harness.report --latest-run --format markdown --output eval-report.md

# Calibrate the judge against human gold labels
python -m eval_harness.judge.calibrate --gold evals/gold_labels.csv

# Show metric trend
python -m eval_harness.report --trend --last 30 --metric faithfulness
```

### 15.2 SUT Adapter Contract (HTTP)

If the RAG system under test exposes an HTTP endpoint, the `RAGSUTAdapter` calls it:

```
POST /answer
Content-Type: application/json

{
  "question": "How do I get a refund for a duplicate charge?"
}

→ 200 OK
{
  "answer": "To get a refund...",
  "retrieved_contexts": [
    {"chunk_id": "billing-faq#chunk-04", "content": "...", "score": 0.92}
  ],
  "citations": [
    {"chunk_id": "billing-faq#chunk-04", "snippet": "Duplicate charges..."}
  ],
  "latency_ms": 1850,
  "tokens_in": 1200,
  "tokens_out": 85
}
```

The adapter is a thin client over this contract. If the SUT doesn't return `tokens_in`/`tokens_out`, cost is estimated from character count (rough: 1 token ≈ 4 chars).

---

## 16. Testing Strategy

### 16.1 Test Pyramid

| Layer | What | Tool | Count |
|-------|------|------|-------|
| Unit | Metric computation with fixtures | pytest | ~30 |
| Unit | Dataset loader, schema validation | pytest | ~5 |
| Unit | Judge client (mocked OpenAI) | pytest | ~5 |
| Unit | Assertor threshold logic | pytest | ~8 |
| Integration | Runner end-to-end (mock SUT + mock judge) | pytest | ~3 |
| Integration | Storage DB round-trip | pytest + testcontainers | ~5 |
| Eval | Regression suite (real SUT + real judge) | pytest | 1 per case |

### 16.2 Critical Test Pseudocode

```python
# tests/unit/test_metrics.py

class TestFaithfulness:
    """Faithfulness metric: grounded answers score high, hallucinations score low."""

    @pytest.fixture
    def mock_judge(self):
        """Mock judge that returns canned verdicts."""
        class MockJudge:
            async def verify_claims(self, answer, context):
                # Simulate: claim "14 days" supported, "free shipping" not supported
                return {
                    "claims": [
                        {"claim": "The refund window is 14 days", "supported": True, "reasoning": "Context states 14-day window."},
                        {"claim": "Shipping is free", "supported": False, "reasoning": "Not in context."},
                    ],
                    "faithfulness_score": 0.5,
                    "summary": "1 of 2 claims supported.",
                }
        return MockJudge()

    @pytest.mark.asyncio
    async def test_grounded_answer_scores_high(self, mock_judge):
        case = EvalCase(id="t1", question="?", expected_answer="",
                        expected_contexts=[])
        sut_output = SUTOutput(answer="The refund window is 14 days.",
                               retrieved_contexts=[RetrievedContext(chunk_id="c1", content="14-day window")],
                               citations=[], latency_ms=100)
        metric = Faithfulness(judge=mock_judge)
        result = await metric.compute(case, sut_output)
        assert result.score == 0.5  # 1 of 2 claims supported
        assert "1/2" in result.reasoning

    @pytest.mark.asyncio
    async def test_hallucinated_answer_scores_low(self, mock_judge):
        # Answer with a fabricated claim not in context
        ...
        assert result.score < 0.5


class TestContextPrecisionRecall:
    def test_perfect_precision_and_recall(self):
        case = EvalCase(id="t", question="?", expected_answer="",
            expected_contexts=[ExpectedContext(chunk_id="c1", content="x")])
        sut_output = SUTOutput(answer="",
            retrieved_contexts=[RetrievedContext(chunk_id="c1", content="x")],
            citations=[], latency_ms=10)
        assert ContextPrecision().compute(case, sut_output).score == 1.0
        assert ContextRecall().compute(case, sut_output).score == 1.0

    def test_low_recall_missing_chunk(self):
        case = EvalCase(id="t", question="?", expected_answer="",
            expected_contexts=[
                ExpectedContext(chunk_id="c1", content="x"),
                ExpectedContext(chunk_id="c2", content="y"),
            ])
        sut_output = SUTOutput(answer="",
            retrieved_contexts=[RetrievedContext(chunk_id="c1", content="x")],  # missing c2
            citations=[], latency_ms=10)
        assert ContextRecall().compute(case, sut_output).score == 0.5
        assert "c2" in ContextRecall().compute(case, sut_output).details["missing"]

    def test_low_precision_junk_retrieved(self):
        case = EvalCase(id="t", question="?", expected_answer="",
            expected_contexts=[ExpectedContext(chunk_id="c1", content="x")])
        sut_output = SUTOutput(answer="",
            retrieved_contexts=[
                RetrievedContext(chunk_id="c1", content="x"),
                RetrievedContext(chunk_id="junk", content="irrelevant"),
            ], citations=[], latency_ms=10)
        assert ContextPrecision().compute(case, sut_output).score == 0.5


class TestCitationAccuracy:
    def test_fabricated_citation_scores_zero(self):
        case = EvalCase(id="t", question="?", expected_answer="",
            expected_contexts=[ExpectedContext(chunk_id="c1", content="x")],
            expected_citations=[ExpectedContext(chunk_id="c1", content="x")])
        sut_output = SUTOutput(answer="",
            retrieved_contexts=[RetrievedContext(chunk_id="c1", content="x")],
            citations=[Citation(chunk_id="FAKED-ID", snippet="...")],  # not in retrieved
            latency_ms=10)
        assert CitationAccuracy().compute(case, sut_output).score == 0.0

    def test_correct_citation_scores_one(self):
        ...
        assert CitationAccuracy().compute(case, sut_output).score == 1.0


class TestAssertor:
    def test_below_threshold_fails(self):
        scores = {"faithfulness": MetricResult(name="faithfulness", score=0.60),
                  "answer_relevance": MetricResult(name="answer_relevance", score=4.5),
                  ...}
        thresholds = Thresholds(faithfulness=0.85, ...)
        passed, failures = check(scores, thresholds)
        assert not passed
        assert any("faithfulness" in f for f in failures)

    def test_above_threshold_passes(self):
        scores = {"faithfulness": MetricResult(name="faithfulness", score=0.95), ...}
        passed, failures = check(scores, Thresholds())
        assert passed
        assert failures == []


class TestDatasetLoader:
    def test_valid_yaml_loads(self, tmp_path):
        yaml_content = """
metadata: {name: test, version: "1.0.0"}
thresholds: {faithfulness: 0.85, answer_relevance: 4.0, context_precision: 0.75, context_recall: 0.80, citation_accuracy: 0.90, p95_latency_ms: 3000}
cases:
  - id: t1
    question: "What?"
    expected_answer: "Answer"
    expected_contexts: [{chunk_id: c1, content: "ctx"}]
"""
        p = tmp_path / "test.yaml"
        p.write_text(yaml_content)
        ds = load_dataset(str(p))
        assert len(ds.cases) == 1
        assert ds.cases[0].id == "t1"

    def test_malformed_yaml_raises(self, tmp_path):
        yaml_content = """
metadata: {name: test, version: "1.0.0"}
threshlds: {faithfulness: 0.85}   # typo: 'threshlds' not 'thresholds'
cases: []
"""
        p = tmp_path / "bad.yaml"
        p.write_text(yaml_content)
        with pytest.raises(ValidationError):
            load_dataset(str(p))
```

### 16.3 Integration Test

```python
# tests/integration/test_runner_e2e.py
@pytest.mark.asyncio
async def test_full_run_with_mocks(test_db):
    """End-to-end: load dataset → run mock SUT → score with mock judge → persist → assert."""
    dataset = load_dataset("evals/datasets/customer_support_faq.yaml")
    sut = MockSUTAdapter(responses={...})          # no API calls
    judge = MockJudge(...)                          # no API calls
    metrics = [Faithfulness(judge), AnswerRelevance(judge),
               ContextPrecision(), ContextRecall(), CitationAccuracy()]

    result = await run_eval(dataset, sut, judge, metrics, dataset.thresholds,
                            repo=test_db, environment="test")

    assert result.status == "completed"
    assert len(result.case_results) == len(dataset.cases)
    # Verify persisted
    persisted = await test_db.get_run(result.run_id)
    assert persisted is not None
    assert persisted.mean_faithfulness is not None
```

---

## 17. Deployment

### 17.1 Dockerfile

```dockerfile
# Dockerfile
FROM python:3.11-slim AS base

WORKDIR /app

# System deps for asyncpg
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential libpq-dev curl \
    && rm -rf /var/lib/apt/lists/*

COPY pyproject.toml alembic.ini ./
COPY src/ ./src/
COPY evals/ ./evals/
COPY alembic/ ./alembic/
COPY dashboards/ ./dashboards/
COPY tests/ ./tests/

RUN pip install --no-cache-dir -e ".[dev]"

# Default: run the eval suite
CMD ["pytest", "tests/eval/test_regression.py", "-v", "--tb=short"]
```

### 17.2 docker-compose.yml

```yaml
# docker-compose.yml
version: "3.9"

services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: eval
      POSTGRES_PASSWORD: eval
      POSTGRES_DB: eval_harness
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U eval"]
      interval: 10s
      timeout: 5s
      retries: 5

  harness:
    build: .
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      DATABASE_URL: postgresql://eval:eval@postgres:5432/eval_harness
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      JUDGE_MODEL: ${JUDGE_MODEL:-gpt-4o}
      SUT_MODEL: ${SUT_MODEL:-gpt-4o-mini}
      SUT_BASE_URL: ${SUT_BASE_URL:-http://host.docker.internal:8000}
      EVAL_DATASET_PATH: evals/datasets/customer_support_faq.yaml
      ENVIRONMENT: local
    volumes:
      - ./evals:/app/evals
      - ./reports:/app/reports
    command: >
      sh -c "alembic upgrade head &&
             python -m eval_harness.runner
             --dataset evals/datasets/customer_support_faq.yaml
             --sut rag --output reports/eval-report.json"

  eval-dashboard:
    build: .
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      DATABASE_URL: postgresql://eval:eval@postgres:5432/eval_harness
    ports:
      - "8501:8501"
    command: streamlit run dashboards/eval_dashboard.py --server.port 8501 --server.address 0.0.0.0

  human-feedback:
    build: .
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      DATABASE_URL: postgresql://eval:eval@postgres:5432/eval_harness
    ports:
      - "8502:8502"
    command: streamlit run dashboards/human_feedback.py --server.port 8502 --server.address 0.0.0.0

volumes:
  pgdata:
```

### 17.3 .env.example

```bash
# .env.example — copy to .env and fill in. NEVER commit .env.

# ── Database ──
DATABASE_URL=postgresql://eval:eval@localhost:5432/eval_harness

# ── OpenAI ──
OPENAI_API_KEY=sk-...

# ── Judge (stronger model) ──
JUDGE_MODEL=gpt-4o

# ── System Under Test (weaker model being evaluated) ──
SUT_MODEL=gpt-4o-mini
SUT_BASE_URL=http://localhost:8000   # RAG system's /answer endpoint

# ── Eval config ──
EVAL_DATASET_PATH=evals/datasets/customer_support_faq.yaml
ENVIRONMENT=local
GIT_SHA=local-dev

# ── Cost guardrail ──
MAX_EVAL_COST_USD=5.00
```

### 17.4 Quick Start

```bash
# 1. Clone & install
git clone <repo> && cd llm-eval-harness
cp .env.example .env  # fill in OPENAI_API_KEY

# 2. Start Postgres + dashboards
docker compose up -d postgres eval-dashboard human-feedback

# 3. Run migrations
docker compose run --rm harness alembic upgrade head

# 4. Run the eval suite (against your RAG system)
docker compose run --rm harness python -m eval_harness.runner \
  --dataset evals/datasets/customer_support_faq.yaml --sut rag

# 5. View results
open http://localhost:8501  # eval dashboard
open http://localhost:8502  # human feedback

# 6. Run as pytest (CI mode)
docker compose run --rm harness pytest tests/eval/test_regression.py -v
```

---

## 18. Roadmap & Milestones

| Milestone | Date (relative) | Deliverable | Exit Criteria |
|-----------|-----------------|-------------|---------------|
| M1: Foundation | Day 3 | Schema, loader, deterministic metrics | `pytest tests/unit/test_metrics.py` green |
| M2: LLM-as-Judge | Day 6 | Faithfulness + relevance + runner | Full run with mock SUT produces persisted results; judge κ ≥ 0.6 |
| M3: CI Integration | Day 7 | pytest suite + GitHub Actions | PR triggers eval; failing metric blocks merge |
| M4: Dashboards | Day 9 | Eval dashboard + human feedback | Both Streamlit apps render live data from Postgres |
| M5: Ship | Day 10 | Dockerized, documented, smoke-tested | `docker compose up` → eval → dashboards show data |

### Post-v1 Extensions (future, not in this plan)

- **Online eval:** sample 1% of production traffic, run eval metrics, alert on drift
- **Auto-generated eval datasets:** mine production Q&A logs for candidate test cases
- **A/B eval:** evaluate two SUT versions side-by-side, statistical significance test
- **Multi-judge ensemble:** use 3 judge models, majority vote to reduce single-judge bias
- **RAGAS-style answer similarity:** semantic similarity between generated and expected answer (embedding cosine)
- **Trace-level eval:** evaluate intermediate retrieval steps, not just final answer

---

## 19. Risk Register

| # | Risk | Likelihood | Impact | Mitigation |
|---|------|:---:|:---:|-----------|
| R1 | **Judge unreliability** — GPT-4 gives inconsistent scores across runs | Medium | High | `temperature=0.0` for determinism; pin model version; calibrate against human gold (κ ≥ 0.6); ensemble judges in v2 |
| R2 | **Judge cost explosion** — large datasets × many claims = high API bill | Medium | Medium | Cost guardrail (`MAX_EVAL_COST_USD`); batch claims per judge call; use cheaper judge for trivial metrics |
| R3 | **Eval dataset rot** — test cases become stale as the product evolves | High | Medium | Dataset is git-tracked + PR-reviewed; quarterly review of dataset relevance; tag stale cases |
| R4 | **Overfitting to the eval set** — team tunes the RAG to pass specific cases, not generalize | Medium | High | Holdout set not in the main dataset; rotate cases quarterly; include edge cases; human feedback catches real-world gaps |
| R5 | **SUT adapter contract drift** — RAG system changes its response format | Medium | Medium | Adapter validates response with Pydantic; contract test in SUT repo; version-pin adapter |
| R6 | **CI flakiness** — LLM API latency/timeouts cause intermittent CI failures | Medium | Medium | Retry with exponential backoff; timeout per case; separate eval job from build job; `pytest --retry` |
| R7 | **Threshold set too low** — tests pass but quality is poor | Medium | High | Start conservative, tighten over time; human feedback as ground truth; review threshold changes in PRs |
| R8 | **Threshold set too high** — tests always fail, team ignores them | Medium | High | Calibrate thresholds against baseline run; if > 10% failure rate, investigate before tightening |
| R9 | **Position/verbosity bias in judge** | Medium | Medium | Evaluate answers independently (not pairwise); rubric penalizes irrelevant content; randomize claim order |
| R10 | **Postgres as bottleneck** — many concurrent CI runs write to same DB | Low | Low | One DB per environment; CI uses ephemeral Postgres service container; prod dashboard reads from dedicated DB |

---

## 20. Appendix

### 20.1 Key Design Decisions

| Decision | Choice | Alternative | Rationale |
|----------|--------|-------------|-----------|
| Judge vs SUT model | GPT-4o judges GPT-4o-mini | Same model judges itself | Stronger model catches weaker model's errors; avoids self-preference bias |
| Eval dataset format | YAML | JSON, CSV | Human-readable, git-diffable, PR-reviewable; supports multi-line strings for answers |
| Results storage | PostgreSQL | JSON files, SQLite | Time-series queries for trend charts; concurrent CI writes; same tooling instinct as a warehouse |
| Judge output | Structured JSON (`response_format`) | Free text parsing | Machine-parseable, CI-assertable; no regex on LLM output |
| CI integration | pytest parametrized over cases | Standalone script | One test per case = granular failure reporting; `pytest -m critical_eval` for high-priority subset; integrates with existing pytest CI |
| Dashboard | Streamlit | Grafana, Metabase | Python-native, no JS; fastest path for internal tool; reads directly from Postgres via pandas |
| Metric: context precision/recall | Chunk ID set overlap | Embedding similarity | Deterministic, no LLM cost, exact; embedding similarity is fuzzy and adds a model dependency for a simple set operation |
| Metric: faithfulness | Claim decomposition + per-claim verification | Single holistic score | Granular: you know *which* claim failed; the per-claim breakdown is stored and shown in the dashboard |

### 20.2 The dbt → AI Eval Cheat Sheet (for the resume / interview)

> "I built an evaluation harness for RAG outputs that's the AI equivalent of dbt tests. Eval cases are version-controlled YAML — like dbt test definitions. The suite runs in GitHub Actions on every PR and blocks merge when faithfulness drops below 0.85 — like a failing dbt test blocking production. Metrics include faithfulness (LLM-as-judge verifies each claim against context), context precision/recall (retrieval quality, same precision/recall as classification), and citation accuracy (referential integrity for citations). Results land in Postgres and render on a Streamlit dashboard showing metric trends per prompt version — like a data quality dashboard in Metabase."

### 20.3 Glossary

| Term | Definition |
|------|-----------|
| **SUT** | System Under Test — the RAG/LLM pipeline being evaluated |
| **Eval case** | A single test case: question + expected answer + expected contexts + expected citations + thresholds |
| **Eval run** | One execution of the harness over a dataset; produces aggregate + per-case metrics |
| **Faithfulness** | Fraction of answer claims supported by retrieved context (grounding) |
| **LLM-as-judge** | Using a stronger LLM to grade a weaker LLM's outputs against a rubric |
| **Rubric** | A structured scoring guide (e.g. 1–5 scale with per-score definitions) used by the judge |
| **Context precision** | Fraction of retrieved chunks that are relevant (no junk) |
| **Context recall** | Fraction of needed chunks that were retrieved (nothing missed) |
| **Citation accuracy** | Fraction of citations that are both real (in retrieved) and correct (in expected) |
| **Calibration** | Measuring judge-vs-human agreement (Cohen's κ) to validate the judge is trustworthy |
| **Threshold** | The minimum acceptable metric score; below it, the test fails |
