# Herdr Multi-Agent Orchestration Guide — Tool-Using Data Analysis Agent

> **Project:** LLM agent that connects to PostgreSQL (Northwind), introspects schema, generates & validates SQL via function calling, recovers from errors, and responds with natural language + auto-generated charts behind a read-only safety layer.
>
> **Plan reference:** `IMPLEMENTATION_PLAN.md` in this directory.
> **Timeline:** 7 phases, 10 days.

---

## 1. Agent Roster

| Agent | Kind | Responsibility | Owns |
|-------|------|----------------|------|
| **core** | codex | Agent ReAct loop, tool definitions, tool router, LLM client, schema introspection, SQL executor, system/SQL/interpretation prompts | `src/agent/orchestrator.py`, `src/agent/tool_definitions.py`, `src/agent/tool_router.py`, `src/agent/prompts.py`, `src/tools/schema_introspection.py`, `src/tools/sql_executor.py`, `src/llm/client.py` |
| **safety** | codex | SQL validation pipeline (sqlglot syntax → AST safety → LIMIT enforcement → EXPLAIN cost), read-only DB connection, error categorization, Docker infra, config, DB init scripts | `src/validation/` (all files), `src/db/` (all files), `src/config.py`, `src/main.py`, `docker-compose.yml`, `docker/Dockerfile.api`, `docker/postgres/init.sql`, `scripts/`, `data/northwind/` |
| **ui** | codex | Streamlit chat UI, FastAPI routes, Pydantic models, conversation memory store, chart generator | `src/ui/streamlit_app.py`, `src/api/` (all files), `src/agent/memory.py`, `src/tools/chart_generator.py`, `docker/Dockerfile.ui` |
| **reviewer** | codex | Phase-gate code review, safety audit, test coverage verification, cross-agent integration checks | `tests/` (writes test files; reviews all `src/`) |

---

## 2. Pane Layout

```
┌─────────────────────────────────────────────────────────────┐
│              Herdr Pane Layout — Data Analysis Agent        │
├──────────────────┬──────────────────┬───────────────────────┤
│                  │                  │                       │
│      core        │     safety       │      reviewer         │
│   (pane 1)       │    (pane 2)      │     (pane 3)          │
│  agent loop      │  validation      │  code review          │
│  tools, SQL gen  │  DB, safety      │  phase gates          │
│  LLM client      │  Docker, config  │  tests                │
│                  │                  │                       │
├──────────────────┼──────────────────┤                       │
│                  │                  │                       │
│      ui          │   terminal       │                       │
│   (pane 4)       │   (pane 5)       │                       │
│  Streamlit       │  git, docker     │                       │
│  charts, API     │  pytest, curl    │                       │
│  memory          │                  │                       │
│                  │                  │                       │
└──────────────────┴──────────────────┴───────────────────────┘
```

---

## 3. Setup Commands

```bash
# --- One-time: initialize Herdr workspace ---
cd "/home/sagar/Projects/AI-Engineering Roadmap/Tier 2 - LLM Orchestration & Agents/Project 4 - Data Analysis Agent"

# Create project scaffold directory if it doesn't exist yet
mkdir -p data-analysis-agent && cd data-analysis-agent

# --- Split panes (5-pane grid) ---
herdr pane split --direction vertical --cwd "$PWD" --no-focus          # pane 2 (right of pane 1)
herdr pane split --direction vertical --cwd "$PWD" --no-focus          # pane 3 (right of pane 2)
herdr pane focus 1 && herdr pane split --direction horizontal --cwd "$PWD" --no-focus  # pane 4 (below pane 1)
herdr pane focus 4 && herdr pane split --direction horizontal --cwd "$PWD" --no-focus  # pane 5 (below pane 4)

# --- Start agents ---
herdr agent start core    --kind codex --pane 1
herdr agent start safety  --kind codex --pane 2
herdr agent start reviewer --kind codex --pane 3
herdr agent start ui      --kind codex --pane 4

# Pane 5 stays as a manual terminal for git, docker, pytest, curl

# --- Verify all agents are running ---
herdr agent list
```

---

## 4. Phase-by-Phase Orchestration

### Phase 1: Foundation — DB, Read-Only User, FastAPI Skeleton (Days 1-2)

**Goal:** Running PostgreSQL with Northwind data, read-only user, FastAPI skeleton, health check.

#### Parallel Work

**safety:**
```
Read IMPLEMENTATION_PLAN.md sections 1-5 (project structure) and 4.1-4.4 (database schema, read-only user, metadata tables).

Create the project scaffold and all infrastructure files:

1. Write `docker-compose.yml` with three services:
   - `postgres` (PostgreSQL 16, volume for persistence, loads `docker/postgres/init.sql`)
   - `api` (FastAPI backend, builds from `docker/Dockerfile.api`, depends on postgres)
   - `ui` (Streamlit, builds from `docker/Dockerfile.ui`, depends on api)
   Reference section 3 (Tech Stack) for image/port choices. API on port 8000, UI on port 8501, Postgres on 5432.

2. Write `docker/postgres/init.sql` — exactly as specified in section 4.3:
   - CREATE USER data_agent_reader WITH PASSWORD from env
   - GRANT CONNECT, USAGE, SELECT on all tables
   - ALTER DEFAULT PRIVILEGES for future tables
   - ALTER USER SET statement_timeout = '30s'
   - ALTER USER SET work_mem = '64MB'
   - REVOKE TEMP ON DATABASE

3. Write `src/config.py` — Pydantic Settings with:
   - DATABASE_URL_READONLY (read-only connection string)
   - DATABASE_URL_META (read-write for agent_meta)
   - OPENAI_API_KEY
   - LLM_MODEL (default gpt-4o-mini)
   - MAX_ROWS=1000, MAX_COST=10000.0, MAX_ITERATIONS=10, MAX_SQL_RETRIES=3

4. Write `src/db/connection.py` — two SQLAlchemy async engines:
   - `get_readonly_engine()` → async engine for data_agent_reader
   - `get_meta_engine()` → async engine for agent_meta schema
   - Session factories for both

5. Write `src/db/readonly_session.py` and `src/db/meta_session.py` — session factories with proper cleanup.

6. Write `src/main.py` — FastAPI app factory with lifespan that initializes both DB engines, includes API routers.

7. Write `data/northwind/northwind_schema.sql` and `data/northwind/northwind_data.sql` — Northwind DDL + INSERT data (9 tables: categories, suppliers, customers, employees, products, orders, order_details, shippers, us_states per section 4.2).

8. Write `scripts/init_northwind.py` — Python script to load Northwind schema + data into the Postgres container.

9. Write `scripts/seed_agent_meta.py` — creates agent_meta schema with sessions, messages, query_log tables per section 4.4.

10. Write `docker/Dockerfile.api` — Python 3.11+, installs deps from pyproject.toml, runs uvicorn.

11. Write `pyproject.toml` with all dependencies from section 3: fastapi, uvicorn, sqlalchemy[asyncio], asyncpg, sqlglot, openai, matplotlib, plotly, streamlit, pydantic-settings, structlog, slowapi, pytest, pytest-asyncio, httpx, ruff.

12. Write `.env.example` with all env vars documented.
```

