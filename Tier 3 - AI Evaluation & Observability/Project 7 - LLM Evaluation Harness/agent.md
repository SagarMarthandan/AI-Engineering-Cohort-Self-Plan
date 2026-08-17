# LLM Evaluation Harness — Herdr Multi-Agent Orchestration Guide

> **Runbook for orchestrating 3 implementation agents + 1 reviewer across 5 phases (10 days).**
> Each phase maps to a milestone in IMPLEMENTATION_PLAN.md §18. Prompts reference exact sections, pseudocode, and file paths from the plan.

---

## 1. Agent Roster

| Agent Name | Kind | Responsibility | Owns (files / dirs) |
|---|---|---|---|
| `metrics` | codex | All 7 metric implementations + LLM-as-judge engine + rubric definitions + calibration | `src/eval_harness/metrics/**`, `src/eval_harness/judge/**`, `evals/rubrics/**`, `tests/unit/test_metrics.py`, `tests/unit/test_judge.py` |
| `infra` | codex | Project scaffold, Pydantic models, dataset loader, config, SUT adapters, storage/DB layer, runner orchestrator, assertor, report, pytest integration, GitHub Actions, Docker | `pyproject.toml`, `alembic.ini`, `.env.example`, `Dockerfile`, `docker-compose.yml`, `README.md`, `src/eval_harness/{models,dataset,config,runner,assertor,report,__init__}.py`, `src/eval_harness/sut/**`, `src/eval_harness/storage/**`, `evals/datasets/**`, `evals/thresholds/**`, `tests/unit/{test_dataset_loader,test_assertor}.py`, `tests/integration/**`, `tests/eval/test_regression.py`, `.github/workflows/eval-suite.yml` |
| `dashboard` | codex | Streamlit eval dashboard (metric trends, version comparison, failing cases, cost/latency) + human feedback interface (thumbs up/down, comments) | `dashboards/eval_dashboard.py`, `dashboards/human_feedback.py` |
| `reviewer` | codex | Read-only review after each phase: verify deliverables match plan, check test coverage, validate file ownership boundaries, approve or request changes | _(no file ownership — read-only)_ |

---

## 2. Pane Layout

```
┌─────────────────────────────────────────────────────────────┐
│                    Herdr Session                             │
├──────────────────────┬──────────────────────────────────────┤
│                      │                                      │
│   Pane 1: infra      │   Pane 2: metrics                    │
│   (scaffold, DB,     │   (7 metrics, LLM-as-judge,          │
│    runner, CI)       │    rubrics, calibration)             │
│                      │                                      │
├──────────────────────┼──────────────────────────────────────┤
│                      │                                      │
│   Pane 3: dashboard  │   Pane 4: reviewer                   │
│   (Streamlit apps)   │   (phase review checkpoints)         │
│                      │                                      │
└──────────────────────┴──────────────────────────────────────┘
```

---

## 3. Setup Commands

```bash
# ── Initialize Herdr session ──
herdr init

# ── Split into 4 panes (2×2 grid) ──
herdr pane split --direction vertical --cwd "$PWD" --no-focus          # Pane 1 (top-left)
herdr pane split --direction horizontal --cwd "$PWD" --no-focus        # Pane 2 (top-right)
herdr pane select 0
herdr pane split --direction horizontal --cwd "$PWD" --no-focus        # Pane 3 (bottom-left)
herdr pane select 1
herdr pane split --direction horizontal --cwd "$PWD" --no-focus        # Pane 4 (bottom-right)

# ── Start agents ──
herdr agent start infra    --kind codex --pane 0
herdr agent start metrics  --kind codex --pane 1
herdr agent start dashboard --kind codex --pane 2
herdr agent start reviewer --kind codex --pane 3

# ── Verify all agents are running ──
herdr agent list
```

---

## 4. Phase-by-Phase Orchestration

### Phase 1: Foundation — Scaffold, Models, Dataset Loader, Config (Day 1)

> **Milestone M1 (partial).** Sets up the project skeleton. Only `infra` works this phase — no parallelism needed since all files are in its ownership.

#### Sequential Work

**Agent `infra`:**

```
You are building the foundation of the LLM Evaluation Harness. Read IMPLEMENTATION_PLAN.md sections §3 (Tech Stack), §5 (Eval Dataset Format), §6 (Project Structure) for full context.

Create these files exactly as specified in the plan:

1. pyproject.toml — dependencies: pydantic v2, pydantic-settings, pyyaml, asyncpg, openai, streamlit, plotly, pytest, pytest-asyncio, ruff, alembic. Use src/ layout with package name "eval_harness". Include a [dev] extra with pytest, pytest-asyncio, ruff, testcontainers.

2. src/eval_harness/__init__.py — empty package init.

3. src/eval_harness/models.py — ALL Pydantic v2 models from §5.2 and §8.1:
   - ExpectedContext (chunk_id, content, extra="forbid")
   - EvalCase (id, question, expected_answer, expected_contexts, expected_citations=[], tags=[], thresholds=None, extra="forbid")
   - EvalDatasetMetadata (name, description="", version, author="")
   - Thresholds (faithfulness=0.85, answer_relevance=4.0, context_precision=0.75, context_recall=0.80, citation_accuracy=0.90, p95_latency_ms=3000)
   - EvalDataset (metadata, thresholds, cases, extra="forbid")
   - RetrievedContext (chunk_id, content, score=0.0)
   - Citation (chunk_id, snippet="")
   - SUTOutput (answer, retrieved_contexts, citations=[], latency_ms, tokens_in=0, tokens_out=0)
   - MetricResult (name, score, details={}, reasoning=None)
   - CaseResult — all fields from §4.2 eval_case_results table mapped to Python (run_id, case_id, question, expected_answer, generated_answer, retrieved_contexts, expected_contexts, citations, faithfulness, answer_relevance, context_precision, context_recall, citation_accuracy, latency_ms, tokens_in, tokens_out, cost_usd, judge_reasoning, judge_raw, passed, failure_reasons)
   - EvalRunResult (run_id, status, case_results, mean_faithfulness, mean_answer_relevance, ... aggregate fields from §4.2 eval_runs, plus a from_case_results classmethod)

4. src/eval_harness/dataset.py — YAML loader with Pydantic schema validation:
   - load_dataset(path: str) -> EvalDataset: reads YAML, validates with EvalDataset model (extra="forbid" catches typos like "threshlds")
   - Use PyYAML to parse, then EvalDataset.model_validate()

5. src/eval_harness/config.py — pydantic-settings Settings class loading from env:
   - database_url, openai_api_key, judge_model (default "gpt-4o"), sut_model (default "gpt-4o-mini"), sut_base_url, eval_dataset_path, environment (default "local"), git_sha, max_eval_cost_usd (default 5.00)

6. evals/datasets/customer_support_faq.yaml — seed dataset with 10 cases following the format in §5.1. Include the 3 example cases from the plan (cs-billing-001, cs-returns-002, cs-account-003) and add 7 more covering billing, returns, and account access topics. Include metadata, thresholds, and per-case tags. Some cases should have "critical" tag.

7. evals/thresholds/default.yaml — default thresholds matching the Thresholds model defaults.

8. tests/unit/test_dataset_loader.py — tests from §16.2 TestDatasetLoader:
   - test_valid_yaml_loads: writes valid YAML to tmp_path, loads it, asserts 1 case with id "t1"
   - test_malformed_yaml_raises: writes YAML with "threshlds" typo, asserts ValidationError raised

Verify: python -c "from eval_harness.dataset import load_dataset; ds = load_dataset('evals/datasets/customer_support_faq.yaml'); print(len(ds.cases))" should print 10. Malformed YAML should raise ValidationError.

Do NOT create files outside your ownership. Do NOT touch src/eval_harness/metrics/ or src/eval_harness/judge/ — those belong to the metrics agent.
```

