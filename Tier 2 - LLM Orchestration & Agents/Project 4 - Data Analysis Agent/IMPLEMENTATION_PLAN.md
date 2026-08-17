# Tool-Using Data Analysis Agent — Detailed Implementation Plan

> **Elevator pitch:** An LLM agent that connects to a PostgreSQL database, autonomously explores the schema, writes and validates SQL to answer business questions, recovers from its own errors, and responds in natural language with auto-generated charts — all behind a read-only safety layer that makes it production-grade, not a demo.

---

## Table of Contents

1. [Project Goals & Non-Goals](#1-project-goals--non-goals)
2. [Architecture Overview](#2-architecture-overview)
3. [Tech Stack](#3-tech-stack)
4. [Database Schema](#4-database-schema)
5. [Project Structure](#5-project-structure)
6. [Implementation Phases](#6-implementation-phases)
7. [Component Specifications](#7-component-specifications)
8. [Safety & Validation Layer](#8-safety--validation-layer)
9. [Error Recovery Loop](#9-error-recovery-loop)
10. [API Specification](#10-api-specification)
11. [Testing Strategy](#11-testing-strategy)
12. [Deployment](#12-deployment)
13. [Roadmap & Milestones](#13-roadmap--milestones)
14. [Risk Register](#14-risk-register)
15. [Appendix A: Quick Start](#appendix-a-quick-start)
16. [Appendix B: Key Design Decisions](#appendix-b-key-design-decisions)
17. [Appendix C: Sample Conversation Trace](#appendix-c-sample-conversation-trace)

---

## 1. Project Goals & Non-Goals

### Goals

| # | Goal | Success Metric |
|---|------|----------------|
| G1 | **Autonomous SQL generation** — agent writes SQL from natural language business questions using function calling / tool use | ≥ 80% of test questions produce syntactically valid, executable SQL on first or second attempt |
| G2 | **Schema introspection** — agent reads tables, columns, types, foreign keys, and sample values to understand the data before querying | Agent always calls `get_schema` before `execute_sql`; schema context included in every SQL generation prompt |
| G3 | **Multi-layer SQL validation** — syntax check (sqlglot), safety check (no DDL/DML), LIMIT enforcement, cost estimation (EXPLAIN) | 100% of generated SQL passes through validation; zero DDL/DML statements reach the database |
| G4 | **Error recovery loop** — if SQL fails, feed error back to LLM → regenerate → retry (max 3 attempts) | Agent recovers from ≥ 60% of syntax/runtime errors within 3 retries without human intervention |
| G5 | **Natural language interpretation** — LLM summarizes query results in plain language with business context | Every answer includes a prose summary, not just raw rows |
| G6 | **Chart generation** — agent can produce matplotlib/plotly charts from query results | Agent generates appropriate chart type (bar, line, pie, scatter) for ≥ 70% of questions where a chart is meaningful |
| G7 | **Conversation memory** — multi-turn conversations with context carryover | Agent references prior turns ("as we saw earlier…") and can refine queries based on follow-up questions |
| G8 | **Production-grade safety** — read-only DB user, statement timeout, row limits, no DDL/DML, cost guardrails | Safety test suite passes 100%; no query can modify data or exceed resource limits |
| G9 | **Deployable end-to-end** — Docker Compose with Postgres (Northwind data), FastAPI backend, Streamlit chat UI | `docker compose up` starts the full system; user can chat, get SQL + charts, all in browser |

### Non-Goals (explicitly out of scope)

- Write operations (INSERT/UPDATE/DELETE) — v1 is strictly read-only analytics
- Multi-database support — v1 connects to a single PostgreSQL instance
- Natural language to complex OLAP (cube/rollup/window function heavy queries) — v1 targets standard analytical SQL
- Fine-tuning a custom SQL generation model — v1 uses prompt engineering + function calling with a frontier model
- Real-time streaming of partial results — v1 returns complete results
- Multi-user tenancy / authentication — v1 is a single-user local tool (auth is a v2 concern)
- Support for non-PostgreSQL databases — v1 is PostgreSQL-only (sqlglot transpilation is a v2 possibility)

---

## 2. Architecture Overview

```
                              ┌──────────────────────────────────────────────────┐
                              │                 User (Browser)                    │
                              │   Streamlit Chat UI                               │
                              │   - Ask business questions in natural language    │
                              │   - See SQL, results, charts, prose answers       │
                              │   - Multi-turn conversation                       │
                              └─────────────────┬────────────────────────────────┘
                                                │ HTTP (POST /chat)
                              ┌─────────────────▼────────────────────────────────┐
                              │              FastAPI Backend                      │
                              │  ┌──────────────────────────────────────────┐    │
                              │  │           Agent Orchestrator              │    │
                              │  │  (ReAct loop: Reason → Act → Observe)     │    │
                              │  │                                           │    │
                              │  │  ┌─────────┐  ┌──────────┐  ┌─────────┐  │    │
                              │  │  │ Memory  │  │  LLM     │  │ Tool    │  │    │
                              │  │  │ Store   │  │  Client  │  │ Router  │  │    │
                              │  │  └─────────┘  └────┬─────┘  └────┬────┘  │    │
                              │  │                      │              │       │
                              │  │                      │   ┌──────────┼──┐    │
                              │  │                      │   │          │  │    │
                              │  │               ┌──────▼───▼──┐  ┌───▼──▼─┐  │    │
                              │  │               │  Tools      │  │ Chart │  │    │
                              │  │               │             │  │ Gen   │  │    │
                              │  │               │ ┌─────────┐ │  └───────┘  │    │
                              │  │               │ │get_schema│ │             │    │
                              │  │               │ │execute_  │ │             │    │
                              │  │               │ │sql       │ │             │    │
                              │  │               │ │generate_ │ │             │    │
                              │  │               │ │chart     │ │             │    │
                              │  │               │ └────┬────┘ │             │    │
                              │  │               └──────┼──────┘             │    │
                              │  └──────────────────────┼──────────────────────┘    │
                              │                         │                           │
                              │           ┌─────────────▼──────────────┐            │
                              │           │    SQL Validation Layer     │            │
                              │           │  ┌────────┐ ┌────────────┐ │            │
                              │           │  │sqlglot │ │ Safety      │ │            │
                              │           │  │ parse  │ │ Check       │ │            │
                              │           │  └────────┘ └────────────┘ │            │
                              │           │  ┌────────────┐ ┌────────┐ │            │
                              │           │  │ LIMIT      │ │ EXPLAIN │ │            │
                              │           │  │ Enforce    │ │ Cost Est│ │            │
                              │           │  └────────────┘ └────────┘ │            │
                              │           └─────────────┬──────────────┘            │
                              └─────────────────────────┼───────────────────────────┘
                                                        │
                              ┌─────────────────────────▼───────────────────────────┐
                              │              PostgreSQL (Read-Only)                 │
                              │  - Northwind sample data (customers, orders,        │
                              │    products, employees, suppliers, etc.)            │
                              │  - Read-only user: SELECT only                      │
                              │  - statement_timeout: 30s                           │
                              │  - Row limit enforced at app + DB level             │
                              └─────────────────────────────────────────────────────┘
```

### Agent Request Flow (Single Turn)

```
1. User sends question via Streamlit UI → POST /chat {message, session_id}
2. Agent Orchestrator loads conversation memory for session_id
3. Agent enters ReAct loop:
   a. REASON: LLM decides which tool to call (usually get_schema first)
   b. ACT:    Tool router dispatches to get_schema → returns schema JSON
   c. OBSERVE: LLM receives schema, reasons about which tables/columns to use
   d. REASON: LLM decides to call execute_sql with generated SQL
   e. ACT:    SQL passes through validation layer:
              i.   sqlglot parse (syntax check)
              ii.  safety check (no DDL/DML, read-only)
              iii. LIMIT enforcement (inject LIMIT if missing)
              iv.  EXPLAIN cost estimation (reject if too expensive)
              v.   execute via read-only connection with statement_timeout
   f. OBSERVE:
      - SUCCESS: LLM receives query results → reasons about interpretation
      - FAILURE: LLM receives error message → regenerates SQL → retry (max 3)
   g. REASON: LLM decides whether to call generate_chart
   h. ACT:    Chart tool generates matplotlib/plotly figure from results
   i. REASON: LLM composes final natural language answer
4. Agent returns: {answer, sql, results, chart_path, reasoning_trace}
5. Memory store appends the full turn to conversation history
6. Streamlit UI renders: prose answer + SQL block + results table + chart
```

---

## 3. Tech Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **Language** | Python 3.11+ | Ecosystem maturity for AI/ML, async support, type hints |
| **Web Framework (API)** | FastAPI | Async, auto OpenAPI docs, Pydantic validation, WebSocket support for future streaming |
| **Chat UI** | Streamlit | Fastest path to a conversational UI; chat components built-in; renders charts natively |
| **Database** | PostgreSQL 16 | Mature, reliable, rich system catalog for schema introspection; Northwind dataset available |
| **ORM / DB Driver** | SQLAlchemy 2.0 (async) + asyncpg | SQLAlchemy's `inspect()` and `text()` for schema introspection and query execution; async for non-blocking |
| **SQL Parsing** | sqlglot | Pure Python SQL parser; supports PostgreSQL dialect; can parse, transpile, and validate SQL without a DB connection |
| **LLM Orchestration** | Raw OpenAI function calling (primary) + LangChain (optional abstraction) | Raw function calling gives full control over the agent loop; LangChain is an option for those who prefer its abstractions |
| **LLM** | `gpt-4o-mini` (OpenAI) | Fast, cheap, strong at SQL generation and function calling. Alternative: Llama 3.1 70B via Ollama for on-prem |
| **Charting** | matplotlib + plotly | matplotlib for static PNG charts (simple, reliable); plotly for interactive HTML charts (hover, zoom) |
| **Conversation Memory** | In-memory dict + SQLite/Postgres for persistence | Simple in-memory for v1; persisted to DB for cross-session recall |
| **Containerization** | Docker + Docker Compose | Reproducible dev + prod environment; multi-service orchestration |
| **Testing** | pytest + pytest-asyncio + httpx | Async test support, FastAPI-native test client |
| **Linting** | ruff | Fast, replaces flake8 + isort + black |
| **Sample Data** | Northwind (or Northwind-style) | Classic business dataset: customers, orders, products, employees, suppliers — realistic schema with FKs |

### Why raw OpenAI function calling over LangChain agents?

**Control and transparency.** The agent loop is the heart of this project — we want to see exactly how the LLM reasons, which tools it calls, and how errors are fed back. LangChain's `AgentExecutor` wraps this in abstractions that obscure the loop and make error handling harder to customize. By implementing the ReAct loop directly, we:

1. Have full control over the error recovery loop (max retries, error message formatting)
2. Can log every tool call and reasoning step for debugging
3. Avoid LangChain's abstraction leaks and version churn
4. Keep the codebase small and understandable (~500 lines of agent logic)

LangChain is offered as an optional abstraction layer for those who prefer it, but the primary implementation uses raw function calling.

### Why sqlglot for validation instead of just running EXPLAIN?

**Defense in depth.** `EXPLAIN` only works if the SQL is syntactically valid and the connection is live. sqlglot parses SQL without a database connection, catching syntax errors before we even open a connection. It also lets us inspect the AST to enforce safety rules (no DDL/DML, no subqueries with writes, etc.) structurally rather than with regex. The validation pipeline is: **sqlglot parse → AST safety check → LIMIT injection → EXPLAIN cost estimation → execute**.

---

## 4. Database Schema

### 4.1 Northwind Sample Data

The Northwind dataset is a classic Microsoft sample database for a small trading company. It contains realistic business data with foreign key relationships — perfect for testing an agent's ability to navigate a schema.

```
Core Tables (simplified):
┌──────────────┐     ┌──────────────┐     ┌──────────────────┐
│  customers   │     │   orders     │     │  order_details   │
│──────────────│     │──────────────│     │──────────────────│
│ customer_id  │◄──┐ │ order_id     │◄──┐ │ order_id (FK)    │
│ company_name │  │ │ customer_id   │───┘ │ product_id (FK)  │──┐
│ contact_name │  └─│ employee_id   │──┐  │ unit_price       │  │
│ country      │    │ order_date    │  │  │ quantity         │  │
│ city         │    │ shipped_date  │  │  │ discount         │  │
│ phone        │    │ ship_country  │  │  └──────────────────┘  │
└──────────────┘    └──────────────┘  │                          │
                                      │     ┌──────────────┐     │
                                      │     │   products   │     │
                                      │     │──────────────│     │
                                      │     │ product_id   │◄────┘
                                      │     │ product_name │
                                      │     │ category_id  │──┐
                                      │     │ unit_price   │  │
                                      │     │ units_in_stock│ │
                                      │     │ supplier_id  │──┼─┐
                                      │     └──────────────┘  │ │
                                      │                       │ │
                                ┌─────▼─────┐          ┌──────▼─▼──┐
                                │ employees │          │ categories│
                                │───────────│          │───────────│
                                │ employee_id│         │ category_id│
                                │ last_name │          │ category_  │
                                │ first_name│          │  name      │
                                │ reports_to│──┐       └───────────┘
                                │ title     │  │
                                └───────────┘  │
                                     └─────────┘ (self-referencing FK)
                                               ┌──────────────┐
                                               │   suppliers  │
                                               │──────────────│
                                               │ supplier_id  │
                                               │ company_name │
                                               │ country      │
                                               └──────────────┘
```

### 4.2 Full Table List (Northwind)

| Table | Rows (approx) | Key Columns | Foreign Keys |
|-------|---------------|-------------|--------------|
| `categories` | 8 | category_id, category_name, description | — |
| `suppliers` | 29 | supplier_id, company_name, country | — |
| `customers` | 91 | customer_id, company_name, country, city | — |
| `employees` | 9 | employee_id, last_name, first_name, reports_to, title | reports_to → employees.employee_id (self-ref) |
| `products` | 77 | product_id, product_name, category_id, supplier_id, unit_price, units_in_stock | category_id → categories, supplier_id → suppliers |
| `orders` | 830 | order_id, customer_id, employee_id, order_date, shipped_date, ship_country | customer_id → customers, employee_id → employees |
| `order_details` | 2,155 | order_id, product_id, unit_price, quantity, discount | order_id → orders, product_id → products |
| `shippers` | 3 | shipper_id, company_name | — |
| `us_states` | 51 | state_id, state_name, state_abbr | — |

### 4.3 Read-Only Database User

```sql
-- Create a dedicated read-only user for the agent
-- This is the SAFETY FOUNDATION: even if all app-level checks fail,
-- the database itself rejects any write operation.

CREATE USER data_agent_reader WITH PASSWORD '${DB_READONLY_PASSWORD}';

-- Grant connect to the database
GRANT CONNECT ON DATABASE northwind TO data_agent_reader;

-- Grant usage on schema
GRANT USAGE ON SCHEMA public TO data_agent_reader;

-- Grant SELECT on all existing tables
GRANT SELECT ON ALL TABLES IN SCHEMA public TO data_agent_reader;

-- Grant SELECT on all future tables (if schema evolves)
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT ON TABLES TO data_agent_reader;

-- Set statement timeout (30 seconds) at the user level
ALTER USER data_agent_reader SET statement_timeout = '30s';

-- Set work memory limit to prevent resource exhaustion
ALTER USER data_agent_reader SET work_mem = '64MB';

-- Prevent creating temp tables (reduces attack surface)
-- (Note: this requires revoking TEMP privilege on the database)
REVOKE TEMP ON DATABASE northwind FROM data_agent_reader;
```

### 4.4 Agent Metadata Tables (Separate Schema)

The agent itself needs a small metadata schema for conversation persistence. This lives in a separate `agent_meta` schema, accessed by a **different** DB user with write privileges (the app backend, not the agent's read-only connection).

```sql
CREATE SCHEMA IF NOT EXISTS agent_meta;

-- Conversation sessions
CREATE TABLE agent_meta.sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title           TEXT,                          -- auto-generated from first message
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Conversation messages (all turns: user + assistant + tool calls)
CREATE TABLE agent_meta.messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES agent_meta.sessions(id) ON DELETE CASCADE,
    role            TEXT NOT NULL,                 -- 'user', 'assistant', 'tool', 'system'
    content         TEXT NOT NULL,                 -- message text or JSON for tool calls
    tool_name       TEXT,                          -- which tool was called (if role='tool')
    tool_call_id    TEXT,                          -- OpenAI function call ID
    sql_generated   TEXT,                          -- SQL if this turn involved a query
    row_count       INT,                           -- rows returned (if applicable)
    execution_ms    INT,                           -- query execution time
    attempt_num     INT DEFAULT 1,                 -- which retry attempt (1-3)
    error_message   TEXT,                          -- if the attempt failed
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_messages_session ON agent_meta.messages(session_id, created_at);
CREATE INDEX idx_messages_session_role ON agent_meta.messages(session_id, role);

-- Query log (audit trail of all SQL executed)
CREATE TABLE agent_meta.query_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID REFERENCES agent_meta.sessions(id),
    sql_text        TEXT NOT NULL,
    validated       BOOLEAN NOT NULL DEFAULT FALSE,
    validation_errors TEXT[],                      -- list of validation failures
    executed        BOOLEAN NOT NULL DEFAULT FALSE,
    row_count       INT,
    execution_ms    INT,
    cost_estimate   TEXT,                          -- EXPLAIN output summary
    success         BOOLEAN NOT NULL DEFAULT FALSE,
    error_message   TEXT,
    attempt_num     INT NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_query_log_session ON agent_meta.query_log(session_id, created_at);
CREATE INDEX idx_query_log_created ON agent_meta.query_log(created_at DESC);
```

---

## 5. Project Structure

```
data-analysis-agent/
├── docker-compose.yml
├── .env.example
├── pyproject.toml
├── README.md
├── IMPLEMENTATION_PLAN.md              ← this file
├── agent.md                      # Herdr multi-agent orchestration guide
│
├── src/
│   ├── __init__.py
│   ├── main.py                         # FastAPI app factory + lifespan
│   ├── config.py                       # Pydantic Settings (env vars)
│   │
│   ├── agent/
│   │   ├── __init__.py
│   │   ├── orchestrator.py             # ReAct loop: reason → act → observe → repeat
│   │   ├── tool_definitions.py         # OpenAI function/tool schemas (JSON)
│   │   ├── tool_router.py              # Dispatches tool calls to implementations
│   │   ├── memory.py                   # Conversation memory store (session-scoped)
│   │   └── prompts.py                  # System prompt, SQL generation prompt, interpretation prompt
│   │
│   ├── tools/
│   │   ├── __init__.py
│   │   ├── schema_introspection.py     # get_schema: read tables, columns, types, FKs, samples
│   │   ├── sql_executor.py             # execute_sql: validate → execute → return results
│   │   └── chart_generator.py          # generate_chart: matplotlib/plotly from query results
│   │
│   ├── validation/
│   │   ├── __init__.py
│   │   ├── syntax_check.py             # sqlglot parse + dialect validation
│   │   ├── safety_check.py             # AST analysis: no DDL/DML, no dangerous functions
│   │   ├── limit_enforcer.py           # Inject LIMIT if missing; cap at max rows
│   │   └── cost_estimator.py           # EXPLAIN → parse cost → reject if too expensive
│   │
│   ├── db/
│   │   ├── __init__.py
│   │   ├── connection.py               # SQLAlchemy async engine (read-only + meta)
│   │   ├── readonly_session.py         # Read-only session factory with timeout
│   │   └── meta_session.py             # Metadata session (read-write, for agent_meta)
│   │
│   ├── llm/
│   │   ├── __init__.py
│   │   └── client.py                   # OpenAI client wrapper (function calling, retries)
│   │
│   ├── api/
│   │   ├── __init__.py
│   │   ├── routes/
│   │   │   ├── __init__.py
│   │   │   ├── chat.py                 # POST /chat (main endpoint)
│   │   │   ├── sessions.py             # GET /sessions, GET /sessions/{id}
│   │   │   ├── schema.py               # GET /schema (view DB schema)
│   │   │   └── health.py               # GET /health
│   │   └── models.py                   # Pydantic request/response models
│   │
│   └── ui/
│       ├── __init__.py
│       └── streamlit_app.py            # Streamlit chat UI
│
├── data/
│   └── northwind/
│       ├── northwind_schema.sql        # DDL for Northwind tables
│       └── northwind_data.sql          # INSERT statements for sample data
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py                     # Fixtures: test DB, test client, seed data
│   ├── test_validation_syntax.py       # sqlglot parse tests
│   ├── test_validation_safety.py       # DDL/DML rejection, dangerous function blocking
│   ├── test_validation_limit.py        # LIMIT injection tests
│   ├── test_validation_cost.py         # EXPLAIN cost estimation tests
│   ├── test_schema_introspection.py    # Schema reading correctness
│   ├── test_sql_executor.py            # End-to-end: validate → execute → return
│   ├── test_chart_generator.py         # Chart generation from result sets
│   ├── test_agent_orchestrator.py      # Agent loop: tool selection, error recovery
│   ├── test_agent_memory.py            # Multi-turn conversation context
│   ├── test_safety_integration.py      # Full safety pipeline (critical)
│   └── test_api_chat.py                # End-to-end API tests
│
├── scripts/
│   ├── init_readonly_user.sql          # Create read-only DB user
│   ├── init_northwind.py               # Load Northwind schema + data
│   └── seed_agent_meta.py              # Create agent_meta schema
│
└── docker/
    ├── Dockerfile.api                  # FastAPI backend image
    ├── Dockerfile.ui                   # Streamlit UI image
    └── postgres/
        └── init.sql                    # Postgres init: extensions + readonly user
```

---

## 6. Implementation Phases

### Phase 1: Foundation — DB, Read-Only User, FastAPI Skeleton (Days 1-2)

**Goal:** Running PostgreSQL with Northwind data, read-only user, FastAPI skeleton, health check.

| Step | Task | Deliverable |
|------|------|-------------|
| 1.1 | Write `docker-compose.yml` with Postgres, API, and UI services | `docker compose up` starts all three |
| 1.2 | Write `docker/postgres/init.sql` (extensions, read-only user, statement timeout) | DB initializes with read-only user on first boot |
| 1.3 | Load Northwind schema + data (`data/northwind/*.sql`) | Northwind tables populated (830 orders, 2155 order_details, etc.) |
| 1.4 | Write `src/config.py` (Pydantic Settings) | Env-driven config with two DB URLs (readonly + meta) |
| 1.5 | Write `src/db/connection.py` (two SQLAlchemy async engines) | Read-only engine + metadata engine both connect |
| 1.6 | Write `src/main.py` (FastAPI app factory + lifespan) | App starts, `GET /health` responds |
| 1.7 | Write `src/api/routes/health.py` | Health check verifies both DB connections |
| 1.8 | Write `tests/conftest.py` (test DB fixtures) | Test fixtures available |

**Verification:** `docker compose up` → `curl localhost:8000/health` → `{"status": "healthy", "database": "connected", "readonly_user": "active"}`. Connect as `data_agent_reader` → attempt `DROP TABLE customers` → permission denied.

---

### Phase 2: Schema Introspection Tool (Day 3)

**Goal:** Agent can read the full database schema — tables, columns, types, foreign keys, and sample values.

| Step | Task | Deliverable |
|------|------|-------------|
| 2.1 | Write `src/tools/schema_introspection.py` | `get_schema()` returns structured schema JSON |
| 2.2 | Implement table discovery (information_schema.tables) | All Northwind tables listed |
| 2.3 | Implement column discovery (information_schema.columns) | Columns with name, type, nullable, default |
| 2.4 | Implement foreign key discovery (pg_constraint) | All FK relationships mapped |
| 2.5 | Implement sample value discovery (SELECT DISTINCT ... LIMIT 5) | 5 sample values per text/enum column |
| 2.6 | Implement row count estimation (pg_class.reltuples) | Approximate row counts per table |
| 2.7 | Write `src/api/routes/schema.py` (`GET /schema`) | Schema viewable via API |
| 2.8 | Write `tests/test_schema_introspection.py` | Schema introspection tests pass |

**Verification:** `GET /schema` returns JSON with all 9 Northwind tables, their columns, types, foreign keys, and sample values. Verify FK from `orders.customer_id` → `customers.customer_id` is present.

---

### Phase 3: SQL Validation Layer (Days 4-5)

**Goal:** Multi-layer validation pipeline: syntax → safety → LIMIT → cost. This is the production-grade safety core.

| Step | Task | Deliverable |
|------|------|-------------|
| 3.1 | Write `src/validation/syntax_check.py` (sqlglot parse) | Invalid SQL → `SyntaxError` with message |
| 3.2 | Write `src/validation/safety_check.py` (AST analysis) | DDL/DML → `SafetyError`; dangerous functions blocked |
| 3.3 | Write `src/validation/limit_enforcer.py` | Missing LIMIT → inject `LIMIT 1000`; existing LIMIT > max → cap |
| 3.4 | Write `src/validation/cost_estimator.py` (EXPLAIN) | Parse EXPLAIN output; reject if cost > threshold |
| 3.5 | Compose validation pipeline: `validate_sql(sql) → ValidationResult` | All four checks run in sequence |
| 3.6 | Write `tests/test_validation_syntax.py` | Syntax check tests pass |
| 3.7 | Write `tests/test_validation_safety.py` | Safety check tests pass (critical) |
| 3.8 | Write `tests/test_validation_limit.py` | LIMIT enforcement tests pass |
| 3.9 | Write `tests/test_validation_cost.py` | Cost estimation tests pass |

**Verification:** Run validation suite:
- `DROP TABLE customers` → rejected (safety)
- `DELETE FROM orders WHERE 1=1` → rejected (safety)
- `SELECT * FROM orders` → passes with injected `LIMIT 1000`
- `SELECT * FROM orders o JOIN order_details od ON o.order_id = od.order_id` → passes, EXPLAIN cost < threshold
- `SELCT * FROM orders` → rejected (syntax: `selct` is not a valid keyword)

---

### Phase 4: SQL Execution Tool + Error Recovery (Days 6-7)

**Goal:** Agent can execute validated SQL via the read-only connection and recover from errors.

| Step | Task | Deliverable |
|------|------|-------------|
| 4.1 | Write `src/tools/sql_executor.py` (validate → execute → return) | SQL execution with full validation pipeline |
| 4.2 | Implement result formatting (columns + rows as list of dicts) | Results returned as structured JSON |
| 4.3 | Implement error capture (psycopg2/asyncpg errors → structured message) | Errors include type, message, hint, position |
| 4.4 | Write `src/agent/tool_definitions.py` (OpenAI function schemas) | `get_schema`, `execute_sql`, `generate_chart` tool definitions |
| 4.5 | Write `src/agent/tool_router.py` (dispatch tool calls) | Tool calls dispatched to correct implementation |
| 4.6 | Write `src/llm/client.py` (OpenAI function calling wrapper) | LLM calls with tool definitions, response parsing |
| 4.7 | Write `src/agent/orchestrator.py` (ReAct loop with error recovery) | Full agent loop: reason → act → observe → retry |
| 4.8 | Write `tests/test_sql_executor.py` | Execution + validation integration tests pass |
| 4.9 | Write `tests/test_agent_orchestrator.py` (mocked LLM) | Agent loop tests with mocked LLM responses pass |

**Verification:** Send a test question through the agent: "What are the top 5 products by revenue?" → agent calls `get_schema` → calls `execute_sql` with valid SQL → returns results. Then test error recovery: inject a deliberately broken SQL → agent receives error → regenerates → succeeds within 3 attempts.

---

### Phase 5: Result Interpretation + Chart Generation (Day 8)

**Goal:** Agent interprets results in natural language and generates appropriate charts.

| Step | Task | Deliverable |
|------|------|-------------|
| 5.1 | Write interpretation prompt in `src/agent/prompts.py` | LLM instructed to summarize results in business language |
| 5.2 | Wire interpretation into agent orchestrator (final step before response) | Every answer includes prose summary |
| 5.3 | Write `src/tools/chart_generator.py` (matplotlib + plotly) | Chart generation from query results |
| 5.4 | Implement chart type selection (bar, line, pie, scatter, table) | Agent specifies chart type via tool parameter |
| 5.5 | Implement chart data extraction (columns → x/y axes) | Chart data structured from result rows |
| 5.6 | Save charts as PNG (matplotlib) and HTML (plotly) to temp directory | Chart files accessible by UI |
| 5.7 | Add `generate_chart` tool to agent tool definitions | Agent can call chart generation autonomously |
| 5.8 | Write `tests/test_chart_generator.py` | Chart generation tests pass |

**Verification:** Ask "Show me monthly sales trends for 2023" → agent generates SQL → executes → interprets results → calls `generate_chart` with `chart_type: "line"` → PNG chart saved. Ask "What's the revenue breakdown by category?" → agent generates pie chart. Ask "How many orders did each employee process?" → agent generates bar chart.

---

### Phase 6: Conversation Memory + Streamlit UI (Day 9)

**Goal:** Multi-turn conversations with context + full chat UI in the browser.

| Step | Task | Deliverable |
|------|------|-------------|
| 6.1 | Write `src/agent/memory.py` (session-scoped memory store) | Conversation history persisted per session |
| 6.2 | Implement memory loading (last N messages as context for LLM) | Agent has context from prior turns |
| 6.3 | Implement memory persistence (agent_meta.messages table) | Conversations survive restarts |
| 6.4 | Write `src/api/routes/chat.py` (`POST /chat` with session_id) | Chat endpoint with session management |
| 6.5 | Write `src/api/routes/sessions.py` (list/get sessions) | Session history viewable via API |
| 6.6 | Write `src/ui/streamlit_app.py` (chat interface) | Full chat UI: messages, SQL display, results table, charts |
| 6.7 | Implement streaming-like display (show reasoning steps as they happen) | User sees agent's tool calls and reasoning in real-time |
| 6.8 | Write `tests/test_agent_memory.py` | Memory tests pass |
| 6.9 | Write `tests/test_api_chat.py` | End-to-end API tests pass |

**Verification:** Open Streamlit UI → ask "What are the top 5 customers by order volume?" → see answer + SQL + table + chart. Then ask follow-up "What about just German customers?" → agent references prior context, adds `WHERE ship_country = 'Germany'`, returns refined results.

---

### Phase 7: Hardening, Testing, Documentation (Day 10)

**Goal:** Production-ready, fully tested, documented, edge cases handled.

| Step | Task | Deliverable |
|------|------|-------------|
| 7.1 | Write `tests/test_safety_integration.py` (full safety pipeline) | Critical: no DDL/DML ever reaches DB |
| 7.2 | Add rate limiting (slowapi) to API | Prevents abuse |
| 7.3 | Add structured logging (structlog) with tool call tracing | Every tool call logged with timing |
| 7.4 | Add error handling for LLM API failures (retry, fallback) | Graceful degradation when LLM is unavailable |
| 7.5 | Add query log audit trail (agent_meta.query_log) | Every SQL execution logged with validation status |
| 7.6 | Run full test suite end-to-end | All tests green |
| 7.7 | Write `README.md` (setup, usage, architecture, screenshots) | Someone else can run it |
| 7.8 | Add `.env.example` with all env vars | Configuration documented |
| 7.9 | Manual end-to-end testing with 20+ diverse questions | Agent handles edge cases gracefully |

**Verification:** Fresh `docker compose up` → open Streamlit → ask 10 diverse questions (aggregations, joins, time series, rankings, filters) → all produce valid SQL, correct results, appropriate charts, and clear prose answers. Run `pytest` → all tests green. Attempt to inject malicious SQL via chat → all blocked by validation layer.

---

## 7. Component Specifications

### 7.1 Tool Definitions (`src/agent/tool_definitions.py`)

These are the OpenAI function calling tool schemas that the LLM uses to interact with the database. The LLM sees these definitions and decides which tool to call based on the user's question.

```python
"""
OpenAI function calling tool definitions.

These schemas are sent to the LLM as 'tools' in the chat completion request.
The LLM decides which tool to call and with what arguments, based on the
user's question and the conversation context.

The agent orchestrator executes the requested tool and returns the result
as a 'tool' role message, which the LLM uses to reason about the next step.
"""

TOOL_DEFINITIONS = [
    {
        "type": "function",
        "function": {
            "name": "get_schema",
            "description": (
                "Get the database schema: tables, columns, data types, "
                "foreign key relationships, sample values, and approximate "
                "row counts. ALWAYS call this first before writing SQL to "
                "understand the available data structure."
            ),
            "parameters": {
                "type": "object",
                "properties": {
                    "table_names": {
                        "type": "array",
                        "items": {"type": "string"},
                        "description": (
                            "Optional: specific tables to inspect. "
                            "If omitted, returns all tables."
                        ),
                    }
                },
                "required": [],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "execute_sql",
            "description": (
                "Execute a read-only SQL query against the PostgreSQL "
                "database. The SQL must be a SELECT statement (no DDL/DML). "
                "A LIMIT clause will be enforced if missing. The query "
                "passes through syntax validation, safety checks, and "
                "cost estimation before execution. If the query fails, "
                "the error message will be returned — use it to fix the "
                "SQL and retry."
            ),
            "parameters": {
                "type": "object",
                "properties": {
                    "sql": {
                        "type": "string",
                        "description": (
                            "The SQL SELECT query to execute. Must be "
                            "valid PostgreSQL syntax. Do not include "
                            "semicolons."
                        ),
                    },
                    "explanation": {
                        "type": "string",
                        "description": (
                            "Brief explanation of what this query does "
                            "and why it answers the user's question."
                        ),
                    },
                },
                "required": ["sql", "explanation"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "generate_chart",
            "description": (
                "Generate a chart from query results. Call this after "
                "execute_sql returns results that would benefit from "
                "visualization. Choose the chart type based on the data: "
                "bar for comparisons, line for trends over time, pie for "
                "proportions, scatter for correlations."
            ),
            "parameters": {
                "type": "object",
                "properties": {
                    "chart_type": {
                        "type": "string",
                        "enum": ["bar", "line", "pie", "scatter", "table"],
                        "description": "Type of chart to generate.",
                    },
                    "title": {
                        "type": "string",
                        "description": "Chart title.",
                    },
                    "x_column": {
                        "type": "string",
                        "description": (
                            "Column name for the x-axis (or labels for pie)."
                        ),
                    },
                    "y_column": {
                        "type": "string",
                        "description": (
                            "Column name for the y-axis (or values for pie)."
                        ),
                    },
                    "sql": {
                        "type": "string",
                        "description": (
                            "The SQL query that produced the data. "
                            "The chart tool will re-execute it to get "
                            "fresh data."
                        ),
                    },
                },
                "required": ["chart_type", "title", "x_column", "y_column", "sql"],
            },
        },
    },
]
```

### 7.2 Agent Orchestrator — ReAct Loop (`src/agent/orchestrator.py`)

```python
"""
The agent orchestrator implements a ReAct (Reason + Act) loop:

1. REASON: The LLM receives the user's question + conversation history +
   tool definitions, and decides which tool to call (or gives a final answer).
2. ACT: The orchestrator executes the requested tool.
3. OBSERVE: The tool result is fed back to the LLM as a 'tool' role message.
4. REPEAT until the LLM produces a final answer (no tool call) or max iterations.

Error recovery is built into the loop: if execute_sql fails, the error
message is returned as the tool result. The LLM sees the error and can
regenerate the SQL with a fix. This continues up to MAX_RETRIES (3) per
SQL attempt, and MAX_ITERATIONS (10) for the overall loop.

The orchestrator logs every step to the query_log and messages tables
for full auditability.
"""

MAX_ITERATIONS = 10  # Total reasoning steps per user turn
MAX_SQL_RETRIES = 3  # Max retries for a failed SQL query

async def run_agent(
    user_message: str,
    session_id: str,
    db_readonly: AsyncSession,   # Read-only DB connection
    db_meta: AsyncSession,       # Metadata DB (for logging)
) -> AgentResponse:
    # 1. Load conversation memory
    history = await memory_store.load(session_id, limit=10)
    messages = [
        {"role": "system", "content": SYSTEM_PROMPT},
        *history,
        {"role": "user", "content": user_message},
    ]

    # 2. Persist user message
    await memory_store.append(session_id, role="user", content=user_message)

    # 3. ReAct loop
    reasoning_trace = []
    sql_attempts = 0  # Track SQL retries across the whole turn

    for iteration in range(MAX_ITERATIONS):
        # REASON: Ask LLM what to do next
        response = await llm_client.chat_completion(
            messages=messages,
            tools=TOOL_DEFINITIONS,
            tool_choice="auto",
        )

        # Case A: LLM produced a final answer (no tool call)
        if not response.tool_calls:
            final_answer = response.content
            reasoning_trace.append({
                "step": iteration + 1,
                "type": "final_answer",
                "content": final_answer,
            })
            # Persist assistant message
            await memory_store.append(
                session_id, role="assistant", content=final_answer
            )
            return AgentResponse(
                answer=final_answer,
                sql=last_sql,           # last SQL executed (if any)
                results=last_results,   # last results (if any)
                chart_path=last_chart,  # last chart path (if any)
                reasoning_trace=reasoning_trace,
            )

        # Case B: LLM wants to call a tool
        for tool_call in response.tool_calls:
            tool_name = tool_call.function.name
            tool_args = json.loads(tool_call.function.arguments)

            reasoning_trace.append({
                "step": iteration + 1,
                "type": "tool_call",
                "tool": tool_name,
                "args": tool_args,
            })

            # ACT: Execute the tool
            try:
                if tool_name == "get_schema":
                    result = await schema_introspection.get_schema(
                        db_readonly,
                        table_names=tool_args.get("table_names"),
                    )

                elif tool_name == "execute_sql":
                    sql_attempts += 1
                    result = await sql_executor.execute(
                        db_readonly,
                        sql=tool_args["sql"],
                        explanation=tool_args.get("explanation", ""),
                        attempt_num=sql_attempts,
                    )
                    if result.success:
                        last_sql = tool_args["sql"]
                        last_results = result.rows
                    # If failed, result.error_message is included —
                    # the LLM will see it and can retry

                elif tool_name == "generate_chart":
                    result = await chart_generator.generate(
                        db_readonly,
                        chart_type=tool_args["chart_type"],
                        title=tool_args["title"],
                        x_column=tool_args["x_column"],
                        y_column=tool_args["y_column"],
                        sql=tool_args["sql"],
                    )
                    if result.success:
                        last_chart = result.chart_path

                else:
                    result = {"error": f"Unknown tool: {tool_name}"}

            except Exception as e:
                result = {"error": str(e)}

            # OBSERVE: Feed tool result back to LLM
            messages.append({
                "role": "assistant",
                "content": None,
                "tool_calls": [tool_call],
            })
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": json.dumps(result, default=str),
            })

            # Persist tool call to memory
            await memory_store.append(
                session_id,
                role="tool",
                content=json.dumps(result, default=str),
                tool_name=tool_name,
                tool_call_id=tool_call.id,
            )

    # Max iterations reached without final answer
    return AgentResponse(
        answer="I wasn't able to complete this analysis within the step limit. "
               "Could you try rephrasing your question or breaking it into parts?",
        reasoning_trace=reasoning_trace,
    )
```

### 7.3 Schema Introspection Tool (`src/tools/schema_introspection.py`)

```python
"""
Reads the database schema and returns a structured JSON representation.

The output includes:
  - Table names with approximate row counts
  - Column names, data types, nullability, defaults
  - Foreign key relationships (table.column → referenced_table.column)
  - Sample values (5 distinct values per text/enum column)

This is the agent's "eyes" — it must call this before writing SQL.
The output is designed to be compact enough to fit in the LLM context
window while being detailed enough for accurate SQL generation.
"""

async def get_schema(
    db: AsyncSession,
    table_names: list[str] | None = None,
) -> dict:
    # 1. Get all tables (or filtered subset)
    tables = await db.execute(text("""
        SELECT table_name
        FROM information_schema.tables
        WHERE table_schema = 'public'
          AND table_type = 'BASE TABLE'
          AND (:table_names IS NULL OR table_name = ANY(:table_names))
        ORDER BY table_name
    """), {"table_names": table_names})

    schema = {"tables": []}

    for (table_name,) in tables:
        # 2. Get columns
        columns = await db.execute(text("""
            SELECT column_name, data_type, is_nullable,
                   column_default, character_maximum_length
            FROM information_schema.columns
            WHERE table_schema = 'public' AND table_name = :tbl
            ORDER BY ordinal_position
        """), {"tbl": table_name})

        # 3. Get foreign keys
        fks = await db.execute(text("""
            SELECT kcu.column_name,
                   ccu.table_name AS foreign_table,
                   ccu.column_name AS foreign_column
            FROM information_schema.table_constraints tc
            JOIN information_schema.key_column_usage kcu
              ON tc.constraint_name = kcu.constraint_name
            JOIN information_schema.constraint_column_usage ccu
              ON ccu.constraint_name = tc.constraint_name
            WHERE tc.constraint_type = 'FOREIGN KEY'
              AND tc.table_name = :tbl
        """), {"tbl": table_name})

        # 4. Get approximate row count (from pg_class, no COUNT(*))
        row_count = await db.execute(text("""
            SELECT reltuples::bigint AS approx_rows
            FROM pg_class
            WHERE relname = :tbl
        """), {"tbl": table_name})

        # 5. Get sample values for text columns (5 distinct values)
        column_list = []
        for col in columns:
            col_info = {
                "name": col.column_name,
                "type": col.data_type,
                "nullable": col.is_nullable == "YES",
                "default": col.column_default,
            }

            # Sample values for text/varchar columns only (avoid heavy sampling)
            if col.data_type in ("text", "character varying", "character"):
                samples = await db.execute(text(
                    f"SELECT DISTINCT {col.column_name} "
                    f"FROM {table_name} "
                    f"WHERE {col.column_name} IS NOT NULL "
                    f"LIMIT 5"
                ))
                col_info["sample_values"] = [r[0] for r in samples]

            column_list.append(col_info)

        # 6. Assemble table info
        schema["tables"].append({
            "table_name": table_name,
            "approx_rows": int(row_count.scalar() or 0),
            "columns": column_list,
            "foreign_keys": [
                {
                    "column": fk.column_name,
                    "references_table": fk.foreign_table,
                    "references_column": fk.foreign_column,
                }
                for fk in fks
            ],
        })

    return schema
```

### 7.4 SQL Validation Pipeline (`src/validation/`)

The validation pipeline runs four checks in sequence. Each check can reject the SQL, preventing execution. The pipeline returns a structured result with all validation errors.

```python
"""
SQL Validation Pipeline

Order of checks (fail fast):
  1. SYNTAX:    sqlglot.parse(sql, dialect="postgres")
                → catches typos, missing keywords, unbalanced parens
  2. SAFETY:    AST analysis — only SELECT allowed
                → blocks DDL (CREATE, DROP, ALTER), DML (INSERT, UPDATE, DELETE),
                  dangerous functions (pg_read_file, lo_import, etc.)
  3. LIMIT:     Inject LIMIT if missing; cap at MAX_ROWS if present
                → prevents accidental full table scans returning millions of rows
  4. COST:      EXPLAIN the query; parse total cost
                → rejects queries estimated to be too expensive (full table scans
                  on large tables, cartesian products, etc.)

If any check fails, the pipeline returns immediately with the error.
The error message is designed to be LLM-readable so the agent can fix the SQL.
"""

from dataclasses import dataclass
from enum import Enum

class ValidationStage(Enum):
    SYNTAX = "syntax"
    SAFETY = "safety"
    LIMIT = "limit"
    COST = "cost"

@dataclass
class ValidationResult:
    valid: bool
    sql: str                          # possibly modified (LIMIT injected)
    original_sql: str
    errors: list[dict]                # [{stage, message, detail}]
    cost_estimate: str | None = None
    cost_value: float | None = None

MAX_ROWS = 1000                       # hard cap on returned rows
MAX_COST = 10000.0                    # EXPLAIN cost threshold

# --- Stage 1: Syntax Check ---
# src/validation/syntax_check.py

import sqlglot
from sqlglot.errors import ParseError

def check_syntax(sql: str) -> tuple[bool, str | None, object | None]:
    """
    Parse SQL with sqlglot using PostgreSQL dialect.
    Returns (valid, error_message, ast).
    """
    try:
        ast = sqlglot.parse_one(sql, dialect="postgres")
        return True, None, ast
    except ParseError as e:
        return False, f"SQL syntax error: {e}", None
    except Exception as e:
        return False, f"SQL parse error: {e}", None


# --- Stage 2: Safety Check ---
# src/validation/safety_check.py

from sqlglot import exp

# Statement types that are allowed (read-only)
ALLOWED_STATEMENT_TYPES = {exp.Select}

# Explicitly blocked statement types
BLOCKED_STATEMENT_TYPES = {
    exp.Insert, exp.Update, exp.Delete, exp.Drop, exp.Create,
    exp.Alter, exp.TruncateTable, exp.Merge, exp.Command,
}

# Dangerous functions that could exfiltrate data or escalate privileges
BLOCKED_FUNCTIONS = {
    "pg_read_file", "pg_read_binary_file", "pg_ls_dir",
    "lo_import", "lo_export", "pg_sleep",
    "pg_terminate_backend", "pg_cancel_backend",
    "dblink", "dblink_query", "dblink_exec",
    "pg_extension", "pg_reload_conf",
}

def check_safety(ast: object) -> tuple[bool, str | None]:
    """
    Analyze the SQL AST to ensure it's a read-only SELECT with no
    dangerous operations.
    """
    # Check top-level statement type
    if type(ast) not in ALLOWED_STATEMENT_TYPES:
        if type(ast) in BLOCKED_STATEMENT_TYPES:
            stmt_name = type(ast).__name__.upper()
            return False, (
                f"Safety violation: {stmt_name} statements are not allowed. "
                f"Only SELECT queries are permitted."
            )
        return False, (
            f"Safety violation: statement type {type(ast).__name__} is not recognized "
            f"as a read-only query. Only SELECT is allowed."
        )

    # Walk the AST for dangerous constructs
    for node in ast.walk():
        # Check for blocked functions
        if isinstance(node, exp.Anonymous):
            func_name = node.name.lower()
            if func_name in BLOCKED_FUNCTIONS:
                return False, (
                    f"Safety violation: function '{func_name}' is blocked. "
                    f"This function can access the filesystem or interfere "
                    f"with database internals."
                )

        # Check for INTO (SELECT ... INTO creates a table)
        if isinstance(node, exp.Into):
            return False, (
                "Safety violation: SELECT ... INTO is not allowed "
                "(it creates a new table)."
            )

        # Check for DDL within CTEs (rare but possible)
        if isinstance(node, (exp.Create, exp.Drop, exp.Alter)):
            return False, (
                "Safety violation: DDL statement found within query. "
                "Only read-only SELECT is allowed."
            )

    return True, None


# --- Stage 3: LIMIT Enforcer ---
# src/validation/limit_enforcer.py

def enforce_limit(ast: object, sql: str, max_rows: int = MAX_ROWS) -> tuple[str, object]:
    """
    Ensure the query has a LIMIT clause. If missing, inject LIMIT max_rows.
    If present but > max_rows, cap it.
    Returns (modified_sql, modified_ast).
    """
    existing_limit = ast.args.get("limit")

    if existing_limit is None:
        # No LIMIT — inject one
        ast = ast.limit(max_rows)
        modified_sql = ast.sql(dialect="postgres")
        return modified_sql, ast
    else:
        # LIMIT exists — check if it's within bounds
        limit_expr = existing_limit.expression
        if isinstance(limit_expr, exp.Literal):
            limit_val = int(limit_expr.this)
            if limit_val > max_rows:
                # Cap the limit
                ast = ast.set("limit", exp.Limit(
                    expression=exp.Literal.number(max_rows)
                ))
                modified_sql = ast.sql(dialect="postgres")
                return modified_sql, ast
        return sql, ast


# --- Stage 4: Cost Estimator ---
# src/validation/cost_estimator.py

async def estimate_cost(
    db: AsyncSession,
    sql: str,
    max_cost: float = MAX_COST,
) -> tuple[bool, str | None, float | None]:
    """
    Run EXPLAIN on the query and parse the total cost.
    Reject if the estimated total cost exceeds max_cost.
    """
    explain_sql = f"EXPLAIN (FORMAT JSON) {sql}"
    try:
        result = await db.execute(text(explain_sql))
        plan = result.scalar()
        # EXPLAIN FORMAT JSON returns a JSON array with plan info
        total_cost = plan[0]["Plan"]["Total Cost"]
        cost_str = f"Total cost: {total_cost:.2f}"

        if total_cost > max_cost:
            return False, (
                f"Cost estimate too high: {total_cost:.2f} > {max_cost}. "
                f"The query may cause a full table scan or cartesian product. "
                f"Consider adding filters or joins to reduce the result set."
            ), total_cost

        return True, cost_str, total_cost
    except Exception as e:
        # If EXPLAIN fails, it might be a syntax error that sqlglot missed
        return False, f"EXPLAIN failed: {e}", None


# --- Full Pipeline ---
# src/validation/__init__.py

async def validate_sql(
    sql: str,
    db: AsyncSession,
) -> ValidationResult:
    """
    Run the full validation pipeline. Returns ValidationResult.
    """
    errors = []
    original_sql = sql

    # Stage 1: Syntax
    valid, err_msg, ast = check_syntax(sql)
    if not valid:
        errors.append({
            "stage": ValidationStage.SYNTAX.value,
            "message": err_msg,
        })
        return ValidationResult(
            valid=False, sql=sql, original_sql=original_sql, errors=errors
        )

    # Stage 2: Safety
    valid, err_msg = check_safety(ast)
    if not valid:
        errors.append({
            "stage": ValidationStage.SAFETY.value,
            "message": err_msg,
        })
        return ValidationResult(
            valid=False, sql=sql, original_sql=original_sql, errors=errors
        )

    # Stage 3: LIMIT enforcement
    sql, ast = enforce_limit(ast, sql)
    # (This stage doesn't fail — it modifies the SQL)

    # Stage 4: Cost estimation
    valid, cost_msg, cost_val = await estimate_cost(db, sql)
    if not valid:
        errors.append({
            "stage": ValidationStage.COST.value,
            "message": cost_msg,
        })
        return ValidationResult(
            valid=False, sql=sql, original_sql=original_sql, errors=errors
        )

    # All checks passed
    return ValidationResult(
        valid=True,
        sql=sql,
        original_sql=original_sql,
        errors=[],
        cost_estimate=cost_msg,
        cost_value=cost_val,
    )
```

### 7.5 SQL Executor with Error Recovery (`src/tools/sql_executor.py`)

```python
"""
Executes validated SQL against the read-only database connection.

The executor wraps the validation pipeline and the actual query execution.
On failure, it returns a structured error message that the agent orchestrator
feeds back to the LLM for error recovery.

Error types are categorized so the LLM can reason about the fix:
  - SYNTAX_ERROR:    SQL syntax is invalid (shouldn't happen post-validation,
                     but DB may catch what sqlglot misses)
  - SAFETY_VIOLATION: SQL contains blocked operations
  - COST_TOO_HIGH:   Query is too expensive
  - RUNTIME_ERROR:   Query executes but fails (column not found, type mismatch,
                     division by zero, timeout, etc.)
  - TIMEOUT:         Query exceeded statement_timeout
"""

async def execute(
    db: AsyncSession,
    sql: str,
    explanation: str,
    attempt_num: int = 1,
) -> ExecutionResult:
    # 1. Validate
    validation = await validate_sql(sql, db)

    if not validation.valid:
        # Format errors for LLM consumption
        error_msgs = [e["message"] for e in validation.errors]
        return ExecutionResult(
            success=False,
            error_type=validation.errors[0]["stage"],
            error_message=" | ".join(error_msgs),
            sql=validation.original_sql,
            attempt_num=attempt_num,
        )

    # 2. Execute (using the possibly-modified SQL with LIMIT)
    try:
        start_time = time.monotonic()
        result = await db.execute(text(validation.sql))
        rows = result.fetchall()
        execution_ms = int((time.monotonic() - start_time) * 1000)

        # 3. Format results
        columns = list(result.keys())
        formatted_rows = [dict(zip(columns, row)) for row in rows]

        return ExecutionResult(
            success=True,
            sql=validation.sql,
            columns=columns,
            rows=formatted_rows,
            row_count=len(formatted_rows),
            execution_ms=execution_ms,
            cost_estimate=validation.cost_estimate,
            attempt_num=attempt_num,
        )

    except asyncio.TimeoutError:
        return ExecutionResult(
            success=False,
            error_type="TIMEOUT",
            error_message=(
                "Query timed out after 30 seconds. The query may be scanning "
                "a large table without adequate filtering. Try adding WHERE "
                "conditions or reducing the scope."
            ),
            sql=validation.sql,
            attempt_num=attempt_num,
        )
    except Exception as e:
        # Categorize the runtime error
        error_str = str(e)
        if "column" in error_str.lower() and "does not exist" in error_str.lower():
            error_type = "RUNTIME_ERROR"
            hint = "Check the column name against the schema. Use get_schema to verify."
        elif "relation" in error_str.lower() and "does not exist" in error_str.lower():
            error_type = "RUNTIME_ERROR"
            hint = "Check the table name. Use get_schema to see available tables."
        elif "function" in error_str.lower() and "does not exist" in error_str.lower():
            error_type = "RUNTIME_ERROR"
            hint = "The function name may be wrong. Check PostgreSQL documentation."
        elif "division by zero" in error_str.lower():
            error_type = "RUNTIME_ERROR"
            hint = "Add NULLIF or a CASE statement to handle zero denominators."
        else:
            error_type = "RUNTIME_ERROR"
            hint = ""

        return ExecutionResult(
            success=False,
            error_type=error_type,
            error_message=f"{error_str}. {hint}".strip(),
            sql=validation.sql,
            attempt_num=attempt_num,
        )
```

### 7.6 Chart Generator (`src/tools/chart_generator.py`)

```python
"""
Generates charts from SQL query results using matplotlib (static PNG)
or plotly (interactive HTML).

The agent specifies:
  - chart_type: bar, line, pie, scatter, table
  - title: chart title
  - x_column: column for x-axis (or labels for pie)
  - y_column: column for y-axis (or values for pie)
  - sql: the SQL to re-execute for fresh data

The generator re-executes the SQL (through the validation pipeline)
to get the data, then creates the appropriate chart.
"""

import matplotlib
matplotlib.use("Agg")  # Non-interactive backend for server-side rendering
import matplotlib.pyplot as plt
import plotly.graph_objects as go
from pathlib import Path

CHART_DIR = Path("/tmp/agent_charts")
CHART_DIR.mkdir(exist_ok=True)

async def generate(
    db: AsyncSession,
    chart_type: str,
    title: str,
    x_column: str,
    y_column: str,
    sql: str,
) -> ChartResult:
    # 1. Re-execute the SQL (with validation) to get fresh data
    exec_result = await sql_executor.execute(db, sql, explanation="chart data")
    if not exec_result.success:
        return ChartResult(
            success=False,
            error=f"Cannot generate chart: query failed — {exec_result.error_message}",
        )

    # 2. Extract x and y data
    rows = exec_result.rows
    x_data = [row[x_column] for row in rows]
    y_data = [row[y_column] for row in rows]

    # 3. Generate chart based on type
    chart_id = uuid4().hex[:8]
    png_path = CHART_DIR / f"{chart_id}.png"
    html_path = CHART_DIR / f"{chart_id}.html"

    if chart_type == "bar":
        _generate_bar(x_data, y_data, title, x_column, y_column, png_path)
    elif chart_type == "line":
        _generate_line(x_data, y_data, title, x_column, y_column, png_path)
    elif chart_type == "pie":
        _generate_pie(x_data, y_data, title, png_path)
    elif chart_type == "scatter":
        _generate_scatter(x_data, y_data, title, x_column, y_column, png_path)
    elif chart_type == "table":
        _generate_table(rows, title, png_path)
    else:
        return ChartResult(success=False, error=f"Unknown chart type: {chart_type}")

    # 4. Also generate interactive plotly version
    _generate_plotly(chart_type, x_data, y_data, title, x_column, y_column, html_path)

    return ChartResult(
        success=True,
        chart_path=str(png_path),
        chart_html_path=str(html_path),
        chart_type=chart_type,
    )


def _generate_bar(x, y, title, xlabel, ylabel, path):
    fig, ax = plt.subplots(figsize=(10, 6))
    ax.bar(x, y, color="#4C72B0")
    ax.set_title(title, fontsize=14, fontweight="bold")
    ax.set_xlabel(xlabel)
    ax.set_ylabel(ylabel)
    plt.xticks(rotation=45, ha="right")
    plt.tight_layout()
    plt.savefig(path, dpi=150)
    plt.close()


def _generate_line(x, y, title, xlabel, ylabel, path):
    fig, ax = plt.subplots(figsize=(10, 6))
    ax.plot(x, y, marker="o", color="#4C72B0", linewidth=2)
    ax.set_title(title, fontsize=14, fontweight="bold")
    ax.set_xlabel(xlabel)
    ax.set_ylabel(ylabel)
    plt.xticks(rotation=45, ha="right")
    plt.tight_layout()
    plt.savefig(path, dpi=150)
    plt.close()


def _generate_pie(labels, values, title, path):
    fig, ax = plt.subplots(figsize=(8, 8))
    ax.pie(values, labels=labels, autopct="%1.1f%%", startangle=90)
    ax.set_title(title, fontsize=14, fontweight="bold")
    plt.tight_layout()
    plt.savefig(path, dpi=150)
    plt.close()


def _generate_scatter(x, y, title, xlabel, ylabel, path):
    fig, ax = plt.subplots(figsize=(10, 6))
    ax.scatter(x, y, color="#4C72B0", alpha=0.6)
    ax.set_title(title, fontsize=14, fontweight="bold")
    ax.set_xlabel(xlabel)
    ax.set_ylabel(ylabel)
    plt.tight_layout()
    plt.savefig(path, dpi=150)
    plt.close()
```

### 7.7 Conversation Memory (`src/agent/memory.py`)

```python
"""
Session-scoped conversation memory.

Stores every message (user, assistant, tool) in the agent_meta.messages table.
On each new turn, loads the last N messages as context for the LLM.

Memory serves two purposes:
  1. CONTEXT: The LLM sees prior turns, enabling follow-up questions like
     "What about just German customers?" after "Top 5 customers by volume?"
  2. AUDIT: Every SQL query, tool call, and error is logged for debugging
     and compliance.
"""

CONTEXT_WINDOW = 20  # Max messages to include as LLM context

async def load(session_id: str, limit: int = CONTEXT_WINDOW) -> list[dict]:
    """Load recent messages as OpenAI-format chat messages."""
    rows = await db_meta.execute(text("""
        SELECT role, content, tool_name, tool_call_id
        FROM agent_meta.messages
        WHERE session_id = :sid
        ORDER BY created_at DESC
        LIMIT :limit
    """), {"sid": session_id, "limit": limit})

    messages = []
    for row in reversed(rows.fetchall()):
        msg = {"role": row.role, "content": row.content}
        if row.tool_call_id:
            msg["tool_call_id"] = row.tool_call_id
        messages.append(msg)

    return messages


async def append(
    session_id: str,
    role: str,
    content: str,
    tool_name: str | None = None,
    tool_call_id: str | None = None,
    sql_generated: str | None = None,
    row_count: int | None = None,
    execution_ms: int | None = None,
    attempt_num: int | None = None,
    error_message: str | None = None,
):
    """Append a message to the conversation history."""
    await db_meta.execute(text("""
        INSERT INTO agent_meta.messages
            (session_id, role, content, tool_name, tool_call_id,
             sql_generated, row_count, execution_ms, attempt_num, error_message)
        VALUES
            (:sid, :role, :content, :tool_name, :tool_call_id,
             :sql, :row_count, :exec_ms, :attempt, :error)
    """), {
        "sid": session_id, "role": role, "content": content,
        "tool_name": tool_name, "tool_call_id": tool_call_id,
        "sql": sql_generated, "row_count": row_count,
        "exec_ms": execution_ms, "attempt": attempt_num,
        "error": error_message,
    })
    await db_meta.commit()
```

### 7.8 System Prompt (`src/agent/prompts.py`)

```python
SYSTEM_PROMPT = """\
You are a data analysis agent connected to a PostgreSQL database containing \
the Northwind trading company's business data (customers, orders, products, \
employees, suppliers, categories, shippers).

Your job is to answer business questions by:
1. Exploring the database schema using the get_schema tool
2. Writing SQL queries using the execute_sql tool
3. Interpreting the results in clear, business-friendly language
4. Generating charts when the data would benefit from visualization

RULES:
- ALWAYS call get_schema before writing SQL, unless you already have the \
schema from a previous turn in this conversation.
- Write ONLY read-only SELECT queries. Never use INSERT, UPDATE, DELETE, \
DROP, CREATE, ALTER, or TRUNCATE.
- Write PostgreSQL-compatible SQL.
- If a query fails, read the error message carefully, fix the SQL, and \
retry. You have up to 3 retry attempts per query.
- If you cannot answer the question after 3 retries, explain what went \
wrong and suggest how the user might rephrase the question.
- When interpreting results, use business language, not technical jargon. \
Instead of "The query returned 5 rows," say "There are 5 products that \
meet your criteria."
- When results would benefit from visualization, call generate_chart with \
an appropriate chart type:
  - Bar chart: comparing categories (e.g., revenue by product category)
  - Line chart: trends over time (e.g., monthly sales)
  - Pie chart: proportions of a whole (e.g., market share by country)
  - Scatter plot: correlations (e.g., price vs. units sold)
- If the user asks a follow-up question, use the conversation context. \
For example, if they previously asked about "top customers" and now ask \
"what about just German ones?", modify the previous query rather than \
starting from scratch.
- Be transparent: always show the SQL you wrote and explain what it does.
- If the question is ambiguous, ask for clarification rather than guessing.

FORMAT YOUR FINAL ANSWER AS:
1. A direct answer to the question in 2-4 sentences.
2. The SQL query you used (in a code block).
3. A brief interpretation of the results.
4. A chart if applicable.
"""
```

---

## 8. Safety & Validation Layer

This is the core differentiator between a production-grade agent and a demo. The safety layer ensures that no matter what SQL the LLM generates, it cannot modify data, exhaust resources, or access forbidden functionality.

### 8.1 Safety Constraints Table

| # | Constraint | Enforcement Layer | Mechanism | What It Prevents |
|---|-----------|-------------------|-----------|------------------|
| S1 | **Read-only access** | Database (GRANT) | `data_agent_reader` user has only `SELECT` privilege; no `INSERT`/`UPDATE`/`DELETE`/`CREATE` | Any data modification, even if app validation is bypassed |
| S2 | **No DDL** | Validation (AST) + Database | sqlglot AST analysis blocks `CREATE`, `DROP`, `ALTER`, `TRUNCATE`; DB user lacks DDL privilege | Schema changes, table deletion |
| S3 | **No DML** | Validation (AST) + Database | sqlglot AST analysis blocks `INSERT`, `UPDATE`, `DELETE`, `MERGE`; DB user lacks DML privilege | Data corruption, data loss |
| S4 | **SELECT only** | Validation (AST) | Only `exp.Select` statement type allowed; `SELECT ... INTO` blocked | Table creation via SELECT INTO |
| S5 | **Statement timeout** | Database (ALTER USER) | `statement_timeout = 30s` set at user level | Long-running queries blocking resources |
| S6 | **Row limit** | Validation (AST rewrite) + App | LIMIT injected if missing (max 1000 rows); existing LIMIT capped at 1000 | Accidental full table scans returning millions of rows |
| S7 | **Cost guardrail** | Validation (EXPLAIN) | `EXPLAIN (FORMAT JSON)` → parse total cost → reject if > 10,000 | Expensive queries (cartesian products, full scans on large tables) |
| S8 | **No dangerous functions** | Validation (AST) | Blocked: `pg_read_file`, `pg_ls_dir`, `lo_import`, `dblink`, `pg_sleep`, `pg_terminate_backend`, etc. | Filesystem access, inter-database connections, DoS via sleep |
| S9 | **No temp tables** | Database (REVOKE) | `REVOKE TEMP ON DATABASE` from read-only user | Temp table abuse, disk space exhaustion |
| S10 | **Work memory limit** | Database (ALTER USER) | `work_mem = 64MB` per user | Memory exhaustion via complex sorts/hashes |
| S11 | **Max iterations** | Agent (orchestrator) | `MAX_ITERATIONS = 10` per user turn | Infinite reasoning loops, excessive LLM cost |
| S12 | **Max SQL retries** | Agent (orchestrator) | `MAX_SQL_RETRIES = 3` per failed query | Repeated failed attempts, excessive LLM cost |
| S13 | **No semicolons** | Validation (pre-parse) | Strip trailing semicolons; reject multiple statements | SQL injection via statement chaining |
| S14 | **Query audit log** | App (query_log table) | Every SQL (valid or invalid) logged with validation status, execution result, error | Forensic analysis, compliance, debugging |
| S15 | **Rate limiting** | App (slowapi) | Max 20 requests/minute per IP | DoS via rapid question flooding |

### 8.2 Defense in Depth

```
Layer 1: Agent Orchestrator
  ├── Max 10 iterations per turn (prevents infinite loops)
  ├── Max 3 SQL retries per failed query (prevents retry storms)
  └── System prompt enforces read-only intent

Layer 2: SQL Validation Pipeline
  ├── Stage 1: sqlglot syntax parse (catches invalid SQL before DB connection)
  ├── Stage 2: AST safety analysis (blocks DDL/DML/dangerous functions)
  ├── Stage 3: LIMIT enforcement (caps returned rows at 1000)
  └── Stage 4: EXPLAIN cost estimation (rejects expensive queries)

Layer 3: Database (PostgreSQL)
  ├── Read-only user (GRANT SELECT only — no DDL/DML possible)
  ├── statement_timeout = 30s (kills long queries)
  ├── work_mem = 64MB (limits per-query memory)
  ├── REVOKE TEMP (no temp table creation)
  └── No superuser access (least privilege)

Layer 4: Audit & Observability
  ├── Every SQL logged to agent_meta.query_log
  ├── Every tool call logged to agent_meta.messages
  └── Structured logging with tool call tracing
```

### 8.3 What Happens If Each Layer Fails

| Failure Scenario | Layer That Catches It | Result |
|-----------------|----------------------|--------|
| LLM generates `DROP TABLE customers` | Layer 2 (AST safety) rejects | SQL never reaches DB |
| LLM generates `DROP TABLE customers` AND validation has a bug | Layer 3 (DB user lacks DROP privilege) | DB returns permission denied |
| LLM generates `SELECT * FROM huge_table` (no LIMIT) | Layer 2 (LIMIT enforcer injects LIMIT 1000) | Only 1000 rows returned |
| LLM generates a 5-table cartesian join | Layer 2 (EXPLAIN cost > 10,000) | Query rejected with cost explanation |
| LLM generates `SELECT pg_sleep(300)` | Layer 2 (dangerous function blocked) | SQL rejected |
| LLM generates `SELECT pg_sleep(300)` AND validation misses it | Layer 3 (statement_timeout = 30s) | Query killed after 30s |
| Attacker floods the API with requests | Layer 4 (rate limiting) | Requests throttled |
| LLM enters infinite reasoning loop | Layer 1 (MAX_ITERATIONS = 10) | Loop terminates, graceful fallback message |

---

## 9. Error Recovery Loop

The error recovery loop is the agent's ability to fix its own mistakes. When `execute_sql` fails, the error message is returned to the LLM as a tool result. The LLM reads the error, reasons about the fix, and regenerates the SQL. This continues up to 3 retries.

### 9.1 Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Error Recovery Loop                             │
│                                                                     │
│  ┌──────────┐                                                      │
│  │  User    │                                                      │
│  │  Question│                                                      │
│  └────┬─────┘                                                      │
│       │                                                             │
│       ▼                                                             │
│  ┌─────────────────┐                                               │
│  │  Agent reasons  │                                               │
│  │  about schema   │                                               │
│  └────────┬────────┘                                               │
│           │                                                         │
│           ▼                                                         │
│  ┌─────────────────┐    Attempt 1                                  │
│  │  Generate SQL   │◄──────────────────────────────────────┐      │
│  └────────┬────────┘                                      │      │
│           │                                                │      │
│           ▼                                                │      │
│  ┌─────────────────┐                                      │      │
│  │  Validate SQL   │                                      │      │
│  │  (4 stages)     │                                      │      │
│  └────────┬────────┘                                      │      │
│           │                                                │      │
│       ┌───┴───┐                                            │      │
│       │       │                                            │      │
│    PASS       FAIL                                         │      │
│       │       │                                            │      │
│       ▼       ▼                                            │      │
│  ┌────────┐  ┌──────────────────┐                         │      │
│  │Execute │  │ Return error to  │                         │      │
│  │ Query  │  │ LLM as tool      │                         │      │
│  └───┬────┘  │ result           │                         │      │
│      │       └────────┬─────────┘                         │      │
│      │                │                                    │      │
│   ┌──┴──┐             │                                    │      │
│   │     │             │                                    │      │
│ OK      ERROR         │                                    │      │
│   │     │             │                                    │      │
│   │     ▼             │                                    │      │
│   │  ┌──────────────────────────┐                         │      │
│   │  │  LLM reads error message │                         │      │
│   │  │  & reasons about fix     │                         │      │
│   │  └────────────┬─────────────┘                         │      │
│   │               │                                        │      │
│   │               ▼                                        │      │
│   │        ┌──────────────┐                                │      │
│   │        │  Attempts <  │─── YES ───────────────────────┘      │
│   │        │  MAX_RETRIES │                                      │
│   │        │  (3)?        │                                      │
│   │        └──────┬───────┘                                      │
│   │               │ NO                                            │
│   │               ▼                                               │
│   │        ┌──────────────────┐                                  │
│   │        │  Return failure  │                                  │
│   │        │  to user with    │                                  │
│   │        │  explanation     │                                  │
│   │        └──────────────────┘                                  │
│   │                                                              │
│   ▼                                                              │
│  ┌─────────────────┐                                            │
│  │  LLM interprets │                                            │
│  │  results →      │                                            │
│  │  natural lang   │                                            │
│  └────────┬────────┘                                            │
│           │                                                      │
│           ▼                                                      │
│  ┌─────────────────┐                                            │
│  │  Generate chart │                                            │
│  │  (if useful)    │                                            │
│  └────────┬────────┘                                            │
│           │                                                      │
│           ▼                                                      │
│  ┌─────────────────┐                                            │
│  │  Return answer  │                                            │
│  │  to user        │                                            │
│  └─────────────────┘                                            │
└─────────────────────────────────────────────────────────────────────┘
```

### 9.2 Error Recovery Examples

| Attempt | SQL | Error | Fix Applied | Result |
|---------|-----|-------|-------------|--------|
| 1 | `SELECT product_name, SUM(quantity * unit_price) AS revenue FROM products p JOIN order_details od ON p.product_id = od.product_id GROUP BY product_name ORDER BY revenue DESC LIMIT 5` | `column "product_name" is ambiguous` (exists in both `products` and another joined table) | Add table alias: `p.product_name` | Passes |
| 1 | `SELECT customer_name, COUNT(*) FROM customers c JOIN orders o ON c.customer_id = o.customer_id GROUP BY customer_name` | `column "customer_name" does not exist` | Check schema: column is `company_name`, not `customer_name` | Passes |
| 1 | `SELECT EXTRACT(MONTH FROM order_date) AS month, SUM(quantity * unit_price) AS revenue FROM orders o JOIN order_details od ON o.order_id = od.order_id GROUP BY month ORDER BY month` | `column "o.order_date" must appear in GROUP BY` | Add `order_date` to GROUP BY or use the alias correctly | Passes |
| 1 | `SELECT * FROM orders WHERE order_date BETWEEN '2023-01-01' AND '2023-12-31'` | (No error — but no LIMIT) | LIMIT enforcer injects `LIMIT 1000` | Passes with LIMIT |
| 1 | `SELECT c.company_name, p.product_name, SUM(od.quantity * od.unit_price) AS total FROM customers c, orders o, order_details od, products p WHERE c.customer_id = o.customer_id AND o.order_id = od.order_id AND od.product_id = p.product_id GROUP BY c.company_name, p.product_name` | `Cost estimate too high: 15432.50 > 10000.0` (implicit cross join before filters) | Rewrite with explicit JOINs and add a filter (e.g., top 10 customers) | Passes |

---

## 10. API Specification

### 10.1 `POST /chat`

Send a message to the data analysis agent. Returns the agent's response with SQL, results, chart, and reasoning trace.

```
Body:
{
  "message": "What are the top 5 products by total revenue?",
  "session_id": "uuid"              // optional; creates new session if omitted
}

Response 200:
{
  "session_id": "uuid",
  "answer": "The top 5 products by total revenue are: \
             1. Côte de Blaye ($141,396), 2. Thüringer Rostbratwurst ($80,368), \
             3. Raclette Courdavault ($71,190), 4. Tarte au sucre ($47,840), \
             5. Camembert Pierrot ($46,597). Côte de Blaye dominates with \
             nearly double the revenue of the second product.",
  "sql": "SELECT p.product_name, SUM(od.quantity * od.unit_price) AS total_revenue \
          FROM products p JOIN order_details od ON p.product_id = od.product_id \
          GROUP BY p.product_name ORDER BY total_revenue DESC LIMIT 5",
  "results": [
    {"product_name": "Côte de Blaye", "total_revenue": 141396.00},
    {"product_name": "Thüringer Rostbratwurst", "total_revenue": 80368.00},
    ...
  ],
  "chart_path": "/tmp/agent_charts/a1b2c3d4.png",
  "chart_type": "bar",
  "reasoning_trace": [
    {"step": 1, "type": "tool_call", "tool": "get_schema", "args": {}},
    {"step": 2, "type": "tool_call", "tool": "execute_sql",
     "args": {"sql": "SELECT p.product_name, ...", "explanation": "..."}},
    {"step": 3, "type": "tool_call", "tool": "generate_chart",
     "args": {"chart_type": "bar", "title": "Top 5 Products by Revenue", ...}},
    {"step": 4, "type": "final_answer", "content": "The top 5 products..."}
  ],
  "latency_ms": 3420
}

Response 200 (error recovery — SQL failed and was fixed):
{
  "session_id": "uuid",
  "answer": "The top 5 products by total revenue are...",
  "sql": "SELECT p.product_name, SUM(od.quantity * od.unit_price) AS total_revenue \
          FROM products p JOIN order_details od ON p.product_id = od.product_id \
          GROUP BY p.product_name ORDER BY total_revenue DESC LIMIT 5",
  "results": [...],
  "reasoning_trace": [
    {"step": 1, "type": "tool_call", "tool": "get_schema", "args": {}},
    {"step": 2, "type": "tool_call", "tool": "execute_sql",
     "args": {"sql": "SELECT product_name, SUM(...) ...", "explanation": "..."},
     "error": "column 'product_name' is ambiguous"},
    {"step": 3, "type": "tool_call", "tool": "execute_sql",
     "args": {"sql": "SELECT p.product_name, SUM(...) ...", "explanation": "Fixed: added table alias"},
     "success": true},
    {"step": 4, "type": "final_answer", "content": "..."}
  ],
  "latency_ms": 5180
}
```

### 10.2 `GET /schema`

View the database schema (same as what the agent sees).

```
Query params:
  ?table=orders              // optional: filter to specific table

Response 200:
{
  "tables": [
    {
      "table_name": "orders",
      "approx_rows": 830,
      "columns": [
        {"name": "order_id", "type": "integer", "nullable": false, "default": null},
        {"name": "customer_id", "type": "character varying", "nullable": false, "default": null},
        ...
      ],
      "foreign_keys": [
        {"column": "customer_id", "references_table": "customers", "references_column": "customer_id"},
        {"column": "employee_id", "references_table": "employees", "references_column": "employee_id"}
      ]
    }
  ]
}
```

### 10.3 `GET /sessions`

List conversation sessions.

```
Response 200:
{
  "sessions": [
    {
      "id": "uuid",
      "title": "Top products by revenue",
      "created_at": "2025-01-15T10:30:00Z",
      "updated_at": "2025-01-15T10:35:00Z",
      "message_count": 6
    }
  ]
}
```

### 10.4 `GET /sessions/{id}`

Get full conversation history for a session.

```
Response 200:
{
  "session": {
    "id": "uuid",
    "title": "Top products by revenue",
    "created_at": "..."
  },
  "messages": [
    {"role": "user", "content": "What are the top 5 products by revenue?", "created_at": "..."},
    {"role": "assistant", "content": null, "tool_calls": [...]},
    {"role": "tool", "tool_name": "get_schema", "content": "{...}"},
    {"role": "assistant", "content": null, "tool_calls": [...]},
    {"role": "tool", "tool_name": "execute_sql", "content": "{...}"},
    {"role": "assistant", "content": "The top 5 products by total revenue are..."}
  ]
}
```

### 10.5 `GET /health`

```
Response 200:
{
  "status": "healthy",
  "database": "connected",
  "readonly_user": "active",
  "llm": "available",
  "version": "1.0.0"
}
```

---

## 11. Testing Strategy

### 11.1 Test Categories

| Category | What It Proves | Priority |
|----------|---------------|----------|
| **Safety tests** | No DDL/DML reaches the database; validation catches all dangerous SQL | P0 — must pass before any merge |
| **Validation tests** | Each validation stage works correctly (syntax, safety, LIMIT, cost) | P0 |
| **Agent loop tests** | Agent selects correct tools, recovers from errors, respects limits | P0 |
| **Integration tests** | Full pipeline: question → schema → SQL → validate → execute → interpret → chart | P1 |
| **API tests** | HTTP endpoints return correct status codes + payloads | P1 |
| **Unit tests** | Individual components (schema introspection, chart generation, memory) | P2 |

### 11.2 Critical Safety Tests (`tests/test_safety_integration.py`)

```python
"""
These tests verify that the safety layer is impenetrable.
If ANY of these fail, the system is not safe to deploy.
"""

import pytest
from src.validation import validate_sql


async def test_drop_table_rejected_by_validation():
    """DROP TABLE must be caught by AST safety check before reaching DB."""
    result = await validate_sql("DROP TABLE customers", db)
    assert not result.valid
    assert any(e["stage"] == "safety" for e in result.errors)


async def test_delete_rejected_by_validation():
    """DELETE must be caught by AST safety check."""
    result = await validate_sql("DELETE FROM orders WHERE 1=1", db)
    assert not result.valid
    assert any(e["stage"] == "safety" for e in result.errors)


async def test_insert_rejected_by_validation():
    """INSERT must be caught by AST safety check."""
    result = await validate_sql(
        "INSERT INTO customers (customer_id) VALUES ('TEST')", db
    )
    assert not result.valid
    assert any(e["stage"] == "safety" for e in result.errors)


async def test_update_rejected_by_validation():
    """UPDATE must be caught by AST safety check."""
    result = await validate_sql(
        "UPDATE customers SET company_name = 'HACKED' WHERE customer_id = 'ALFKI'", db
    )
    assert not result.valid
    assert any(e["stage"] == "safety" for e in result.errors)


async def test_select_into_rejected():
    """SELECT ... INTO creates a table — must be blocked."""
    result = await validate_sql(
        "SELECT * INTO temp_customers FROM customers LIMIT 10", db
    )
    assert not result.valid
    assert any(e["stage"] == "safety" for e in result.errors)


async def test_dangerous_function_blocked():
    """pg_read_file can read filesystem — must be blocked."""
    result = await validate_sql(
        "SELECT pg_read_file('/etc/passwd')", db
    )
    assert not result.valid
    assert any(e["stage"] == "safety" for e in result.errors)


async def test_pg_sleep_blocked():
    """pg_sleep can cause DoS — must be blocked."""
    result = await validate_sql("SELECT pg_sleep(300)", db)
    assert not result.valid
    assert any(e["stage"] == "safety" for e in result.errors)


async def test_dblink_blocked():
    """dblink can connect to other databases — must be blocked."""
    result = await validate_sql(
        "SELECT * FROM dblink('dbname=postgres', 'SELECT 1') AS t(a int)", db
    )
    assert not result.valid
    assert any(e["stage"] == "safety" for e in result.errors)


async def test_limit_injected_when_missing():
    """SELECT without LIMIT must get LIMIT 1000 injected."""
    result = await validate_sql("SELECT * FROM orders", db)
    assert result.valid
    assert "LIMIT" in result.sql.upper()
    assert "1000" in result.sql


async def test_limit_capped_when_too_high():
    """LIMIT > 1000 must be capped at 1000."""
    result = await validate_sql("SELECT * FROM orders LIMIT 100000", db)
    assert result.valid
    assert "1000" in result.sql
    assert "100000" not in result.sql


async def test_drop_table_rejected_by_db_user():
    """Even if validation has a bug, the DB user cannot DROP."""
    # Bypass validation and try to execute directly as readonly user
    with pytest.raises(Exception) as exc_info:
        await readonly_db.execute(text("DROP TABLE customers"))
    assert "permission denied" in str(exc_info.value).lower()


async def test_statement_timeout_enforced():
    """Query exceeding 30s must be killed by statement_timeout."""
    # pg_sleep is blocked by validation, but if we bypass it:
    # This tests the DB-level timeout directly
    with pytest.raises(Exception) as exc_info:
        await readonly_db.execute(text(
            "SELECT count(*) FROM generate_series(1, 1000000000)"
        ))
    # Should timeout, not complete
    assert "timeout" in str(exc_info.value).lower() or \
           "canceling statement" in str(exc_info.value).lower()


async def test_multiple_statements_rejected():
    """SQL injection via semicolons must be blocked."""
    result = await validate_sql(
        "SELECT * FROM customers; DROP TABLE customers", db
    )
    assert not result.valid


async def test_truncate_rejected():
    """TRUNCATE must be caught by AST safety check."""
    result = await validate_sql("TRUNCATE TABLE orders", db)
    assert not result.valid
    assert any(e["stage"] == "safety" for e in result.errors)
```

### 11.3 Agent Loop Tests (`tests/test_agent_orchestrator.py`)

```python
"""
Tests for the agent orchestrator's ReAct loop and error recovery.
Uses mocked LLM responses to test specific scenarios.
"""

from unittest.mock import AsyncMock, patch


async def test_agent_calls_get_schema_first():
    """Agent must call get_schema before execute_sql."""
    mock_llm = MockLLM([
        # Step 1: LLM decides to call get_schema
        MockResponse(tool_calls=[MockToolCall("get_schema", {})]),
        # Step 2: LLM decides to call execute_sql
        MockResponse(tool_calls=[MockToolCall("execute_sql", {
            "sql": "SELECT COUNT(*) FROM customers",
            "explanation": "Count customers"
        })]),
        # Step 3: LLM gives final answer
        MockResponse(content="There are 91 customers in the database."),
    ])

    result = await run_agent("How many customers?", "test-session", db, db_meta)

    assert mock_llm.call_count == 3
    assert mock_llm.tool_calls[0].name == "get_schema"
    assert mock_llm.tool_calls[1].name == "execute_sql"
    assert "91" in result.answer


async def test_agent_recovers_from_sql_error():
    """Agent receives SQL error, fixes it, and succeeds on retry."""
    mock_llm = MockLLM([
        # Step 1: get_schema
        MockResponse(tool_calls=[MockToolCall("get_schema", {})]),
        # Step 2: execute_sql with ambiguous column
        MockResponse(tool_calls=[MockToolCall("execute_sql", {
            "sql": "SELECT product_name, SUM(quantity * unit_price) AS rev "
                   "FROM products p JOIN order_details od ON p.product_id = od.product_id "
                   "GROUP BY product_name ORDER BY rev DESC LIMIT 5",
            "explanation": "Top 5 products by revenue"
        })]),
        # Step 3: LLM sees error, fixes SQL with table alias
        MockResponse(tool_calls=[MockToolCall("execute_sql", {
            "sql": "SELECT p.product_name, SUM(od.quantity * od.unit_price) AS rev "
                   "FROM products p JOIN order_details od ON p.product_id = od.product_id "
                   "GROUP BY p.product_name ORDER BY rev DESC LIMIT 5",
            "explanation": "Fixed: added table alias p. to product_name"
        })]),
        # Step 4: Final answer
        MockResponse(content="The top 5 products by revenue are..."),
    ])

    result = await run_agent("Top 5 products by revenue?", "test-session", db, db_meta)

    assert result.answer
    assert len(result.reasoning_trace) == 4
    # Verify error recovery happened
    tool_calls = [s for s in result.reasoning_trace if s["type"] == "tool_call"]
    assert tool_calls[1]["tool"] == "execute_sql"
    assert tool_calls[2]["tool"] == "execute_sql"  # retry


async def test_agent_gives_up_after_max_retries():
    """After 3 failed SQL attempts, agent should explain the failure."""
    mock_llm = MockLLM([
        MockResponse(tool_calls=[MockToolCall("get_schema", {})]),
        # 3 failed attempts with the same broken SQL
        MockResponse(tool_calls=[MockToolCall("execute_sql", {
            "sql": "SELECT nonexistent_column FROM customers",
            "explanation": "..."
        })]),
        MockResponse(tool_calls=[MockToolCall("execute_sql", {
            "sql": "SELECT nonexistent_column FROM customers",
            "explanation": "retry"
        })]),
        MockResponse(tool_calls=[MockToolCall("execute_sql", {
            "sql": "SELECT nonexistent_column FROM customers",
            "explanation": "retry 2"
        })]),
        # Final answer explaining failure
        MockResponse(content="I wasn't able to answer this question after 3 attempts..."),
    ])

    result = await run_agent("Query that always fails", "test-session", db, db_meta)

    assert "3 attempts" in result.answer or "unable" in result.answer.lower()


async def test_agent_respects_max_iterations():
    """Agent must stop after MAX_ITERATIONS and return a fallback message."""
    # LLM that always calls get_schema, never gives a final answer
    mock_llm = MockLLM([
        MockResponse(tool_calls=[MockToolCall("get_schema", {})])
    ] * 15)  # More than MAX_ITERATIONS (10)

    result = await run_agent("Infinite loop test", "test-session", db, db_meta)

    assert "step limit" in result.answer.lower() or "wasn't able" in result.answer.lower()
    assert len(result.reasoning_trace) <= 10
```

### 11.4 Test Fixtures (`tests/conftest.py`)

```python
"""
Test setup:
  - Spin up a test PostgreSQL with Northwind data
  - Create read-only user
  - Provide async DB sessions (readonly + meta)
  - Provide FastAPI TestClient
"""

import pytest
import pytest_asyncio
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession

TEST_DB_URL = "postgresql+asyncpg://data_agent_reader:testpass@localhost:5432/northwind_test"
META_DB_URL = "postgresql+asyncpg://test_admin:testpass@localhost:5432/northwind_test"


@pytest_asyncio.fixture
async def readonly_db():
    """Read-only DB session (same as what the agent uses)."""
    engine = create_async_engine(TEST_DB_URL)
    async with AsyncSession(engine) as session:
        yield session
    await engine.dispose()


@pytest_asyncio.fixture
async def meta_db():
    """Metadata DB session (for agent_meta tables)."""
    engine = create_async_engine(META_DB_URL)
    async with AsyncSession(engine) as session:
        # Clean agent_meta tables before each test
        await session.execute(text("DELETE FROM agent_meta.messages"))
        await session.execute(text("DELETE FROM agent_meta.sessions"))
        await session.execute(text("DELETE FROM agent_meta.query_log"))
        await session.commit()
        yield session
    await engine.dispose()


@pytest.fixture
def sample_questions():
    """Business questions for end-to-end testing."""
    return [
        "What are the top 5 products by total revenue?",
        "How many orders did each employee process in 2023?",
        "What's the revenue breakdown by product category?",
        "Show me monthly sales trends for the most recent year.",
        "Which 10 customers have the highest total order value?",
        "What is the average order value by country?",
        "Which products have low stock (less than 10 units)?",
        "What's the revenue difference between Q1 and Q4?",
        "Which suppliers provide the most products?",
        "What is the average discount by product category?",
    ]
```

---

## 12. Deployment

### 12.1 `docker-compose.yml`

```yaml
version: "3.9"

services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: northwind
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-changeme}
      POSTGRES_READ_ONLY_PASSWORD: ${DB_READONLY_PASSWORD:-readonlypass}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./data/northwind/northwind_schema.sql:/docker-entrypoint-initdb.d/01_schema.sql
      - ./data/northwind/northwind_data.sql:/docker-entrypoint-initdb.d/02_data.sql
      - ./docker/postgres/init.sql:/docker-entrypoint-initdb.d/03_init.sql
      - ./scripts/init_readonly_user.sql:/docker-entrypoint-initdb.d/04_readonly_user.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d northwind"]
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
      DATABASE_READONLY_URL: postgresql+asyncpg://data_agent_reader:${DB_READONLY_PASSWORD:-readonlypass}@postgres:5432/northwind
      DATABASE_META_URL: postgresql+asyncpg://admin:${POSTGRES_PASSWORD:-changeme}@postgres:5432/northwind
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      LLM_MODEL: gpt-4o-mini
      MAX_ROWS: "1000"
      MAX_COST: "10000"
      STATEMENT_TIMEOUT: "30s"
      LOG_LEVEL: info
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 10s
      timeout: 5s
      retries: 3

  ui:
    build:
      context: .
      dockerfile: docker/Dockerfile.ui
    ports:
      - "8501:8501"
    environment:
      API_URL: http://api:8000
    depends_on:
      api:
        condition: service_healthy

volumes:
  postgres_data:
```

### 12.2 `docker/Dockerfile.api`

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install system deps for matplotlib and PostgreSQL
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential libpq-dev curl libffi-dev \
    && rm -rf /var/lib/apt/lists/*

# Install Python deps
COPY pyproject.toml .
RUN pip install --no-cache-dir -e .

# Copy source
COPY src/ ./src/
COPY scripts/ ./scripts/
COPY data/ ./data/

# Create chart output directory
RUN mkdir -p /tmp/agent_charts

EXPOSE 8000

CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 12.3 `docker/Dockerfile.ui`

```dockerfile
FROM python:3.12-slim

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential libpq-dev \
    && rm -rf /var/lib/apt/lists/*

COPY pyproject.toml .
RUN pip install --no-cache-dir streamlit httpx plotly matplotlib

COPY src/ui/ ./src/ui/

EXPOSE 8501

CMD ["streamlit", "run", "src/ui/streamlit_app.py", "--server.port=8501", "--server.address=0.0.0.0"]
```

### 12.4 `docker/postgres/init.sql`

```sql
-- Create agent_meta schema (for conversation persistence)
CREATE SCHEMA IF NOT EXISTS agent_meta;

-- Enable gen_random_uuid (pgcrypto or pg16 built-in)
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Agent metadata tables (created by app on startup or scripts/seed_agent_meta.py)
```

### 12.5 `.env.example`

```env
# Database
POSTGRES_PASSWORD=changeme-in-production
DB_READONLY_PASSWORD=readonlypass-in-production

# OpenAI
OPENAI_API_KEY=sk-...
LLM_MODEL=gpt-4o-mini

# Alternative: Local LLM via Ollama
# OPENAI_API_KEY=ollama
# OPENAI_BASE_URL=http://localhost:11434/v1
# LLM_MODEL=llama3.1:70b

# Safety limits
MAX_ROWS=1000
MAX_COST=10000
STATEMENT_TIMEOUT=30s

# Agent limits
MAX_ITERATIONS=10
MAX_SQL_RETRIES=3

# Logging
LOG_LEVEL=info
```

### 12.6 Production Considerations

| Concern | Recommendation |
|---------|---------------|
| **TLS** | Terminate TLS at reverse proxy (nginx/traefik); API and UI listen on HTTP |
| **DB backups** | `pg_dump` cron job; Northwind is sample data but agent_meta contains conversation history |
| **Secrets** | Use Docker secrets or a vault; never bake API keys into images |
| **DB user privileges** | `data_agent_reader` has SELECT only; app backend uses separate user for agent_meta writes |
| **Rate limiting** | slowapi middleware (20 requests/min per IP) |
| **Monitoring** | Structured JSON logs → Loki/ELK; `/health` endpoint for k8s probes |
| **LLM cost control** | Track token usage per session; alert if cost exceeds threshold |
| **Chart storage** | v1 uses /tmp; v2 should use S3 with lifecycle policies |
| **Session cleanup** | Cron job to archive sessions older than 30 days |

---

## 13. Roadmap & Milestones

```
Week 1 (Days 1-5)
├── Phase 1: Foundation + DB + Read-Only User  [██░░░░░░░░] Days 1-2
├── Phase 2: Schema Introspection Tool         [░░██░░░░░░] Day 3
└── Phase 3: SQL Validation Layer              [░░░░██░░░░] Days 4-5

Week 2 (Days 6-10)
├── Phase 4: SQL Execution + Error Recovery    [░░░░░░██░░] Days 6-7
├── Phase 5: Interpretation + Charts           [░░░░░░░░██] Day 8
├── Phase 6: Memory + Streamlit UI             [░░░░░░░░██] Day 9  (parallel with Phase 5)
└── Phase 7: Hardening + Testing + Docs        [░░░░░░░░░█] Day 10

Future (Post-v1)
├── Multi-database support (MySQL, Snowflake, BigQuery via sqlglot transpilation)
├── Write operations with approval workflow (agent proposes, human approves)
├── Saved queries + query library
├── Scheduled reports (agent runs queries on a schedule, emails results)
├── Multi-user with authentication + per-user query history
├── Streaming responses (SSE for real-time reasoning display)
├── Fine-tuned SQL generation model for domain-specific schemas
├── Natural language to dbt model selection (leverage dbt metrics)
├── Query result caching (identical SQL → cached response)
└── Kubernetes deployment manifests
```

### Milestone Summary

| Milestone | Day | Deliverable |
|-----------|-----|-------------|
| M1: Foundation | Day 2 | Docker Compose up, Northwind loaded, read-only user active, health check passes |
| M2: Schema introspection | Day 3 | `GET /schema` returns full Northwind schema with FKs and sample values |
| M3: Validation layer | Day 5 | All 4 validation stages working; safety test suite green (no DDL/DML passes) |
| M4: Agent loop + error recovery | Day 7 | Agent answers questions autonomously; recovers from SQL errors within 3 retries |
| M5: Charts + interpretation | Day 8 | Agent generates appropriate charts and prose summaries |
| M6: Full chat UI | Day 9 | Streamlit chat interface with multi-turn conversations, SQL display, charts |
| M7: Production-ready | Day 10 | Full test suite green, documented, 20+ questions tested manually, deployable |

---

## 14. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| **LLM generates SQL injection / DDL** | High | Critical | Multi-layer defense: AST safety check + DB read-only user. Either layer alone is sufficient; both together provide defense in depth. Safety test suite verifies both layers. |
| **LLM generates syntactically invalid SQL** | High | Low | sqlglot parse catches syntax errors before DB connection; error message fed back to LLM for self-correction (max 3 retries) |
| **LLM hallucinates column/table names** | Medium | Medium | Schema introspection provides ground truth; error recovery loop catches `column does not exist` errors and feeds them back to LLM |
| **Expensive query causes resource exhaustion** | Medium | High | EXPLAIN cost estimation rejects queries above threshold; statement_timeout = 30s kills long queries; work_mem = 64MB limits memory |
| **LLM enters infinite reasoning loop** | Low | Medium | MAX_ITERATIONS = 10 hard cap; agent returns fallback message if limit reached |
| **Prompt injection via user input** | Medium | High | User input is in the "user" role message only; system prompt is immutable and sets strict rules; validation layer catches dangerous SQL regardless of prompt |
| **OpenAI API downtime / rate limits** | Low | High | Implement retry with exponential backoff; fallback to local Llama 3.1 via Ollama; graceful error message to user |
| **LLM cost exceeds budget** | Medium | Medium | Track token usage per session; MAX_ITERATIONS and MAX_SQL_RETRIES cap reasoning steps; consider gpt-4o-mini over gpt-4o for cost |
| **Northwind dataset too small for meaningful cost testing** | High | Low | Generate synthetic large tables (1M+ rows) for cost estimation testing; adjust MAX_COST threshold based on real-world schema |
| **Chart generation fails for unexpected data types** | Medium | Low | Wrap chart generation in try/except; return text-only answer if chart fails; log failure for debugging |
| **Conversation memory grows unbounded** | High | Low | Load only last 20 messages as LLM context; archive old sessions; add retention policy (30 days) |
| **Multiple semicolons allow statement chaining** | Low | Critical | Pre-parse strips semicolons; sqlglot rejects multiple statements; DB user lacks privileges for non-SELECT statements |
| **Agent leaks sensitive data in responses** | Low | High | Read-only user has no access to pg_catalog system tables; sample values limited to 5 per column; agent prompt instructs business-language responses |

---

## Appendix A: Quick Start

```bash
# 1. Clone
git clone <repo-url> data-analysis-agent
cd data-analysis-agent

# 2. Configure
cp .env.example .env
# Edit .env: set OPENAI_API_KEY, POSTGRES_PASSWORD, DB_READONLY_PASSWORD

# 3. Start all services
docker compose up -d

# 4. Wait for health
curl http://localhost:8000/health
# Expected: {"status": "healthy", "database": "connected", "readonly_user": "active", ...}

# 5. Open the chat UI
open http://localhost:8501
# Or use the API directly:

# 6. Ask a question via API
curl -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "What are the top 5 products by total revenue?"}'

# 7. Ask a follow-up (multi-turn)
curl -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "What about just German customers?", "session_id": "<from step 6>"}'

# 8. View the database schema
curl http://localhost:8000/schema

# 9. View conversation history
curl http://localhost:8000/sessions
curl http://localhost:8000/sessions/<session_id>

# 10. Run tests
docker compose exec api pytest tests/ -v
```

---

## Appendix B: Key Design Decisions

| Decision | Chosen | Rejected Alternative | Why |
|----------|--------|----------------------|-----|
| Agent loop | Raw OpenAI function calling (ReAct loop) | LangChain AgentExecutor | Full control over error recovery, retry logic, and logging; no abstraction leaks; smaller codebase; easier to debug |
| SQL validation | sqlglot AST analysis + EXPLAIN | Regex-based filtering | AST analysis is structural (can't be bypassed with clever syntax); regex is fragile and misses edge cases. EXPLAIN adds cost-based guardrail |
| Read-only safety | DB-level GRANT SELECT only | App-level check only | DB-level is the hard boundary — even if app has a bug, DB rejects writes. App-level is the first line (better UX with error messages) |
| LIMIT enforcement | AST rewrite (inject LIMIT if missing) | Post-processing (fetch all, truncate in app) | AST rewrite prevents the DB from scanning the full table; app-level truncation still wastes DB resources |
| Cost estimation | EXPLAIN (FORMAT JSON) + parse total cost | Query timeout only | EXPLAIN catches expensive queries before they start; timeout kills them mid-execution (wasteful). Both together = defense in depth |
| Chart library | matplotlib (PNG) + plotly (HTML) | Only one | matplotlib is reliable for static PNGs in Streamlit; plotly adds interactivity (hover, zoom). Both are industry standard |
| Sample data | Northwind | Synthetic data, Chinook, Sakila | Northwind is the classic business DB: realistic schema with FKs, recognizable entities (customers, orders, products), and manageable size (~3000 rows) |
| Conversation memory | PostgreSQL agent_meta schema | Redis, in-memory only | PostgreSQL persists across restarts; queryable via SQL; no additional service to run. Redis is faster but adds operational complexity |
| UI framework | Streamlit | Gradio, custom React | Streamlit has built-in chat components, renders charts natively, and requires minimal frontend code. Gradio is simpler but less flexible; React is overkill for v1 |
| LLM model | gpt-4o-mini | gpt-4o, Llama 3.1 70B | gpt-4o-mini is fast, cheap, and strong at SQL generation + function calling. gpt-4o is better but 10x more expensive. Llama 3.1 is the on-prem fallback |
| Schema introspection | information_schema + pg_class + sample queries | SQLAlchemy inspect() only | information_schema gives us FKs and column metadata directly; pg_class.reltuples gives fast row count estimates; sample values help the LLM understand data content |
| Error message format | Structured (type + message + hint) | Raw exception string | Structured messages help the LLM reason about fixes (e.g., "column does not exist → check schema"); raw strings are harder for the LLM to interpret |
| Max retries | 3 | 5, 10 | 3 is enough for most fixable errors (ambiguous column, wrong name, missing alias); more retries waste LLM tokens on unfixable errors |
| Max iterations | 10 | 5, 20 | 10 allows: get_schema + 3 SQL attempts + chart + interpretation + buffer. Too few → agent can't complete complex questions. Too many → cost and latency |

---

## Appendix C: Sample Conversation Trace

This trace shows the agent's full multi-step reasoning for a complex business question, including an error recovery cycle. The trace is formatted as the `reasoning_trace` array returned by the `POST /chat` endpoint.

### User Question

> "Which product categories had the biggest revenue drop from Q3 to Q4 in the most recent year, and can you show me a chart?"

### Agent Reasoning Trace

```json
[
  {
    "step": 1,
    "type": "tool_call",
    "tool": "get_schema",
    "args": {},
    "reasoning": "I need to understand the database schema before writing SQL. Let me get all tables, their columns, and foreign key relationships."
  },
  {
    "step": 1,
    "type": "tool_result",
    "tool": "get_schema",
    "result": {
      "tables": [
        {
          "table_name": "categories",
          "approx_rows": 8,
          "columns": [
            {"name": "category_id", "type": "integer", "nullable": false},
            {"name": "category_name", "type": "character varying", "nullable": false,
             "sample_values": ["Beverages", "Condiments", "Confections", "Dairy Products", "Grains/Cereals"]},
            {"name": "description", "type": "text", "nullable": true}
          ],
          "foreign_keys": []
        },
        {
          "table_name": "orders",
          "approx_rows": 830,
          "columns": [
            {"name": "order_id", "type": "integer", "nullable": false},
            {"name": "customer_id", "type": "character varying", "nullable": false},
            {"name": "employee_id", "type": "integer", "nullable": false},
            {"name": "order_date", "type": "date", "nullable": false},
            {"name": "ship_country", "type": "character varying", "nullable": false}
          ],
          "foreign_keys": [
            {"column": "customer_id", "references_table": "customers", "references_column": "customer_id"},
            {"column": "employee_id", "references_table": "employees", "references_column": "employee_id"}
          ]
        },
        {
          "table_name": "order_details",
          "approx_rows": 2155,
          "columns": [
            {"name": "order_id", "type": "integer", "nullable": false},
            {"name": "product_id", "type": "integer", "nullable": false},
            {"name": "unit_price", "type": "numeric", "nullable": false},
            {"name": "quantity", "type": "smallint", "nullable": false},
            {"name": "discount", "type": "real", "nullable": false}
          ],
          "foreign_keys": [
            {"column": "order_id", "references_table": "orders", "references_column": "order_id"},
            {"column": "product_id", "references_table": "products", "references_column": "product_id"}
          ]
        },
        {
          "table_name": "products",
          "approx_rows": 77,
          "columns": [
            {"name": "product_id", "type": "integer", "nullable": false},
            {"name": "product_name", "type": "character varying", "nullable": false},
            {"name": "category_id", "type": "integer", "nullable": false},
            {"name": "unit_price", "type": "numeric", "nullable": false}
          ],
          "foreign_keys": [
            {"column": "category_id", "references_table": "categories", "references_column": "category_id"}
          ]
        }
      ]
    }
  },
  {
    "step": 2,
    "type": "reasoning",
    "content": "To answer this question, I need to: 1) Find the most recent year in the orders data. 2) Calculate Q3 and Q4 revenue by category. 3) Compute the difference. 4) Sort by biggest drop. I need to join orders → order_details → products → categories, filter by date quarters, group by category, and compare. Let me first find the most recent year."
  },
  {
    "step": 3,
    "type": "tool_call",
    "tool": "execute_sql",
    "args": {
      "sql": "SELECT EXTRACT(YEAR FROM order_date) AS year, COUNT(*) AS order_count FROM orders GROUP BY year ORDER BY year DESC LIMIT 5",
      "explanation": "Find the most recent years in the data to determine which year to analyze."
    }
  },
  {
    "step": 3,
    "type": "tool_result",
    "tool": "execute_sql",
    "result": {
      "success": true,
      "columns": ["year", "order_count"],
      "rows": [
        {"year": 2024, "order_count": 152},
        {"year": 2023, "order_count": 270},
        {"year": 2022, "order_count": 270},
        {"year": 2021, "order_count": 138}
      ],
      "row_count": 4,
      "execution_ms": 12
    }
  },
  {
    "step": 4,
    "type": "reasoning",
    "content": "The most recent year is 2024. Now I need to calculate Q3 (Jul-Sep) and Q4 (Oct-Dec) revenue by category for 2024. Revenue = SUM(quantity * unit_price * (1 - discount)). I need to join orders, order_details, products, and categories."
  },
  {
    "step": 5,
    "type": "tool_call",
    "tool": "execute_sql",
    "args": {
      "sql": "SELECT c.category_name, EXTRACT(QUARTER FROM o.order_date) AS quarter, SUM(od.quantity * od.unit_price * (1 - od.discount)) AS revenue FROM orders o JOIN order_details od ON o.order_id = od.order_id JOIN products p ON od.product_id = p.product_id JOIN categories c ON p.category_id = c.category_id WHERE EXTRACT(YEAR FROM o.order_date) = 2024 AND EXTRACT(QUARTER FROM o.order_date) IN (3, 4) GROUP BY c.category_name, quarter ORDER BY c.category_name, quarter",
      "explanation": "Calculate Q3 and Q4 2024 revenue by product category, including discount adjustment."
    }
  },
  {
    "step": 5,
    "type": "tool_result",
    "tool": "execute_sql",
    "result": {
      "success": false,
      "error_type": "RUNTIME_ERROR",
      "error_message": "column 'quarter' does not exist. LINE 1: ...GROUP BY c.category_name, quarter. HINT: It may need to be GROUP BY EXTRACT(QUARTER FROM o.order_date) or use a subquery."
    }
  },
  {
    "step": 6,
    "type": "reasoning",
    "content": "The error is because I used the alias 'quarter' in the GROUP BY clause, but PostgreSQL doesn't allow column aliases in GROUP BY. I need to repeat the full expression EXTRACT(QUARTER FROM o.order_date) in the GROUP BY, or use a subquery. Let me use the full expression."
  },
  {
    "step": 7,
    "type": "tool_call",
    "tool": "execute_sql",
    "args": {
      "sql": "SELECT c.category_name, EXTRACT(QUARTER FROM o.order_date) AS quarter, SUM(od.quantity * od.unit_price * (1 - od.discount)) AS revenue FROM orders o JOIN order_details od ON o.order_id = od.order_id JOIN products p ON od.product_id = p.product_id JOIN categories c ON p.category_id = c.category_id WHERE EXTRACT(YEAR FROM o.order_date) = 2024 AND EXTRACT(QUARTER FROM o.order_date) IN (3, 4) GROUP BY c.category_name, EXTRACT(QUARTER FROM o.order_date) ORDER BY c.category_name, quarter",
      "explanation": "Fixed: replaced alias 'quarter' in GROUP BY with full expression EXTRACT(QUARTER FROM o.order_date)."
    }
  },
  {
    "step": 7,
    "type": "tool_result",
    "tool": "execute_sql",
    "result": {
      "success": true,
      "columns": ["category_name", "quarter", "revenue"],
      "rows": [
        {"category_name": "Beverages", "quarter": 3, "revenue": 28560.00},
        {"category_name": "Beverages", "quarter": 4, "revenue": 22100.00},
        {"category_name": "Condiments", "quarter": 3, "revenue": 12340.50},
        {"category_name": "Condiments", "quarter": 4, "revenue": 9870.25},
        {"category_name": "Confections", "quarter": 3, "revenue": 18900.00},
        {"category_name": "Confections", "quarter": 4, "revenue": 21500.00},
        {"category_name": "Dairy Products", "quarter": 3, "revenue": 31200.00},
        {"category_name": "Dairy Products", "quarter": 4, "revenue": 24800.00},
        {"category_name": "Grains/Cereals", "quarter": 3, "revenue": 8900.00},
        {"category_name": "Grains/Cereals", "quarter": 4, "revenue": 7200.00}
      ],
      "row_count": 10,
      "execution_ms": 28
    }
  },
  {
    "step": 8,
    "type": "reasoning",
    "content": "I have Q3 and Q4 revenue by category. Now I need to pivot this to compare Q3 vs Q4 and compute the difference. Let me write a query that shows each category with Q3 revenue, Q4 revenue, and the difference (Q4 - Q3), sorted by biggest drop."
  },
  {
    "step": 9,
    "type": "tool_call",
    "tool": "execute_sql",
    "args": {
      "sql": "WITH quarterly_revenue AS (SELECT c.category_name, EXTRACT(QUARTER FROM o.order_date) AS quarter, SUM(od.quantity * od.unit_price * (1 - od.discount)) AS revenue FROM orders o JOIN order_details od ON o.order_id = od.order_id JOIN products p ON od.product_id = p.product_id JOIN categories c ON p.category_id = c.category_id WHERE EXTRACT(YEAR FROM o.order_date) = 2024 AND EXTRACT(QUARTER FROM o.order_date) IN (3, 4) GROUP BY c.category_name, EXTRACT(QUARTER FROM o.order_date)) SELECT q3.category_name, q3.revenue AS q3_revenue, q4.revenue AS q4_revenue, q4.revenue - q3.revenue AS revenue_change, ROUND(((q4.revenue - q3.revenue) / q3.revenue) * 100, 2) AS pct_change FROM quarterly_revenue q3 JOIN quarterly_revenue q4 ON q3.category_name = q4.category_name AND q3.quarter = 3 AND q4.quarter = 4 ORDER BY revenue_change ASC LIMIT 5",
      "explanation": "Pivot Q3 and Q4 revenue using a CTE and self-join to compute the revenue change and percentage change per category, sorted by biggest drop."
    }
  },
  {
    "step": 9,
    "type": "tool_result",
    "tool": "execute_sql",
    "result": {
      "success": true,
      "columns": ["category_name", "q3_revenue", "q4_revenue", "revenue_change", "pct_change"],
      "rows": [
        {"category_name": "Dairy Products", "q3_revenue": 31200.00, "q4_revenue": 24800.00, "revenue_change": -6400.00, "pct_change": -20.51},
        {"category_name": "Beverages", "q3_revenue": 28560.00, "q4_revenue": 22100.00, "revenue_change": -6460.00, "pct_change": -22.62},
        {"category_name": "Grains/Cereals", "q3_revenue": 8900.00, "q4_revenue": 7200.00, "revenue_change": -1700.00, "pct_change": -19.10},
        {"category_name": "Condiments", "q3_revenue": 12340.50, "q4_revenue": 9870.25, "revenue_change": -2470.25, "pct_change": -20.02}
      ],
      "row_count": 4,
      "execution_ms": 35
    }
  },
  {
    "step": 10,
    "type": "reasoning",
    "content": "I have the data. Four categories saw revenue drops from Q3 to Q4 2024. Beverages had the biggest percentage drop (-22.62%), while Dairy Products had the biggest absolute drop (-$6,400). Confections actually grew. Let me generate a grouped bar chart to visualize Q3 vs Q4 by category."
  },
  {
    "step": 11,
    "type": "tool_call",
    "tool": "generate_chart",
    "args": {
      "chart_type": "bar",
      "title": "Q3 vs Q4 2024 Revenue by Product Category",
      "x_column": "category_name",
      "y_column": "q3_revenue",
      "sql": "WITH quarterly_revenue AS (...) SELECT ... ORDER BY revenue_change ASC"
    }
  },
  {
    "step": 11,
    "type": "tool_result",
    "tool": "generate_chart",
    "result": {
      "success": true,
      "chart_path": "/tmp/agent_charts/a1b2c3d4.png",
      "chart_type": "bar"
    }
  },
  {
    "step": 12,
    "type": "final_answer",
    "content": "Four product categories saw revenue declines from Q3 to Q4 in 2024:\n\n1. **Beverages**: -$6,460 (-22.62%) — the largest percentage drop\n2. **Dairy Products**: -$6,400 (-20.51%) — the largest absolute drop\n3. **Condiments**: -$2,470 (-20.02%)\n4. **Grains/Cereals**: -$1,700 (-19.10%)\n\nNotably, **Confections** was the only category that grew (+$2,600, +13.76%).\n\nThe chart below visualizes the Q3 vs Q4 comparison across all categories. Beverages and Dairy Products stand out as the categories needing the most attention — together they account for $12,860 in lost quarterly revenue.\n\n**SQL used:**\n```sql\nWITH quarterly_revenue AS (\n  SELECT c.category_name,\n         EXTRACT(QUARTER FROM o.order_date) AS quarter,\n         SUM(od.quantity * od.unit_price * (1 - od.discount)) AS revenue\n  FROM orders o\n  JOIN order_details od ON o.order_id = od.order_id\n  JOIN products p ON od.product_id = p.product_id\n  JOIN categories c ON p.category_id = c.category_id\n  WHERE EXTRACT(YEAR FROM o.order_date) = 2024\n    AND EXTRACT(QUARTER FROM o.order_date) IN (3, 4)\n  GROUP BY c.category_name, EXTRACT(QUARTER FROM o.order_date)\n)\nSELECT q3.category_name,\n       q3.revenue AS q3_revenue,\n       q4.revenue AS q4_revenue,\n       q4.revenue - q3.revenue AS revenue_change,\n       ROUND(((q4.revenue - q3.revenue) / q3.revenue) * 100, 2) AS pct_change\nFROM quarterly_revenue q3\nJOIN quarterly_revenue q4\n  ON q3.category_name = q4.category_name\n  AND q3.quarter = 3 AND q4.quarter = 4\nORDER BY revenue_change ASC\n```"
  }
]
```

### Key Observations from the Trace

1. **Schema-first approach**: The agent calls `get_schema` before writing any SQL (step 1). This is enforced by the system prompt and is the foundation of accurate SQL generation.

2. **Multi-step reasoning**: The agent breaks the complex question into sub-problems:
   - First, find the most recent year (step 3)
   - Then, calculate quarterly revenue by category (step 5)
   - Finally, pivot and compute the difference (step 9)

3. **Error recovery in action**: Step 5 fails because PostgreSQL doesn't allow column aliases in GROUP BY. The agent reads the error message, reasons about the fix (step 6), and regenerates the SQL with the correct expression (step 7). This is attempt 1 → fix → attempt 2 = success.

4. **Chart generation**: The agent decides a bar chart is appropriate for comparing Q3 vs Q4 across categories (step 11) and calls `generate_chart` autonomously.

5. **Business-language interpretation**: The final answer (step 12) uses business language ("Beverages had the biggest percentage drop"), not technical jargon ("The query returned 4 rows"). It also highlights the actionable insight (Confections grew while others declined).

6. **Transparency**: The SQL is shown in the answer so the user can verify the logic. The reasoning trace is available for debugging and auditing.

### Follow-up Turn (Multi-Turn Conversation)

If the user then asks: *"What about just German customers?"*

```json
[
  {
    "step": 1,
    "type": "reasoning",
    "content": "The user is asking a follow-up to the previous question about Q3 vs Q4 revenue drops. They want to filter to only German customers. From the previous turn, I know the schema and the SQL structure. I just need to add a filter for ship_country = 'Germany' to the orders table. I can reuse the previous query structure."
  },
  {
    "step": 2,
    "type": "tool_call",
    "tool": "execute_sql",
    "args": {
      "sql": "WITH quarterly_revenue AS (SELECT c.category_name, EXTRACT(QUARTER FROM o.order_date) AS quarter, SUM(od.quantity * od.unit_price * (1 - od.discount)) AS revenue FROM orders o JOIN order_details od ON o.order_id = od.order_id JOIN products p ON od.product_id = p.product_id JOIN categories c ON p.category_id = c.category_id WHERE EXTRACT(YEAR FROM o.order_date) = 2024 AND EXTRACT(QUARTER FROM o.order_date) IN (3, 4) AND o.ship_country = 'Germany' GROUP BY c.category_name, EXTRACT(QUARTER FROM o.order_date)) SELECT q3.category_name, q3.revenue AS q3_revenue, q4.revenue AS q4_revenue, q4.revenue - q3.revenue AS revenue_change, ROUND(((q4.revenue - q3.revenue) / q3.revenue) * 100, 2) AS pct_change FROM quarterly_revenue q3 JOIN quarterly_revenue q4 ON q3.category_name = q4.category_name AND q3.quarter = 3 AND q4.quarter = 4 ORDER BY revenue_change ASC LIMIT 5",
      "explanation": "Same query as before but filtered to German customers (ship_country = 'Germany'). Reusing the previous query structure from conversation context."
    }
  },
  {
    "step": 2,
    "type": "tool_result",
    "tool": "execute_sql",
    "result": {
      "success": true,
      "columns": ["category_name", "q3_revenue", "q4_revenue", "revenue_change", "pct_change"],
      "rows": [
        {"category_name": "Beverages", "q3_revenue": 8200.00, "q4_revenue": 5100.00, "revenue_change": -3100.00, "pct_change": -37.80},
        {"category_name": "Dairy Products", "q3_revenue": 9800.00, "q4_revenue": 7200.00, "revenue_change": -2600.00, "pct_change": -26.53}
      ],
      "row_count": 2,
      "execution_ms": 18
    }
  },
  {
    "step": 3,
    "type": "final_answer",
    "content": "For German customers specifically, the Q3-to-Q4 2024 revenue drops are more pronounced:\n\n1. **Beverages**: -$3,100 (-37.80%) — a much steeper decline than the global average of -22.62%\n2. **Dairy Products**: -$2,600 (-26.53%) — also worse than the global -20.51%\n\nOnly 2 categories had declines among German customers (vs 4 globally), suggesting the German market is more concentrated. The Beverages decline in Germany is particularly concerning — it's nearly double the global percentage drop.\n\nI've reused the previous query with an added `WHERE o.ship_country = 'Germany'` filter."
  }
]
```

This follow-up demonstrates **conversation memory**: the agent reuses the previous query structure, doesn't need to call `get_schema` again, and references the prior results for comparison ("much steeper decline than the global average of -22.62%").