**ui:**
```
Read IMPLEMENTATION_PLAN.md sections 5 (project structure) and 10 (API specification).

Create the API foundation files:

1. Write `src/api/__init__.py` and `src/api/models.py` — Pydantic models:
   - ChatRequest: {message: str, session_id: str | None}
   - ChatResponse: {session_id, answer, sql, results, chart_path, chart_type, reasoning_trace, latency_ms} per section 10.1
   - HealthResponse: {status, database, readonly_user}
   - SchemaResponse: {tables: [...]} per section 10.2
   - SessionListResponse, SessionDetailResponse

2. Write `src/api/routes/__init__.py` and `src/api/routes/health.py`:
   - GET /health → checks both DB connections (readonly + meta)
   - Returns {"status": "healthy", "database": "connected", "readonly_user": "active"}

3. Write `docker/Dockerfile.ui` — Python 3.11+, installs streamlit, runs `streamlit run src/ui/streamlit_app.py --server.port=8501 --server.address=0.0.0.0`.

4. Write `src/ui/__init__.py` (empty placeholder for now; full Streamlit app comes in Phase 6).

Do NOT write src/main.py — safety owns that. Your health route will be imported by safety's main.py.
```

#### Sequential (after parallel)

**safety:**
```
Now that ui has written src/api/routes/health.py and src/api/models.py, update src/main.py to import and register the health router. Verify the import path works: `from src.api.routes.health import router as health_router`.

Write `tests/conftest.py` with fixtures:
   - test_db: async Postgres fixture with Northwind data loaded
   - test_client: httpx AsyncClient fixture for FastAPI
   - readonly_session: async session using data_agent_reader
   - meta_session: async session for agent_meta

Write `tests/__init__.py`.
```

#### Review

**reviewer:**
```
Review Phase 1 deliverables against IMPLEMENTATION_PLAN.md section 6 Phase 1:

1. Verify `docker compose up` starts all three services (postgres, api, ui).
2. Run `curl localhost:8000/health` — must return {"status": "healthy", "database": "connected", "readonly_user": "active"}.
3. Connect as data_agent_reader and attempt `DROP TABLE customers` — must get permission denied.
4. Verify Northwind tables are populated: SELECT COUNT(*) FROM orders should return ~830, order_details ~2155.
5. Verify agent_meta schema exists with sessions, messages, query_log tables.
6. Check pyproject.toml has all dependencies from section 3.
7. Verify .env.example documents all config vars.

Report any failures to the responsible agent via hub message. Do not commit until all checks pass.
```

#### Commit

```bash
git add -A && git commit -m "Phase 1: Foundation — Docker Compose, PostgreSQL with Northwind, read-only user, FastAPI skeleton, health check"
```

---

### Phase 2: Schema Introspection Tool (Day 3)

**Goal:** Agent can read the full database schema — tables, columns, types, foreign keys, sample values.

#### Parallel Work

**core:**
```
Read IMPLEMENTATION_PLAN.md section 7.3 (Schema Introspection Tool) — the full pseudocode for get_schema().

Write `src/tools/__init__.py` and `src/tools/schema_introspection.py`:

Implement `async def get_schema(db: AsyncSession, table_names: list[str] | None = None) -> dict` exactly as specified in section 7.3:
1. Query information_schema.tables for all public BASE TABLEs (or filtered subset)
2. For each table, query information_schema.columns for column_name, data_type, is_nullable, column_default, character_maximum_length
3. Query information_schema for foreign key relationships (table_constraints + key_column_usage + constraint_column_usage)
4. Query pg_class.reltuples for approximate row counts (NOT COUNT(*))
5. For text/varchar/character columns, run SELECT DISTINCT col FROM table LIMIT 5 to get sample values
6. Assemble into structured dict: {tables: [{table_name, approx_rows, columns: [{name, type, nullable, default, sample_values?}], foreign_keys: [{column, references_table, references_column}]}]}

IMPORTANT: Use parameterized queries (text() with bind params) for all user-controlled inputs. The sample value query uses f-strings for column/table names — validate these against the schema first to prevent injection.

Write `tests/test_schema_introspection.py`:
- Test get_schema() returns all 9 Northwind tables
- Test FK from orders.customer_id → customers.customer_id is present
- Test products has FKs to both categories and suppliers
- Test sample_values are returned for text columns (e.g., customers.company_name)
- Test approx_rows for orders is ~830
- Test table_names filter works (pass ["orders"] → only orders returned)
```

**ui:**
```
Read IMPLEMENTATION_PLAN.md section 10.2 (GET /schema API spec).

Write `src/api/routes/schema.py`:
- GET /schema endpoint
- Optional query param ?table=orders to filter to a specific table
- Calls core's get_schema() function (import from src.tools.schema_introspection)
- Returns SchemaResponse per section 10.2 format: {tables: [{table_name, approx_rows, columns, foreign_keys}]}

Write `src/api/routes/sessions.py` (basic stub for now, full impl in Phase 6):
- GET /sessions → list sessions (returns empty list for now)
- GET /sessions/{id} → session detail (returns 404 for now)

Create `src/api/routes/chat.py` as a stub (POST /chat returning 501 Not Implemented — full impl in Phase 6).

Register all new routers. Do NOT modify src/main.py — message safety via hub to add the new router imports.
```

#### Sequential (after parallel)

**safety:**
```
ui has created new API route files (schema.py, sessions.py, chat.py). Update src/main.py to import and register these routers:
- from src.api.routes.schema import router as schema_router
- from src.api.routes.sessions import router as sessions_router  
- from src.api.routes.chat import router as chat_router
Add them to the FastAPI app with appropriate prefixes/tags.
```

#### Review

