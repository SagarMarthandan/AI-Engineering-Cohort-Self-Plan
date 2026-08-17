# Agent Orchestration Guide — Multi-Agent Systems

> **Project:** LangGraph-orchestrated multi-agent system with Router → Research → Writer → Critic → Revision loop, Orchestrator-Worker pattern, Map-Reduce parallel agents, Pydantic AI comparison agent, short-term memory with summarization in Redis, episodic memory in pgvector, per-agent cost tracking, and a Streamlit dashboard.
>
> **Plan reference:** `IMPLEMENTATION_PLAN.md` in this directory — the single source of truth.
> **Timeline:** 8 phases, 10 working days, 4-5 Herdr agents.

---

## Tier 2 — Project 6

**Prerequisites:** Project 4 (Data Analysis Agent) — tool-use patterns, function calling, LLM client with retry. Project 5 (Document Processing Pipeline) — pipeline orchestration, structured output, Docker Compose patterns.

**Downstream:** Project 11 (Customer Support AI System) consumes the multi-agent orchestration patterns, cost tracking, and memory management built here.

---

## 1. Agent Roster

| Agent | Kind | Responsibility | Owns |
|-------|------|----------------|------|
| **graph** | codex | LangGraph StateGraph definition, state TypedDict, graph builder, conditional edges, all graph variants (default, orchestrator-worker, critic-loop, map-reduce), checkpointing | `src/graph/` (all files), `src/agents/prompts.py` |
| **agents** | codex | All agent node functions: router, researcher (raw + Pydantic AI), writer, critic, orchestrator, worker, synthesizer; LLM client, cost tracker, tools | `src/agents/` (all files except prompts.py), `src/llm/` (all files), `src/tools/` (all files) |
| **infra** | codex | Docker Compose, Dockerfiles, PostgreSQL schema DDL, Redis setup, config, FastAPI app, API routes, Streamlit dashboard, memory stores, observability/tracing | `docker-compose.yml`, `docker/`, `src/config.py`, `src/main.py`, `src/db/`, `src/memory/`, `src/observability/`, `src/api/`, `streamlit/`, `scripts/`, `pyproject.toml`, `.env.example` |
| **reviewer** | codex | Phase-gate code review, graph correctness verification, test coverage, no stubs/placeholders, cross-agent integration checks | `tests/` (writes test files; reviews all `src/`) |

---

## 2. Pane Layout

```
┌─────────────────────────────────────────────────────────────┐
│           Herdr Pane Layout — Multi-Agent Systems           │
├──────────────────┬──────────────────┬───────────────────────┤
│                  │                  │                       │
│      graph       │     agents       │      reviewer         │
│   (pane 1)       │    (pane 2)      │     (pane 3)          │
│  StateGraph      │  Router, Writer  │  code review          │
│  edges, state    │  Critic, Workers │  phase gates          │
│  checkpointing   │  LLM, cost       │  tests                │
│                  │  tools, Pydantic │                       │
│                  │                  │                       │
├──────────────────┼──────────────────┤                       │
│                  │                  │                       │
│     infra        │   terminal       │                       │
│   (pane 4)       │   (pane 5)       │                       │
│  Docker, DB      │  git, docker     │                       │
│  API, Streamlit  │  pytest, curl    │                       │
│  memory, trace   │                  │                       │
│                  │                  │                       │
└──────────────────┴──────────────────┴───────────────────────┘
```

---

## 3. Setup Commands

```bash
# --- One-time: initialize Herdr workspace ---
cd "/home/sagar/Projects/AI-Engineering Roadmap/Tier 2 - LLM Orchestration & Agents/Project 6 - Multi-Agent Systems"

# Create project scaffold directory
mkdir -p multi-agent-systems && cd multi-agent-systems

# --- Split panes (5-pane grid) ---
# Pane 1: graph (top-left)
herdr pane split --direction vertical --cwd "$PWD" --no-focus          # pane 2 (right of pane 1)
herdr pane split --direction vertical --cwd "$PWD" --no-focus          # pane 3 (right of pane 2)
herdr pane focus 1 && herdr pane split --direction horizontal --cwd "$PWD" --no-focus  # pane 4 (below pane 1)
herdr pane focus 4 && herdr pane split --direction horizontal --cwd "$PWD" --no-focus  # pane 5 (below pane 4)

# --- Start agents ---
herdr agent start graph     --kind codex --pane 1
herdr agent start agents    --kind codex --pane 2
herdr agent start reviewer  --kind codex --pane 3
herdr agent start infra     --kind codex --pane 4

# Pane 5 stays as a manual terminal for git, docker, pytest, curl

# --- Verify all agents are running ---
herdr agent list
```

> **Pane IDs:** `1` = graph, `2` = agents, `3` = reviewer, `4` = infra, `5` = terminal.
> Adjust IDs if your Herdr session numbers differently — verify with `herdr pane list`.

---

## 4. Phase-by-Phase Orchestration

### Phase 1: Environment Setup + LangGraph Basics (Day 1)

**Goal:** Docker Compose stack running (Postgres + Redis + API + Streamlit), LangGraph state graph compiles and visualizes.

**Plan reference:** §14 Phase 1, §2 (Architecture), §5 (LangGraph Implementation), §12 (Database Schema), §13 (Project Structure), §17 (Deployment).

#### Parallel Work