#### Review

**Agent `reviewer`:**

```
Review Phase 1 deliverables for the LLM Evaluation Harness. Check:

1. pyproject.toml has all dependencies from §3 (pydantic v2, pydantic-settings, pyyaml, asyncpg, openai, streamlit, plotly, pytest, pytest-asyncio, ruff, alembic).
2. src/eval_harness/models.py contains ALL Pydantic models listed in §5.2 and §8.1 — every model has extra="forbid" where specified.
3. src/eval_harness/dataset.py load_dataset() validates with Pydantic (not just yaml.safe_load).
4. evals/datasets/customer_support_faq.yaml has exactly 10 cases, valid format per §5.1, at least 2 cases tagged "critical".
5. tests/unit/test_dataset_loader.py has both test_valid_yaml_loads and test_malformed_yaml_raises.
6. No files created outside infra's ownership (no metrics/, judge/, dashboards/ files).
7. Run: pytest tests/unit/test_dataset_loader.py -v — must pass.

Report PASS or list specific issues with file paths and line numbers.
```

#### Commit

```bash
git add -A && git commit -m "Phase 1: project scaffold, Pydantic models, YAML dataset loader, config, seed dataset"
```

---

### Phase 2: Database Schema, Migrations, Storage Layer (Day 2)

> **Milestone M1 (continued).** Only `infra` works — DB layer is entirely in its ownership.

#### Sequential Work

**Agent `infra`:**

```
You are building the database and storage layer for the LLM Evaluation Harness. Read IMPLEMENTATION_PLAN.md §4 (Database Schema) for the full schema.

Create these files:

1. docker-compose.yml — Postgres 16 service only (for now):
   - Service "postgres": image postgres:16, env POSTGRES_USER=eval, POSTGRES_PASSWORD=eval, POSTGRES_DB=eval_harness, port 5432, volume pgdata, healthcheck pg_isready.
   - Define pgdata volume.
   - (Other services will be added in Phase 5.)

2. alembic.ini — standard Alembic config pointing to migrations in src/eval_harness/storage/migrations/. Use DATABASE_URL from environment for the sqlalchemy.url.

3. src/eval_harness/storage/__init__.py — empty.

4. src/eval_harness/storage/migrations/env.py — Alembic env that reads DATABASE_URL from environment, uses asyncpg. Target metadata can be empty (we use raw SQL in migrations).

5. src/eval_harness/storage/migrations/versions/001_create_eval_tables.py — Alembic migration creating ALL 5 tables from §4.2:
   - prompt_versions (id UUID PK, version_tag TEXT UNIQUE, system_prompt, user_prompt_tpl, description, created_at, created_by)
   - eval_datasets (id UUID PK, name, version, case_count, file_path, file_sha, created_at, UNIQUE(name, version))
   - eval_runs (id UUID PK, started_at, finished_at, prompt_version_id FK, eval_dataset_id FK, git_sha, environment, status CHECK in running/completed/failed/aborted, mean_* aggregate columns, p50/p95 latency, total_tokens, total_cost_usd, pass_count, fail_count)
   - eval_case_results (id UUID PK, run_id FK CASCADE, case_id, question, expected_answer, generated_answer, retrieved_contexts JSONB, expected_contexts JSONB, citations JSONB, faithfulness, answer_relevance, context_precision, context_recall, citation_accuracy, latency_ms, tokens_in, tokens_out, cost_usd, judge_reasoning, judge_raw JSONB, passed BOOLEAN, failure_reasons TEXT[], created_at)
   - human_feedback (id UUID PK, case_result_id FK CASCADE, run_id FK CASCADE, annotator, rating CHECK in up/down, comment, created_at)
   - All indexes from §4.2: idx_eval_runs_time, idx_eval_runs_prompt, idx_case_results_run, idx_case_results_case, idx_case_results_pass, idx_feedback_case, idx_feedback_run
   - CREATE EXTENSION IF NOT EXISTS "uuid-ossp" in upgrade, DROP in downgrade.
   - Use uuid_generate_v4() for DEFAULT on all UUID PKs.

6. src/eval_harness/storage/db.py — asyncpg connection pool:
   - async def get_pool() -> asyncpg.Pool: creates pool from DATABASE_URL
   - async def close_pool(pool): closes pool
   - Use settings.database_url from eval_harness.config

7. src/eval_harness/storage/repository.py — EvalRepository class (or module-level async functions) with:
   - insert_run(prompt_version_id, eval_dataset_id, git_sha, environment, status) -> UUID
   - update_run_status(run_id, status)
   - update_run_aggregates(run_id, run_result: EvalRunResult) — writes mean_* columns
   - insert_case_result(cr: CaseResult)
   - insert_feedback(case_result_id, run_id, annotator, rating, comment)
   - get_runs(dataset, prompt, limit) -> list[dict] — for dashboard trend queries (§4.3)
   - get_failing_cases(run_id) -> list[dict] — SELECT from eval_case_results WHERE run_id=$1 AND passed=FALSE (§4.3)
   - get_latest_run_cases(run_id) -> list[dict] — for human feedback dashboard
   - get_run(run_id) -> dict | None
   All methods use asyncpg parameterized queries ($1, $2, ...).

8. tests/integration/test_storage.py — DB round-trip tests (§16.1):
   - Use testcontainers or skip if no DB available.
   - test_insert_and_retrieve_run: insert a run, retrieve it, verify fields.
   - test_insert_case_result: insert run + case result, verify.

Verify: docker compose up -d postgres && alembic upgrade head should create all 5 tables. python -c "import asyncio; from eval_harness.storage.repository import insert_run; asyncio.run(insert_run(...))" should insert a row.

Do NOT touch metrics/, judge/, or dashboards/ directories.
```

#### Review

**Agent `reviewer`:**

```
Review Phase 2 deliverables. Check:

1. docker-compose.yml has postgres:16 service with correct env vars and healthcheck.
2. Alembic migration creates ALL 5 tables (prompt_versions, eval_datasets, eval_runs, eval_case_results, human_feedback) with correct columns, types, FKs, CHECK constraints, and indexes per §4.2.
3. uuid-ossp extension created in migration upgrade().
4. src/eval_harness/storage/db.py uses asyncpg pool (not psycopg2).
5. src/eval_harness/storage/repository.py has all methods: insert_run, update_run_status, update_run_aggregates, insert_case_result, insert_feedback, get_runs, get_failing_cases, get_latest_run_cases, get_run. All use parameterized queries.
6. tests/integration/test_storage.py exists with at least 2 round-trip tests.
7. Run: docker compose up -d postgres && alembic upgrade head — must succeed without errors.
8. No files outside infra ownership.

Report PASS or list specific issues.
```