**reviewer:**
```
Review Phase 2 against IMPLEMENTATION_PLAN.md section 6 Phase 2:

1. Run `curl localhost:8000/schema` — verify JSON returns all 9 Northwind tables.
2. Verify each table has columns with name, type, nullable, default.
3. Verify foreign_keys array for orders includes customer_id → customers.customer_id and employee_id → employees.employee_id.
4. Verify products has FKs to categories (category_id) and suppliers (supplier_id).
5. Verify text columns have sample_values arrays with ≤5 entries.
6. Verify approx_rows for orders is ~830, order_details ~2155.
7. Run `curl localhost:8000/schema?table=orders` — verify only orders is returned.
8. Run pytest tests/test_schema_introspection.py — all tests pass.
9. Check that parameterized queries are used (no SQL injection in table_names filter).

Report failures to responsible agent. Do not commit until all pass.
```

#### Commit

```bash
git add -A && git commit -m "Phase 2: Schema introspection tool — get_schema with tables, columns, FKs, sample values, row counts; GET /schema API"
```

---

### Phase 3: SQL Validation Layer (Days 4-5)

**Goal:** Multi-layer validation pipeline: syntax → safety → LIMIT → cost. Production-grade safety core.

#### Parallel Work

**safety:**
```
Read IMPLEMENTATION_PLAN.md sections 7.4 (SQL Validation Pipeline) and 8 (Safety & Validation Layer) — the full pseudocode for all four validation stages and the safety constraints table.

Write all validation modules:

1. `src/validation/__init__.py` — the composed pipeline:
   - `async def validate_sql(sql: str, db: AsyncSession) -> ValidationResult`
   - Runs stages in sequence (fail fast): syntax → safety → limit → cost
   - Returns ValidationResult dataclass with: valid, sql (possibly modified), original_sql, errors, cost_estimate, cost_value
   - Define ValidationStage enum, ValidationResult dataclass, MAX_ROWS=1000, MAX_COST=10000.0
   - Strip trailing semicolons before parsing (constraint S13)

2. `src/validation/syntax_check.py`:
   - `def check_syntax(sql: str) -> tuple[bool, str | None, object | None]`
   - Uses sqlglot.parse_one(sql, dialect="postgres")
   - Returns (True, None, ast) on success, (False, error_msg, None) on ParseError

3. `src/validation/safety_check.py`:
   - `def check_safety(ast: object) -> tuple[bool, str | None]`
   - ALLOWED_STATEMENT_TYPES = {exp.Select} — only SELECT allowed
   - BLOCKED_STATEMENT_TYPES = {Insert, Update, Delete, Drop, Create, Alter, TruncateTable, Merge, Command}
   - BLOCKED_FUNCTIONS = {pg_read_file, pg_read_binary_file, pg_ls_dir, lo_import, lo_export, pg_sleep, pg_terminate_backend, pg_cancel_backend, dblink, dblink_query, dblink_exec, pg_extension, pg_reload_conf}
   - Walk AST: check for blocked functions (exp.Anonymous), SELECT...INTO (exp.Into), DDL within CTEs
   - Error messages must be LLM-readable (explain what's wrong and what's allowed)

4. `src/validation/limit_enforcer.py`:
   - `def enforce_limit(ast: object, sql: str, max_rows: int = MAX_ROWS) -> tuple[str, object]`
   - If no LIMIT: inject LIMIT max_rows via ast.limit(max_rows)
   - If LIMIT exists and > max_rows: cap it
   - Return (modified_sql, modified_ast)

5. `src/validation/cost_estimator.py`:
   - `async def estimate_cost(db: AsyncSession, sql: str, max_cost: float = MAX_COST) -> tuple[bool, str | None, float | None]`
   - Run EXPLAIN (FORMAT JSON) on the query
   - Parse total_cost from plan[0]["Plan"]["Total Cost"]
   - Reject if total_cost > max_cost with LLM-readable message

Write ALL test files:
- `tests/test_validation_syntax.py`: valid SELECT passes, SELCT typo fails, unbalanced parens fail, empty string fails
- `tests/test_validation_safety.py` (CRITICAL): DROP TABLE rejected, DELETE FROM rejected, INSERT rejected, UPDATE rejected, CREATE TABLE rejected, ALTER TABLE rejected, TRUNCATE rejected, pg_read_file blocked, pg_sleep blocked, dblink blocked, SELECT...INTO blocked, normal SELECT passes, SELECT with CTE passes, SELECT with subquery passes
- `tests/test_validation_limit.py`: no LIMIT → LIMIT 1000 injected, LIMIT 50 → stays 50, LIMIT 5000 → capped to 1000, LIMIT in subquery not affected
- `tests/test_validation_cost.py`: simple SELECT passes, cross join rejected (cost > 10000), EXPLAIN failure handled gracefully

These tests are the safety contract — they MUST be comprehensive.
```

**core:**
```
Read IMPLEMENTATION_PLAN.md section 7.1 (Tool Definitions) — the OpenAI function calling schemas.

Write `src/agent/__init__.py` and `src/agent/tool_definitions.py`:
- Define TOOL_DEFINITIONS list with three tool schemas exactly as specified in section 7.1:
  1. get_schema: {table_names: array of strings (optional)}
  2. execute_sql: {sql: string (required), explanation: string (required)}
  3. generate_chart: {chart_type: enum[bar,line,pie,scatter,table], title, x_column, y_column, sql (all required)}
- Each tool has type "function" with proper JSON Schema parameters

Write `src/agent/tool_router.py`:
- `async def route_tool(tool_name: str, tool_args: dict, db_readonly, db_meta, session_id) -> dict`
- Dispatches to: schema_introspection.get_schema, sql_executor.execute, chart_generator.generate
- Returns dict result (success or error)
- Handles unknown tool names with error dict

Write `src/llm/__init__.py` and `src/llm/client.py`:
- OpenAI client wrapper class with:
  - `async def chat_completion(messages, tools, tool_choice="auto") -> LLMResponse`
  - Parses response: extracts content + tool_calls
  - LLMResponse dataclass: {content, tool_calls, finish_reason}
  - Retry logic for API failures (max 3 retries with backoff)
  - Uses config.LLM_MODEL and config.OPENAI_API_KEY

Do NOT write orchestrator.py yet — that's Phase 4. Do NOT write sql_executor.py yet — that's Phase 4. Just the tool definitions, router, and LLM client.
```

#### Review