**infra** (Docker + DB + config — no dependencies):
```
Read IMPLEMENTATION_PLAN.md §17 (Deployment), §12 (Database Schema), §13 (Project Structure), §3 (Tech Stack).

Create the project infrastructure:
1. `pyproject.toml` — Python 3.11+, deps: langgraph, langgraph-checkpoint-redis, pydantic-ai, fastapi, uvicorn, asyncpg, sqlalchemy[asyncio], redis, openai, pydantic, pydantic-settings, tenacity, streamlit, httpx, pytest, pytest-asyncio, ruff. Use src/ layout.
2. `docker-compose.yml` — follow §17.1 exactly: services postgres (image postgres:16, healthcheck, loads init.sql), redis (image redis:7-alpine, healthcheck), api (build from docker/Dockerfile.api, port 8000, env vars), streamlit (build from docker/Dockerfile.streamlit, port 8501). Include postgres_data and redis_data volumes.
3. `docker/Dockerfile.api` — follow §17.2: python:3.12-slim, install build-essential + libpq-dev, pip install -e ., CMD uvicorn.
4. `docker/Dockerfile.streamlit` — follow §17.3: python:3.12-slim, pip install streamlit httpx pydantic, CMD streamlit run.
5. `docker/postgres/init.sql` — follow §12.2 exactly: CREATE EXTENSION uuid-ossp, pgcrypto, vector; CREATE SCHEMA agent_meta, cost_tracking, episodic_memory, execution_logs; all tables with columns, types, constraints, indexes as specified.
6. `.env.example` — follow §17.5: POSTGRES_PASSWORD, OPENAI_API_KEY, LLM_MODEL, DATABASE_URL, REDIS_URL, LOG_LEVEL, optional LANGSMITH/LANGFUSE keys.
7. `src/config.py` — Pydantic Settings reading: DATABASE_URL, REDIS_URL, OPENAI_API_KEY, LLM_MODEL (default gpt-4o-mini), LOG_LEVEL, BUFFER_LIMIT (default 10), SUMMARIZE_THRESHOLD (default 10), MAX_REVISIONS (default 3).
8. `src/db/connection.py` — asyncpg connection pool. Init pool from settings.DATABASE_URL. Provide get_db() dependency and module-level pool accessor.
9. `src/main.py` — FastAPI app factory with lifespan (init DB pool + Redis client on startup, close on shutdown). Include API routers (will be added in later phases).

Do NOT create src/graph/ or src/agents/ files — those are owned by graph and agents agents.
```

**graph** (state + minimal graph — no dependencies):
```
Read IMPLEMENTATION_PLAN.md §5 (LangGraph Implementation), §2.2 (State Graph Topology), §13 (Project Structure).

Create the LangGraph state definition and a minimal compilable graph:
1. `src/graph/__init__.py` — empty.
2. `src/graph/state.py` — AgentState TypedDict exactly as specified in §5.1: all fields (query, session_id, pattern, query_type, router_confidence, research_notes, subtasks, worker_results, draft, revision_count, critic_feedback, critic_score, critic_approved, score_history, final_output, quality_warning, messages, memory_context, cost_log, status, error). Use total=False.
3. `src/graph/builder.py` — minimal build_graph() with stub node functions (each just returns {} or a minimal dict). Add all nodes from §5.4: router, research_technical, research_creative, research_analytical, research_fallback, writer, critic, synthesizer. Add edges: START→router, conditional edges from router to research nodes, all research→writer, writer→critic, conditional from critic (synthesizer/writer), synthesizer→END. Use placeholder edge functions that return "synthesizer" for now.
4. `src/graph/edges.py` — placeholder route_after_router() and route_after_critic() functions (return "research_fallback" and "synthesizer" respectively for now — will be implemented in Phase 3 and 4).

The graph MUST compile and `graph.get_graph().draw_mermaid()` MUST produce valid Mermaid output.
```

**agents** (LLM client + cost tracker — no dependencies):
```
Read IMPLEMENTATION_PLAN.md §11 (Cost Tracking & Observability), §15.1 (LLM Client pseudocode), §15.10 (Cost Tracker pseudocode).

Create the LLM client and cost tracker:
1. `src/llm/__init__.py` — empty.
2. `src/llm/cost_tracker.py` — CostTracker and CostRecord exactly as specified in §11.1 and §15.10. CostRecord dataclass with agent, timestamp, model, prompt_tokens, completion_tokens, total_tokens, cost_usd, latency_ms. CostTracker with gpt-4o-mini pricing ($0.15/1M input, $0.60/1M output), record() method, summary() method aggregating by agent.
3. `src/llm/client.py` — LLMClient exactly as specified in §15.1: AsyncOpenAI wrapper with tenacity retry (stop_after_attempt=3, wait_exponential min=2 max=30, retry on RateLimitError/APITimeoutError/APIConnectionError). Two methods: complete() returning tuple[str, dict], structured_complete() returning tuple[T, dict] using beta.chat.completions.parse().

Do NOT create agent node functions yet — those come in Phase 3+.
```

#### Sequential (after parallel)

**infra** (wire routers into main.py):
```
graph and agents have created their modules. Verify imports work:
1. `python -c "from src.graph.builder import build_graph; g = build_graph(); print(g.get_graph().draw_mermaid())"` — must print valid Mermaid.
2. `python -c "from src.llm.client import LLMClient; from src.llm.cost_tracker import CostTracker"` — must import without error.
3. Write `src/api/__init__.py`, `src/api/routes/__init__.py`, `src/api/dependencies.py` (FastAPI DI: get_db, get_settings), `src/api/routes/health.py` (GET /health → check DB + Redis connectivity).
4. Update src/main.py to include health router.
5. Write `tests/conftest.py` with fixtures: mock_llm (AsyncMock spec LLMClient), sample_state (minimal AgentState), test_db (asyncpg connect, run schema.sql, yield, drop), test_client (httpx AsyncClient with create_app).
```

#### Review

**reviewer**:
```
Review Phase 1 deliverables against IMPLEMENTATION_PLAN.md §14 Phase 1:

1. Run `docker compose up` — all 4 services (postgres, redis, api, streamlit) start without errors.
2. Run `curl localhost:8000/health` — must return {"status":"healthy","database":"connected","redis":"connected"}.
3. Connect to Postgres and verify all 4 schemas exist: agent_meta, cost_tracking, episodic_memory, execution_logs. Verify pgvector extension is installed.
4. Run `python -c "from src.graph.builder import build_graph; g = build_graph(); print(g.get_graph().draw_mermaid())"` — must print valid Mermaid with all nodes (router, research_*, writer, critic, synthesizer).
5. Verify src/llm/client.py has tenacity retry with correct exception types and both complete() and structured_complete() methods.
6. Verify src/llm/cost_tracker.py has correct gpt-4o-mini pricing ($0.15/1M input, $0.60/1M output).
7. Verify no stubs, TODOs, or placeholder code in LLM client or cost tracker (graph stubs are expected for Phase 1).
8. Verify file ownership: graph owns src/graph/; agents owns src/llm/; infra owns docker/, docker-compose.yml, pyproject.toml, .env.example, src/config.py, src/db/, src/main.py, src/api/.

Report any failures to the responsible agent via hub message. Do not commit until all checks pass.
```