#### Commit

```bash
git add -A && git commit -m "Phase 2: PostgreSQL schema, Alembic migrations, asyncpg storage repository"
```

---

### Phase 3: SUT Adapters & Deterministic Metrics (Day 3)

> **Milestone M1 (complete).** `infra` builds SUT adapters; `metrics` builds 5 deterministic metrics. These are independent — run in PARALLEL.

#### Parallel Work

**Agent `infra`:**

```
You are building the SUT (System Under Test) adapter layer for the LLM Evaluation Harness. Read IMPLEMENTATION_PLAN.md §8.1 (SUT Adapter ABC), §8.2 (Mock Adapter), §15.2 (SUT HTTP contract).

Create these files:

1. src/eval_harness/sut/__init__.py — empty.

2. src/eval_harness/sut/base.py — SUTAdapter ABC from §8.1:
   - abstract async def answer(self, question: str) -> SUTOutput
   - abstract property name(self) -> str

3. src/eval_harness/sut/mock_adapter.py — MockSUTAdapter from §8.2:
   - __init__(self, responses: dict[str, SUTOutput])
   - name property returns "mock-sut"
   - async answer(question): matches question prefix to canned responses, falls back to generic "I don't have information about that." with empty contexts, latency_ms=50.

4. src/eval_harness/sut/rag_adapter.py — RAGSUTAdapter:
   - __init__(self, base_url: str, model: str)
   - name property returns f"rag-{model}"
   - async answer(question): POST to {base_url}/answer with {"question": question}, parse response per §15.2 contract into SUTOutput. Validate response with Pydantic. If tokens_in/tokens_out missing, estimate from char count (1 token ≈ 4 chars). Wrap call with time.perf_counter() to measure latency_ms if not provided. Use httpx.AsyncClient.

5. tests/unit/test_assertor.py — tests from §16.2 TestAssertor:
   - test_below_threshold_fails: faithfulness=0.60 < 0.85 → not passed, "faithfulness" in failures
   - test_above_threshold_passes: faithfulness=0.95 → passed, failures == []
   - Add tests for latency threshold (p95_latency_ms > threshold → fail) and per-case threshold override.

6. src/eval_harness/assertor.py — from §8.5:
   - def check(metric_scores: dict[str, MetricResult], thresholds: Thresholds) -> tuple[bool, list[str]]
   - Checks: faithfulness >=, answer_relevance >=, context_precision >=, context_recall >=, citation_accuracy >=, p95_latency_ms <=
   - Returns (all_passed, failure_reasons_list) where each failure is "metric_name: score < threshold"

Verify: pytest tests/unit/test_assertor.py -v must pass. The mock adapter must work without any API calls.

Do NOT create metric implementations — those belong to the metrics agent. Only create the SUT adapters and assertor.
```

**Agent `metrics`:**

```
You are building the 5 deterministic metrics for the LLM Evaluation Harness. Read IMPLEMENTATION_PLAN.md §8.3 (Metric Base Class), §9.3 (Context Precision), §9.4 (Context Recall), §9.5 (Citation Accuracy), §9.6 (Latency), §9.7 (Cost).

IMPORTANT: The infra agent is simultaneously creating src/eval_harness/sut/ and src/eval_harness/assertor.py. Do NOT create those. You ONLY create files under src/eval_harness/metrics/ and your test files.

Create these files:

1. src/eval_harness/metrics/__init__.py — export all metric classes:
   from .faithfulness import Faithfulness  (will be created in Phase 4 — import lazily or leave as TODO comment for now)
   from .answer_relevance import AnswerRelevance  (Phase 4)
   from .context_precision import ContextPrecision
   from .context_recall import ContextRecall
   from .citation_accuracy import CitationAccuracy
   from .latency import LatencyMetric
   from .cost import CostMetric
   NOTE: Faithfulness and AnswerRelevance will be added in Phase 4. For now, import only the 5 deterministic ones. Add a comment noting Phase 4 will add the LLM-as-judge imports.

2. src/eval_harness/metrics/base.py — from §8.3:
   - Metric ABC with: name: str (class attr), abstract async def compute(self, case: EvalCase, sut_output: SUTOutput) -> MetricResult
   - MetricResult is already defined in models.py by the infra agent — import it: from eval_harness.models import MetricResult

3. src/eval_harness/metrics/context_precision.py — from §9.3:
   - class ContextPrecision(Metric), name="context_precision"
   - async def compute: set overlap of retrieved chunk_ids vs expected chunk_ids. If no retrieved: 1.0 if no expected, else 0.0. Else: |retrieved ∩ expected| / |retrieved|. Details dict with retrieved, expected, relevant_retrieved lists.

4. src/eval_harness/metrics/context_recall.py — from §9.4:
   - class ContextRecall(Metric), name="context_recall"
   - async def compute: if no expected_ids: 1.0. Else: |retrieved ∩ expected| / |expected|. Details dict with "missing" list (expected - retrieved).

5. src/eval_harness/metrics/citation_accuracy.py — from §9.5:
   - class CitationAccuracy(Metric), name="citation_accuracy"
   - async def compute: citation is "correct" if chunk_id exists in retrieved_contexts AND is in expected_citations. If no citations: 1.0 if no expected, else 0.0. Else: correct/total. Details with per_citation breakdown.

6. src/eval_harness/metrics/latency.py — from §9.6:
   - class LatencyMetric(Metric), name="latency_ms"
   - async def compute: return MetricResult(name="latency_ms", score=float(sut_output.latency_ms)). Passthrough only.

7. src/eval_harness/metrics/cost.py — from §9.7:
   - class CostMetric(Metric), name="cost_usd"
   - __init__(self, model: str = "gpt-4o-mini") — stores model for pricing lookup
   - PRICING dict per 1M tokens: gpt-4o-mini {in:0.150, out:0.600}, gpt-4o {in:2.50, out:10.00}, gpt-4 {in:30.00, out:60.00}
   - async def compute: cost = (tokens_in * price_in/1M) + (tokens_out * price_out/1M). Return MetricResult with score=cost.

8. tests/unit/test_metrics.py — tests from §16.2:
   - TestContextPrecisionRecall:
     - test_perfect_precision_and_recall: retrieved={c1}, expected={c1} → both 1.0
     - test_low_recall_missing_chunk: expected={c1,c2}, retrieved={c1} → recall=0.5, "c2" in missing
     - test_low_precision_junk_retrieved: expected={c1}, retrieved={c1,junk} → precision=0.5
   - TestCitationAccuracy:
     - test_fabricated_citation_scores_zero: citation chunk_id="FAKED-ID" not in retrieved → 0.0
     - test_correct_citation_scores_one: citation matches expected and retrieved → 1.0
   - TestLatencyMetric:
     - test_latency_passthrough: latency_ms=1500 → score=1500.0
   - TestCostMetric:
     - test_cost_calculation: tokens_in=1000000, tokens_out=1000000, model="gpt-4o-mini" → cost = 0.150 + 0.600 = 0.750
   - All deterministic metric tests must use NO API calls, NO mock judge — just direct compute() calls with SUTOutput fixtures.

Verify: pytest tests/unit/test_metrics.py -v must pass. All known-good fixtures score 1.0, all known-bad score 0.0 or low.

Do NOT create files outside src/eval_harness/metrics/ and tests/unit/test_metrics.py. Do NOT touch src/eval_harness/models.py (infra owns it — import from it).
```