**reviewer:**
```
Review Phase 3 against IMPLEMENTATION_PLAN.md sections 6 Phase 3, 7.4, and 8.

CRITICAL SAFETY REVIEW — this is the production-grade safety core:

1. Run pytest tests/test_validation_syntax.py — all pass
2. Run pytest tests/test_validation_safety.py — ALL pass. This is the most critical test file. Verify:
   - DROP TABLE customers → rejected
   - DELETE FROM orders WHERE 1=1 → rejected
   - INSERT INTO orders VALUES (...) → rejected
   - UPDATE customers SET ... → rejected
   - CREATE TABLE foo (...) → rejected
   - ALTER TABLE orders ... → rejected
   - TRUNCATE orders → rejected
   - SELECT pg_read_file('/etc/passwd') → rejected
   - SELECT pg_sleep(300) → rejected
   - SELECT * FROM dblink(...) → rejected
   - SELECT * INTO new_table FROM orders → rejected
   - SELECT * FROM orders → passes
   - SELECT * FROM orders WHERE order_id IN (SELECT order_id FROM orders) → passes
3. Run pytest tests/test_validation_limit.py — all pass
4. Run pytest tests/test_validation_cost.py — all pass
5. Verify the full pipeline: validate_sql("SELECT * FROM orders") returns valid=True with LIMIT 1000 injected.
6. Verify validate_sql("DROP TABLE customers") returns valid=False with safety error.
7. Verify validate_sql("SELCT * FROM orders") returns valid=False with syntax error.
8. Check tool_definitions.py matches section 7.1 exactly (3 tools, correct schemas).
9. Check llm/client.py handles API errors gracefully with retry.

If ANY safety test fails, block the commit and message safety immediately. This is non-negotiable.
```

#### Commit

```bash
git add -A && git commit -m "Phase 3: SQL validation layer — sqlglot syntax check, AST safety check, LIMIT enforcement, EXPLAIN cost estimation; tool definitions, LLM client"
```

---

### Phase 4: SQL Execution Tool + Error Recovery (Days 6-7)

**Goal:** Agent can execute validated SQL via read-only connection and recover from errors.

#### Parallel Work

**core:**
```
Read IMPLEMENTATION_PLAN.md sections 7.5 (SQL Executor with Error Recovery) and 7.2 (Agent Orchestrator — ReAct Loop).

Write `src/tools/sql_executor.py`:
- `async def execute(db: AsyncSession, sql: str, explanation: str, attempt_num: int = 1) -> ExecutionResult`
- Follow section 7.5 pseudocode exactly:
  1. Call validate_sql(sql, db) from safety's validation pipeline
  2. If invalid: return ExecutionResult(success=False, error_type=stage, error_message=joined errors, sql=original)
  3. If valid: execute the (possibly LIMIT-modified) SQL via db.execute(text(validation.sql))
  4. Format results: columns = list(result.keys()), rows = [dict(zip(columns, row)) for row in result.fetchall()]
  5. Measure execution_ms with time.monotonic()
  6. On success: return ExecutionResult(success=True, sql, columns, rows, row_count, execution_ms, cost_estimate, attempt_num)
  7. On asyncio.TimeoutError: return ExecutionResult with error_type="TIMEOUT", message about 30s limit
  8. On other exceptions: categorize error (column not found, relation not found, function not found, division by zero) with LLM-readable hints
- Define ExecutionResult dataclass with all fields from section 7.5

Write `src/agent/orchestrator.py` — the ReAct loop:
- `async def run_agent(user_message, session_id, db_readonly, db_meta) -> AgentResponse`
- Follow section 7.2 pseudocode EXACTLY:
  1. Load conversation memory (call memory_store.load — ui's module; use a lazy import or accept as param)
  2. Build messages: [system prompt, *history, user message]
  3. Persist user message
  4. ReAct loop for MAX_ITERATIONS (10):
     a. Call llm_client.chat_completion(messages, tools=TOOL_DEFINITIONS, tool_choice="auto")
     b. If no tool_calls: persist final answer, return AgentResponse
     c. If tool_calls: dispatch each via tool_router, append assistant+tool messages, persist tool calls
     d. Track sql_attempts (increment on execute_sql calls)
  5. If max iterations reached: return graceful fallback message
- Define AgentResponse dataclass: {answer, sql, results, chart_path, reasoning_trace}
- MAX_ITERATIONS = 10, MAX_SQL_RETRIES = 3
- The orchestrator depends on: safety's validation (via sql_executor), ui's memory (via memory_store), core's tool_definitions/router/llm_client
- For memory: import from src.agent.memory (ui owns this file, but it should be ready by Phase 4 — if not, create a minimal interface and message ui)

Write `tests/test_sql_executor.py`:
- Test valid SELECT executes and returns rows
- Test invalid SQL (syntax error) returns ExecutionResult with error
- Test DDL (DROP TABLE) is rejected by validation before execution
- Test results are formatted as list of dicts with correct column names
- Test execution_ms is recorded
- Test LIMIT is enforced (SELECT * FROM orders returns max 1000 rows)

Write `tests/test_agent_orchestrator.py` (mocked LLM):
- Mock LLM to first call get_schema, then execute_sql with valid SQL, then return final answer
- Verify reasoning_trace has correct steps
- Verify AgentResponse has answer, sql, results
- Mock LLM to call execute_sql with broken SQL, then fixed SQL → verify error recovery works
- Mock LLM to exhaust MAX_ITERATIONS → verify graceful fallback
- Mock LLM to retry SQL 3 times (all fail) → verify it stops retrying
```

**safety:**
```
Read IMPLEMENTATION_PLAN.md section 9 (Error Recovery Loop) and section 8.3 (What Happens If Each Layer Fails).

Your validation pipeline from Phase 3 is the foundation. Now ensure error messages are optimized for LLM consumption:

1. Review and refine error messages in all validation modules to ensure they are:
   - Clear about what went wrong
   - Actionable (tell the LLM how to fix it)
   - Structured (include stage name, error type)

2. Write `src/db/readonly_session.py` enhancements:
   - Ensure the read-only session factory properly sets statement_timeout
   - Add connection pool settings (pool_size=5, max_overflow=10)
   - Add a context manager for clean session lifecycle

3. Verify the EXPLAIN cost estimator handles edge cases:
   - EXPLAIN on invalid SQL → graceful error
   - EXPLAIN on query with missing table → graceful error
   - Very expensive query (cross join) → rejected with clear message

4. Write integration test `tests/test_safety_integration.py` (basic version, expanded in Phase 7):
   - Full pipeline test: generate malicious SQL → validate → verify rejected
   - Test that NO query can reach db.execute without passing validation
   - Test that the read-only DB user rejects writes even if validation is bypassed

Do NOT modify sql_executor.py or orchestrator.py — core owns those. Your job is to ensure the validation pipeline is robust and error messages are LLM-readable.
```

#### Sequential (after parallel)