#### Commit

```bash
git add -A && git commit -m "Phase 1: Docker stack (Postgres+Redis+API+Streamlit), LangGraph state graph compiles, LLM client with retry, cost tracker"
```

---

### Phase 2: LLM Client + Cost Tracker + Tools (Day 2)

**Goal:** LLM client functional, cost tracker verified, stubbed tools (web_search, knowledge_base), all system prompts defined.

**Plan reference:** §14 Phase 2, §11 (Cost Tracking), §15.1 (LLM Client), §15.15 (Agent Prompts).

#### Parallel Work

**agents** (tools + prompts — depends on Phase 1 LLM client):
```
Read IMPLEMENTATION_PLAN.md §15.15 (Agent Prompts — all system prompts), §13 (Project Structure for tools/).

Create the tools and all system prompts:
1. `src/tools/__init__.py` — empty.
2. `src/tools/search.py` — async web_search_stub(query: str) -> str. Returns simulated search results. Include Redis caching with 5-min TTL (check cache first, store on miss). Use the Redis client from app state.
3. `src/tools/knowledge_base.py` — async knowledge_base_lookup(topic: str, query: str) -> str. Queries the knowledge_base table in PostgreSQL via asyncpg. Returns content or "No KB entry found."
4. `src/agents/prompts.py` — ALL system prompts exactly as specified in §15.15:
   - ROUTER_SYSTEM_PROMPT (classify into technical/creative/analytical/unknown with confidence)
   - WRITER_SYSTEM_PROMPT (produce content, revise based on feedback)
   - CRITIC_SYSTEM_PROMPT (review against rubric, scoring guide 90-100 approved, 75-89 approved, 60-74 not approved, 0-59 not approved)
   - ORCHESTRATOR_SYSTEM_PROMPT (decompose into 2-5 independent subtasks)
   - WORKER_SYSTEM_PROMPT (specialist worker, {role} placeholder)
   - SYNTHESIZER_SYSTEM_PROMPT (combine worker results into coherent response)
   - SUMMARIZATION_SYSTEM_PROMPT (summarize conversation preserving key facts, under 500 words)
   - FALLBACK_SYSTEM_PROMPT (generalist agent for low-confidence queries)
5. `tests/test_cost_tracker.py` — test cost calculation with known token counts (e.g., 1000 prompt + 500 completion → verify cost = (1000/1M)*0.15 + (500/1M)*0.60). Test summary() aggregation by agent. Test multiple agents tracked separately.

Write `scripts/seed_kb.py` — script to insert sample knowledge_base entries (topics: "rag", "transformers", "postgresql", "docker", "python-async") with placeholder content.
```

**graph** (edge functions — depends on Phase 1 state):
```
Read IMPLEMENTATION_PLAN.md §5.3 (Conditional Edges), §7.4 (Routing Conditional Edge), §8.4 (Max Iterations and Convergence Detection).

Implement the real conditional edge functions (replacing Phase 1 placeholders):
1. Update `src/graph/edges.py`:
   - route_after_router(state) — follow §7.4: if confidence < 0.6 → "research_fallback"; else map query_type to research node. Return "research_fallback" for unknown.
   - route_after_critic(state) — follow §8.4: if critic_approved → "synthesizer"; if revision_count >= 3 → "synthesizer"; convergence detection (if score_history[-1] <= score_history[-2] → "synthesizer"); else → "writer".

These are pure functions of state — no LLM calls. They will be used by the graph builder in Phase 3+.
```

#### Review

**reviewer**:
```
Review Phase 2 deliverables against IMPLEMENTATION_PLAN.md §14 Phase 2:

1. Run `pytest tests/test_cost_tracker.py -v` — all tests pass, cost calculation correct for gpt-4o-mini pricing.
2. Verify src/tools/search.py has Redis caching (check-then-set with TTL).
3. Verify src/tools/knowledge_base.py queries the knowledge_base table via asyncpg.
4. Verify ALL 8 system prompts in src/agents/prompts.py match §15.15 exactly (ROUTER, WRITER, CRITIC, ORCHESTRATOR, WORKER, SYNTHESIZER, SUMMARIZATION, FALLBACK).
5. Verify CRITIC_SYSTEM_PROMPT has the correct scoring guide (≥75 = approved, <75 = not approved).
6. Verify route_after_router returns "research_fallback" when confidence < 0.6.
7. Verify route_after_critic has convergence detection (score not improving → stop).
8. Verify no stubs or TODOs in tools, prompts, or edge functions.

Report failures to responsible agent. Do not commit until all pass.
```

#### Commit

```bash
git add -A && git commit -m "Phase 2: LLM client with retry, cost tracker verified, stubbed tools (search+KB), all system prompts, conditional edge functions"
```

---

### Phase 3: Router Agent + Routing (Day 3)

**Goal:** Router classifies queries and routes to the correct research node via conditional edges.

**Plan reference:** §14 Phase 3, §7 (Routing Agent), §5.2 (Node Functions — router_node), §15.2 (Router Agent pseudocode).

#### Parallel Work

**agents** (router + research nodes — depends on Phase 2 prompts + tools):
```
Read IMPLEMENTATION_PLAN.md §7 (Routing Agent), §15.2 (Router Agent pseudocode), §5.2 (router_node, research_node).

Create the router agent and research nodes:
1. `src/agents/__init__.py` — empty.
2. `src/agents/router.py` — follow §15.2 exactly:
   - QueryCategory enum (TECHNICAL, CREATIVE, ANALYTICAL, UNKNOWN)
   - QueryClassification BaseModel (category, confidence 0.0-1.0, reasoning)
   - router_node(state) async function: calls llm_client.structured_complete with QueryClassification schema, records cost, returns query_type + router_confidence + cost_log update.
3. `src/agents/researcher.py` — research_node(state) async function (raw function-calling version):
   - Calls web_search_stub(query) and knowledge_base_lookup(query_type, query)
   - Combines results into research_notes string
   - Returns {"research_notes": ...}
   This is a shared function used by all research_* nodes (technical, creative, analytical, fallback).
4. Write `tests/test_router.py`:
   - Mock llm_client.structured_complete to return QueryClassification(category=TECHNICAL, confidence=0.9) → verify query_type="technical"
   - Mock returning confidence=0.4 → verify route_after_router returns "research_fallback"
   - Mock returning category=CREATIVE → verify routes to "research_creative"
   - Mock returning category=ANALYTICAL → verify routes to "research_analytical"
   - Mock returning category=UNKNOWN → verify routes to "research_fallback"
```