#### Sequential (after parallel)

**Agent `infra`:**

```
Phase 3 parallel work is complete. The metrics agent has created the 5 deterministic metrics. Now verify the integration:

1. Confirm src/eval_harness/metrics/__init__.py exports ContextPrecision, ContextRecall, CitationAccuracy, LatencyMetric, CostMetric.
2. Run: pytest tests/unit/test_metrics.py tests/unit/test_assertor.py -v — all must pass.
3. Run: pytest tests/unit/test_dataset_loader.py -v — must still pass (regression check).
4. Fix any import issues between metrics/ and models.py if the metrics agent's imports don't match your models.py exports. Coordinate field names — ensure MetricResult fields match what metrics agent expects.

Report which tests pass and which fail (if any).
```

#### Review

**Agent `reviewer`:**

```
Review Phase 3 deliverables. Check:

1. src/eval_harness/sut/base.py has SUTAdapter ABC with abstract async answer() and abstract name property.
2. src/eval_harness/sut/mock_adapter.py returns canned responses, no API calls, has fallback default.
3. src/eval_harness/sut/rag_adapter.py POSTs to /answer endpoint, validates with Pydantic, estimates tokens from char count if missing.
4. src/eval_harness/assertor.py implements check() per §8.5 with all 6 threshold checks and correct >= / <= operators.
5. src/eval_harness/metrics/ has all 5 deterministic metrics (context_precision, context_recall, citation_accuracy, latency, cost) with correct formulas per §9.3-§9.7.
6. CostMetric uses PRICING table per 1M tokens with correct prices for gpt-4o-mini, gpt-4o, gpt-4.
7. tests/unit/test_metrics.py has known-good (score=1.0) and known-bad (score=0.0) fixtures for each metric.
8. tests/unit/test_assertor.py tests both pass and fail cases.
9. Run: pytest tests/unit/ -v — ALL unit tests must pass.
10. No file ownership violations: metrics agent only touched metrics/ and test_metrics.py; infra agent only touched sut/, assertor.py, test_assertor.py.

Report PASS or list specific issues.
```

#### Commit

```bash
git add -A && git commit -m "Phase 3: SUT adapters (mock + RAG), 5 deterministic metrics, assertor threshold checks"
```

---

### Phase 4: LLM-as-Judge & Eval Runner (Days 4–6)

> **Milestone M2.** `metrics` builds faithfulness + answer relevance + judge client + calibration. `infra` builds the runner orchestrator + report generator. These can run in PARALLEL for Days 4–5, then `infra` needs the metrics on Day 6 for the runner integration.

#### Parallel Work (Days 4–5)

**Agent `metrics`:**

```
You are building the LLM-as-judge metrics for the LLM Evaluation Harness. Read IMPLEMENTATION_PLAN.md §9.1 (Faithfulness), §9.2 (Answer Relevance), §10 (LLM-as-Judge), §10.2 (Faithfulness prompt), §10.3 (Relevance prompt), §10.4 (Judge Client), §10.5 (Calibration).

The infra agent has already created src/eval_harness/models.py with MetricResult, EvalCase, SUTOutput, etc. Import from there.

Create these files:

1. src/eval_harness/judge/__init__.py — empty.

2. src/eval_harness/judge/prompts.py — from §10.2 and §10.3:
   - FAITHFULNESS_SYSTEM: the exact system prompt from §10.2 (strict evaluator, rules about supported claims, numeric matching, JSON schema with claims array, faithfulness_score, summary).
   - FAITHFULNESS_USER: template with {context} and {answer} placeholders.
   - ANSWER_RELEVANCE_SYSTEM: the exact system prompt from §10.3 (1-5 rubric scoring with per-score definitions, JSON schema with score, reasoning, missing_aspects).
   - ANSWER_RELEVANCE_USER: template with {question} and {answer} placeholders.

3. src/eval_harness/judge/llm_judge.py — from §10.4:
   - class LLMJudge:
     - __init__(self, model: str = "gpt-4o", client: AsyncOpenAI | None = None): uses settings.openai_api_key if no client
     - async def _judge(self, system: str, user: str) -> dict: calls client.chat.completions.create with response_format={"type": "json_object"}, temperature=0.0, parses JSON from response
     - async def verify_claims(self, answer: str, context: str) -> dict: uses FAITHFULNESS_SYSTEM/USER
     - async def score_relevance(self, question: str, answer: str) -> dict: uses ANSWER_RELEVANCE_SYSTEM/USER

4. src/eval_harness/metrics/faithfulness.py — from §9.1:
   - class Faithfulness(Metric), name="faithfulness"
   - __init__(self, judge: LLMJudge): stores judge
   - async def compute(self, case, sut_output):
     - Step 1: call judge.verify_claims(answer, context) where context = "\n".join(c.content for c in sut_output.retrieved_contexts)
     - Step 2: parse claims from judge response, count supported
     - score = supported / total_claims if claims else 1.0
     - Return MetricResult(name="faithfulness", score=score, details={"claims": per_claim}, reasoning=f"{supported}/{total} claims supported by context")

5. src/eval_harness/metrics/answer_relevance.py — from §9.2:
   - class AnswerRelevance(Metric), name="answer_relevance"
   - __init__(self, judge: LLMJudge): stores judge
   - async def compute(self, case, sut_output):
     - call judge.score_relevance(question=case.question, answer=sut_output.answer)
     - score = raw_score / 5.0 (normalized for aggregation)
     - Return MetricResult(name="answer_relevance", score=normalized, details={"raw_score": raw_score}, reasoning=judge_reasoning)

6. Update src/eval_harness/metrics/__init__.py — add imports for Faithfulness and AnswerRelevance:
   from .faithfulness import Faithfulness
   from .answer_relevance import AnswerRelevance

7. evals/rubrics/faithfulness.yaml — YAML version of the faithfulness rubric (system prompt + rules, versioned). Include version: "1.0" field.

8. evals/rubrics/answer_relevance.yaml — YAML version of the 1-5 relevance rubric from §9.2 table. Include version: "1.0", the 5 score levels with definitions.

9. src/eval_harness/judge/calibrate.py — from §10.5:
   - async def calibrate(judge: LLMJudge, gold_csv: str) -> dict
   - Reads CSV with columns: question, answer, context, human_faithfulness, human_relevance
   - Runs judge.verify_claims on each, binarizes at 0.85 threshold
   - Computes cohen_kappa_score from sklearn
   - Returns {"cohen_kappa": kappa, "n_cases": len(gold), "passed": kappa >= 0.6}
   - Add __main__ block so it can be run as: python -m eval_harness.judge.calibrate --gold evals/gold_labels.csv

10. tests/unit/test_judge.py — mock judge tests (§16.1, §16.2):
    - TestFaithfulness with MockJudge (from §16.2): mock judge returns canned verdicts, test_grounded_answer_scores_high (score=0.5 for 1/2 claims), test_hallucinated_answer_scores_low (score < 0.5).
    - TestAnswerRelevance: mock judge returns {"score": 4, "reasoning": "..."}, verify normalized score = 0.8.
    - TestLLMJudge: mock AsyncOpenAI client, verify _judge parses JSON correctly, verify temperature=0.0 and response_format used.
    - All tests use mocked judge — NO real API calls.

Verify: pytest tests/unit/test_judge.py tests/unit/test_metrics.py -v must pass. Mock judge tests must work without any OpenAI API key.

Do NOT create runner.py, report.py, or integration tests — those belong to the infra agent.
```