**ui:**
```
core's orchestrator.py depends on src/agent/memory.py for conversation memory. Write it now if not already written:

Write `src/agent/memory.py`:
- Follow section 7.7 pseudocode:
  - CONTEXT_WINDOW = 20
  - `async def load(session_id, db_meta, limit=20) -> list[dict]` — loads recent messages as OpenAI chat format
  - `async def append(session_id, db_meta, role, content, tool_name=None, tool_call_id=None, sql_generated=None, row_count=None, execution_ms=None, attempt_num=None, error_message=None)` — inserts to agent_meta.messages
- The memory store uses db_meta (the read-write connection to agent_meta schema)
- Messages are returned in chronological order (reverse the DESC LIMIT query result)

If core's orchestrator already has a different interface for memory, coordinate via hub to align the interface. The memory module must export load() and append() with signatures matching what orchestrator.py expects.
```

#### Review

**reviewer:**
```
Review Phase 4 against IMPLEMENTATION_PLAN.md sections 6 Phase 4, 7.2, 7.5, and 9.

1. Run pytest tests/test_sql_executor.py — all pass:
   - Valid SELECT returns rows as list of dicts
   - Invalid SQL rejected by validation
   - DDL rejected before reaching DB
   - LIMIT enforced
   - Execution time recorded

2. Run pytest tests/test_agent_orchestrator.py — all pass:
   - Happy path: get_schema → execute_sql → final answer
   - Error recovery: broken SQL → error fed back → fixed SQL → success
   - Max iterations: graceful fallback after 10 steps
   - Max retries: stops after 3 failed SQL attempts

3. Run pytest tests/test_safety_integration.py — all pass:
   - No DDL/DML reaches the database
   - Read-only user rejects writes even if validation bypassed

4. Manual test: Send a question through the agent (via test or API):
   - "What are the top 5 products by revenue?" → agent calls get_schema → execute_sql with valid SQL → returns results
   - Inject deliberately broken SQL → agent receives error → regenerates → succeeds within 3 attempts

5. Verify reasoning_trace is populated with each step (tool_call type, tool name, args).

6. Verify memory.load() and memory.append() work with the agent_meta.messages table.

Report failures to responsible agent. Do not commit until all pass.
```

#### Commit

```bash
git add -A && git commit -m "Phase 4: SQL execution tool with error recovery, ReAct agent orchestrator, conversation memory store"
```

---

### Phase 5: Result Interpretation + Chart Generation (Day 8)

**Goal:** Agent interprets results in natural language and generates appropriate charts.

#### Parallel Work

**core:**
```
Read IMPLEMENTATION_PLAN.md section 7.8 (System Prompt) and section 7.2 (orchestrator — final answer step).

1. Write `src/agent/prompts.py`:
   - SYSTEM_PROMPT: exactly as specified in section 7.8 — the full system prompt with rules for schema exploration, read-only SQL, error recovery, business language, chart selection, conversation context, transparency, and answer format.
   - Add an INTERPRETATION_PROMPT template that instructs the LLM to:
     * Summarize query results in 2-4 sentences of business language
     * Reference specific numbers and comparisons
     * Avoid technical jargon ("The query returned 5 rows" → "There are 5 products...")
     * Suggest follow-up questions when appropriate

2. Update `src/agent/orchestrator.py`:
   - Import SYSTEM_PROMPT from src.agent.prompts
   - Replace the placeholder system message with SYSTEM_PROMPT
   - Ensure the final answer step (when LLM produces no tool_calls) captures the LLM's natural language interpretation
   - The orchestrator already handles this — just wire in the actual prompt

3. Update `src/agent/tool_definitions.py` if needed:
   - Ensure generate_chart tool definition is complete (it was written in Phase 3, verify it's correct)

Do NOT modify chart_generator.py — ui owns that. Your job is the prompts and orchestrator wiring.
```

**ui:**
```
Read IMPLEMENTATION_PLAN.md section 7.6 (Chart Generator) — the full pseudocode.

Write `src/tools/chart_generator.py`:
- Follow section 7.6 pseudocode exactly:
  - `async def generate(db, chart_type, title, x_column, y_column, sql) -> ChartResult`
  - Re-execute the SQL (via sql_executor.execute with validation) to get fresh data
  - Extract x_data and y_data from result rows
  - Generate chart based on type:
    * bar: _generate_bar() — ax.bar with #4C72B0, rotation=45, tight_layout, savefig dpi=150
    * line: _generate_line() — ax.plot with marker="o", linewidth=2
    * pie: _generate_pie() — ax.pie with autopct="%1.1f%%", startangle=90
    * scatter: _generate_scatter() — ax.scatter with alpha=0.6
    * table: _generate_table() — render rows as a table figure
  - Also generate interactive plotly HTML version
  - Save to CHART_DIR (/tmp/agent_charts)
  - Return ChartResult(success, chart_path, chart_html_path, chart_type)
- Use matplotlib.use("Agg") for non-interactive backend
- Handle errors: query failure → ChartResult(success=False, error=msg), unknown chart type → error

Write `tests/test_chart_generator.py`:
- Test bar chart generation from query results → PNG file exists
- Test line chart generation → PNG file exists
- Test pie chart generation → PNG file exists
- Test scatter chart generation → PNG file exists
- Test chart with invalid SQL → returns error
- Test chart with unknown type → returns error
- Test HTML version is also generated
```

#### Sequential (after parallel)

**core:**
```
Verify the orchestrator can call generate_chart through the tool_router. The tool_router (written in Phase 3) should already dispatch to chart_generator.generate. If the import path needs updating (e.g., chart_generator was just written by ui), update tool_router.py to import from src.tools.chart_generator.

Test the full flow manually:
- Mock LLM to: call get_schema → call execute_sql → call generate_chart(chart_type="bar") → return final answer
- Verify chart_path is set in AgentResponse
- Verify the chart PNG file exists at the returned path
```

#### Review

**reviewer:**
```
Review Phase 5 against IMPLEMENTATION_PLAN.md section 6 Phase 5 and 7.6, 7.8.

1. Run pytest tests/test_chart_generator.py — all pass:
   - All 4 chart types (bar, line, pie, scatter) generate PNG files
   - HTML versions generated
   - Error cases handled (invalid SQL, unknown type)

2. Verify SYSTEM_PROMPT in src/agent/prompts.py matches section 7.8 exactly:
   - Rules for get_schema first, read-only SQL, error recovery, business language, chart selection, conversation context, transparency
   - Answer format: direct answer + SQL code block + interpretation + chart

3. Manual integration test (via test or API):
   - "Show me monthly sales trends" → agent generates SQL → executes → calls generate_chart(chart_type="line") → PNG saved
   - "What's the revenue breakdown by category?" → agent generates pie chart
   - "How many orders did each employee process?" → agent generates bar chart
   - Verify each answer includes prose summary, not just raw rows

4. Verify the orchestrator passes chart_path through to AgentResponse.

Report failures to responsible agent. Do not commit until all pass.
```