**graph** (wire router into graph — depends on Phase 2 edges):
```
Read IMPLEMENTATION_PLAN.md §5.4 (Building and Compiling the Graph), §7.4 (Routing Conditional Edge).

Update src/graph/builder.py to use real node functions and edge functions:
1. Replace stub router node with real router_node from src.agents.router.
2. Replace stub research nodes with real research_node from src.agents.researcher (same function for all 4 research nodes).
3. Replace placeholder route_after_router with real function from src.graph.edges.
4. Keep placeholder writer_node, critic_node, synthesizer_node for now (Phase 4).
5. Keep placeholder route_after_critic returning "synthesizer" for now.
6. Verify graph compiles and Mermaid output shows: START→router→{research_technical|research_creative|research_analytical|research_fallback}→writer→critic→{synthesizer|writer}→END.

Write `tests/test_graph.py`:
- Test graph compiles without error
- Test all nodes are present in the graph
- Test edges: START→router, router has 4 conditional edges, each research→writer, writer→critic, critic has 2 conditional edges, synthesizer→END
- Test Mermaid output contains all node names
```

#### Review

**reviewer**:
```
Review Phase 3 against IMPLEMENTATION_PLAN.md §14 Phase 3:

1. Run `pytest tests/test_router.py tests/test_graph.py -v` — all tests pass.
2. Verify router_node uses llm_client.structured_complete with QueryClassification schema.
3. Verify route_after_router routes to "research_fallback" when confidence < 0.6.
4. Verify all 4 research nodes (technical, creative, analytical, fallback) are wired in the graph.
5. Verify graph compiles and Mermaid output is valid.
6. Verify cost is tracked for the router agent (cost_log entry with agent="router").
7. Verify no stubs or TODOs in router.py or researcher.py.

Report failures to responsible agent. Do not commit until all pass.
```

#### Commit

```bash
git add -A && git commit -m "Phase 3: Router agent with classification, conditional routing to 4 research nodes, graph wired with real nodes"
```

---

### Phase 4: Writer + Critic/Revision Loop (Days 4-5)

**Goal:** Writer produces drafts, Critic reviews with structured feedback, revision loop with max 3 rounds and convergence detection.

**Plan reference:** §14 Phase 4, §8 (Critic/Revision Loop), §15.3 (Writer Agent), §15.4 (Critic Agent), §15.7 (Synthesizer Agent).

#### Parallel Work

**agents** (writer + critic + synthesizer — depends on Phase 2 prompts):
```
Read IMPLEMENTATION_PLAN.md §8 (Critic/Revision Loop), §15.3 (Writer), §15.4 (Critic), §15.7 (Synthesizer).

Create the writer, critic, and synthesizer agents:
1. `src/agents/writer.py` — follow §15.3 exactly:
   - writer_node(state) async function
   - If revision_count == 0: initial draft prompt (query + research notes)
   - If revision_count > 0: revision prompt (query + research + previous draft + critic feedback)
   - Calls llm_client.complete with WRITER_SYSTEM_PROMPT
   - Returns draft, revision_count+1, cost_log update
2. `src/agents/critic.py` — follow §15.4 exactly:
   - CriticFeedback BaseModel (feedback_text, score 0-100, strengths, weaknesses, suggestions)
   - critic_node(state) async function
   - Calls llm_client.structured_complete with CriticFeedback schema
   - approved = score >= 75
   - Updates score_history
   - Returns critic_feedback, critic_score, critic_approved, score_history, cost_log update
3. `src/agents/synthesizer.py` — follow §15.7 exactly:
   - synthesizer_node(state): if approved → final_output = draft, quality_warning = False; else → quality_warning = True. Stores episodic memory. Returns final_output, quality_warning, status="completed".
   - synthesizer_orchestrator_node(state): for orchestrator-worker pattern (Phase 5). Merges worker_results into coherent output.

Write `tests/test_critic_loop.py`:
- Test: critic approves (score=85) → loop terminates, quality_warning=False
- Test: critic rejects (score=50) → revision_count increments, loops back to writer
- Test: max 3 revisions reached without approval → loop terminates, quality_warning=True
- Test: convergence detection — score goes 60→58 → loop stops early (not improving)
- Test: score improves 60→70→80→85 → loop continues until approval
- Mock llm_client for all tests (deterministic).
```

**graph** (wire writer/critic/synthesizer — depends on agents creating node functions):
```
Read IMPLEMENTATION_PLAN.md §5.4 (Building and Compiling the Graph), §8.5 (Critic/Revision Graph).

Update src/graph/builder.py:
1. Replace stub writer_node with real writer_node from src.agents.writer.
2. Replace stub critic_node with real critic_node from src.agents.critic.
3. Replace stub synthesizer_node with real synthesizer_node from src.agents.synthesizer.
4. Replace placeholder route_after_critic with real function from src.graph.edges (already implemented in Phase 2).
5. Verify the full default graph: START→router→research_*→writer→critic→{synthesizer|writer(loop)}→END.
6. Also build build_critic_loop_graph() as specified in §8.5 (standalone critic/revision loop: START→writer→critic→{synthesizer|writer}→END).

Update tests/test_graph.py to verify the full graph with real nodes.
```

#### Review