**Agent `infra`:**

```
You are building the eval runner orchestrator and report generator for the LLM Evaluation Harness. Read IMPLEMENTATION_PLAN.md §8.4 (Eval Runner), §2 (Architecture — Eval Run Flow), §15.1 (CLI).

The metrics agent is simultaneously building faithfulness.py, answer_relevance.py, and the judge client. You can build runner.py and report.py now — they import from metrics/ and judge/ which will exist by the time you need to test.

Create these files:

1. src/eval_harness/runner.py — from §8.4:
   - async def run_eval(dataset, sut, judge, metrics, thresholds, repo, prompt_version_id=None, git_sha=None, environment="local") -> EvalRunResult
   - Flow per §2 Eval Run Flow:
     a. insert_run(status="running")
     b. for each case: sut.answer(question) → score each metric → assertor.check() → assemble CaseResult → insert_case_result()
     c. aggregate: EvalRunResult.from_case_results(run_id, case_results)
     d. update_run_aggregates(run_id, run_result)
     e. update_run_status(run_id, "completed")
     f. return run_result
   - Cost guardrail: estimate cost before running, abort if > settings.max_eval_cost_usd (§14.4). Estimate = case_count * avg_tokens * price. If exceeds, raise ValueError with message.
   - Add __main__ block with argparse CLI per §15.1:
     --dataset, --sut (mock|rag), --sut-url, --judge-model, --sut-model, --environment, --output
   - When --sut mock: use MockSUTAdapter with canned responses. When --sut rag: use RAGSUTAdapter.

2. src/eval_harness/report.py — report generation:
   - def generate_json_report(run_result: EvalRunResult, output_path: str): writes JSON with run metadata + per-case results
   - def generate_markdown_report(run_result: EvalRunResult, output_path: str): writes Markdown with summary table (mean metrics), failing cases table, per-case breakdown
   - async def generate_report_from_latest_run(output_path: str, format: str): queries DB for latest run, generates report
   - Add __main__ block per §15.1: --latest-run, --format (json|markdown), --output, --trend, --last, --metric

3. tests/integration/test_runner_e2e.py — from §16.3:
   - @pytest.mark.asyncio
   - async def test_full_run_with_mocks(test_db): load dataset → MockSUTAdapter → MockJudge → run_eval → verify result.status == "completed", len(case_results) == len(dataset.cases), persisted in DB.
   - Use mock judge that returns canned faithfulness/relevance scores.
   - Use a test DB fixture (testcontainers or in-memory mock repository).
   - NO real API calls.

4. Update tests/unit/test_assertor.py if needed to ensure it still passes with any model changes.

Verify: pytest tests/integration/test_runner_e2e.py -v must pass with mock SUT + mock judge (no API calls, no real DB needed if using mock repository).

IMPORTANT: Your runner.py imports from metrics (Faithfulness, AnswerRelevance, etc.) and judge (LLMJudge). These files are being created by the metrics agent in parallel. Write the imports assuming the metrics agent's interface:
  from eval_harness.metrics import Faithfulness, AnswerRelevance, ContextPrecision, ContextRecall, CitationAccuracy, LatencyMetric, CostMetric
  from eval_harness.judge.llm_judge import LLMJudge
If tests fail due to missing imports, wait for the metrics agent to finish, then re-run.

Do NOT create metric implementations or judge client — those belong to the metrics agent.
```

#### Sequential (Day 6 — after parallel)

**Agent `infra`:**

```
Phase 4 parallel work is complete. The metrics agent has created faithfulness.py, answer_relevance.py, llm_judge.py, prompts.py, calibrate.py, and rubric YAMLs. Now integrate and verify:

1. Verify src/eval_harness/metrics/__init__.py exports all 7 metrics: Faithfulness, AnswerRelevance, ContextPrecision, ContextRecall, CitationAccuracy, LatencyMetric, CostMetric.
2. Verify src/eval_harness/judge/llm_judge.py LLMJudge class exists with verify_claims and score_relevance methods.
3. Run: pytest tests/unit/test_metrics.py tests/unit/test_judge.py tests/unit/test_assertor.py tests/unit/test_dataset_loader.py -v — all unit tests must pass.
4. Run: pytest tests/integration/test_runner_e2e.py -v — integration test with mock SUT + mock judge must pass.
5. Run the CLI end-to-end: python -m eval_harness.runner --dataset evals/datasets/customer_support_faq.yaml --sut mock --output /tmp/eval-report.json — should produce a JSON report.
6. Fix any integration issues between runner.py and the metrics/judge modules.

Report which tests pass and which fail (if any). Include the CLI output.
```

#### Review

**Agent `reviewer`:**

```
Review Phase 4 deliverables. Check:

1. src/eval_harness/judge/prompts.py has FAITHFULNESS_SYSTEM, FAITHFULNESS_USER, ANSWER_RELEVANCE_SYSTEM, ANSWER_RELEVANCE_USER — all matching §10.2 and §10.3 exactly.
2. src/eval_harness/judge/llm_judge.py LLMJudge uses response_format={"type": "json_object"}, temperature=0.0, parses JSON from response. Has verify_claims and score_relevance methods.
3. src/eval_harness/metrics/faithfulness.py decomposes answer into claims, verifies each against context, computes supported/total. Score 0.0-1.0.
4. src/eval_harness/metrics/answer_relevance.py scores 1-5, normalizes to 0.0-1.0 by dividing by 5.0.
5. src/eval_harness/judge/calibrate.py computes Cohen's kappa, returns dict with cohen_kappa, n_cases, passed. Has __main__ block.
6. evals/rubrics/faithfulness.yaml and answer_relevance.yaml exist with versioned rubric definitions.
7. src/eval_harness/runner.py implements the full flow from §8.4: insert_run → loop cases (sut.answer → score metrics → assertor.check → assemble CaseResult → insert_case_result) → aggregate → update_run_aggregates → update_run_status. Has cost guardrail. Has CLI with argparse.
8. src/eval_harness/report.py generates JSON and Markdown reports. Has --latest-run and --trend CLI options.
9. tests/integration/test_runner_e2e.py runs full flow with mock SUT + mock judge, verifies results persisted.
10. tests/unit/test_judge.py has mock judge tests for faithfulness (grounded + hallucinated) and relevance.
11. Run: pytest tests/unit/ tests/integration/ -v — ALL tests must pass.
12. Run: python -m eval_harness.runner --dataset evals/datasets/customer_support_faq.yaml --sut mock — must produce a report without errors.
13. No file ownership violations.

Report PASS or list specific issues.
```