#### Commit

```bash
git add -A && git commit -m "Phase 5: Result interpretation prompts, chart generation (bar/line/pie/scatter), orchestrator wiring"
```

---

### Phase 6: Conversation Memory + Streamlit UI (Day 9)

**Goal:** Multi-turn conversations with context + full chat UI in the browser.

#### Parallel Work

**ui:**
```
Read IMPLEMENTATION_PLAN.md sections 7.7 (Conversation Memory — already written in Phase 4, verify/enhance), 10.1 (POST /chat API), and 6 Phase 6.

1. Verify `src/agent/memory.py` is complete (written in Phase 4):
   - load() returns last N messages as OpenAI chat format
   - append() persists to agent_meta.messages with all metadata fields
   - CONTEXT_WINDOW = 20

2. Write `src/api/routes/chat.py` (replace the Phase 2 stub):
   - POST /chat endpoint
   - Accept ChatRequest: {message: str, session_id: str | None}
   - If no session_id: create new session in agent_meta.sessions, generate UUID
   - Call orchestrator.run_agent(user_message, session_id, db_readonly, db_meta)
   - Return ChatResponse: {session_id, answer, sql, results, chart_path, chart_type, reasoning_trace, latency_ms}
   - Measure latency_ms with time.monotonic()

3. Write `src/api/routes/sessions.py` (replace the Phase 2 stub):
   - GET /sessions → list all sessions (id, title, created_at, updated_at) from agent_meta.sessions
   - GET /sessions/{id} → session detail with messages from agent_meta.messages

4. Write `src/ui/streamlit_app.py` — the full Streamlit chat UI:
   - Chat interface using st.chat_message and st.chat_input
   - Session management: sidebar with session list, new session button
   - On user message: POST to /chat via httpx, display response
   - Render response components:
     * st.markdown for prose answer
     * st.code for SQL display (language="sql")
     * st.dataframe for results table
     * st.image for chart PNG (load from chart_path)
   - Show reasoning trace as expandable section (st.expander)
     * Display each step: tool name, args, result
   - Multi-turn: maintain session_id across messages
   - Streaming-like display: show "Thinking..." and reasoning steps as they happen
     (Since the API returns the full response, simulate by displaying reasoning trace after response)
   - Title: "Northwind Data Analysis Agent"
   - Sidebar: session history, schema viewer (GET /schema)

Write `tests/test_agent_memory.py`:
- Test append() + load() roundtrip: append 3 messages, load returns all 3 in order
- Test load with limit: append 25 messages, load(limit=20) returns last 20
- Test messages include tool calls with tool_name and tool_call_id
- Test session isolation: messages from session A don't appear in session B

Write `tests/test_api_chat.py`:
- Test POST /chat with new session (no session_id) → returns session_id + response
- Test POST /chat with existing session_id → uses same session
- Test response includes answer, sql, results, reasoning_trace
- Test follow-up question: first ask "top 5 customers", then "what about German ones?" → second response references prior context
- Test error handling: invalid message → graceful error response
```

**core:**
```
The orchestrator is already complete from Phase 4. Now ensure it properly integrates with memory for multi-turn conversations:

1. Verify orchestrator.py calls memory_store.load(session_id) at the start of each turn
2. Verify orchestrator.py calls memory_store.append() for user messages, tool calls, and final answers
3. Verify the conversation history is passed to the LLM as context (messages = [system, *history, user])

4. Update tool_router.py if needed to ensure all three tools (get_schema, execute_sql, generate_chart) are properly dispatched. The generate_chart dispatch should call chart_generator.generate with the correct arguments.

5. Write a simple integration test in tests/test_agent_orchestrator.py (add to existing):
   - Multi-turn test: mock LLM for turn 1 (get_schema + execute_sql + answer), then turn 2 (execute_sql with modified query + answer)
   - Verify turn 2 doesn't call get_schema (uses conversation context)
   - Verify memory is loaded and appended correctly

Do NOT modify memory.py, chart_generator.py, or any API routes — ui owns those.
```

#### Review

**reviewer:**
```
Review Phase 6 against IMPLEMENTATION_PLAN.md section 6 Phase 6, 7.7, 10.1.

1. Run pytest tests/test_agent_memory.py — all pass:
   - Roundtrip append/load works
   - Limit respected
   - Tool calls persisted with metadata
   - Session isolation enforced

2. Run pytest tests/test_api_chat.py — all pass:
   - New session creation works
   - Existing session reuse works
   - Response includes all fields (answer, sql, results, reasoning_trace)
   - Follow-up question uses conversation context

3. Run pytest tests/test_agent_orchestrator.py — all pass including new multi-turn test.

4. MANUAL UI TEST (critical):
   - Open Streamlit UI at localhost:8501
   - Ask "What are the top 5 customers by order volume?" → see answer + SQL + table + chart
   - Ask follow-up "What about just German customers?" → agent references prior context, adds WHERE ship_country = 'Germany', returns refined results
   - Verify reasoning trace is visible in expandable section
   - Verify session appears in sidebar
   - Verify schema viewer works in sidebar

5. Verify the full end-to-end flow: docker compose up → Streamlit UI → chat → SQL + results + chart + prose answer.

Report failures to responsible agent. Do not commit until all pass.
```

#### Commit

```bash
git add -A && git commit -m "Phase 6: Conversation memory persistence, POST /chat endpoint, session management, full Streamlit chat UI with charts and reasoning trace"
```

---

### Phase 7: Hardening, Testing, Documentation (Day 10)

**Goal:** Production-ready, fully tested, documented, edge cases handled.

#### Parallel Work