**reviewer**:
```
Review Phase 4 against IMPLEMENTATION_PLAN.md §14 Phase 4:

1. Run `pytest tests/test_critic_loop.py tests/test_graph.py -v` — all tests pass.
2. Verify writer_node has two modes: initial draft (revision_count=0) and revision (revision_count>0 with critic feedback).
3. Verify critic_node uses CriticFeedback schema with score 0-100 and approved = score >= 75.
4. Verify route_after_critic has THREE termination conditions: approved, max 3 revisions, convergence (score not improving).
5. Verify quality_warning is True when max revisions hit without approval.
6. Verify synthesizer_node stores episodic memory.
7. Verify build_critic_loop_graph() exists as a standalone graph.
8. Verify the full default graph compiles with all real nodes.
9. Verify no stubs or TODOs in writer.py, critic.py, synthesizer.py.

Report failures to responsible agent. Do not commit until all pass.
```

#### Commit

```bash
git add -A && git commit -m "Phase 4: Writer + Critic/Revision loop with max 3 rounds, convergence detection, quality warning, synthesizer with episodic memory storage"
```

---

### Phase 5: Orchestrator-Worker + Map-Reduce (Days 6-7)

**Goal:** Orchestrator decomposes tasks, workers execute in parallel via asyncio.gather, synthesizer merges results.

**Plan reference:** §14 Phase 5, §6 (Orchestrator-Worker Pattern), §4.4 (Map-Reduce Parallel Agents), §15.5 (Orchestrator), §15.6 (Worker), §15.7 (Synthesizer — orchestrator variant).

#### Parallel Work

**agents** (orchestrator + worker — depends on Phase 2 prompts):
```
Read IMPLEMENTATION_PLAN.md §6 (Orchestrator-Worker Pattern), §4.4 (Map-Reduce), §15.5 (Orchestrator), §15.6 (Worker).

Create the orchestrator and worker agents:
1. `src/agents/orchestrator.py` — follow §15.5 exactly:
   - DecompositionPlan BaseModel (subtasks: list[str] min_length=2 max_length=5, strategy: str)
   - orchestrator_node(state) async function: calls llm_client.structured_complete with DecompositionPlan, records cost, returns subtasks + cost_log update.
2. `src/agents/worker.py` — follow §15.6 exactly:
   - worker_node(state) async function
   - Inner execute_single(subtask, index) function
   - Uses asyncio.gather(*[execute_single(st, i) for i, st in enumerate(subtasks)], return_exceptions=True)
   - Handles worker failures: if isinstance(result, Exception) → append "[Worker {i} failed: {result}]"
   - Returns worker_results + cost_log update
3. Update `src/agents/synthesizer.py` — add synthesizer_orchestrator_node(state) as specified in §15.7:
   - Pairs subtasks with worker_results
   - Calls llm_client.complete with SYNTHESIZER_SYSTEM_PROMPT
   - Returns final_output, status="completed", cost_log update

Write `tests/test_orchestrator.py`:
- Test: mock orchestrator returns 3 subtasks → verify 3 worker calls via asyncio.gather
- Test: worker failure handling — mock one worker to raise → verify partial results with failure note
- Test: synthesizer merges worker results into coherent output
- Test: parallel execution (verify timing — parallel < sequential for 3+ workers)
- Test: subtasks count is 2-5 (DecompositionPlan validation)
```

**graph** (orchestrator-worker graph — depends on agents creating node functions):
```
Read IMPLEMENTATION_PLAN.md §6.5 (Orchestrator-Worker Graph), §4.4 (Map-Reduce Parallel Agents).

Update src/graph/builder.py:
1. Add build_orchestrator_worker_graph() as specified in §6.5:
   - Nodes: orchestrator, workers, synthesizer (orchestrator variant)
   - Edges: START→orchestrator→workers→synthesizer→END
2. Add build_map_reduce_graph() for the map-reduce pattern (§4.4):
   - Mapper node splits task into N subtasks (same as orchestrator decomposition)
   - Worker node processes all subtasks in parallel (same as worker_node)
   - Reducer node merges results (same as synthesizer_orchestrator_node)
   - Edges: START→mapper→workers→reducer→END

Update tests/test_graph.py to verify both new graph variants compile.
```

#### Review

**reviewer**:
```
Review Phase 5 against IMPLEMENTATION_PLAN.md §14 Phase 5:

1. Run `pytest tests/test_orchestrator.py -v` — all tests pass.
2. Verify orchestrator_node produces DecompositionPlan with 2-5 subtasks.
3. Verify worker_node uses asyncio.gather with return_exceptions=True (parallel execution + failure handling).
4. Verify worker failure is handled gracefully (partial results + "[Worker {i} failed: ...]" note).
5. Verify synthesizer_orchestrator_node merges worker results into coherent output (not just concatenation).
6. Verify build_orchestrator_worker_graph() compiles: START→orchestrator→workers→synthesizer→END.
7. Verify build_map_reduce_graph() compiles: START→mapper→workers→reducer→END.
8. Verify no stubs or TODOs in orchestrator.py, worker.py.

Report failures to responsible agent. Do not commit until all pass.
```

#### Commit

```bash
git add -A && git commit -m "Phase 5: Orchestrator-Worker pattern with parallel asyncio.gather, Map-Reduce graph, worker failure handling, synthesizer merge"
```

---

### Phase 6: Pydantic AI Agent (Day 8)

**Goal:** Researcher agent built with Pydantic AI; comparison table documented; both versions produce equivalent results.

**Plan reference:** §14 Phase 6, §9 (Pydantic AI Integration), §9.5 (Comparison Table).

#### Parallel Work

**agents** (Pydantic AI researcher — depends on Phase 3 researcher.py):
```
Read IMPLEMENTATION_PLAN.md §9 (Pydantic AI Integration) in full: §9.1 (Agent Definition), §9.2 (Structured Output), §9.3 (Tool Definitions), §9.4 (Dependency Injection), §9.5 (Comparison Table).

Create the Pydantic AI version of the researcher agent:
1. `src/agents/researcher_pydantic.py` — follow §9.1-§9.4 exactly:
   - ResearchDependencies dataclass (db_pool, redis, cost_tracker)
   - ResearchResult BaseModel (summary, key_points, sources, confidence)
   - research_agent = Agent(model=OpenAIModel("gpt-4o-mini"), deps_type=ResearchDependencies, output_type=ResearchResult, system_prompt=...)
   - @research_agent.tool decorators for web_search and knowledge_base_lookup (follow §9.3)
   - run_pydantic_research(query, deps) async function that calls research_agent.run() and returns result.output (follow §9.2)
2. Verify the existing src/agents/researcher.py (raw function-calling version from Phase 3) is the comparison baseline.
3. Add a comparison comment block at the top of researcher_pydantic.py documenting the trade-offs from §9.5 (boilerplate, type safety, streaming, error handling, dependency injection, control, best for, LangGraph integration).

Write `tests/test_pydantic_ai.py`:
- Test: Pydantic AI agent produces ResearchResult with validated fields (summary, key_points, sources, confidence)
- Test: tool calls work (mock web_search_stub and knowledge_base_lookup)
- Test: dependency injection (ResearchDependencies passed via deps=)
- Test: structured output validation — if LLM returns invalid output, Pydantic AI retries (mock to verify retry behavior)
- Test: both raw and Pydantic AI versions produce equivalent results for the same query (mock both to return same research_notes)
```