#### Commit

```bash
git add -A && git commit -m "Phase 4: LLM-as-judge (faithfulness + relevance), judge calibration, eval runner orchestrator, report generator"
```

---

### Phase 5: CI Integration, Dashboards & Ship (Days 7–10)

> **Milestones M3, M4, M5.** `infra` builds pytest regression suite + GitHub Actions + Docker. `dashboard` builds both Streamlit apps. These are independent — run in PARALLEL for Days 7–9, then `infra` finalizes Docker/docs on Day 10.

#### Parallel Work (Days 7–9)

**Agent `infra`:**

```
You are building the CI regression suite and GitHub Actions workflow for the LLM Evaluation Harness. Read IMPLEMENTATION_PLAN.md §13 (CI Integration), §13.1 (pytest Regression Suite), §13.2 (GitHub Actions Workflow), §13.3 (What Blocks Merge).

Create these files:

1. tests/eval/test_regression.py — from §13.1:
   - pytestmark = pytest.mark.eval
   - DATASET_PATH from settings.eval_dataset_path
   - Session-scoped fixtures: eval_dataset (load_dataset), sut (RAGSUTAdapter), judge (LLMJudge), metrics (list of all 7 metrics), run_result (asyncio.run(run_eval(...)) with environment="ci")
   - _case_ids(): returns list of case IDs from dataset
   - _case_result(run_result, case_id): finds matching CaseResult
   - test_case_passes_thresholds: @pytest.mark.parametrize over case_ids, asserts each case passes thresholds (per-case override → dataset default). On failure, pytest.fail with detailed message (question, answer[:200], failures, judge_reasoning).
   - test_run_aggregate_passes: asserts mean_faithfulness >= threshold, mean_context_recall >= threshold, p95_latency_ms <= threshold.
   - test_critical_cases: @pytest.mark.critical_eval, parametrize over cases with "critical" tag, asserts cr.passed.
   - Register markers in pyproject.toml: eval, critical_eval.

2. .github/workflows/eval-suite.yml — from §13.2:
   - Name: "LLM Eval Suite"
   - Triggers: pull_request (paths: evals/**, src/eval_harness/**, tests/eval/**, .github/workflows/eval-suite.yml), push to main
   - Job "eval": ubuntu-latest, timeout 30 min
   - Services: postgres:16 with healthcheck
   - Env: DATABASE_URL, OPENAI_API_KEY from secrets, JUDGE_MODEL=gpt-4o, SUT_MODEL=gpt-4o-mini, SUT_BASE_URL from secrets, EVAL_DATASET_PATH, GIT_SHA, ENVIRONMENT=ci
   - Steps: checkout, setup-python 3.11 with pip cache, install -e ".[dev]", alembic upgrade head, pytest tests/eval/test_regression.py -v --tb=short --junitxml=eval-report.xml, generate markdown report (if: always()), upload artifact (if: always()), comment PR with sticky-pull-request-comment (if: pull_request && always())

3. Update pyproject.toml — add pytest markers section:
   [tool.pytest.ini_options]
   markers = [
     "eval: end-to-end eval regression tests",
     "critical_eval: critical-tagged eval cases (stricter bar)",
   ]
   asyncio_mode = "auto"

Verify: pytest tests/eval/test_regression.py -v should run (may fail if no OpenAI key or SUT available, but should not error on import/collection). The GitHub Actions YAML must be valid.

Do NOT touch dashboards/ — the dashboard agent owns those files.
```

**Agent `dashboard`:**

```
You are building the two Streamlit dashboards for the LLM Evaluation Harness. Read IMPLEMENTATION_PLAN.md §11 (Human Feedback Interface), §11.1 (Wireframe), §11.2 (Implementation Sketch), §12 (Evaluation Dashboard), §12.1 (Wireframe), §12.2 (Implementation Sketch).

The infra agent has already created src/eval_harness/storage/repository.py with these functions you will import:
  from eval_harness.storage.repository import get_runs, get_failing_cases, get_latest_run_cases, insert_feedback

Create these files:

1. dashboards/eval_dashboard.py — from §12.2:
   - st.set_page_config(page_title="Eval Dashboard", layout="wide")
   - Title: "📊 Eval Dashboard — LLM Evaluation Harness"
   - Filters (3 columns): Dataset selectbox, Prompt Version selectbox (All + versions), Last N runs slider (5-100, default 30)
   - 📈 Metric Trends section: Plotly line chart with mean_faithfulness, mean_answer_relevance, mean_context_recall over started_at. Add horizontal threshold line at y=0.85 for faithfulness (red dashed). Metric selector buttons.
   - 🔀 Prompt Version Comparison: DataFrame comparing latest vs previous run (Faithfulness, Ctx Recall, Cost/run, p95 Latency) with Δ column. Color-code deltas (green for improvement, red for regression).
   - ❌ Failing Cases (latest run): DataFrame from get_failing_cases(latest_run_id) showing case_id, question, faithfulness, failure_reasons. Expander for judge reasoning on first failure. Success message if no failures.
   - 💰 Cost & Latency Trends: Plotly dual-axis chart — cost ($USD) on y1, p95 latency (ms) on y2, both over started_at.
   - Use pandas DataFrames from repository query results.
   - Handle empty state gracefully (no runs yet → show info message).

2. dashboards/human_feedback.py — from §11.2:
   - st.set_page_config(page_title="Human Feedback", layout="wide")
   - Title: "🧑‍⚖️ Human Feedback — Eval Harness"
   - Annotator text input (default "sagar")
   - Run selector: dropdown of recent runs (from get_runs), store selected run_id in session_state
   - Case navigation: Prev/Next buttons, "Case X of N" display, case_idx in session_state
   - Display per case: Question, Expected Answer, Generated Answer, faithfulness score + judge reasoning caption
   - Expander for Retrieved Contexts: show each chunk_id + content[:200]
   - Rating: 👍 Good / 👎 Bad buttons (two columns)
   - Comment: text_area (height=100)
   - Submit Feedback button: calls insert_feedback(case_result_id, run_id, annotator, rating, comment), shows success, advances to next case, st.rerun()
   - Skip button: advances to next case without submitting
   - Judge vs. You comparison: after submitting, show whether human rating agrees with judge
   - Handle empty state (no runs/cases → show info message)

Both dashboards must:
- Import from eval_harness.storage.repository (already created by infra agent)
- Use st.session_state for navigation state
- Handle database connection errors gracefully (try/except with st.error)
- Work with the repository function signatures: get_runs(dataset, prompt, limit), get_failing_cases(run_id), get_latest_run_cases(run_id), insert_feedback(case_result_id, run_id, annotator, rating, comment)

Verify: streamlit run dashboards/eval_dashboard.py should launch without import errors. streamlit run dashboards/human_feedback.py should launch without import errors. (They may show empty state if no DB data, but must not crash on import.)

Do NOT create any files outside dashboards/. Do NOT modify storage/repository.py — if function signatures don't match what you need, note it for the reviewer but do not edit infra's files.
```