**safety:**
```
Read IMPLEMENTATION_PLAN.md section 8 (Safety & Validation Layer — full constraints table S1-S15) and section 6 Phase 7.

1. Expand `tests/test_safety_integration.py` to full safety pipeline:
   - Test every constraint S1-S15 from section 8.1:
     * S1: Read-only access (DROP TABLE → permission denied at DB level)
     * S2: No DDL (CREATE/DROP/ALTER/TRUNCATE → rejected by validation)
     * S3: No DML (INSERT/UPDATE/DELETE/MERGE → rejected by validation)
     * S4: SELECT only (SELECT...INTO → rejected)
     * S5: Statement timeout (long query → killed after 30s)
     * S6: Row limit (no LIMIT → injected; LIMIT > 1000 → capped)
     * S7: Cost guardrail (expensive query → rejected)
     * S8: No dangerous functions (pg_read_file, pg_sleep, dblink → blocked)
     * S9: No temp tables (CREATE TEMP TABLE → permission denied)
     * S10: Work memory limit (verify work_mem=64MB set)
     * S11: Max iterations (orchestrator stops at 10)
     * S12: Max SQL retries (orchestrator stops at 3)
     * S13: No semicolons (trailing semicolons stripped; multiple statements rejected)
     * S14: Query audit log (every SQL logged to agent_meta.query_log)
     * S15: Rate limiting (max 20 req/min per IP)
   - This is THE critical test file — it must be exhaustive.

2. Add rate limiting to the API:
   - Install slowapi, configure limiter with 20 requests/minute per IP
   - Apply to POST /chat endpoint
   - Write test: 21 rapid requests → 429 Too Many Requests

3. Add query audit logging:
   - In sql_executor.py (coordinate with core via hub), after each execution attempt, log to agent_meta.query_log:
     {session_id, sql_text, validated, validation_errors, executed, row_count, execution_ms, cost_estimate, success, error_message, attempt_num}
   - Write test: execute a query → verify query_log entry exists with correct fields

4. Add structured logging with structlog:
   - Configure structlog in src/main.py
   - Log every tool call with timing (tool_name, duration_ms, success)
   - Log every validation failure (stage, error)
   - Write test: verify log output contains tool call traces
```

**core:**
```
Read IMPLEMENTATION_PLAN.md section 6 Phase 7 (steps 7.4, 7.6).

1. Add error handling for LLM API failures in src/llm/client.py:
   - Retry with exponential backoff (1s, 2s, 4s) for 429/500/503 errors
   - After max retries: return a graceful error message, not a crash
   - Handle timeout (30s per request)
   - Handle invalid API key → clear error message
   - Write test: mock OpenAI to return 429 → verify retry logic
   - Write test: mock OpenAI to return 500 three times → verify graceful degradation

2. Update orchestrator.py to handle LLM client failures:
   - If llm_client.chat_completion raises after retries: return AgentResponse with error answer
   - If tool_router raises unexpectedly: catch, return error as tool result, continue loop

3. Run the full test suite and fix any failures in core-owned test files:
   - tests/test_agent_orchestrator.py
   - tests/test_sql_executor.py (if core owns it — coordinate with safety)

4. Manual end-to-end testing: prepare 10 diverse test questions for the reviewer:
   - Aggregation: "What is the total revenue by category?"
   - Join: "Which employees have sold the most products?"
   - Time series: "Show monthly sales trends for 2023"
   - Ranking: "Top 5 customers by order volume"
   - Filter: "How many orders were shipped to Germany?"
   - Comparison: "Compare revenue from beverages vs. condiments"
   - Subquery: "Which products have never been ordered?"
   - Group by: "Average order value by ship country"
   - Follow-up: "What about just German customers?" (after #5)
   - Edge case: "What is the meaning of life?" (non-SQL question → agent should explain it can't answer with data)
```

**ui:**
```
Read IMPLEMENTATION_PLAN.md section 6 Phase 7 (steps 7.7, 7.8, 7.9).

1. Write `README.md`:
   - Project overview (elevator pitch from section 1)
   - Architecture diagram (from section 2)
   - Quick start: docker compose up → open localhost:8501
   - API documentation (endpoints from section 10)
   - Configuration (.env vars from .env.example)
   - Testing: pytest commands
   - Architecture decisions (from section 3 — why raw function calling, why sqlglot)
   - Safety layer description (from section 8)

2. Verify `.env.example` is complete with all env vars:
   - DATABASE_URL_READONLY
   - DATABASE_URL_META
   - DB_READONLY_PASSWORD
   - OPENAI_API_KEY
   - LLM_MODEL
   - MAX_ROWS, MAX_COST, MAX_ITERATIONS, MAX_SQL_RETRIES
   - API_PORT, UI_PORT

3. Polish Streamlit UI:
   - Add loading spinner during agent processing
   - Add error display for API failures
   - Add "New Session" button in sidebar
   - Add chart type indicator badge
   - Ensure responsive layout
   - Add footer with project info

4. Run the full test suite for ui-owned test files:
   - tests/test_agent_memory.py
   - tests/test_api_chat.py
   - tests/test_chart_generator.py
   Fix any failures.
```

#### Sequential (after parallel)

**reviewer:**
```
Final comprehensive review against IMPLEMENTATION_PLAN.md section 6 Phase 7.

1. Run FULL test suite: `pytest -v` — ALL tests green:
   - tests/test_validation_syntax.py
   - tests/test_validation_safety.py (CRITICAL)
   - tests/test_validation_limit.py
   - tests/test_validation_cost.py
   - tests/test_schema_introspection.py
   - tests/test_sql_executor.py
   - tests/test_chart_generator.py
   - tests/test_agent_orchestrator.py
   - tests/test_agent_memory.py
   - tests/test_safety_integration.py (CRITICAL)
   - tests/test_api_chat.py

2. Fresh `docker compose up` → open Streamlit → ask 10 diverse questions:
   - Aggregations, joins, time series, rankings, filters, comparisons, subqueries, group by, follow-ups, edge cases
   - All produce: valid SQL, correct results, appropriate charts, clear prose answers
   - No errors, no crashes, no safety violations

3. Safety penetration test:
   - Attempt to inject malicious SQL via chat: "DROP TABLE customers" → blocked
   - "Delete all orders" → blocked
   - "SELECT pg_sleep(300)" → blocked
   - "SELECT * FROM pg_tables" → verify behavior (should be blocked by read-only user)
   - Rapid-fire 25 requests → rate limited after 20

4. Verify README.md is complete and someone else could run the project from it.

5. Verify .env.example documents all configuration.

6. Verify query_log audit trail: check agent_meta.query_log has entries for all executed SQL.

7. Verify structured logging: check API logs include tool call traces with timing.

Report any failures. This is the final gate — do not commit until everything is green.
```

#### Commit

```bash
git add -A && git commit -m "Phase 7: Hardening — full safety integration tests, rate limiting, audit logging, LLM error handling, structured logging, README documentation"
```

---

## 5. Coordination Rules

### File Ownership Boundaries