**graph** (wire Pydantic AI into graph — depends on agents creating researcher_pydantic.py):
```
Read IMPLEMENTATION_PLAN.md §9.4 (Dependency Injection — FastAPI route integration).

Update src/graph/builder.py:
1. Add a pydantic_ai_research_node(state) that calls run_pydantic_research() instead of the raw research_node.
2. Add build_pydantic_ai_graph() that uses pydantic_ai_research_node instead of the raw research nodes.
3. The graph topology is the same as the default graph but with the Pydantic AI research node.
4. Verify the graph compiles.

Update tests/test_graph.py to verify build_pydantic_ai_graph() compiles.
```

#### Review

**reviewer**:
```
Review Phase 6 against IMPLEMENTATION_PLAN.md §14 Phase 6:

1. Run `pytest tests/test_pydantic_ai.py -v` — all tests pass.
2. Verify researcher_pydantic.py uses Pydantic AI Agent class with deps_type=ResearchDependencies and output_type=ResearchResult.
3. Verify @research_agent.tool decorators are used for web_search and knowledge_base_lookup.
4. Verify ResearchResult has fields: summary, key_points, sources, confidence.
5. Verify the comparison table from §9.5 is documented (in code comments or README).
6. Verify build_pydantic_ai_graph() compiles and uses the Pydantic AI research node.
7. Verify both raw and Pydantic AI versions exist and can be selected via the pattern parameter.
8. Verify no stubs or TODOs in researcher_pydantic.py.

Report failures to responsible agent. Do not commit until all pass.
```

#### Commit

```bash
git add -A && git commit -m "Phase 6: Pydantic AI researcher agent with typed tools, structured output, dependency injection, comparison with raw function calling"
```

---

### Phase 7: Memory (Short-Term + Episodic) (Day 9)

**Goal:** Short-term memory buffer in Redis with summarization; episodic memory in pgvector; Redis checkpointing for graph state.

**Plan reference:** §14 Phase 7, §10 (Agent Memory), §15.8 (Short-Term Memory), §15.9 (Episodic Memory), §5.5 (Checkpointing).

#### Parallel Work

**infra** (memory stores + checkpointing — depends on Phase 1 Redis + DB):
```
Read IMPLEMENTATION_PLAN.md §10 (Agent Memory) in full: §10.1 (Short-Term), §10.2 (Working Memory), §10.3 (Summarization), §10.4 (Redis Key Structure), §10.5 (Episodic Memory), §5.5 (Checkpointing).

Create the memory subsystem:
1. `src/memory/__init__.py` — empty.
2. `src/memory/short_term.py` — ShortTermMemory class exactly as specified in §10.1 and §15.8:
   - BUFFER_LIMIT = 10, SUMMARIZE_THRESHOLD = 10
   - __init__(redis: redis.asyncio.Redis)
   - async get_messages(session_id) → list[dict] from Redis list
   - async add_message(session_id, message) → RPUSH + summarize if threshold exceeded
   - async _summarize(session_id) → summarize older messages, keep recent, store summary
   - async get_context(session_id) → summary + recent messages as string
3. `src/memory/summarizer.py` — summarization logic using LLM client with SUMMARIZATION_SYSTEM_PROMPT. Called by ShortTermMemory._summarize().
4. `src/memory/episodic.py` — EpisodicMemoryStore class exactly as specified in §10.5 and §15.9:
   - __init__(db_pool, embedding_dim=1536)
   - async store_episodic(session_id, query, output) → generate embedding, insert into episodic_memory.memories
   - async retrieve_relevant(query, top_k=3) → generate embedding, cosine similarity search via pgvector, return formatted string
   - Use OpenAI text-embedding-3-small model for embeddings.
5. Integrate Redis checkpointing into the graph builder:
   - In src/graph/builder.py, update all build_*_graph() functions to compile with checkpointer=RedisSaver(redis_client).
   - The redis_client should be obtained from app state or config.

Write `tests/test_memory.py`:
- Test: ShortTermMemory add_message → verify message stored in Redis list
- Test: buffer holds last 10 messages → add 12 messages, verify only 10 remain
- Test: summarization triggers at threshold 10 → verify summary stored in Redis, buffer trimmed
- Test: get_context returns summary + recent messages
- Test: EpisodicMemoryStore.store_episodic → verify row in episodic_memory.memories with embedding
- Test: EpisodicMemoryStore.retrieve_relevant → verify returns memories with cosine similarity (mock embedding API)
- Test: Redis checkpoint persists graph state → run graph, check checkpoint exists in Redis
```

**graph** (integrate memory into graph flow — depends on infra creating memory stores):
```
Read IMPLEMENTATION_PLAN.md §2.3 (Agent Request Flow — steps 2-3, 6), §10.2 (Working Memory).

Update the graph to load and store memory:
1. Add a memory_loader_node(state) that runs at START:
   - Loads short-term context via ShortTermMemory.get_context(session_id)
   - Loads episodic context via EpisodicMemoryStore.retrieve_relevant(query)
   - Sets memory_context in state
2. Add memory_context to the router_node prompt (it already reads state.get("memory_context", "")).
3. The synthesizer_node already stores episodic memory (from Phase 4) — verify it works with the real EpisodicMemoryStore.
4. Update build_graph() to include memory_loader_node: START→memory_loader→router→...

Coordinate with infra on the memory store interfaces: ShortTermMemory(redis) and EpisodicMemoryStore(db_pool).
```