#### Sequential (Day 10 — after parallel)

**Agent `infra`:**

```
Phase 5 parallel work is complete. The dashboard agent has created both Streamlit apps. Now finalize Dockerization, docs, and end-to-end smoke test. Read IMPLEMENTATION_PLAN.md §17 (Deployment), §17.1 (Dockerfile), §17.2 (docker-compose.yml), §17.3 (.env.example), §17.4 (Quick Start).

Create/update these files:

1. Dockerfile — from §17.1:
   - FROM python:3.11-slim AS base
   - WORKDIR /app
   - Install build-essential, libpq-dev, curl
   - COPY pyproject.toml alembic.ini, src/, evals/, alembic/, dashboards/, tests/
   - RUN pip install -e ".[dev]"
   - CMD: pytest tests/eval/test_regression.py -v --tb=short

2. Update docker-compose.yml — from §17.2, add services beyond postgres:
   - postgres (already exists from Phase 2 — keep it)
   - harness: build ., depends_on postgres healthy, env DATABASE_URL/OPENAI_API_KEY/JUDGE_MODEL/SUT_MODEL/SUT_BASE_URL/EVAL_DATASET_PATH/ENVIRONMENT, volumes for evals and reports, command: alembic upgrade head && python -m eval_harness.runner --dataset ... --sut rag --output reports/eval-report.json
   - eval-dashboard: build ., depends_on postgres healthy, env DATABASE_URL, port 8501, command: streamlit run dashboards/eval_dashboard.py --server.port 8501 --server.address 0.0.0.0
   - human-feedback: build ., depends_on postgres healthy, env DATABASE_URL, port 8502, command: streamlit run dashboards/human_feedback.py --server.port 8502 --server.address 0.0.0.0
   - pgdata volume

3. .env.example — from §17.3:
   - DATABASE_URL, OPENAI_API_KEY, JUDGE_MODEL=gpt-4o, SUT_MODEL=gpt-4o-mini, SUT_BASE_URL, EVAL_DATASET_PATH, ENVIRONMENT=local, GIT_SHA=local-dev, MAX_EVAL_COST_USD=5.00
   - Comment: "NEVER commit .env"

4. README.md — quick start from §17.4:
   - Project description (1 paragraph from the elevator pitch in §1)
   - Quick Start: clone, cp .env.example .env, docker compose up -d postgres eval-dashboard human-feedback, alembic upgrade head, run eval, open dashboards
   - Architecture overview (brief)
   - Metric definitions table (7 metrics with range)
   - CLI commands reference
   - CI integration description

5. End-to-end smoke test:
   - Run: docker compose up -d
   - Run: docker compose run harness python -m eval_harness.runner --dataset evals/datasets/customer_support_faq.yaml --sut mock
   - Verify: results in Postgres, dashboards show data at localhost:8501 and localhost:8502
   - Run: pytest tests/eval/test_regression.py -v (with mock SUT if possible)

Report the smoke test results. Include any errors encountered.
```

#### Review

**Agent `reviewer`:**

```
Review Phase 5 deliverables — FINAL REVIEW. Check:

CI INTEGRATION:
1. tests/eval/test_regression.py has session-scoped fixtures (eval_dataset, sut, judge, metrics, run_result), parametrized test_case_passes_thresholds, test_run_aggregate_passes, test_critical_cases with @pytest.mark.critical_eval.
2. .github/workflows/eval-suite.yml triggers on PR (paths: evals/**, src/eval_harness/**, tests/eval/**) and push to main. Has postgres service, env vars, all steps (checkout, setup-python, install, migrate, pytest, report, upload artifact, PR comment).
3. pyproject.toml has pytest markers registered (eval, critical_eval) and asyncio_mode="auto".

DASHBOARDS:
4. dashboards/eval_dashboard.py has: metric trend chart (plotly), prompt version comparison table, failing cases table, cost & latency trends. Filters for dataset/prompt/last-n.
5. dashboards/human_feedback.py has: annotator input, run selector, case navigation (prev/next), question/expected/generated display, thumbs up/down, comment, submit feedback (calls insert_feedback), skip button.
6. Both dashboards import from eval_harness.storage.repository, handle empty state, use st.session_state.

DOCKER & DOCS:
7. Dockerfile builds from python:3.11-slim, installs deps, copies all needed dirs.
8. docker-compose.yml has all 4 services (postgres, harness, eval-dashboard, human-feedback) with correct ports (8501, 8502), env vars, depends_on, healthcheck.
9. .env.example documents all env vars with defaults.
10. README.md has quick start, architecture, metric table, CLI reference.

FULL SUITE:
11. Run: pytest tests/unit/ tests/integration/ -v — ALL must pass (eval tests may skip without API key).
12. Run: streamlit run dashboards/eval_dashboard.py --headless — must not crash on import.
13. Run: streamlit run dashboards/human_feedback.py --headless — must not crash on import.
14. No file ownership violations across all 5 phases.
15. All 7 metrics implemented and tested. All 5 DB tables created. Runner produces reports. CI workflow is valid.

Report PASS or list specific issues. This is the final gate before the project is considered complete.
```

#### Commit

```bash
git add -A && git commit -m "Phase 5: pytest regression suite, GitHub Actions CI, Streamlit dashboards, Docker, docs"
```

---

## 5. Coordination Rules

### File Ownership Boundaries

| Directory / File | Owner | Notes |
|---|---|---|
| `src/eval_harness/metrics/**` | `metrics` | All 7 metric implementations + base.py |
| `src/eval_harness/judge/**` | `metrics` | LLM judge client, prompts, calibration |
| `evals/rubrics/**` | `metrics` | Rubric YAML definitions |
| `tests/unit/test_metrics.py` | `metrics` | Metric unit tests |
| `tests/unit/test_judge.py` | `metrics` | Judge unit tests |
| `src/eval_harness/models.py` | `infra` | All Pydantic models — metrics agent imports from here |
| `src/eval_harness/dataset.py` | `infra` | YAML loader |
| `src/eval_harness/config.py` | `infra` | Settings |
| `src/eval_harness/runner.py` | `infra` | Eval orchestrator |
| `src/eval_harness/assertor.py` | `infra` | Threshold checks |
| `src/eval_harness/report.py` | `infra` | Report generation |
| `src/eval_harness/sut/**` | `infra` | SUT adapters |
| `src/eval_harness/storage/**` | `infra` | DB pool, repository, migrations |
| `evals/datasets/**` | `infra` | Eval dataset YAMLs |
| `evals/thresholds/**` | `infra` | Default thresholds |
| `tests/unit/test_dataset_loader.py` | `infra` | Dataset loader tests |
| `tests/unit/test_assertor.py` | `infra` | Assertor tests |
| `tests/integration/**` | `infra` | E2E and storage integration tests |
| `tests/eval/test_regression.py` | `infra` | pytest regression suite |
| `.github/workflows/**` | `infra` | GitHub Actions |
| `pyproject.toml`, `alembic.ini` | `infra` | Project config |
| `Dockerfile`, `docker-compose.yml` | `infra` | Containerization |
| `.env.example`, `README.md` | `infra` | Docs |
| `dashboards/**` | `dashboard` | Both Streamlit apps |