| Directory/File | Owner | Notes |
|----------------|-------|-------|
| `src/agent/orchestrator.py` | core | ReAct loop, error recovery retry logic |
| `src/agent/tool_definitions.py` | core | OpenAI function schemas |
| `src/agent/tool_router.py` | core | Tool dispatch |
| `src/agent/prompts.py` | core | System prompt, interpretation prompt |
| `src/agent/memory.py` | ui | Conversation memory store |
| `src/tools/schema_introspection.py` | core | get_schema implementation |
| `src/tools/sql_executor.py` | core | Execute + error categorization (calls safety's validation) |
| `src/tools/chart_generator.py` | ui | matplotlib/plotly chart generation |
| `src/validation/` | safety | All 4 validation stages + pipeline |
| `src/db/` | safety | Connection engines, session factories |
| `src/llm/client.py` | core | OpenAI client wrapper |
| `src/api/` | ui | All routes and Pydantic models |
| `src/ui/streamlit_app.py` | ui | Streamlit chat interface |
| `src/config.py` | safety | Pydantic Settings |
| `src/main.py` | safety | FastAPI app factory, router registration |
| `docker-compose.yml` | safety | Multi-service orchestration |
| `docker/Dockerfile.api` | safety | API container |
| `docker/Dockerfile.ui` | ui | Streamlit container |
| `docker/postgres/init.sql` | safety | DB init, read-only user |
| `scripts/` | safety | DB init scripts |
| `data/northwind/` | safety | Northwind DDL + data |
| `tests/` | reviewer | All test files (with input from domain agents) |
| `pyproject.toml` | safety | Dependencies |
| `.env.example` | safety | Environment documentation |
| `README.md` | ui | Project documentation (Phase 7) |

### Conflict Avoidance Rules

1. **Never edit another agent's files without permission.** If you need a change in a file you don't own, message the owner via hub.
2. **`src/main.py` is safety's.** When ui creates new API routes, message safety to register them in main.py. Do not edit main.py directly.
3. **`tests/` are reviewer's.** Domain agents write test content and hand it to reviewer for integration. Reviewer owns the test files.
4. **Interface contracts must be agreed before parallel work.** Key interfaces:
   - `validate_sql(sql, db) → ValidationResult` — safety defines, core consumes
   - `get_schema(db, table_names) → dict` — core defines, ui's API route consumes
   - `execute(db, sql, explanation, attempt_num) → ExecutionResult` — core defines, core's orchestrator consumes
   - `generate(db, chart_type, title, x_column, y_column, sql) → ChartResult` — ui defines, core's tool_router consumes
   - `load(session_id, db_meta, limit) → list[dict]` and `append(session_id, db_meta, role, content, ...)` — ui defines, core's orchestrator consumes
5. **The orchestrator (core) is the integration point.** It imports from all three domains. If an interface changes, coordinate via hub.

### Parallel vs Sequential

- **Parallel:** Phases 1, 2, 3, 5, 6, 7 have parallel work segments where agents work on non-overlapping files.
- **Sequential:** Within each phase, the sequential segment handles cross-agent dependencies (e.g., ui creates routes → safety registers them in main.py).
- **Rule of thumb:** If agent A's output is agent B's input, B runs in the sequential segment. If they touch disjoint file sets, they run in parallel.

### Blocked Agent Protocol

1. If an agent is blocked waiting for another agent's file/interface:
   - Message the blocking agent via hub with the specific file and interface needed.
   - While waiting, work on any independent tasks in your domain.
   - If blocked for the entire phase, create a minimal stub/mock to unblock yourself and note it for the reviewer.
2. If an agent discovers a bug in another agent's code:
   - Message the owner via hub with the file, line, and description.
   - Do not fix it yourself unless the owner is unresponsive and the fix is trivial.
3. If safety tests fail during review:
   - This is a hard block. The responsible agent must fix before commit.
   - No phase commits with failing safety tests.

---

## 6. Quick Reference

### Herdr Commands

```bash
# Pane management
herdr pane split --direction vertical --cwd "$PWD" --no-focus
herdr pane split --direction horizontal --cwd "$PWD" --no-focus
herdr pane focus <id>
herdr pane list

# Agent management
herdr agent start <name> --kind codex --pane <id>
herdr agent list
herdr agent stop <name>
herdr agent send <name> "your prompt here"

# Monitoring
herdr agent logs <name>
herdr agent status <name>
```

### Development Commands

```bash
# Docker
docker compose up -d              # Start all services
docker compose down               # Stop all services
docker compose logs -f api        # Follow API logs
docker compose logs -f ui         # Follow UI logs

# Database
docker compose exec postgres psql -U postgres -d northwind
docker compose exec postgres psql -U data_agent_reader -d northwind  # Read-only user

# Testing
pytest -v                         # Full test suite
pytest tests/test_validation_safety.py -v  # Safety tests only
pytest --tb=short                 # Concise tracebacks
pytest -k "test_safety"           # Filter by name

# API testing
curl localhost:8000/health
curl localhost:8000/schema | jq .
curl -X POST localhost:8000/chat -H "Content-Type: application/json" -d '{"message": "What are the top 5 products by revenue?"}'

# UI
open http://localhost:8501        # Streamlit

# Linting
ruff check src/                   # Lint
ruff format src/                  # Format
```

### Key Constants

| Constant | Value | Location |
|----------|-------|----------|
| MAX_ROWS | 1000 | `src/validation/limit_enforcer.py` |
| MAX_COST | 10000.0 | `src/validation/cost_estimator.py` |
| MAX_ITERATIONS | 10 | `src/agent/orchestrator.py` |
| MAX_SQL_RETRIES | 3 | `src/agent/orchestrator.py` |
| CONTEXT_WINDOW | 20 | `src/agent/memory.py` |
| STATEMENT_TIMEOUT | 30s | `docker/postgres/init.sql` |
| WORK_MEM | 64MB | `docker/postgres/init.sql` |
| RATE_LIMIT | 20 req/min | `src/main.py` (slowapi) |

### Phase Summary

| Phase | Days | Parallel Agents | Key Deliverable |
|-------|------|-----------------|-----------------|
| 1. Foundation | 1-2 | safety, ui | Docker Compose, PostgreSQL, read-only user, FastAPI skeleton |
| 2. Schema Introspection | 3 | core, ui | get_schema tool, GET /schema API |
| 3. SQL Validation | 4-5 | safety, core | 4-stage validation pipeline, tool definitions, LLM client |
| 4. SQL Execution + Error Recovery | 6-7 | core, safety | sql_executor, ReAct orchestrator, error recovery loop |
| 5. Charts + Interpretation | 8 | core, ui | System prompt, chart generator, interpretation wiring |
| 6. Memory + Streamlit UI | 9 | ui, core | Conversation memory, POST /chat, full Streamlit chat UI |
| 7. Hardening | 10 | safety, core, ui | Full safety tests, rate limiting, audit logging, README |