#### Sequential (after parallel)

**infra** (wire memory into API — after graph integrates memory):
```
Read IMPLEMENTATION_PLAN.md §15.13 (API Routes — run.py).

Update the API to initialize memory stores and pass them to the graph:
1. In src/main.py lifespan: initialize ShortTermMemory(redis) and EpisodicMemoryStore(db_pool) and store in app.state.
2. In src/api/routes/run.py (create if not exists): POST /run endpoint that:
   - Loads memory context (short-term + episodic)
   - Builds the appropriate graph based on pattern parameter
   - Invokes graph with initial_state including memory_context
   - Persists cost log to PostgreSQL
   - Adds user query and assistant response to short-term memory
   - Returns RunResponse (session_id, final_output, quality_warning, cost_summary, trace)
```

#### Review

**reviewer**:
```
Review Phase 7 against IMPLEMENTATION_PLAN.md §14 Phase 7:

1. Run `pytest tests/test_memory.py -v` — all tests pass.
2. Verify ShortTermMemory has BUFFER_LIMIT=10 and SUMMARIZE_THRESHOLD=10.
3. Verify summarization triggers when buffer exceeds 10 messages.
4. Verify summarized context preserves key facts (test with mock LLM summarizer).
5. Verify EpisodicMemoryStore uses pgvector cosine similarity search (embedding <=> query embedding).
6. Verify Redis checkpointing is enabled in all build_*_graph() functions.
7. Verify memory_loader_node loads both short-term and episodic context.
8. Verify POST /run loads memory before graph invocation and stores after.
9. Verify no stubs or TODOs in memory modules.

Report failures to responsible agent. Do not commit until all pass.
```

#### Commit

```bash
git add -A && git commit -m "Phase 7: Short-term memory (Redis buffer + summarization), episodic memory (pgvector), Redis checkpointing, memory integrated into graph and API"
```

---

### Phase 8: Cost Tracking + Observability + Dashboard (Day 10)

**Goal:** Per-agent cost logged to Postgres, execution traces, Streamlit dashboard showing graph + costs + traces, full E2E pipeline working.

**Plan reference:** §14 Phase 8, §11 (Cost Tracking & Observability), §15.11 (Node Tracer), §15.12 (FastAPI App), §15.13 (API Routes), §15.14 (Streamlit Dashboard).

#### Parallel Work

**infra** (observability + API routes + Streamlit — depends on all prior phases):
```
Read IMPLEMENTATION_PLAN.md §11 (Cost Tracking & Observability), §15.11 (Node Tracer), §15.12 (FastAPI App), §15.13 (API Routes), §15.14 (Streamlit Dashboard), §17 (Deployment).

Create the observability layer, complete API, and Streamlit dashboard:
1. `src/observability/__init__.py` — empty.
2. `src/observability/tracer.py` — @trace_node decorator exactly as specified in §11.3 and §15.11. Wraps async node functions, logs entry/exit/duration/status to execution_logs.node_traces.
3. `src/observability/trace_logger.py` — TraceLogger class that persists to execution_logs.node_traces via asyncpg.
4. Apply @trace_node decorator to ALL node functions (coordinate with agents and graph agents — send hub messages requesting they add the decorator to their node functions).
5. `src/api/routes/run.py` — POST /run endpoint as specified in §15.13 (if not already created in Phase 7, complete it now with full cost persistence).
6. `src/api/routes/costs.py` — GET /costs/{session_id} → query cost_tracking.agent_costs, return per-agent breakdown.
7. `src/api/routes/traces.py` — GET /traces/{session_id} → query execution_logs.node_traces, return node execution timeline.
8. `src/api/routes/sessions.py` — session CRUD: POST /sessions (create), GET /sessions (list), GET /sessions/{id} (detail).
9. `src/api/routes/health.py` — update to check DB + Redis + LLM connectivity.
10. Update src/main.py to include ALL routers.
11. `streamlit/dashboard.py` — follow §15.14 exactly:
    - Query input (text area) + pattern selector + session ID
    - POST /run on button click
    - Display final output (with quality warning if applicable)
    - Cost summary: total cost, total tokens, total calls (3 metrics)
    - Per-agent cost breakdown table (agent, calls, tokens, cost, latency)
    - Agent execution trace display
    - Session ID stored in st.session_state for continuity
12. Write `tests/test_api.py`:
    - Test POST /run with mock graph → returns final_output + cost_summary
    - Test GET /costs/{session_id} → returns per-agent cost breakdown
    - Test GET /traces/{session_id} → returns node traces
    - Test GET /health → returns healthy status
    - Test session CRUD
    - Mock the graph invocation for all tests.
```

**agents** (apply tracer decorator — depends on infra creating tracer):
```
Read IMPLEMENTATION_PLAN.md §15.11 (Node Tracer).

Apply the @trace_node decorator to ALL agent node functions:
1. In src/agents/router.py: add @trace_node to router_node
2. In src/agents/researcher.py: add @trace_node to research_node
3. In src/agents/writer.py: add @trace_node to writer_node
4. In src/agents/critic.py: add @trace_node to critic_node
5. In src/agents/synthesizer.py: add @trace_node to synthesizer_node and synthesizer_orchestrator_node
6. In src/agents/orchestrator.py: add @trace_node to orchestrator_node
7. In src/agents/worker.py: add @trace_node to worker_node

Import: from src.observability.tracer import trace_node

The decorator wraps the function, logs to execution_logs.node_traces, and re-raises on error.
```

**graph** (apply tracer to graph-level nodes — depends on infra creating tracer):
```
Apply @trace_node to any graph-level node functions (memory_loader_node, etc.) in src/graph/builder.py.

Verify all graphs (default, orchestrator_worker, critic_loop, map_reduce, pydantic_ai) still compile with the decorated nodes.
```

#### Sequential (after parallel)