### Conflict Avoidance Rules

1. **Never edit another agent's files.** If you need a change to a file you don't own (e.g., `metrics` needs a field added to `models.py`), message the owning agent via `hub send` and request the change. Do not edit it directly.
2. **`src/eval_harness/models.py` is the shared contract.** `infra` creates it in Phase 1. `metrics` imports `MetricResult`, `EvalCase`, `SUTOutput` from it. If `metrics` needs additional fields on these models, request via hub message before Phase 3.
3. **`src/eval_harness/storage/repository.py` is the dashboard contract.** `infra` creates it in Phase 2. `dashboard` imports `get_runs`, `get_failing_cases`, `get_latest_run_cases`, `insert_feedback` from it. If `dashboard` needs different signatures, request via hub message before Phase 5.
4. **`src/eval_harness/metrics/__init__.py` exports are the runner contract.** `metrics` creates it in Phase 3 and updates it in Phase 4. `infra`'s runner.py imports all 7 metrics from it. The export names must match: `Faithfulness, AnswerRelevance, ContextPrecision, ContextRecall, CitationAccuracy, LatencyMetric, CostMetric`.
5. **Parallel phases only run when there are zero file overlaps.** Phases 3, 4, and 5 have parallel work — verify no agent writes to the same file.

### Parallel vs Sequential

| Phase | Mode | Reason |
|---|---|---|
| Phase 1 (Day 1) | Sequential (infra only) | Foundation — no other agent has prerequisites yet |
| Phase 2 (Day 2) | Sequential (infra only) | DB layer — entirely infra-owned |
| Phase 3 (Day 3) | **Parallel** (infra + metrics) | SUT adapters (infra) and deterministic metrics (metrics) have zero file overlap |
| Phase 4 (Days 4–5) | **Parallel** (infra + metrics) | Judge metrics (metrics) and runner/report (infra) have zero file overlap; integration on Day 6 is sequential |
| Phase 5 (Days 7–9) | **Parallel** (infra + dashboard) | CI/pytest (infra) and Streamlit apps (dashboard) have zero file overlap; Docker/docs on Day 10 is sequential (infra only) |

### Handling Blocked Agents

1. **If an agent is blocked waiting for another agent's file:** Send a `hub send` message to the owning agent with the specific request (file path, field name, function signature needed). Continue working on other tasks in your ownership while waiting.
2. **If the blocking agent is in a parallel phase:** The blocked agent should stub the expected interface (e.g., write the import assuming the function exists) and continue. Integration testing happens in the sequential sub-phase after parallel work completes.
3. **If an agent fails or produces broken code:** The reviewer catches it at the phase checkpoint. Do not proceed to the next phase until the reviewer approves. Re-assign the failed work to the same agent with specific fix instructions.
4. **If models.py needs changes after Phase 1:** `infra` owns it. Any agent needing a model change must request it via hub. `infra` makes the change, all agents re-pull.
5. **Cost guardrail triggers:** If the runner aborts due to cost exceeding `MAX_EVAL_COST_USD`, reduce the dataset size or use a cheaper judge model. This is a config change, not a code change.

---

## 6. Quick Reference

### Common Herdr Commands

```bash
# ── Session management ──
herdr init                              # Initialize new session
herdr agent list                        # List all running agents
herdr agent stop <name>                 # Stop a specific agent
herdr session close                     # Close entire session

# ── Pane management ──
herdr pane split --direction vertical --cwd "$PWD" --no-focus
herdr pane split --direction horizontal --cwd "$PWD" --no-focus
herdr pane select <id>                  # Focus a pane
herdr pane close <id>                   # Close a pane

# ── Agent management ──
herdr agent start <name> --kind codex --pane <id>   # Start agent in pane
herdr agent stop <name>                             # Stop agent
herdr agent restart <name>                          # Restart agent
herdr agent send <name> "<message>"                 # Send prompt to agent

# ── Monitoring ──
herdr agent logs <name>                 # View agent output logs
herdr agent status <name>               # Check agent status
herdr pane list                         # List all panes

# ── Inter-agent coordination ──
herdr agent send metrics "Need MetricResult.reasoning field — is it in models.py?"
herdr agent send infra "Please add 'judge_raw' field to CaseResult model"
```

### Phase Summary

| Phase | Days | Mode | Agents | Milestone | Commit Message |
|---|---|---|---|---|---|
| 1: Foundation | 1 | Sequential | infra | M1 (partial) | Phase 1: project scaffold, Pydantic models, YAML dataset loader, config, seed dataset |
| 2: Database | 2 | Sequential | infra | M1 (continued) | Phase 2: PostgreSQL schema, Alembic migrations, asyncpg storage repository |
| 3: SUT + Deterministic Metrics | 3 | Parallel | infra + metrics | M1 (complete) | Phase 3: SUT adapters (mock + RAG), 5 deterministic metrics, assertor threshold checks |
| 4: LLM-as-Judge + Runner | 4–6 | Parallel → Sequential | metrics + infra | M2 | Phase 4: LLM-as-judge (faithfulness + relevance), judge calibration, eval runner orchestrator, report generator |
| 5: CI + Dashboards + Ship | 7–10 | Parallel → Sequential | infra + dashboard | M3, M4, M5 | Phase 5: pytest regression suite, GitHub Actions CI, Streamlit dashboards, Docker, docs |

### Verification Commands (per phase)

```bash
# Phase 1
python -c "from eval_harness.dataset import load_dataset; ds = load_dataset('evals/datasets/customer_support_faq.yaml'); print(len(ds.cases))"
pytest tests/unit/test_dataset_loader.py -v

# Phase 2
docker compose up -d postgres && alembic upgrade head

# Phase 3
pytest tests/unit/test_metrics.py tests/unit/test_assertor.py -v

# Phase 4
pytest tests/unit/ tests/integration/ -v
python -m eval_harness.runner --dataset evals/datasets/customer_support_faq.yaml --sut mock

# Phase 5
pytest tests/eval/test_regression.py -v
streamlit run dashboards/eval_dashboard.py
streamlit run dashboards/human_feedback.py
docker compose up -d
docker compose run harness python -m eval_harness.runner --dataset evals/datasets/customer_support_faq.yaml --sut mock
```