**infra** (E2E smoke test):
```
Run the full end-to-end pipeline:
1. `docker compose up --build` — all 4 services start.
2. `curl -X POST localhost:8000/run -H "Content-Type: application/json" -d '{"query": "Explain how transformer attention works", "pattern": "default"}'` — verify response has final_output, cost_summary, trace.
3. `curl localhost:8000/costs/{session_id}` — verify per-agent cost breakdown.
4. `curl localhost:8000/traces/{session_id}` — verify node traces.
5. Open http://localhost:8501 — verify Streamlit dashboard renders: query input, pattern selector, cost table, trace display.
6. Submit a query via Streamlit — verify full pipeline runs and results display.
7. Verify total pipeline cost < $0.05 per query (check cost_summary.total_cost_usd).
8. Test each pattern: default, orchestrator_worker, critic_loop, map_reduce, pydantic_ai.
```

#### Review

**reviewer**:
```
Review Phase 8 against IMPLEMENTATION_PLAN.md §14 Phase 8:

1. Run `pytest tests/test_api.py -v` — all tests pass.
2. Verify POST /run returns RunResponse with session_id, final_output, quality_warning, cost_summary, trace.
3. Verify GET /costs/{session_id} returns per-agent cost breakdown from cost_tracking.agent_costs.
4. Verify GET /traces/{session_id} returns node traces from execution_logs.node_traces.
5. Verify @trace_node decorator is applied to ALL node functions (router, researcher, writer, critic, synthesizer, orchestrator, worker).
6. Verify Streamlit dashboard renders: query input, pattern selector, final output, cost summary (3 metrics), per-agent cost table, trace display.
7. Run E2E: `docker compose up` → submit query via API → verify full pipeline runs, cost < $0.05.
8. Run E2E: submit query via Streamlit → verify dashboard shows results.
9. Test all 5 patterns via API: default, orchestrator_worker, critic_loop, map_reduce, pydantic_ai.
10. Verify no stubs, TODOs, or placeholder code anywhere in the codebase.

Report failures to responsible agent. Do not commit until all pass.
```

#### Commit

```bash
git add -A && git commit -m "Phase 8: Observability (node tracer), complete API (run/costs/traces/sessions/health), Streamlit dashboard, E2E pipeline verified, all 5 patterns working"
```

---

## 5. Phase Summary

| Phase | Days | Agents Active | Key Deliverable |
|-------|------|---------------|-----------------|
| 1. Environment + LangGraph Basics | 1 | graph, agents, infra | Docker stack, StateGraph compiles, LLM client |
| 2. LLM Client + Cost Tracker + Tools | 1 | agents, graph | Cost tracker verified, stubbed tools, all prompts, edge functions |
| 3. Router Agent + Routing | 1 | agents, graph | Router classifies, conditional routing to 4 research nodes |
| 4. Writer + Critic/Revision Loop | 2 | agents, graph | Critic loop with max 3 + convergence, quality warning |
| 5. Orchestrator-Worker + Map-Reduce | 2 | agents, graph | Parallel workers, synthesis, failure handling |
| 6. Pydantic AI Agent | 1 | agents, graph | Pydantic AI researcher, comparison table |
| 7. Memory | 1 | infra, graph | Short-term (Redis) + episodic (pgvector) + checkpointing |
| 8. Cost + Observability + Dashboard | 1 | infra, agents, graph | Tracer, API, Streamlit dashboard, E2E verified |

---

## 6. Cross-Agent Coordination Notes

- **File ownership is strict.** Each agent owns specific directories/files (see §1 Agent Roster). Do not edit files owned by another agent without sending a hub message first.
- **The graph agent owns `src/graph/builder.py`** but needs node functions from the agents agent. The agents agent creates node functions in `src/agents/`; the graph agent imports them. Coordinate via hub if import paths change.
- **The agents agent owns `src/agents/prompts.py`** — all system prompts live here. If any agent needs a new prompt, request it via hub.
- **The infra agent owns `src/main.py`** — all new API routers must be registered here. When a new router is created, send a hub message to infra requesting registration.
- **The @trace_node decorator (Phase 8)** is owned by infra (`src/observability/tracer.py`) but applied by agents and graph to their node functions. Infra creates the decorator first, then sends a hub message to agents and graph to apply it.
- **Memory stores (Phase 7)** are owned by infra. The graph agent's memory_loader_node imports ShortTermMemory and EpisodicMemoryStore from infra's `src/memory/` modules. Coordinate the interfaces via hub before Phase 7 starts.

---

## 7. Validation Checklist (Final)

Before marking the project complete, verify ALL of the following:

- [ ] `docker compose up` starts postgres, redis, api, streamlit without errors
- [ ] `curl localhost:8000/health` returns healthy status
- [ ] LangGraph StateGraph compiles and `draw_mermaid()` produces valid output
- [ ] Router classifies queries into technical/creative/analytical with confidence
- [ ] Low-confidence queries route to fallback agent
- [ ] Writer produces initial draft and revises based on critic feedback
- [ ] Critic produces structured feedback (score, strengths, weaknesses, suggestions)
- [ ] Critic/Revision loop terminates at max 3 revisions or approval (score ≥ 75)
- [ ] Convergence detection stops loop when score doesn't improve
- [ ] quality_warning flag set when max revisions hit without approval
- [ ] Orchestrator decomposes tasks into 2-5 subtasks
- [ ] Workers execute in parallel via asyncio.gather
- [ ] Worker failures handled gracefully (partial results + note)
- [ ] Synthesizer merges worker results into coherent output
- [ ] Pydantic AI researcher produces validated ResearchResult
- [ ] Pydantic AI vs raw function calling comparison documented
- [ ] Short-term memory buffer holds 10 messages in Redis
- [ ] Summarization triggers at threshold 10
- [ ] Episodic memory stores and retrieves via pgvector cosine similarity
- [ ] Redis checkpointing persists graph state
- [ ] Per-agent cost tracked and logged to PostgreSQL
- [ ] Cost summary shows per-agent breakdown (calls, tokens, cost, latency)
- [ ] Total pipeline cost < $0.05 per query on gpt-4o-mini
- [ ] Node execution traces logged to execution_logs.node_traces
- [ ] Streamlit dashboard renders: query input, pattern selector, output, cost table, traces
- [ ] All 5 patterns work via API: default, orchestrator_worker, critic_loop, map_reduce, pydantic_ai
- [ ] All tests pass: `pytest tests/ -v`
- [ ] No stubs, TODOs, or placeholder code anywhere in the codebase
