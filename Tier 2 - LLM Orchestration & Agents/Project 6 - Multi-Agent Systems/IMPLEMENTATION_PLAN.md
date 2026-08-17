# Multi-Agent Systems — Detailed Implementation Plan

> **Elevator pitch:** A LangGraph-orchestrated multi-agent system where a Router classifies queries, an Orchestrator decomposes complex tasks into parallel worker subtasks, a Writer drafts content, and a Critic reviews and triggers revision loops — all coordinated through a typed state graph with conditional edges, per-agent cost/latency tracking, short-term conversation memory with summarization in Redis, persistent execution state in PostgreSQL, and a Streamlit dashboard showing the full agent topology, token costs, and latency breakdowns. One agent is also built with Pydantic AI to compare frameworks head-to-head.

---

## Tier 2 — Project 6

**Prerequisites:** Project 4 (Data Analysis Agent) — tool-use patterns, ReAct loop, function calling. Project 5 (Document Processing Pipeline) — pipeline orchestration, retry logic, structured output, audit logging.

**Downstream:** Project 11 (Customer Support AI System) — the capstone consumes the multi-agent orchestration patterns, cost tracking, and memory management built here.

---

## Table of Contents

1. [Project Goals & Non-Goals](#1-project-goals--non-goals)
2. [Architecture Overview](#2-architecture-overview)
3. [Tech Stack](#3-tech-stack)
4. [Multi-Agent Patterns Overview](#4-multi-agent-patterns-overview)
5. [LangGraph Implementation](#5-langgraph-implementation)
6. [Orchestrator-Worker Pattern](#6-orchestrator-worker-pattern)
7. [Routing Agent](#7-routing-agent)
8. [Critic/Revision Loop](#8-criticrevision-loop)
9. [Pydantic AI Integration](#9-pydantic-ai-integration)
10. [Agent Memory](#10-agent-memory)
11. [Cost Tracking & Observability](#11-cost-tracking--observability)
12. [Database Schema](#12-database-schema)
13. [Project Structure](#13-project-structure)
14. [Implementation Phases](#14-implementation-phases)
15. [Component Specifications](#15-component-specifications)
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
| G1 | **LangGraph state graph orchestration** — build a typed `StateGraph` with nodes (agents), edges, and conditional routing that coordinates the full Router → Research → Write → Critique → Revise pipeline | Graph compiles, visualizes, and executes; every agent invocation flows through the graph; `graph.get_graph().draw_mermaid()` renders the topology |
| G2 | **Orchestrator-Worker pattern** — one orchestrator agent decomposes a complex task into subtasks and delegates to specialist worker agents, then synthesizes results | Orchestrator produces a decomposition plan; ≥ 2 workers execute in parallel; synthesizer combines worker outputs into a coherent result |
| G3 | **Routing pattern (classifier → specialist)** — a Router agent classifies the input query type (technical, creative, analytical) and routes to the appropriate specialist agent | Router classifies ≥ 90% of test queries correctly; wrong routes are recoverable within 1 re-routing attempt |
| G4 | **Critic/Revision loop** — Writer produces a draft, Critic provides structured feedback, Writer revises; max 3 revision rounds before output | Critic produces structured feedback (strengths, weaknesses, suggestions); ≥ 70% of drafts improve on Critic's rubric after revision; loop terminates at max 3 rounds or Critic approval |
| G5 | **Map-Reduce parallel agents** — fan out a list of subtasks to parallel worker agents, then reduce/synthesize results | Map phase dispatches ≥ 3 subtasks concurrently; reduce phase merges all worker outputs into a single coherent result within 2× the latency of a single worker |
| G6 | **Supervisor pattern** — a supervisor node monitors worker status, decides next steps, and handles failures | Supervisor detects worker failures and routes to fallback; ≥ 1 failure scenario tested and recovered without pipeline crash |
| G7 | **Pydantic AI agent** — build one agent (the Researcher) using Pydantic AI framework with typed tools and structured outputs, and compare it with the raw function-calling approach from P4 | Pydantic AI agent produces equivalent results to the raw function-calling version; comparison table documents trade-offs (boilerplate, type safety, streaming, control) |
| G8 | **Short-term memory (buffer + summarization)** — conversation buffer stores recent turns in Redis; after N turns, older messages are summarized to compress context | Buffer holds last 10 messages; summarization triggers after 10 turns; summarized context preserves key facts with ≥ 80% factual recall in tests |
| G9 | **Cost & latency tracking** — per-agent token usage (prompt + completion), total pipeline cost, latency per node, all logged to Postgres and visualized in Streamlit | Every agent call logs tokens, cost, and latency; Streamlit dashboard shows per-agent and per-pattern breakdowns; total pipeline cost < $0.05 per query on gpt-4o-mini |
| G10 | **Multi-agent failure mode mitigations** — detect and prevent infinite loops, cascading errors, information isolation, and deadlocks | Loop counter caps revisions at 3; error isolation prevents one agent's failure from crashing the pipeline; deadlock detection via graph cycle analysis; all failure modes have unit tests |
| G11 | **Deployable end-to-end** — Docker Compose with Postgres, Redis, FastAPI backend, Streamlit dashboard | `docker compose up` starts all four services; user submits a query via Streamlit, watches the multi-agent graph execute, and sees cost/latency dashboard |

### Non-Goals (explicitly out of scope)

- Autonomous web browsing with real search APIs — v1 uses stubbed/knowledge-base tools; real search integration is a v2 concern
- Multi-tenant isolation — v1 is a single-user local tool
- Fine-tuning agent models — v1 uses prompt engineering + frontier models
- Human-in-the-loop approval gates — v1 is fully autonomous (human review is a v2 concern)
- Agent-to-agent negotiation protocols — v1 uses fixed graph topology, not dynamic agent spawning
- Real-time streaming of agent intermediate results via SSE — v1 returns complete results (streaming is a v2 possibility)
- Production-grade vector DB (Pinecone, Weaviate) — v1 uses pgvector extension in PostgreSQL for episodic memory
- Multi-language agent communication — v1 agents communicate in English
- Custom agent evaluation framework — v1 uses manual + automated rubric checks, not a dedicated eval harness (that is Project 7)

---

## 2. Architecture Overview

### 2.1 System Architecture

```
                              ┌──────────────────────────────────────────────────┐
                              │                 User (Browser)                    │
                              │   Streamlit Dashboard                             │
                              │   - Submit research/writing queries               │
                              │   - Watch multi-agent graph execute               │
                              │   - View cost/latency per agent                   │
                              │   - Inspect agent reasoning traces                │
                              │   - Browse episodic memories                      │
                              └─────────────────┬────────────────────────────────┘
                                                │ HTTP (POST /run)
                              ┌─────────────────▼────────────────────────────────┐
                              │              FastAPI Backend                      │
                              │  ┌──────────────────────────────────────────┐    │
                              │  │         LangGraph Orchestrator            │    │
                              │  │  (StateGraph: nodes, edges, routing)      │    │
                              │  │                                           │    │
                              │  │  ┌─────────┐  ┌──────────┐  ┌─────────┐  │    │
                              │  │  │ Short-  │  │  LLM     │  │ Cost    │  │    │
                              │  │  │ Term    │  │  Client  │  │ Tracker │  │    │
                              │  │  │ Memory  │  │  (OpenAI)│  │         │  │    │
                              │  │  │ (Redis) │  │          │  │         │  │    │
                              │  │  └─────────┘  └────┬─────┘  └────┬────┘  │    │
                              │  │                      │              │       │    │
                              │  │               ┌──────▼──────────────▼───┐  │    │
                              │  │               │     Agent Nodes          │  │    │
                              │  │               │                          │  │    │
                              │  │               │  ┌────────┐  ┌────────┐ │  │    │
                              │  │               │  │ Router │  │Research│ │  │    │
                              │  │               │  │ Agent  │  │ Agent  │ │  │    │
                              │  │               │  └───┬────┘  └───┬────┘ │  │    │
                              │  │               │      │           │      │  │    │
                              │  │               │  ┌───▼────┐  ┌───▼────┐ │  │    │
                              │  │               │  │ Writer │  │ Critic │ │  │    │
                              │  │               │  │ Agent  │  │ Agent  │ │  │    │
                              │  │               │  └───┬────┘  └───┬────┘ │  │    │
                              │  │               │      │           │      │  │    │
                              │  │               │  ┌───▼───────────▼────┐ │  │    │
                              │  │               │  │  Revision Loop      │ │  │    │
                              │  │               │  │  (max 3 rounds)     │ │  │    │
                              │  │               │  └─────────────────────┘ │  │    │
                              │  │               └──────────────────────────┘  │    │
                              │  └──────────────────────────────────────────────┘    │
                              │                         │                           │
                              │           ┌─────────────▼──────────────┐            │
                              │           │    Long-Term Memory         │            │
                              │           │  ┌────────────┐ ┌────────┐ │            │
                              │           │  │ pgvector   │ │ Redis  │ │            │
                              │           │  │ episodic   │ │ check- │ │            │
                              │           │  │ memory     │ │ point  │ │            │
                              │           │  └────────────┘ └────────┘ │            │
                              │           └─────────────┬──────────────┘            │
                              └─────────────────────────┼───────────────────────────┘
                                                        │
                              ┌─────────────────────────▼───────────────────────────┐
                              │              PostgreSQL 16                          │
                              │  - agent_meta schema (sessions, messages)           │
                              │  - cost_tracking schema (per-agent tokens/cost)     │
                              │  - episodic_memory schema (pgvector embeddings)     │
                              │  - execution_logs schema (node traces)              │
                              └─────────────────────────────────────────────────────┘
```

### 2.2 LangGraph State Graph Topology

This is the core of the project — a LangGraph `StateGraph` that defines the multi-agent flow as a typed graph with conditional routing.

```
                        ┌─────────────────────────────────────────────────────────┐
                        │                  LangGraph StateGraph                    │
                        │                                                         │
                        │  State: TypedDict {                                     │
                        │    query: str                                           │
                        │    query_type: str          # technical|creative|analytical
                        │    research_notes: str                                  │
                        │    draft: str                                           │
                        │    critic_feedback: str                                 │
                        │    revision_count: int                                  │
                        │    final_output: str                                    │
                        │    cost_log: list[dict]     # per-agent cost entries    │
                        │    messages: list[dict]     # conversation buffer       │
                        │    memory_context: str      # retrieved episodic mem   │
                        │    status: str              # running|completed|error  │
                        │  }                                                       │
                        │                                                         │
                        └─────────────────────────────────────────────────────────┘

                                    START
                                      │
                                      ▼
                              ┌───────────────┐
                              │  router_node  │
                              │  (Router Agent│
                              │   classifies  │
                              │   query type) │
                              └───────┬───────┘
                                      │
                          ┌───────────┼───────────┐
                          │           │           │
                     technical    creative    analytical
                          │           │           │
                          ▼           ▼           ▼
                   ┌──────────┐ ┌──────────┐ ┌──────────┐
                   │research_ │ │research_ │ │research_ │
                   │node_tech │ │node_creat│ │node_analy│
                   └─────┬────┘ └─────┬────┘ └─────┬────┘
                         │            │            │
                         └────────────┼────────────┘
                                      │
                                      ▼
                              ┌───────────────┐
                              │  writer_node  │
                              │  (Writer Agent│
                              │   produces    │
                              │   draft)      │
                              └───────┬───────┘
                                      │
                                      ▼
                              ┌───────────────┐
                              │  critic_node  │
                              │  (Critic Agent│
                              │   reviews     │
                              │   draft)      │
                              └───────┬───────┘
                                      │
                             ┌────────┴────────┐
                             │                 │
                        approved           needs_revision
                             │                 │
                             │                 ▼
                             │          ┌──────────────┐
                             │          │ revision_count│
                             │          │   < 3?        │
                             │          └─────┬────────┘
                             │                │
                             │           ┌────┴────┐
                             │           │         │
                             │          YES       NO
                             │           │         │
                             │           ▼         │
                             │    ┌────────────┐   │
                             │    │writer_node │   │
                             │    │(revise)    │   │
                             │    └─────┬──────┘   │
                             │          │          │
                             │          ▼          │
                             │    ┌────────────┐   │
                             │    │critic_node │◄──┘
                             │    └─────┬──────┘
                             │          │
                             └──────────┘
                                      │
                                      ▼
                              ┌───────────────┐
                              │ synthesizer_  │
                              │ node          │
                              │ (final output │
                              │  + cost summary)│
                              └───────┬───────┘
                                      │
                                      ▼
                                    END
```

### 2.3 Agent Request Flow (Single Turn)

```
1. User submits query via Streamlit UI → POST /run {query, session_id, pattern}
2. FastAPI backend loads session memory (short-term buffer + summarization from Redis)
3. Retrieve relevant episodic memories from pgvector (long-term memory)
4. LangGraph orchestrator compiles and invokes the state graph:
   a. ROUTER NODE:    Router Agent classifies query → {technical, creative, analytical}
   b. CONDITIONAL EDGE: routes to the appropriate research node
   c. RESEARCH NODE:  Research Agent gathers information using tools:
                      - web_search_stub(query) → simulated search results
                      - knowledge_base_lookup(topic) → retrieve from local KB
                      - (Pydantic AI version: typed tools with structured output)
   d. WRITER NODE:    Writer Agent produces a draft based on research notes
   e. CRITIC NODE:    Critic Agent reviews draft → structured feedback:
                      {strengths, weaknesses, suggestions, approved: bool}
   f. CONDITIONAL EDGE:
      - If approved → synthesizer node
      - If not approved AND revision_count < 3 → back to writer node
      - If not approved AND revision_count >= 3 → synthesizer node (best effort)
   g. SYNTHESIZER NODE: combines final output + cost summary + memory storage
5. Cost tracker logs per-agent tokens, cost, latency to Postgres
6. Episodic memory stores key facts from this interaction as embeddings
7. Streamlit UI renders: final output + agent graph visualization + cost dashboard
```

### 2.4 Three Patterns, One System

The project implements multiple multi-agent design patterns, each exercisable independently via the API `pattern` parameter:

```
Pattern 1: Orchestrator-Worker
┌──────────────┐
│ Orchestrator │──── decompose task
└──────┬───────┘
       │
  ┌────┼────┬─────────┐
  ▼    ▼    ▼         ▼
┌────┐┌────┐┌──────┐┌──────┐
│W1  ││W2  ││W3   ││W4   │
│tech││creat││analy││fact │
└──┬─┘└──┬─┘└──┬───┘└──┬───┘
   │     │     │       │
   └─────┼─────┼───────┘
         ▼     ▼
  ┌──────────────┐
  │ Synthesizer  │──── combine results
  └──────────────┘

Pattern 2: Routing (Classifier → Specialist)
┌──────────┐
│ Router / │──── classify
│Classifier│
└────┬─────┘
     │
     ├── technical ──→ ┌──────────────┐
     │                 │ Technical    │
     │                 │ Specialist   │
     │                 └──────────────┘
     ├── creative  ──→ ┌──────────────┐
     │                 │ Creative     │
     │                 │ Specialist   │
     │                 └──────────────┘
     └── analytical ──→ ┌──────────────┐
                       │ Analytical   │
                       │ Specialist   │
                       └──────────────┘

Pattern 3: Critic/Revision Loop
┌────────┐    ┌────────┐    ┌──────────┐
│ Writer │───→│ Critic │───→│ approved?│
└────────┘    └────────┘    └────┬─────┘
    ▲                            │
    │           NO               │
    └────────────────────────────┘
    │
    │           YES
    └──────────────→ END

Pattern 4: Map-Reduce Parallel
┌──────────┐
│  Mapper  │──── split into subtasks
└────┬─────┘
     │
  ┌──┼──┬──┬──┐
  ▼  ▼  ▼  ▼  ▼  (parallel)
 W1 W2 W3 W4 W5
  │  │  │  │  │
  └──┼──┴──┼──┘
     ▼     ▼
┌──────────┐
│ Reducer  │──── merge results
└──────────┘

Pattern 5: Supervisor
┌────────────┐
│ Supervisor │◄──────────┐
└─────┬──────┘           │
      │ decide next      │
      ▼                  │
┌──────────┐             │
│  Worker  │──── done? ──┘
└──────────┘    (loop until complete)
```

### 2.5 Relationship to Prior Projects

| Project | What We Reuse | What We Extend |
|---------|---------------|----------------|
| **P4 — Data Analysis Agent** | Tool-use patterns, function calling, ReAct loop, LLM client with retry, cost tracking per call | Single agent → multi-agent graph; static tool list → dynamic routing; single cost log → per-agent cost attribution |
| **P5 — Document Processing Pipeline** | Pipeline orchestration, retry logic (tenacity), structured output (Pydantic), audit logging, Docker Compose patterns | Sequential pipeline → conditional graph; fixed stages → dynamic agent routing; single LLM call → multi-agent collaboration |
| **P11 — Customer Support AI (downstream)** | Consumes: orchestrator-worker pattern for ticket routing, critic/revision loop for response quality, cost tracking for budget monitoring, memory for conversation continuity | — |

---

## 3. Tech Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **Language** | Python 3.11+ | Ecosystem maturity for AI/ML, async support, type hints, structural pattern matching |
| **Agent Orchestration** | LangGraph 0.2+ | First-class state graph with typed state, conditional edges, checkpointing, and visualization; the industry standard for multi-agent flows |
| **Structured Agent Framework** | Pydantic AI 0.0.13+ | Type-safe agent definitions with Pydantic v2 validation, dependency injection, and structured outputs; used for the Researcher agent to compare with raw function calling |
| **Web Framework (API)** | FastAPI | Async, auto OpenAPI docs, Pydantic validation, WebSocket support for future streaming |
| **Dashboard UI** | Streamlit | Fastest path to a monitoring dashboard; renders agent graph, cost tables, latency charts natively |
| **Database** | PostgreSQL 16 | Mature, reliable, pgvector extension for episodic memory embeddings, JSONB for flexible agent state |
| **Vector Search** | pgvector 0.7+ | In-database vector similarity search; avoids a separate vector DB service for v1 simplicity |
| **Memory / Cache** | Redis 7 | Short-term conversation buffer, LangGraph checkpoint store, TTL-based memory eviction |
| **ORM / DB Driver** | SQLAlchemy 2.0 (async) + asyncpg | Async DB access; SQLAlchemy for schema management, asyncpg for fast raw queries |
| **Redis Client** | redis-py 5+ (async) | Async Redis access for memory buffer and checkpointing |
| **LLM** | `gpt-4o-mini` (OpenAI) | Fast, cheap, strong at function calling and structured output. Cost target: < $0.05 per query |
| **LLM Client** | OpenAI Python SDK 1.40+ | Native async support, structured output via `beta.chat.completions.parse()`, token usage in response |
| **Structured Output** | Pydantic v2 | Validation, serialization, JSON schema generation for LLM structured output |
| **Retry Logic** | tenacity | Exponential backoff for LLM API calls (rate limits, timeouts) — same pattern as P5 |
| **Observability** | LangSmith (optional) / LangFuse (optional) | Agent execution tracing, token tracking, prompt versioning; optional integration via env vars |
| **Containerization** | Docker + Docker Compose | Reproducible multi-service orchestration: postgres, redis, api, streamlit |
| **Testing** | pytest + pytest-asyncio + httpx | Async test support, FastAPI-native test client, mock LLM responses |
| **Linting** | ruff | Fast, replaces flake8 + isort + black |

### Why LangGraph over raw function-calling or LangChain AgentExecutor?

**P4 used raw function calling for a single agent.** That approach gives full control over the ReAct loop but does not scale to multi-agent coordination — there is no native concept of state, conditional routing, parallel execution, or checkpointing.

**LangGraph provides:**
1. **Typed state** — a `TypedDict` flows through the graph; every node reads and writes typed fields
2. **Conditional edges** — route to different nodes based on state (e.g., `approved` → END, `needs_revision` → writer)
3. **Parallel execution** — fan out to multiple worker nodes simultaneously (map-reduce)
4. **Checkpointing** — persist graph state to Redis; resume from any node after a crash
5. **Visualization** — `graph.get_graph().draw_mermaid()` renders the topology for the dashboard
6. **Cycle support** — the critic/revision loop is a real cycle in the graph, not a `while` loop hidden in code

LangChain's `AgentExecutor` wraps the agent loop in abstractions that obscure routing and make multi-agent coordination awkward. LangGraph exposes the graph as a first-class object, which is exactly what we need.

### Why Pydantic AI for one agent (and not all)?

**Comparison, not replacement.** P4 used raw OpenAI function calling. This project builds the Researcher agent in **both** styles — raw function calling (the default in the LangGraph nodes) and Pydantic AI (an alternative implementation) — to produce a head-to-head comparison table documenting trade-offs in boilerplate, type safety, streaming, error handling, and control. The Pydantic AI version is invoked when `pattern=pydantic_ai` is passed to the API.

---

## 4. Multi-Agent Patterns Overview

This project implements five canonical multi-agent design patterns. Each is exercisable independently via the API `pattern` parameter, and all are integrated into the default pipeline.

### 4.1 Orchestrator-Worker Pattern

```
┌──────────────┐
│ Orchestrator │  1. Receives complex task
│              │  2. Decomposes into subtasks
│              │  3. Assigns subtasks to workers
└──────┬───────┘  4. Waits for all workers
       │          5. Synthesizes results
       │
  ┌────┼────┬─────────┐
  ▼    ▼    ▼         ▼
┌────┐┌────┐┌──────┐┌──────┐
│W1  ││W2  ││W3   ││W4   │  Workers execute
│tech││crea││analy││fact │  subtasks in parallel
└──┬─┘└──┬─┘└──┬───┘└──┬───┘
   │     │     │       │
   └─────┼─────┼───────┘
         ▼     ▼
  ┌──────────────┐
  │ Synthesizer  │  Combines worker outputs
  │ (Orchestrator│  into coherent result
  │  phase 2)    │
  └──────────────┘
```

**When to use:** The task is genuinely decomposable into independent subtasks (e.g., "research the impact of AI on healthcare, education, and finance" → three parallel research workers).

**Key decisions:**
- The orchestrator uses an LLM call to produce a **decomposition plan** (a list of subtask descriptions)
- Workers execute via `asyncio.gather()` for true parallelism
- The synthesizer merges results — it does not just concatenate; it produces a coherent narrative

**Failure modes:**
- Over-decomposition: too many tiny subtasks → high cost, low quality. Mitigation: cap at 5 workers
- Worker failure: one worker errors → orchestrator retries once, then proceeds with partial results and a note

### 4.2 Router Agent (Classification-Based Routing)

```
┌──────────┐
│  Router  │  1. Receives query
│ /Classifier│  2. LLM classifies into category
└────┬─────┘  3. Conditional edge routes to specialist
     │
     ├── technical ──→ Technical Specialist Agent
     ├── creative  ──→ Creative Specialist Agent
     ├── analytical ──→ Analytical Specialist Agent
     └── unknown   ──→ Fallback Agent (generalist)
```

**When to use:** Different query types require different system prompts, tools, or models. Routing avoids a single bloated prompt.

**Key decisions:**
- The router uses a **fast, cheap LLM call** (gpt-4o-mini) with a structured output schema (`QueryClassification`) to classify
- Classification includes a confidence score; low confidence → fallback agent
- The fallback agent is a generalist that handles any query type adequately

**Failure modes:**
- Misclassification: technical query routed to creative agent. Mitigation: fallback agent + re-routing on critic rejection
- Router LLM failure: classification call errors. Mitigation: default to fallback agent

### 4.3 Critic/Revision Loop

```
┌────────┐    ┌────────┐    ┌──────────┐
│ Writer │───→│ Critic │───→│ approved?│
│ Agent  │    │ Agent  │    └────┬─────┘
└────────┘    └────────┘         │
    ▲                     ┌──────┴──────┐
    │                     │             │
    │                    YES           NO
    │                     │             │
    │                     ▼             ▼
    │                   END    ┌──────────────┐
    │                          │ revision_count│
    │                          │  < max (3)?  │
    │                          └──────┬───────┘
    │                            ┌────┴────┐
    │                            │         │
    │                           YES       NO
    └───────────────────────────┘         │
                                          ▼
                                        END (best effort)
```

**When to use:** Quality matters more than latency. The critic enforces a rubric; the writer iterates until the critic approves or the max revision count is reached.

**Key decisions:**
- The critic outputs **structured feedback** (strengths, weaknesses, suggestions, score, approved boolean)
- The writer receives the critic's feedback as additional context for the next draft
- **Convergence detection:** if the critic's score does not improve between rounds, terminate early (avoid wasting tokens on a stuck draft)
- Max 3 revisions — after that, output the best version with a "quality warning" flag

**Failure modes:**
- Infinite loop: critic never approves. Mitigation: hard cap at 3 + convergence detection
- Oscillation: draft quality bounces up and down. Mitigation: track score history; if no improvement over 2 rounds, terminate
- Critic too lenient: always approves. Mitigation: rubric-based scoring with explicit thresholds (score ≥ 75 = approved)

### 4.4 Map-Reduce Parallel Agents

```
┌──────────┐
│  Mapper  │──── split task into N subtasks
└────┬─────┘
     │
  ┌──┼──┬──┬──┐
  ▼  ▼  ▼  ▼  ▼  (asyncio.gather — parallel)
 W1 W2 W3 W4 W5
  │  │  │  │  │
  └──┼──┴──┼──┘
     ▼     ▼
┌──────────┐
│ Reducer  │──── merge all worker results
└──────────┘
```

**When to use:** A task can be split into N independent chunks, each processed the same way, then merged. Example: "summarize these 5 documents" → 5 parallel summarizers → 1 reducer merges summaries.

**Key decisions:**
- The mapper is an LLM call that produces a list of subtask strings
- Workers are the same agent function invoked with different inputs via `asyncio.gather()`
- The reducer is an LLM call that merges results into a single coherent output
- Error handling: if a worker fails, the reducer proceeds with N-1 results and notes the gap

**Difference from Orchestrator-Worker:** In map-reduce, all workers are identical (same agent, different inputs). In orchestrator-worker, workers are specialists (different agents, different capabilities).

### 4.5 Supervisor Pattern

```
┌────────────┐
│ Supervisor │◄──────────┐
│            │           │
│  monitors  │           │
│  worker    │           │
│  status    │           │
└─────┬──────┘           │
      │                  │
      │ decide           │
      │ next step        │
      ▼                  │
┌──────────┐             │
│  Worker  │──── status?─┘
│  Agent   │    (done |
│          │     error |
└──────────┘     retry)
```

**When to use:** The workflow is dynamic — the next step depends on the outcome of the current step, and the set of possible steps is known but the order is not.

**Key decisions:**
- The supervisor is a node that inspects state and decides which worker to invoke next (or terminate)
- Implemented as a conditional edge function that returns the next node name
- Handles failures: if a worker returns an error, the supervisor can retry, route to a fallback, or terminate

**Difference from Orchestrator-Worker:** The supervisor is a **loop controller** (decides next step iteratively), while the orchestrator is a **one-shot decomposer** (splits task upfront, then synthesizes).

### 4.6 Pattern Selection Guide

| Pattern | Use When | Cost | Latency | Complexity |
|---------|----------|------|---------|------------|
| **Orchestrator-Worker** | Task decomposes into independent specialist subtasks | High (N+1 LLM calls) | Medium (parallel) | Medium |
| **Routing** | Different query types need different handling | Low (1 + 1 LLM calls) | Low | Low |
| **Critic/Revision** | Quality > latency; iterative improvement needed | Medium-High (2-8 LLM calls) | High (sequential loop) | Medium |
| **Map-Reduce** | Identical processing on N independent chunks | High (N+1 LLM calls) | Medium (parallel) | Low |
| **Supervisor** | Dynamic workflow; next step depends on outcome | Variable | Variable | High |

---

## 5. LangGraph Implementation

### 5.1 StateGraph Definition

The state graph is the backbone of the system. Every agent is a node; every routing decision is a conditional edge.

```python
from typing import TypedDict, Literal
from langgraph.graph import StateGraph, START, END

class AgentState(TypedDict, total=False):
    """Typed state that flows through every node in the graph."""
    # Input
    query: str
    session_id: str
    pattern: str                    # "default" | "orchestrator_worker" | "routing" | "critic_loop" | "map_reduce" | "supervisor" | "pydantic_ai"

    # Router output
    query_type: str                 # "technical" | "creative" | "analytical" | "unknown"
    router_confidence: float

    # Research output
    research_notes: str
    subtasks: list[str]             # for orchestrator-worker / map-reduce
    worker_results: list[str]       # parallel worker outputs

    # Writer output
    draft: str
    revision_count: int

    # Critic output
    critic_feedback: str
    critic_score: int               # 0-100
    critic_approved: bool
    score_history: list[int]        # for convergence detection

    # Final output
    final_output: str
    quality_warning: bool           # True if max revisions hit without approval

    # Memory
    messages: list[dict]            # conversation buffer
    memory_context: str             # retrieved episodic memories

    # Cost & observability
    cost_log: list[dict]            # per-agent {agent, tokens, cost, latency_ms}
    status: str                     # "running" | "completed" | "error"
    error: str | None
```

### 5.2 Node Functions

Each node is an async function that takes `AgentState` and returns a partial state update:

```python
async def router_node(state: AgentState) -> dict:
    """Classify the query type using a fast LLM call."""
    query = state["query"]
    memory_ctx = state.get("memory_context", "")

    classification, cost = await llm_client.structured_complete(
        response_format=QueryClassification,
        system_prompt=ROUTER_SYSTEM_PROMPT,
        user_prompt=f"Query: {query}\n\nPrior context: {memory_ctx}",
    )

    cost_tracker.record("router", cost)
    return {
        "query_type": classification.category,
        "router_confidence": classification.confidence,
        "cost_log": state.get("cost_log", []) + [cost_tracker.last_entry("router")],
    }


async def research_node(state: AgentState) -> dict:
    """Gather information using tools (web_search_stub, knowledge_base_lookup)."""
    query = state["query"]
    query_type = state["query_type"]

    # Tool calls (stubbed for v1)
    search_results = await web_search_stub(query)
    kb_results = await knowledge_base_lookup(query_type, query)

    research_notes = f"Web results:\n{search_results}\n\nKB results:\n{kb_results}"
    return {"research_notes": research_notes}


async def writer_node(state: AgentState) -> dict:
    """Produce or revise a draft based on research notes and critic feedback."""
    query = state["query"]
    research = state.get("research_notes", "")
    feedback = state.get("critic_feedback", "")
    revision_count = state.get("revision_count", 0)

    if revision_count == 0:
        prompt = f"Write a response to: {query}\n\nResearch: {research}"
    else:
        prompt = (
            f"Revise your previous draft based on critic feedback.\n\n"
            f"Query: {query}\nResearch: {research}\n\n"
            f"Previous draft: {state['draft']}\n\n"
            f"Critic feedback: {feedback}\n\n"
            f"Produce only the revised content."
        )

    draft, cost = await llm_client.complete(WRITER_SYSTEM_PROMPT, prompt)
    cost_tracker.record("writer", cost)
    return {
        "draft": draft,
        "revision_count": revision_count + 1,
        "cost_log": state.get("cost_log", []) + [cost_tracker.last_entry("writer")],
    }


async def critic_node(state: AgentState) -> dict:
    """Review the draft against a rubric; produce structured feedback."""
    draft = state["draft"]
    query = state["query"]

    feedback, cost = await llm_client.structured_complete(
        response_format=CriticFeedback,
        system_prompt=CRITIC_SYSTEM_PROMPT,
        user_prompt=f"Query: {query}\n\nDraft to review:\n{draft}",
    )

    approved = feedback.score >= 75
    score_history = state.get("score_history", []) + [feedback.score]

    cost_tracker.record("critic", cost)
    return {
        "critic_feedback": feedback.feedback_text,
        "critic_score": feedback.score,
        "critic_approved": approved,
        "score_history": score_history,
        "cost_log": state.get("cost_log", []) + [cost_tracker.last_entry("critic")],
    }


async def synthesizer_node(state: AgentState) -> dict:
    """Produce final output and store episodic memory."""
    draft = state["draft"]
    approved = state.get("critic_approved", False)
    revision_count = state.get("revision_count", 0)

    if approved:
        final_output = draft
        quality_warning = False
    else:
        final_output = draft
        quality_warning = True  # max revisions hit without approval

    # Store episodic memory
    await memory_store.store_episodic(
        session_id=state["session_id"],
        query=state["query"],
        output=final_output,
    )

    return {
        "final_output": final_output,
        "quality_warning": quality_warning,
        "status": "completed",
    }
```

### 5.3 Conditional Edges

```python
def route_after_router(state: AgentState) -> str:
    """Route to the appropriate research node based on query type."""
    query_type = state.get("query_type", "unknown")
    confidence = state.get("router_confidence", 0.0)

    if confidence < 0.6:
        return "research_fallback"
    return f"research_{query_type}"


def route_after_critic(state: AgentState) -> str:
    """Route based on critic approval and revision count."""
    if state.get("critic_approved", False):
        return "synthesizer"

    revision_count = state.get("revision_count", 0)
    if revision_count >= 3:
        return "synthesizer"  # best effort

    # Convergence detection: if score hasn't improved in 2 rounds, stop
    score_history = state.get("score_history", [])
    if len(score_history) >= 2:
        if score_history[-1] <= score_history[-2]:
            return "synthesizer"  # not improving, stop

    return "writer"  # revise
```

### 5.4 Building and Compiling the Graph

```python
def build_graph() -> CompiledGraph:
    """Build and compile the LangGraph state graph."""
    graph = StateGraph(AgentState)

    # Add nodes
    graph.add_node("router", router_node)
    graph.add_node("research_technical", research_node)
    graph.add_node("research_creative", research_node)
    graph.add_node("research_analytical", research_node)
    graph.add_node("research_fallback", research_node)
    graph.add_node("writer", writer_node)
    graph.add_node("critic", critic_node)
    graph.add_node("synthesizer", synthesizer_node)

    # Add edges
    graph.add_edge(START, "router")

    # Conditional routing after router
    graph.add_conditional_edges(
        "router",
        route_after_router,
        {
            "research_technical": "research_technical",
            "research_creative": "research_creative",
            "research_analytical": "research_analytical",
            "research_fallback": "research_fallback",
        },
    )

    # All research nodes converge to writer
    for node in ["research_technical", "research_creative", "research_analytical", "research_fallback"]:
        graph.add_edge(node, "writer")

    # Writer → Critic
    graph.add_edge("writer", "critic")

    # Conditional routing after critic (revision loop)
    graph.add_conditional_edges(
        "critic",
        route_after_critic,
        {
            "synthesizer": "synthesizer",
            "writer": "writer",
        },
    )

    # Synthesizer → END
    graph.add_edge("synthesizer", END)

    return graph.compile(
        checkpointer=RedisSaver(redis_client),
        interrupt_before=None,
    )
```

### 5.5 Checkpointing and Persistence

LangGraph supports checkpointing — persisting graph state after each node execution so the graph can be resumed after a crash or interruption.

```python
from langgraph.checkpoint.redis import RedisSaver

# Redis-backed checkpoint store
redis_client = redis.asyncio.Redis(host="localhost", port=6379, db=0)
checkpointer = RedisSaver(redis_client)

# When compiling the graph, pass the checkpointer:
compiled_graph = graph.compile(checkpointer=checkpointer)

# When invoking, pass a thread_id (maps to session_id):
result = await compiled_graph.ainvoke(
    initial_state,
    config={"configurable": {"thread_id": session_id}},
)

# To resume from the last checkpoint:
result = await compiled_graph.ainvoke(
    None,  # None = resume from last checkpoint
    config={"configurable": {"thread_id": session_id}},
)
```

**Checkpointing enables:**
- **Crash recovery:** if the API process restarts mid-execution, the graph resumes from the last completed node
- **Human-in-the-loop (v2):** interrupt before a node, wait for human approval, then resume
- **Time travel:** load a previous checkpoint and re-execute from that point with modified state
- **Debugging:** inspect the state at every node boundary via `checkpointer.aget(config)`

---

## 6. Orchestrator-Worker Pattern

### 6.1 Overview

The orchestrator-worker pattern decomposes a complex task into independent subtasks, dispatches them to specialist worker agents in parallel, and synthesizes the results into a coherent output.

### 6.2 Orchestrator Node

The orchestrator uses an LLM call to produce a **decomposition plan** — a list of subtask descriptions:

```python
class DecompositionPlan(BaseModel):
    """Structured output from the orchestrator's decomposition step."""
    subtasks: list[str] = Field(
        description="List of independent subtask descriptions",
        min_length=2,
        max_length=5,
    )
    strategy: str = Field(description="Brief explanation of the decomposition strategy")


async def orchestrator_node(state: AgentState) -> dict:
    """Decompose the query into subtasks."""
    query = state["query"]

    plan, cost = await llm_client.structured_complete(
        response_format=DecompositionPlan,
        system_prompt=ORCHESTRATOR_SYSTEM_PROMPT,
        user_prompt=f"Decompose this task into 2-5 independent subtasks:\n{query}",
    )

    cost_tracker.record("orchestrator", cost)
    return {
        "subtasks": plan.subtasks,
        "cost_log": state.get("cost_log", []) + [cost_tracker.last_entry("orchestrator")],
    }
```

### 6.3 Worker Nodes (Parallel Execution)

Workers execute subtasks in parallel via `asyncio.gather()`:

```python
async def worker_node(state: AgentState) -> dict:
    """Execute all subtasks in parallel using specialist workers."""
    subtasks = state["subtasks"]
    query_type = state.get("query_type", "general")

    async def execute_single(subtask: str, index: int) -> str:
        """One worker processes one subtask."""
        result, cost = await llm_client.complete(
            system_prompt=WORKER_SYSTEM_PROMPT.format(role=query_type),
            user_prompt=f"Subtask: {subtask}\n\nContext: {state['query']}",
        )
        cost_tracker.record(f"worker_{index}", cost)
        return result

    # Fan out — all workers run concurrently
    results = await asyncio.gather(
        *[execute_single(st, i) for i, st in enumerate(subtasks)],
        return_exceptions=True,
    )

    # Handle worker failures
    worker_results = []
    for i, result in enumerate(results):
        if isinstance(result, Exception):
            worker_results.append(f"[Worker {i} failed: {result}]")
        else:
            worker_results.append(result)

    return {
        "worker_results": worker_results,
        "cost_log": state.get("cost_log", []) + cost_tracker.all_entries_since("orchestrator"),
    }
```

### 6.4 Synthesizer Node

The synthesizer merges worker outputs into a coherent narrative:

```python
async def synthesizer_orchestrator_node(state: AgentState) -> dict:
    """Synthesize parallel worker results into a coherent output."""
    worker_results = state["worker_results"]
    subtasks = state["subtasks"]
    query = state["query"]

    paired = "\n\n".join(
        f"## Subtask: {st}\n{res}" for st, res in zip(subtasks, worker_results)
    )

    synthesis, cost = await llm_client.complete(
        system_prompt=SYNTHESIZER_SYSTEM_PROMPT,
        user_prompt=(
            f"Original query: {query}\n\n"
            f"Worker results:\n{paired}\n\n"
            f"Synthesize these into a single coherent response."
        ),
    )

    cost_tracker.record("synthesizer", cost)
    return {
        "final_output": synthesis,
        "status": "completed",
        "cost_log": state.get("cost_log", []) + [cost_tracker.last_entry("synthesizer")],
    }
```

### 6.5 Orchestrator-Worker Graph

```python
def build_orchestrator_worker_graph() -> CompiledGraph:
    graph = StateGraph(AgentState)
    graph.add_node("orchestrator", orchestrator_node)
    graph.add_node("workers", worker_node)
    graph.add_node("synthesizer", synthesizer_orchestrator_node)

    graph.add_edge(START, "orchestrator")
    graph.add_edge("orchestrator", "workers")
    graph.add_edge("workers", "synthesizer")
    graph.add_edge("synthesizer", END)

    return graph.compile(checkpointer=RedisSaver(redis_client))
```

---

## 7. Routing Agent

### 7.1 Classification-Based Routing

The router agent classifies the query into a category and routes to the appropriate specialist:

```python
from enum import Enum
from pydantic import BaseModel, Field

class QueryCategory(str, Enum):
    TECHNICAL = "technical"
    CREATIVE = "creative"
    ANALYTICAL = "analytical"
    UNKNOWN = "unknown"


class QueryClassification(BaseModel):
    """Structured output from the router's classification step."""
    category: QueryCategory
    confidence: float = Field(ge=0.0, le=1.0)
    reasoning: str = Field(description="Brief explanation of the classification")


ROUTER_SYSTEM_PROMPT = """\
You are a query router. Classify the user's query into one of:
- technical: programming, engineering, science, how-to, technical explanations
- creative: writing, storytelling, brainstorming, marketing copy, creative content
- analytical: data analysis, comparison, evaluation, pros/cons, decision-making
- unknown: if you are unsure

Respond with the category, a confidence score (0.0-1.0), and brief reasoning.
"""
```

### 7.2 Dynamic Routing with LLM Classifier

```python
async def router_node(state: AgentState) -> dict:
    """Classify query and route to the appropriate specialist."""
    query = state["query"]
    memory_ctx = state.get("memory_context", "")

    classification, cost = await llm_client.structured_complete(
        response_format=QueryClassification,
        system_prompt=ROUTER_SYSTEM_PROMPT,
        user_prompt=f"Query: {query}\n\nPrior context: {memory_ctx}",
    )

    cost_tracker.record("router", cost)

    # Log the routing decision
    logger.info(
        "router_classified",
        category=classification.category.value,
        confidence=classification.confidence,
        session_id=state.get("session_id"),
    )

    return {
        "query_type": classification.category.value,
        "router_confidence": classification.confidence,
        "cost_log": state.get("cost_log", []) + [cost_tracker.last_entry("router")],
    }
```

### 7.3 Fallback Agent

When the router's confidence is low (< 0.6), the query routes to a generalist fallback agent:

```python
FALLBACK_SYSTEM_PROMPT = """\
You are a generalist agent. You handle queries that don't clearly fit
a technical, creative, or analytical category. Provide a helpful,
well-structured response covering multiple angles.
"""


async def research_fallback_node(state: AgentState) -> dict:
    """Generalist research node for low-confidence classifications."""
    query = state["query"]
    search_results = await web_search_stub(query)
    kb_results = await knowledge_base_lookup("general", query)

    research_notes = f"Web results:\n{search_results}\n\nKB results:\n{kb_results}"
    return {"research_notes": research_notes}
```

### 7.4 Routing Conditional Edge

```python
def route_after_router(state: AgentState) -> str:
    """Route based on classification and confidence."""
    query_type = state.get("query_type", "unknown")
    confidence = state.get("router_confidence", 0.0)

    # Low confidence → fallback
    if confidence < 0.6:
        logger.warning("router_low_confidence", confidence=confidence, type=query_type)
        return "research_fallback"

    # High confidence → specialist
    route_map = {
        "technical": "research_technical",
        "creative": "research_creative",
        "analytical": "research_analytical",
        "unknown": "research_fallback",
    }
    return route_map.get(query_type, "research_fallback")
```

---

## 8. Critic/Revision Loop

### 8.1 Writer Agent

The writer produces an initial draft and revises it based on critic feedback:

```python
WRITER_SYSTEM_PROMPT = """\
You are a writer agent. Produce clear, well-structured content \
that fully addresses the user's query.

When revising:
- Address each point in the critic's feedback
- Improve clarity, accuracy, structure, and completeness
- Do not just patch errors — improve the overall quality
- Preserve what was good in the previous draft

Produce only the content, no meta-commentary.
"""
```

### 8.2 Critic Agent

The critic evaluates the draft against a rubric and produces structured feedback:

```python
class CriticFeedback(BaseModel):
    """Structured output from the critic agent."""
    feedback_text: str = Field(description="Overall assessment in 2-3 sentences")
    score: int = Field(ge=0, le=100, description="Quality score 0-100")
    strengths: list[str] = Field(description="What the draft does well")
    weaknesses: list[str] = Field(description="What needs improvement")
    suggestions: list[str] = Field(description="Specific, actionable suggestions")


CRITIC_SYSTEM_PROMPT = """\
You are a critic agent. Review the draft and provide structured, \
actionable feedback.

Evaluate on:
1. Accuracy: Are the facts correct and well-supported?
2. Clarity: Is the writing clear and easy to understand?
3. Structure: Is the content well-organized with logical flow?
4. Completeness: Does it address the original query fully?
5. Style: Is the tone appropriate for the query type?

Scoring guide:
- 90-100: Excellent — ready for publication (approved)
- 75-89: Good — minor revisions needed (approved)
- 60-74: Fair — significant revisions needed (not approved)
- 0-59: Poor — major rewrite needed (not approved)

Be specific and actionable. "Add more examples" is better than "needs improvement".
"""
```

### 8.3 Revision Agent

The revision agent is the writer node invoked with critic feedback as additional context (see §5.2 `writer_node`). The key is that the writer receives:

1. The original query
2. The research notes
3. The previous draft
4. The critic's structured feedback (strengths, weaknesses, suggestions)

### 8.4 Max Iterations and Convergence Detection

```python
def route_after_critic(state: AgentState) -> str:
    """Decide whether to revise or finalize."""
    # Approved → done
    if state.get("critic_approved", False):
        return "synthesizer"

    # Max revisions reached → best effort
    revision_count = state.get("revision_count", 0)
    if revision_count >= 3:
        logger.info("critic_max_revisions", count=revision_count)
        return "synthesizer"

    # Convergence detection: score not improving → stop
    score_history = state.get("score_history", [])
    if len(score_history) >= 2:
        latest = score_history[-1]
        previous = score_history[-2]
        if latest <= previous:
            logger.info("critic_convergence_stopped", history=score_history)
            return "synthesizer"

    # Not approved, under max, still improving → revise
    return "writer"
```

### 8.5 Critic/Revision Graph (Standalone)

```python
def build_critic_loop_graph() -> CompiledGraph:
    """Standalone critic/revision loop graph."""
    graph = StateGraph(AgentState)
    graph.add_node("writer", writer_node)
    graph.add_node("critic", critic_node)
    graph.add_node("synthesizer", synthesizer_node)

    graph.add_edge(START, "writer")
    graph.add_edge("writer", "critic")
    graph.add_conditional_edges(
        "critic",
        route_after_critic,
        {"synthesizer": "synthesizer", "writer": "writer"},
    )
    graph.add_edge("synthesizer", END)

    return graph.compile(checkpointer=RedisSaver(redis_client))
```

---

## 9. Pydantic AI Integration

### 9.1 Agent Definition with Pydantic AI

The Researcher agent is built in both raw function-calling style (default) and Pydantic AI style (alternative) for comparison:

```python
from pydantic_ai import Agent
from pydantic_ai.models.openai import OpenAIModel
from dataclasses import dataclass


@dataclass
class ResearchDependencies:
    """Dependencies injected into the Pydantic AI agent."""
    db_pool: object          # asyncpg pool
    redis: object            # redis client
    cost_tracker: object     # CostTracker instance


# Pydantic AI agent with structured output
research_agent = Agent(
    model=OpenAIModel("gpt-4o-mini"),
    deps_type=ResearchDependencies,
    output_type=ResearchResult,
    system_prompt=(
        "You are a research agent. Use the available tools to gather "
        "information relevant to the query. Return structured findings."
    ),
)


class ResearchResult(BaseModel):
    """Structured output from the Pydantic AI research agent."""
    summary: str = Field(description="Summary of findings")
    key_points: list[str] = Field(description="Key points discovered")
    sources: list[str] = Field(description="Sources consulted")
    confidence: float = Field(ge=0.0, le=1.0, description="Confidence in findings")
```

### 9.2 Structured Output Validation

Pydantic AI validates the LLM's output against the `output_type` schema automatically:

```python
async def run_pydantic_research(query: str, deps: ResearchDependencies) -> ResearchResult:
    """Run the Pydantic AI research agent."""
    result = await research_agent.run(query, deps=deps)
    # result.output is already validated as ResearchResult
    # If the LLM output doesn't validate, Pydantic AI retries automatically
    return result.output
```

### 9.3 Tool Definitions

```python
@research_agent.tool
async def web_search(ctx, query: str) -> str:
    """Search the web for information (stubbed for v1)."""
    deps = ctx.deps
    # Check Redis cache first
    cached = await deps.redis.get(f"search:{query}")
    if cached:
        return cached.decode()

    # Stubbed search — returns simulated results
    result = f"Simulated search results for: {query}"
    await deps.redis.setex(f"search:{query}", 300, result)  # 5-min TTL
    return result


@research_agent.tool
async def knowledge_base_lookup(ctx, topic: str) -> str:
    """Look up a topic in the local knowledge base."""
    deps = ctx.deps
    async with deps.db_pool.acquire() as conn:
        row = await conn.fetchrow(
            "SELECT content FROM knowledge_base WHERE topic = $1",
            topic,
        )
    return row["content"] if row else "No KB entry found."
```

### 9.4 Dependency Injection

Pydantic AI's dependency injection separates agent logic from infrastructure:

```python
# In the FastAPI route handler:
@app.post("/run")
async def run_pipeline(request: RunRequest):
    deps = ResearchDependencies(
        db_pool=app.state.db_pool,
        redis=app.state.redis,
        cost_tracker=app.state.cost_tracker,
    )

    if request.pattern == "pydantic_ai":
        # Use Pydantic AI research agent
        research_result = await run_pydantic_research(request.query, deps)
        research_notes = research_result.summary
    else:
        # Use raw function-calling research agent (default)
        research_notes = await run_raw_research(request.query, deps)
```

### 9.5 Pydantic AI vs Raw Function Calling — Comparison

| Dimension | Raw Function Calling (P4 style) | Pydantic AI |
|-----------|-------------------------------|-------------|
| **Boilerplate** | High — manual tool schema, manual output parsing, manual retry | Low — `@agent.tool` decorator, automatic output validation, automatic retry |
| **Type Safety** | Manual — JSON schema from dict, parse errors at runtime | Strong — Pydantic models, type-checked at definition |
| **Streaming** | Full control — stream tokens, partial JSON | Supported but less granular control |
| **Error Handling** | Manual — catch JSON parse errors, retry | Automatic — retries on validation failure |
| **Dependency Injection** | Manual — pass deps as function args | First-class — `deps_type` + `ctx.deps` |
| **Control** | Full — you own the loop | Constrained — framework owns the loop |
| **Best For** | Custom agent loops, maximum control | Standard tool-using agents, rapid prototyping |
| **Integration with LangGraph** | Native — node functions are just async functions | Requires wrapper — call Pydantic AI agent inside a LangGraph node |

**Decision:** Use raw function calling for agents that need custom control flow (router, critic, writer). Use Pydantic AI for standard tool-using agents (researcher) where type safety and reduced boilerplate matter most.

---

## 10. Agent Memory

### 10.1 Short-Term Memory (Conversation Context)

Short-term memory holds the recent conversation turns in a Redis-backed buffer. When the buffer exceeds a threshold, older messages are summarized.

```python
class ShortTermMemory:
    """Redis-backed conversation buffer with summarization."""

    BUFFER_LIMIT = 10           # keep last 10 messages
    SUMMARIZE_THRESHOLD = 10     # summarize when buffer exceeds 10

    def __init__(self, redis: redis.asyncio.Redis):
        self.redis = redis

    async def get_messages(self, session_id: str) -> list[dict]:
        """Get the current conversation buffer for a session."""
        key = f"memory:short:{session_id}"
        raw = await self.lrange(key, 0, -1)
        return [json.loads(m) for m in raw]

    async def add_message(self, session_id: str, message: dict) -> None:
        """Add a message to the buffer; summarize if threshold exceeded."""
        key = f"memory:short:{session_id}"
        await self.redis.rpush(key, json.dumps(message))

        count = await self.redis.llen(key)
        if count > self.SUMMARIZE_THRESHOLD:
            await self._summarize(session_id)

    async def _summarize(self, session_id: str) -> None:
        """Summarize older messages to compress context."""
        key = f"memory:short:{session_id}"
        summary_key = f"memory:summary:{session_id}"

        # Get all messages
        all_raw = await self.redis.lrange(key, 0, -1)
        all_messages = [json.loads(m) for m in all_raw]

        # Keep the most recent BUFFER_LIMIT messages
        recent = all_messages[-self.BUFFER_LIMIT:]
        older = all_messages[:-self.BUFFER_LIMIT]

        if not older:
            return

        # Summarize the older messages
        older_text = "\n".join(f"{m['role']}: {m['content']}" for m in older)
        existing_summary = await self.redis.get(summary_key)
        summary_input = f"Existing summary:\n{existing_summary}\n\nNew messages:\n{older_text}" if existing_summary else older_text

        summary, cost = await llm_client.complete(
            system_prompt=SUMMARIZATION_SYSTEM_PROMPT,
            user_prompt=f"Summarize this conversation, preserving key facts:\n{summary_input}",
        )

        # Store summary, replace buffer with recent messages only
        await self.redis.set(summary_key, summary)
        await self.redis.delete(key)
        for m in recent:
            await self.redis.rpush(key, json.dumps(m))

    async def get_context(self, session_id: str) -> str:
        """Get the full context: summary + recent messages."""
        summary = await self.redis.get(f"memory:summary:{session_id}")
        messages = await self.get_messages(session_id)

        parts = []
        if summary:
            parts.append(f"Conversation summary:\n{summary.decode()}")
        if messages:
            parts.append("Recent messages:\n" + "\n".join(
                f"{m['role']}: {m['content']}" for m in messages
            ))
        return "\n\n".join(parts)
```

### 10.2 Working Memory (Task State)

Working memory is the LangGraph state itself — the `AgentState` TypedDict that flows through the graph. It is persisted via the Redis checkpointer after each node execution.

```
Working Memory = AgentState (TypedDict)
  ├── query, query_type          (router output)
  ├── research_notes             (research output)
  ├── draft, revision_count      (writer output)
  ├── critic_feedback, score     (critic output)
  ├── cost_log                   (per-agent cost entries)
  └── messages                   (conversation buffer)
```

**Persistence:** The Redis checkpointer serializes the full `AgentState` after each node. This means:
- If the process crashes, the graph resumes from the last completed node
- The Streamlit dashboard can read the checkpoint to show intermediate state
- Debugging: inspect state at any node boundary

### 10.3 Memory Summarization for Long Conversations

The summarization system (see §10.1 `_summarize`) compresses older messages into a running summary. The summary is updated incrementally — each summarization pass incorporates the existing summary plus new older messages:

```python
SUMMARIZATION_SYSTEM_PROMPT = """\
You are a conversation summarizer. Summarize the conversation, \
preserving:
- Key facts and decisions
- User preferences and constraints
- Important context for future turns

Discard:
- Pleasantries and filler
- Redundant information
- Details unlikely to be referenced again

Keep the summary under 500 words. Use bullet points for clarity.
"""
```

### 10.4 Memory Persistence in Redis

```
Redis Key Structure:
┌─────────────────────────────────────────────────────┐
│ memory:short:{session_id}    │ List  │ recent messages (JSON)    │
│ memory:summary:{session_id}  │ String│ running summary           │
│ checkpoint:{thread_id}       │ Hash  │ LangGraph checkpoint      │
│ search:{query}               │ String│ cached search results     │
│ cost:session:{session_id}    │ Hash  │ session cost totals       │
└─────────────────────────────────────────────────────┘

TTL Policy:
- memory:short:*     → no TTL (cleared on session end)
- memory:summary:*   → 24h TTL (expires after inactivity)
- checkpoint:*       → 1h TTL (enough for crash recovery)
- search:*           → 5min TTL (short-lived cache)
```

### 10.5 Long-Term Episodic Memory (pgvector)

Episodic memory stores key facts from past interactions as embeddings in PostgreSQL with the pgvector extension:

```python
class EpisodicMemoryStore:
    """pgvector-backed long-term episodic memory."""

    def __init__(self, db_pool, embedding_dim=1536):
        self.db_pool = db_pool
        self.embedding_dim = embedding_dim

    async def store_episodic(self, session_id: str, query: str, output: str) -> None:
        """Store a key interaction as an episodic memory."""
        # Generate embedding from the query
        embedding = await openai_client.embeddings.create(
            input=query, model="text-embedding-3-small"
        )

        async with self.db_pool.acquire() as conn:
            await conn.execute(
                """INSERT INTO episodic_memory.memories
                   (session_id, query, output, embedding)
                   VALUES ($1, $2, $3, $4)""",
                session_id, query, output, embedding.data[0].embedding,
            )

    async def retrieve_relevant(self, query: str, top_k: int = 3) -> str:
        """Retrieve top-k relevant episodic memories via cosine similarity."""
        embedding = await openai_client.embeddings.create(
            input=query, model="text-embedding-3-small"
        )

        async with self.db_pool.acquire() as conn:
            rows = await conn.fetch(
                """SELECT query, output, embedding <=> $1 AS distance
                   FROM episodic_memory.memories
                   ORDER BY embedding <=> $1
                   LIMIT $2""",
                embedding.data[0].embedding, top_k,
            )

        if not rows:
            return ""

        memories = "\n\n".join(
            f"Past query: {r['query']}\nPast response: {r['output'][:200]}..."
            for r in rows if r["distance"] < 0.3  # similarity threshold
        )
        return f"Relevant past interactions:\n{memories}"
```

---

## 11. Cost Tracking & Observability

### 11.1 Per-Agent Token Counting

Every LLM call is wrapped by the `LLMClient`, which extracts token usage from the OpenAI response and records it:

```python
from dataclasses import dataclass, field
from datetime import datetime

@dataclass
class CostRecord:
    """A single cost record for one LLM call."""
    agent: str
    timestamp: str
    model: str
    prompt_tokens: int
    completion_tokens: int
    total_tokens: int
    cost_usd: float
    latency_ms: float


class CostTracker:
    """Tracks per-agent token usage and cost."""

    # gpt-4o-mini pricing (per 1M tokens)
    INPUT_PRICE_PER_M = 0.15
    OUTPUT_PRICE_PER_M = 0.60

    def __init__(self):
        self._entries: list[CostRecord] = []

    def record(self, agent: str, usage: dict, model: str, latency_ms: float) -> CostRecord:
        """Record a single LLM call's cost."""
        prompt_tokens = usage.get("prompt_tokens", 0)
        completion_tokens = usage.get("completion_tokens", 0)
        total_tokens = usage.get("total_tokens", prompt_tokens + completion_tokens)

        cost = (
            (prompt_tokens / 1_000_000) * self.INPUT_PRICE_PER_M
            + (completion_tokens / 1_000_000) * self.OUTPUT_PRICE_PER_M
        )

        record = CostRecord(
            agent=agent,
            timestamp=datetime.utcnow().isoformat(),
            model=model,
            prompt_tokens=prompt_tokens,
            completion_tokens=completion_tokens,
            total_tokens=total_tokens,
            cost_usd=cost,
            latency_ms=latency_ms,
        )
        self._entries.append(record)
        return record

    def summary(self) -> dict:
        """Aggregate cost summary by agent."""
        by_agent: dict[str, dict] = {}
        for entry in self._entries:
            if entry.agent not in by_agent:
                by_agent[entry.agent] = {
                    "calls": 0, "prompt_tokens": 0, "completion_tokens": 0,
                    "total_tokens": 0, "cost_usd": 0.0, "latency_ms": 0.0,
                }
            agg = by_agent[entry.agent]
            agg["calls"] += 1
            agg["prompt_tokens"] += entry.prompt_tokens
            agg["completion_tokens"] += entry.completion_tokens
            agg["total_tokens"] += entry.total_tokens
            agg["cost_usd"] += entry.cost_usd
            agg["latency_ms"] += entry.latency_ms

        total_cost = sum(e.cost_usd for e in self._entries)
        total_tokens = sum(e.total_tokens for e in self._entries)

        return {
            "by_agent": by_agent,
            "total_cost_usd": total_cost,
            "total_tokens": total_tokens,
            "total_calls": len(self._entries),
        }
```

### 11.2 Per-Agent Cost Attribution

Each node in the graph records its cost with an agent label (e.g., `"router"`, `"writer"`, `"critic"`, `"worker_0"`). The cost log accumulates in `AgentState.cost_log` and is persisted to PostgreSQL at the end of the run:

```python
async def persist_cost_log(session_id: str, cost_log: list[dict]) -> None:
    """Persist per-agent cost entries to PostgreSQL."""
    async with db_pool.acquire() as conn:
        for entry in cost_log:
            await conn.execute(
                """INSERT INTO cost_tracking.agent_costs
                   (session_id, agent, model, prompt_tokens, completion_tokens,
                    total_tokens, cost_usd, latency_ms, recorded_at)
                   VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)""",
                session_id, entry["agent"], entry["model"],
                entry["prompt_tokens"], entry["completion_tokens"],
                entry["total_tokens"], entry["cost_usd"], entry["latency_ms"],
                entry["timestamp"],
            )
```

### 11.3 Agent Execution Tracing

Every node execution is traced with: node name, input state snapshot, output state delta, duration, and any errors:

```python
import time
import functools

def trace_node(func):
    """Decorator that traces node execution to PostgreSQL."""
    @functools.wraps(func)
    async def wrapper(state: AgentState) -> dict:
        node_name = func.__name__
        session_id = state.get("session_id", "unknown")
        start = time.monotonic()

        try:
            result = await func(state)
            duration_ms = (time.monotonic() - start) * 1000

            await trace_logger.log(
                session_id=session_id,
                node=node_name,
                input_keys=list(state.keys()),
                output_keys=list(result.keys()) if result else [],
                duration_ms=duration_ms,
                status="success",
            )
            return result

        except Exception as e:
            duration_ms = (time.monotonic() - start) * 1000
            await trace_logger.log(
                session_id=session_id,
                node=node_name,
                input_keys=list(state.keys()),
                output_keys=[],
                duration_ms=duration_ms,
                status="error",
                error=str(e),
            )
            raise

    return wrapper
```

### 11.4 LangSmith / LangFuse Integration Concepts

Both LangSmith and LangFuse are optional integrations enabled via environment variables:

```python
# .env
LANGSMITH_API_KEY=ls__***
LANGSMITH_PROJECT=multi-agent-systems
LANGFUSE_PUBLIC_KEY=lf_pk_***
LANGFUSE_SECRET_KEY=lf_sk_***
LANGFUSE_HOST=https://cloud.langfuse.com

# In the LLM client, wrap calls with tracing:
import os

if os.getenv("LANGSMITH_API_KEY"):
    # LangSmith tracing is automatic when LANGSMITH_TRACING=true
    os.environ["LANGSMITH_TRACING"] = "true"

if os.getenv("LANGFUSE_PUBLIC_KEY"):
    from langfuse.openai import AsyncOpenAI
    llm_client = AsyncOpenAI()  # LangFuse-wrapped OpenAI client
else:
    from openai import AsyncOpenAI
    llm_client = AsyncOpenAI()  # plain OpenAI client
```

**What tracing provides:**
- **LangSmith:** Full LangGraph trace visualization, per-node input/output, token breakdown, latency waterfall, prompt versioning
- **LangFuse:** Open-source alternative; self-hostable; similar trace visualization + cost analytics

Both integrate with LangGraph natively — no code changes needed beyond setting environment variables.

---

## 12. Database Schema

### 12.1 Overview

PostgreSQL stores three categories of data:
1. **Agent metadata** — sessions, messages, conversation history
2. **Cost tracking** — per-agent token usage and cost
3. **Episodic memory** — pgvector embeddings for long-term recall
4. **Execution logs** — per-node traces for debugging and observability

### 12.2 Schema DDL

```sql
-- Enable extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS vector;

-- ============================================================
-- Schema 1: agent_meta — sessions and conversation messages
-- ============================================================
CREATE SCHEMA IF NOT EXISTS agent_meta;

CREATE TABLE agent_meta.sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE agent_meta.messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES agent_meta.sessions(id) ON DELETE CASCADE,
    role            TEXT NOT NULL,             -- 'user', 'assistant', 'system', 'tool'
    content         TEXT NOT NULL,
    agent_name      TEXT,                      -- which agent produced this (router, writer, etc.)
    node_name       TEXT,                      -- which graph node
    tokens          INTEGER,
    cost_usd        NUMERIC(10, 6),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_messages_session ON agent_meta.messages(session_id, created_at);

-- ============================================================
-- Schema 2: cost_tracking — per-agent token usage and cost
-- ============================================================
CREATE SCHEMA IF NOT EXISTS cost_tracking;

CREATE TABLE cost_tracking.agent_costs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id          UUID NOT NULL REFERENCES agent_meta.sessions(id) ON DELETE CASCADE,
    agent               TEXT NOT NULL,          -- 'router', 'writer', 'critic', 'worker_0', etc.
    model               TEXT NOT NULL,
    prompt_tokens       INTEGER NOT NULL,
    completion_tokens   INTEGER NOT NULL,
    total_tokens        INTEGER NOT NULL,
    cost_usd            NUMERIC(10, 6) NOT NULL,
    latency_ms          NUMERIC(10, 2) NOT NULL,
    recorded_at         TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_costs_session ON cost_tracking.agent_costs(session_id);
CREATE INDEX idx_costs_agent ON cost_tracking.agent_costs(agent);

-- ============================================================
-- Schema 3: episodic_memory — pgvector embeddings
-- ============================================================
CREATE SCHEMA IF NOT EXISTS episodic_memory;

CREATE TABLE episodic_memory.memories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES agent_meta.sessions(id) ON DELETE CASCADE,
    query           TEXT NOT NULL,
    output          TEXT NOT NULL,
    embedding       vector(1536) NOT NULL,     -- text-embedding-3-small
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_memories_embedding ON episodic_memory.memories
    USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);

CREATE INDEX idx_memories_session ON episodic_memory.memories(session_id);

-- ============================================================
-- Schema 4: execution_logs — per-node traces
-- ============================================================
CREATE SCHEMA IF NOT EXISTS execution_logs;

CREATE TABLE execution_logs.node_traces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES agent_meta.sessions(id) ON DELETE CASCADE,
    node_name       TEXT NOT NULL,              -- 'router_node', 'writer_node', etc.
    input_keys      TEXT[],                     -- state keys at entry
    output_keys     TEXT[],                     -- state keys modified
    duration_ms     NUMERIC(10, 2) NOT NULL,
    status          TEXT NOT NULL,              -- 'success', 'error'
    error           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_traces_session ON execution_logs.node_traces(session_id, created_at);

-- ============================================================
-- Knowledge base for research tools (stubbed search target)
-- ============================================================
CREATE TABLE knowledge_base (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    topic           TEXT NOT NULL UNIQUE,
    content         TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 12.3 Redis Key Structure

```
Redis Keys (all async, redis-py 5+):
┌──────────────────────────────────────────────────────────────┐
│ Key                            │ Type   │ Purpose             │
│────────────────────────────────│────────│─────────────────────│
│ memory:short:{session_id}      │ List   │ Recent messages     │
│ memory:summary:{session_id}    │ String │ Running summary     │
│ checkpoint:{thread_id}         │ Hash   │ LangGraph state     │
│ search:{query_hash}            │ String │ Cached search       │
│ cost:session:{session_id}      │ Hash   │ Session cost totals │
└──────────────────────────────────────────────────────────────┘
```

---

## 13. Project Structure

```
multi-agent-systems/
├── docker-compose.yml
├── pyproject.toml
├── .env.example
├── README.md
├── docker/
│   ├── Dockerfile.api
│   ├── Dockerfile.streamlit
│   └── postgres/
│       └── init.sql
├── src/
│   ├── __init__.py
│   ├── config.py                     # Pydantic Settings
│   ├── main.py                       # FastAPI app factory
│   ├── graph/
│   │   ├── __init__.py
│   │   ├── state.py                  # AgentState TypedDict
│   │   ├── builder.py                # build_graph(), build_*_graph()
│   │   └── edges.py                  # Conditional edge functions
│   ├── agents/
│   │   ├── __init__.py
│   │   ├── router.py                 # Router agent (classification)
│   │   ├── researcher.py             # Research agent (raw function calling)
│   │   ├── researcher_pydantic.py    # Research agent (Pydantic AI)
│   │   ├── writer.py                 # Writer agent
│   │   ├── critic.py                 # Critic agent
│   │   ├── orchestrator.py           # Orchestrator agent
│   │   ├── worker.py                 # Worker agent (parallel)
│   │   ├── synthesizer.py            # Synthesizer agent
│   │   └── prompts.py                # All system prompts
│   ├── tools/
│   │   ├── __init__.py
│   │   ├── search.py                 # web_search_stub
│   │   └── knowledge_base.py         # knowledge_base_lookup
│   ├── memory/
│   │   ├── __init__.py
│   │   ├── short_term.py             # ShortTermMemory (Redis buffer)
│   │   ├── episodic.py               # EpisodicMemoryStore (pgvector)
│   │   └── summarizer.py             # Memory summarization logic
│   ├── llm/
│   │   ├── __init__.py
│   │   ├── client.py                 # LLMClient (OpenAI wrapper + retry)
│   │   └── cost_tracker.py           # CostTracker, CostRecord
│   ├── db/
│   │   ├── __init__.py
│   │   ├── connection.py             # asyncpg pool
│   │   └── schema.sql                # Full DDL
│   ├── observability/
│   │   ├── __init__.py
│   │   ├── tracer.py                 # Node execution tracer
│   │   └── trace_logger.py           # PostgreSQL trace logger
│   └── api/
│       ├── __init__.py
│       ├── dependencies.py           # FastAPI DI
│       └── routes/
│           ├── __init__.py
│           ├── run.py                # POST /run
│           ├── sessions.py           # Session CRUD
│           ├── costs.py              # GET /costs/{session_id}
│           ├── traces.py             # GET /traces/{session_id}
│           └── health.py             # GET /health
├── streamlit/
│   └── dashboard.py                  # Streamlit dashboard
├── tests/
│   ├── conftest.py
│   ├── test_graph.py                 # Graph compilation, routing
│   ├── test_router.py                # Router classification
│   ├── test_critic_loop.py           # Critic/revision loop, convergence
│   ├── test_orchestrator.py          # Orchestrator-worker
│   ├── test_memory.py                # Short-term + summarization
│   ├── test_cost_tracker.py          # Cost calculation
│   ├── test_pydantic_ai.py           # Pydantic AI agent
│   └── test_api.py                   # FastAPI endpoints
└── scripts/
    ├── seed_kb.py                    # Seed knowledge_base table
    └── run_demo.py                   # CLI demo runner
```

---

## 14. Implementation Phases

### Phase 1: Environment Setup + LangGraph Basics (Day 1)

**Goal:** Docker Compose stack running (Postgres + Redis), LangGraph state graph compiles and visualizes.

**Steps:**
1. Create `pyproject.toml` with all dependencies (langgraph, pydantic-ai, fastapi, asyncpg, redis, openai, streamlit, pytest, ruff)
2. Create `docker-compose.yml` with postgres, redis, api, streamlit services
3. Create `docker/Dockerfile.api` and `docker/Dockerfile.streamlit`
4. Create `docker/postgres/init.sql` with all schema DDL + extensions
5. Create `src/config.py` — Pydantic Settings (DATABASE_URL, REDIS_URL, OPENAI_API_KEY, LLM_MODEL, etc.)
6. Create `src/db/connection.py` — asyncpg pool
7. Create `src/graph/state.py` — `AgentState` TypedDict
8. Create `src/graph/builder.py` — minimal `build_graph()` with stub nodes
9. Verify: `docker compose up` starts all services; `graph.get_graph().draw_mermaid()` renders

**Validation:**
- `docker compose up` starts postgres, redis, api, streamlit without errors
- `python -c "from src.graph.builder import build_graph; g = build_graph(); print(g.get_graph().draw_mermaid())"` prints a valid Mermaid graph
- `psql` connects and all schemas/tables exist

### Phase 2: LLM Client + Cost Tracker (Day 2)

**Goal:** LLM client with retry, structured output, and per-call cost tracking.

**Steps:**
1. Create `src/llm/cost_tracker.py` — `CostTracker`, `CostRecord` (see §11.1)
2. Create `src/llm/client.py` — `LLMClient` wrapping OpenAI with tenacity retry, `complete()` and `structured_complete()` methods
3. Create `src/agents/prompts.py` — all system prompts (router, writer, critic, orchestrator, synthesizer, summarizer)
4. Create `src/tools/search.py` — `web_search_stub()` (returns simulated results)
5. Create `src/tools/knowledge_base.py` — `knowledge_base_lookup()` (queries PostgreSQL)
6. Write `tests/test_cost_tracker.py` — verify cost calculation with known token counts

**Validation:**
- `pytest tests/test_cost_tracker.py` passes
- LLM client completes a simple prompt and returns token usage
- Cost tracker correctly calculates cost for gpt-4o-mini pricing

### Phase 3: Router Agent + Routing (Day 3)

**Goal:** Router classifies queries and routes to the correct research node.

**Steps:**
1. Create `src/agents/router.py` — `router_node()`, `QueryClassification` schema, `route_after_router()` (see §7)
2. Create `src/graph/edges.py` — `route_after_router()`, `route_after_critic()` conditional edge functions
3. Create research nodes (technical, creative, analytical, fallback) — stub implementations that call tools
4. Wire router → conditional edges → research nodes in `build_graph()`
5. Write `tests/test_router.py` — mock LLM returns known classifications; verify routing to correct node

**Validation:**
- `pytest tests/test_router.py` passes
- Router classifies "write a poem about the sea" as `creative` with confidence > 0.7
- Router classifies "explain how TCP handshake works" as `technical`
- Low-confidence query routes to fallback

### Phase 4: Writer + Critic/Revision Loop (Days 4-5)

**Goal:** Writer produces drafts, Critic reviews, revision loop with max 3 rounds and convergence detection.

**Steps:**
1. Create `src/agents/writer.py` — `writer_node()` with initial draft and revision modes (see §8.1)
2. Create `src/agents/critic.py` — `critic_node()`, `CriticFeedback` schema (see §8.2)
3. Create `src/agents/synthesizer.py` — `synthesizer_node()` for final output
4. Wire writer → critic → conditional edge (approve/revise) in `build_graph()`
5. Implement convergence detection in `route_after_critic()` (see §8.4)
6. Write `tests/test_critic_loop.py` — mock critic returns approved/not-approved; verify loop terminates at max 3 or approval; verify convergence detection

**Validation:**
- `pytest tests/test_critic_loop.py` passes
- Loop terminates when critic approves (score ≥ 75)
- Loop terminates at max 3 revisions even if critic never approves
- Loop terminates early when score doesn't improve between rounds
- `quality_warning` flag is set when max revisions hit without approval

### Phase 5: Orchestrator-Worker + Map-Reduce (Days 6-7)

**Goal:** Orchestrator decomposes tasks, workers execute in parallel, synthesizer merges results.

**Steps:**
1. Create `src/agents/orchestrator.py` — `orchestrator_node()`, `DecompositionPlan` schema (see §6.2)
2. Create `src/agents/worker.py` — `worker_node()` with `asyncio.gather()` parallel execution (see §6.3)
3. Create `src/agents/synthesizer.py` — add `synthesizer_orchestrator_node()` (see §6.4)
4. Build `build_orchestrator_worker_graph()` (see §6.5)
5. Implement map-reduce pattern: mapper splits task, workers process in parallel, reducer merges
6. Write `tests/test_orchestrator.py` — mock orchestrator returns 3 subtasks; verify 3 worker calls; verify synthesis

**Validation:**
- `pytest tests/test_orchestrator.py` passes
- Orchestrator produces 2-5 subtasks
- Workers execute in parallel (verify via timing — parallel < sequential)
- Worker failure is handled gracefully (partial results + note)
- Synthesizer produces a coherent merged output

### Phase 6: Pydantic AI Agent (Day 8)

**Goal:** Researcher agent built with Pydantic AI; comparison table documented.

**Steps:**
1. Create `src/agents/researcher_pydantic.py` — Pydantic AI agent with `ResearchResult` output, `@agent.tool` decorators, `ResearchDependencies` (see §9)
2. Create `src/agents/researcher.py` — raw function-calling version for comparison
3. Wire Pydantic AI agent into the graph when `pattern=pydantic_ai`
4. Write `tests/test_pydantic_ai.py` — verify structured output validation, tool calls, dependency injection
5. Document comparison table (see §9.5) in code comments and README

**Validation:**
- `pytest tests/test_pydantic_ai.py` passes
- Pydantic AI agent produces `ResearchResult` with validated fields
- Tool calls work (web_search_stub, knowledge_base_lookup)
- Both versions produce equivalent results for the same query

### Phase 7: Memory (Short-Term + Episodic) (Day 9)

**Goal:** Short-term memory buffer in Redis with summarization; episodic memory in pgvector.

**Steps:**
1. Create `src/memory/short_term.py` — `ShortTermMemory` class (see §10.1)
2. Create `src/memory/summarizer.py` — summarization logic
3. Create `src/memory/episodic.py` — `EpisodicMemoryStore` with pgvector (see §10.5)
4. Integrate memory into the graph: load context at start, store at end
5. Write `tests/test_memory.py` — verify buffer add/get, summarization triggers at threshold, episodic retrieval returns relevant memories

**Validation:**
- `pytest tests/test_memory.py` passes
- Buffer holds last 10 messages; summarization triggers at 10
- Summarized context preserves key facts (≥ 80% recall in test)
- Episodic retrieval returns memories with cosine similarity > 0.7 for related queries
- Redis checkpoint persists graph state across process restarts

### Phase 8: Cost Tracking + Observability + Dashboard (Day 10)

**Goal:** Per-agent cost logged to Postgres, execution traces, Streamlit dashboard showing graph + costs.

**Steps:**
1. Create `src/observability/tracer.py` — `@trace_node` decorator (see §11.3)
2. Create `src/observability/trace_logger.py` — persist traces to `execution_logs.node_traces`
3. Create `src/api/routes/run.py` — `POST /run` endpoint
4. Create `src/api/routes/costs.py` — `GET /costs/{session_id}`
5. Create `src/api/routes/traces.py` — `GET /traces/{session_id}`
6. Create `src/api/routes/sessions.py` — session CRUD
7. Create `src/api/routes/health.py` — `GET /health`
8. Create `streamlit/dashboard.py` — submit query, view graph, cost table, latency chart, traces
9. Write `tests/test_api.py` — test all endpoints with mock LLM
10. End-to-end smoke test: submit query via Streamlit, verify full pipeline runs

**Validation:**
- `pytest tests/test_api.py` passes
- `POST /run` with a query returns final output + cost summary
- `GET /costs/{session_id}` returns per-agent cost breakdown
- Streamlit dashboard renders: agent graph (Mermaid), cost table, latency chart
- Total pipeline cost < $0.05 per query on gpt-4o-mini
- `docker compose up` → full system works end-to-end

---

## 15. Component Specifications

### 15.1 LLM Client (`src/llm/client.py`)

```python
import time
from typing import TypeVar, Type
from pydantic import BaseModel
from openai import AsyncOpenAI, RateLimitError, APITimeoutError, APIConnectionError
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type

T = TypeVar("T", bound=BaseModel)

class LLMClient:
    """Async OpenAI client with retry, structured output, and cost tracking."""

    def __init__(self, api_key: str, model: str = "gpt-4o-mini", timeout: int = 60):
        self.client = AsyncOpenAI(api_key=api_key, timeout=timeout)
        self.model = model
        self.cost_tracker = CostTracker()

    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(min=2, max=30),
        retry=retry_if_exception_type((RateLimitError, APITimeoutError, APIConnectionError)),
        reraise=True,
    )
    async def complete(self, system_prompt: str, user_prompt: str) -> tuple[str, dict]:
        """Plain text completion. Returns (text, cost_entry_dict)."""
        start = time.monotonic()
        response = await self.client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": user_prompt},
            ],
        )
        latency_ms = (time.monotonic() - start) * 1000
        text = response.choices[0].message.content
        usage = response.usage.model_dump()
        return text, {
            "model": self.model, "prompt_tokens": usage["prompt_tokens"],
            "completion_tokens": usage["completion_tokens"],
            "total_tokens": usage["total_tokens"], "latency_ms": latency_ms,
        }

    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(min=2, max=30),
        retry=retry_if_exception_type((RateLimitError, APITimeoutError, APIConnectionError)),
        reraise=True,
    )
    async def structured_complete(
        self, response_format: Type[T], system_prompt: str, user_prompt: str
    ) -> tuple[T, dict]:
        """Structured output completion. Returns (parsed_model, cost_entry_dict)."""
        start = time.monotonic()
        response = await self.client.beta.chat.completions.parse(
            model=self.model,
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": user_prompt},
            ],
            response_format=response_format,
        )
        latency_ms = (time.monotonic() - start) * 1000
        parsed = response.choices[0].message.parsed
        usage = response.usage.model_dump()
        return parsed, {
            "model": self.model, "prompt_tokens": usage["prompt_tokens"],
            "completion_tokens": usage["completion_tokens"],
            "total_tokens": usage["total_tokens"], "latency_ms": latency_ms,
        }
```

### 15.2 Router Agent (`src/agents/router.py`)

```python
from enum import Enum
from pydantic import BaseModel, Field
from src.graph.state import AgentState
from src.llm.client import llm_client
from src.llm.cost_tracker import cost_tracker

class QueryCategory(str, Enum):
    TECHNICAL = "technical"
    CREATIVE = "creative"
    ANALYTICAL = "analytical"
    UNKNOWN = "unknown"

class QueryClassification(BaseModel):
    category: QueryCategory
    confidence: float = Field(ge=0.0, le=1.0)
    reasoning: str

async def router_node(state: AgentState) -> dict:
    query = state["query"]
    memory_ctx = state.get("memory_context", "")
    classification, cost = await llm_client.structured_complete(
        response_format=QueryClassification,
        system_prompt=ROUTER_SYSTEM_PROMPT,
        user_prompt=f"Query: {query}\n\nPrior context: {memory_ctx}",
    )
    cost_tracker.record("router", cost, llm_client.model, cost["latency_ms"])
    return {
        "query_type": classification.category.value,
        "router_confidence": classification.confidence,
        "cost_log": state.get("cost_log", []) + [cost_tracker.last_entry("router")],
    }
```

### 15.3 Writer Agent (`src/agents/writer.py`)

```python
async def writer_node(state: AgentState) -> dict:
    query = state["query"]
    research = state.get("research_notes", "")
    feedback = state.get("critic_feedback", "")
    revision_count = state.get("revision_count", 0)

    if revision_count == 0:
        prompt = f"Write a response to: {query}\n\nResearch: {research}"
    else:
        prompt = (
            f"Revise your previous draft based on critic feedback.\n\n"
            f"Query: {query}\nResearch: {research}\n\n"
            f"Previous draft: {state['draft']}\n\n"
            f"Critic feedback: {feedback}\n\n"
            f"Produce only the revised content."
        )

    draft, cost = await llm_client.complete(WRITER_SYSTEM_PROMPT, prompt)
    cost_tracker.record("writer", cost, llm_client.model, cost["latency_ms"])
    return {
        "draft": draft,
        "revision_count": revision_count + 1,
        "cost_log": state.get("cost_log", []) + [cost_tracker.last_entry("writer")],
    }
```

### 15.4 Critic Agent (`src/agents/critic.py`)

```python
class CriticFeedback(BaseModel):
    feedback_text: str
    score: int = Field(ge=0, le=100)
    strengths: list[str]
    weaknesses: list[str]
    suggestions: list[str]

async def critic_node(state: AgentState) -> dict:
    draft = state["draft"]
    query = state["query"]
    feedback, cost = await llm_client.structured_complete(
        response_format=CriticFeedback,
        system_prompt=CRITIC_SYSTEM_PROMPT,
        user_prompt=f"Query: {query}\n\nDraft to review:\n{draft}",
    )
    approved = feedback.score >= 75
    score_history = state.get("score_history", []) + [feedback.score]
    cost_tracker.record("critic", cost, llm_client.model, cost["latency_ms"])
    return {
        "critic_feedback": feedback.feedback_text,
        "critic_score": feedback.score,
        "critic_approved": approved,
        "score_history": score_history,
        "cost_log": state.get("cost_log", []) + [cost_tracker.last_entry("critic")],
    }
```

### 15.5 Orchestrator Agent (`src/agents/orchestrator.py`)

```python
class DecompositionPlan(BaseModel):
    subtasks: list[str] = Field(min_length=2, max_length=5)
    strategy: str

async def orchestrator_node(state: AgentState) -> dict:
    query = state["query"]
    plan, cost = await llm_client.structured_complete(
        response_format=DecompositionPlan,
        system_prompt=ORCHESTRATOR_SYSTEM_PROMPT,
        user_prompt=f"Decompose this task into 2-5 independent subtasks:\n{query}",
    )
    cost_tracker.record("orchestrator", cost, llm_client.model, cost["latency_ms"])
    return {
        "subtasks": plan.subtasks,
        "cost_log": state.get("cost_log", []) + [cost_tracker.last_entry("orchestrator")],
    }
```

### 15.6 Worker Agent (`src/agents/worker.py`)

```python
import asyncio

async def worker_node(state: AgentState) -> dict:
    subtasks = state["subtasks"]
    query_type = state.get("query_type", "general")

    async def execute_single(subtask: str, index: int) -> str:
        result, cost = await llm_client.complete(
            system_prompt=WORKER_SYSTEM_PROMPT.format(role=query_type),
            user_prompt=f"Subtask: {subtask}\n\nContext: {state['query']}",
        )
        cost_tracker.record(f"worker_{index}", cost, llm_client.model, cost["latency_ms"])
        return result

    results = await asyncio.gather(
        *[execute_single(st, i) for i, st in enumerate(subtasks)],
        return_exceptions=True,
    )

    worker_results = []
    for i, result in enumerate(results):
        if isinstance(result, Exception):
            worker_results.append(f"[Worker {i} failed: {result}]")
        else:
            worker_results.append(result)

    return {
        "worker_results": worker_results,
        "cost_log": state.get("cost_log", []) + cost_tracker.all_entries_since("orchestrator"),
    }
```

### 15.7 Synthesizer Agent (`src/agents/synthesizer.py`)

```python
async def synthesizer_node(state: AgentState) -> dict:
    draft = state["draft"]
    approved = state.get("critic_approved", False)
    final_output = draft
    quality_warning = not approved

    await memory_store.store_episodic(
        session_id=state["session_id"],
        query=state["query"],
        output=final_output,
    )
    return {
        "final_output": final_output,
        "quality_warning": quality_warning,
        "status": "completed",
    }

async def synthesizer_orchestrator_node(state: AgentState) -> dict:
    worker_results = state["worker_results"]
    subtasks = state["subtasks"]
    query = state["query"]
    paired = "\n\n".join(
        f"## Subtask: {st}\n{res}" for st, res in zip(subtasks, worker_results)
    )
    synthesis, cost = await llm_client.complete(
        system_prompt=SYNTHESIZER_SYSTEM_PROMPT,
        user_prompt=f"Original query: {query}\n\nWorker results:\n{paired}\n\nSynthesize into a single coherent response.",
    )
    cost_tracker.record("synthesizer", cost, llm_client.model, cost["latency_ms"])
    return {
        "final_output": synthesis,
        "status": "completed",
        "cost_log": state.get("cost_log", []) + [cost_tracker.last_entry("synthesizer")],
    }
```

### 15.8 Short-Term Memory (`src/memory/short_term.py`)

See §10.1 for full implementation.

### 15.9 Episodic Memory (`src/memory/episodic.py`)

See §10.5 for full implementation.

### 15.10 Cost Tracker (`src/llm/cost_tracker.py`)

See §11.1 for full implementation.

### 15.11 Node Tracer (`src/observability/tracer.py`)

See §11.3 for full implementation.

### 15.12 FastAPI App (`src/main.py`)

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from src.config import settings
from src.db.connection import init_db_pool, close_db_pool
from src.api.routes import run, sessions, costs, traces, health

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.db_pool = await init_db_pool(settings.DATABASE_URL)
    app.state.redis = redis.asyncio.Redis.from_url(settings.REDIS_URL)
    app.state.llm_client = LLMClient(settings.OPENAI_API_KEY, settings.LLM_MODEL)
    app.state.cost_tracker = CostTracker()
    yield
    await close_db_pool(app.state.db_pool)
    await app.state.redis.close()

def create_app() -> FastAPI:
    app = FastAPI(title="Multi-Agent Systems API", version="1.0.0", lifespan=lifespan)
    app.include_router(run.router)
    app.include_router(sessions.router)
    app.include_router(costs.router)
    app.include_router(traces.router)
    app.include_router(health.router)
    return app

app = create_app()
```

### 15.13 API Routes (`src/api/routes/run.py`)

```python
from fastapi import APIRouter, Depends, Request
from pydantic import BaseModel
from src.graph.builder import build_graph
from src.memory.short_term import ShortTermMemory
from src.memory.episodic import EpisodicMemoryStore

router = APIRouter()

class RunRequest(BaseModel):
    query: str
    session_id: str | None = None
    pattern: str = "default"  # default | orchestrator_worker | routing | critic_loop | map_reduce | pydantic_ai

class RunResponse(BaseModel):
    session_id: str
    final_output: str
    quality_warning: bool
    cost_summary: dict
    trace: list[dict]

@router.post("/run", response_model=RunResponse)
async def run_pipeline(request: RunRequest, req: Request):
    session_id = request.session_id or str(uuid.uuid4())
    pool = req.app.state.db_pool
    redis = req.app.state.redis

    # Load memory
    short_term = ShortTermMemory(redis)
    memory_ctx = await short_term.get_context(session_id)
    episodic = EpisodicMemoryStore(pool)
    episodic_ctx = await episodic.retrieve_relevant(request.query)

    # Build and invoke graph
    graph = build_graph()
    initial_state = {
        "query": request.query,
        "session_id": session_id,
        "pattern": request.pattern,
        "memory_context": f"{memory_ctx}\n\n{episodic_ctx}",
        "messages": [],
        "cost_log": [],
        "revision_count": 0,
        "score_history": [],
        "status": "running",
    }

    result = await graph.ainvoke(
        initial_state,
        config={"configurable": {"thread_id": session_id}},
    )

    # Persist cost log
    await persist_cost_log(session_id, result.get("cost_log", []))

    # Add to short-term memory
    await short_term.add_message(session_id, {"role": "user", "content": request.query})
    await short_term.add_message(session_id, {"role": "assistant", "content": result["final_output"]})

    return RunResponse(
        session_id=session_id,
        final_output=result["final_output"],
        quality_warning=result.get("quality_warning", False),
        cost_summary=cost_tracker.summary(),
        trace=result.get("cost_log", []),
    )
```

### 15.14 Streamlit Dashboard (`streamlit/dashboard.py`)

```python
import streamlit as st
import httpx
import json

st.set_page_config(page_title="Multi-Agent Systems", layout="wide")
st.title("Multi-Agent Systems Dashboard")

# Query input
col1, col2 = st.columns([3, 1])
with col1:
    query = st.text_area("Enter your query:", height=100)
with col2:
    pattern = st.selectbox("Pattern", [
        "default", "orchestrator_worker", "routing",
        "critic_loop", "map_reduce", "pydantic_ai",
    ])
    session_id = st.text_input("Session ID (optional)", value="")

if st.button("Run Pipeline") and query:
    with st.spinner("Running multi-agent pipeline..."):
        response = httpx.post(
            "http://localhost:8000/run",
            json={"query": query, "session_id": session_id or None, "pattern": pattern},
            timeout=120,
        )
        data = response.json()

    # Display final output
    st.subheader("Final Output")
    if data.get("quality_warning"):
        st.warning("Quality warning: max revisions reached without critic approval")
    st.write(data["final_output"])

    # Display cost summary
    st.subheader("Cost Summary")
    cost = data["cost_summary"]
    col_a, col_b, col_c = st.columns(3)
    col_a.metric("Total Cost", f"${cost['total_cost_usd']:.4f}")
    col_b.metric("Total Tokens", cost["total_tokens"])
    col_c.metric("Total Calls", cost["total_calls"])

    # Per-agent cost table
    st.subheader("Per-Agent Cost Breakdown")
    agent_costs = cost["by_agent"]
    st.table([
        {
            "Agent": agent,
            "Calls": v["calls"],
            "Tokens": v["total_tokens"],
            "Cost (USD)": f"${v['cost_usd']:.4f}",
            "Latency (ms)": f"{v['latency_ms']:.0f}",
        }
        for agent, v in agent_costs.items()
    ])

    # Display trace
    st.subheader("Agent Execution Trace")
    for entry in data.get("trace", []):
        st.write(f"**{entry.get('agent', 'unknown')}** — {entry.get('total_tokens', 0)} tokens, ${entry.get('cost_usd', 0):.4f}")

    # Session ID for continuity
    st.session_state["session_id"] = data["session_id"]
```

### 15.15 Agent Prompts (`src/agents/prompts.py`)

```python
ROUTER_SYSTEM_PROMPT = """\
You are a query router. Classify the user's query into one of:

Respond with the category, a confidence score (0.0-1.0), and brief reasoning.
"""

WRITER_SYSTEM_PROMPT = """\
You are a writer agent. Produce clear, well-structured content \
that fully addresses the user's query.

When revising:

Produce only the content, no meta-commentary.
"""

CRITIC_SYSTEM_PROMPT = """\
You are a critic agent. Review the draft and provide structured, \
actionable feedback.

Evaluate on:
1. Accuracy: Are the facts correct and well-supported?
2. Clarity: Is the writing clear and easy to understand?
3. Structure: Is the content well-organized with logical flow?
4. Completeness: Does it address the original query fully?
5. Style: Is the tone appropriate for the query type?

Scoring guide:

Be specific and actionable. "Add more examples" is better than "needs improvement".
"""

ORCHESTRATOR_SYSTEM_PROMPT = """\
You are an orchestrator agent. Decompose the given task into 2-5 \
independent subtasks that can be executed in parallel by specialist workers.

Each subtask should be:

Return the list of subtasks and a brief strategy explanation.
"""

WORKER_SYSTEM_PROMPT = """\
You are a specialist worker agent (role: {role}). \
Execute the given subtask thoroughly and return a complete result.

Focus on your specialty. Be detailed and accurate.
Return only the result, no meta-commentary.
"""

SYNTHESIZER_SYSTEM_PROMPT = """\
You are a synthesizer agent. Combine multiple worker results into \
a single coherent, well-structured response.

"""

SUMMARIZATION_SYSTEM_PROMPT = """\
You are a conversation summarizer. Summarize the conversation, \
preserving:

Discard:

Keep the summary under 500 words. Use bullet points for clarity.
"""

FALLBACK_SYSTEM_PROMPT = """\
You are a generalist agent. You handle queries that don't clearly fit \
a technical, creative, or analytical category. Provide a helpful, \
well-structured response covering multiple angles.
"""
```


## 16. Testing Strategy

### 16.1 Testing Principles


### 16.2 Test Categories

| Test File | What It Tests | LLM Mock? |
|-----------|---------------|-----------|
| `test_graph.py` | Graph compiles, all nodes present, edges correct, Mermaid renders | No LLM calls |
| `test_router.py` | Router classifies correctly, low-confidence → fallback, routing edges | Yes |
| `test_critic_loop.py` | Loop terminates on approval, max 3 revisions, convergence detection, quality_warning | Yes |
| `test_orchestrator.py` | Decomposition plan, parallel worker execution, synthesis, worker failure handling | Yes |
| `test_memory.py` | Buffer add/get, summarization triggers, episodic retrieval, Redis checkpoint | No LLM (mock summarizer) |
| `test_cost_tracker.py` | Cost calculation, per-agent attribution, summary aggregation | No LLM |
| `test_pydantic_ai.py` | Structured output validation, tool calls, dependency injection | Yes (mock Pydantic AI agent) |
| `test_api.py` | POST /run, GET /costs, GET /traces, GET /health, session CRUD | Yes (mock graph) |

### 16.3 Key Test Cases

```python
# test_critic_loop.py — convergence detection
async def test_loop_terminates_on_convergence(mock_llm):
    """If score doesn't improve between rounds, loop stops early."""
    mock_llm.structured_complete.side_effect = [
        CriticFeedback(feedback_text="needs work", score=60, ...),  # round 1
        CriticFeedback(feedback_text="still needs work", score=58, ...),  # round 2 (lower)
    ]
    # After round 2, score went down → convergence detection triggers → synthesizer
    result = await graph.ainvoke(initial_state)
    assert result["status"] == "completed"
    assert result["revision_count"] == 2  # stopped after 2, not 3
    assert result["quality_warning"] is True


# test_critic_loop.py — max revisions
async def test_loop_terminates_at_max(mock_llm):
    """Loop terminates at max 3 revisions even if critic never approves."""
    mock_llm.structured_complete.return_value = CriticFeedback(
        feedback_text="not good enough", score=50, approved=False, ...
    )
    result = await graph.ainvoke(initial_state)
    assert result["revision_count"] == 3
    assert result["quality_warning"] is True


# test_orchestrator.py — worker failure
async def test_worker_failure_handled(mock_llm):
    """If one worker fails, orchestrator proceeds with partial results."""
    mock_llm.complete.side_effect = [
        "worker 1 result",
        RuntimeError("worker 2 crashed"),
        "worker 3 result",
    ]
    result = await graph.ainvoke(initial_state)
    assert "[Worker 1 failed" in result["final_output"]
    assert "worker 1 result" in result["final_output"]
    assert "worker 3 result" in result["final_output"]


# test_router.py — low confidence fallback
async def test_low_confidence_routes_to_fallback(mock_llm):
    """Low router confidence routes to fallback agent."""
    mock_llm.structured_complete.return_value = QueryClassification(
        category=QueryCategory.TECHNICAL, confidence=0.4, reasoning="unsure"
    )
    result = await graph.ainvoke(initial_state)
    assert result["query_type"] == "technical"
    assert result["router_confidence"] == 0.4
    # Verify fallback node was executed (check trace)
```

### 16.4 Test Fixtures

```python
# conftest.py
import pytest
from unittest.mock import AsyncMock
from src.llm.client import LLMClient

@pytest.fixture
def mock_llm():
    """Mock LLM client for deterministic tests."""
    client = AsyncMock(spec=LLMClient)
    client.complete = AsyncMock()
    client.structured_complete = AsyncMock()
    client.model = "gpt-4o-mini"
    return client

@pytest.fixture
def sample_state():
    """Minimal AgentState for testing."""
    return {
        "query": "Explain how RAG works",
        "session_id": "test-session",
        "pattern": "default",
        "messages": [],
        "cost_log": [],
        "revision_count": 0,
        "score_history": [],
        "status": "running",
    }
```


## 17. Deployment

### 17.1 Docker Compose

```yaml
# docker-compose.yml
version: "3.9"

services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: multiagent
      POSTGRES_USER: agent
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-agentpass}
    ports:
    volumes:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U agent -d multiagent"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
    volumes:
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

  api:
    build:
      context: .
      dockerfile: docker/Dockerfile.api
    ports:
    environment:
      DATABASE_URL: postgresql://agent:agentpass@postgres:5432/multiagent
      REDIS_URL: redis://redis:6379/0
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      LLM_MODEL: gpt-4o-mini
      LOG_LEVEL: INFO
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: on-failure

  streamlit:
    build:
      context: .
      dockerfile: docker/Dockerfile.streamlit
    ports:
    environment:
      API_URL: http://api:8000
    depends_on:
    restart: on-failure

volumes:
  postgres_data:
  redis_data:
```

### 17.2 Dockerfile.api

```dockerfile
# docker/Dockerfile.api
FROM python:3.12-slim

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential libpq-dev && rm -rf /var/lib/apt/lists/*

COPY pyproject.toml .
RUN pip install --no-cache-dir -e .

COPY src/ src/
COPY scripts/ scripts/

EXPOSE 8000

CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 17.3 Dockerfile.streamlit

```dockerfile
# docker/Dockerfile.streamlit
FROM python:3.12-slim

WORKDIR /app

RUN pip install --no-cache-dir streamlit httpx pydantic

COPY streamlit/ streamlit/

EXPOSE 8501

CMD ["streamlit", "run", "streamlit/dashboard.py", "--server.port=8501", "--server.address=0.0.0.0"]
```

### 17.4 init.sql

See §12.2 for the full DDL. The `docker/postgres/init.sql` file contains all `CREATE EXTENSION`, `CREATE SCHEMA`, and `CREATE TABLE` statements.

### 17.5 .env.example

```bash
# .env.example
POSTGRES_PASSWORD=agentpass
OPENAI_API_KEY=sk-***
LLM_MODEL=gpt-4o-mini
DATABASE_URL=postgresql://agent:agentpass@localhost:5432/multiagent
REDIS_URL=redis://localhost:6379/0
LOG_LEVEL=INFO

# Optional observability
LANGSMITH_API_KEY=
LANGSMITH_PROJECT=multi-agent-systems
LANGFUSE_PUBLIC_KEY=
LANGFUSE_SECRET_KEY=
LANGFUSE_HOST=https://cloud.langfuse.com
```

### 17.6 Quick Start

```bash
# 1. Clone and configure
cp .env.example .env
# Edit .env: set OPENAI_API_KEY

# 2. Start all services
docker compose up --build

# 3. Verify
curl http://localhost:8000/health
# → {"status":"healthy","database":"connected","redis":"connected"}

# 4. Open dashboard
open http://localhost:8501

# 5. Submit a query via API
curl -X POST http://localhost:8000/run \
```


## 18. Roadmap & Milestones

| Milestone | Target | Deliverable | Status |
|-----------|--------|-------------|--------|
| M1: Graph compiles | Day 1 | LangGraph StateGraph builds, Mermaid renders, Docker stack up | — |
| M2: LLM + cost tracking | Day 2 | LLM client with retry, cost tracker, stubbed tools | — |
| M3: Router + routing | Day 3 | Router classifies, conditional edges route correctly | — |
| M4: Critic/revision loop | Day 5 | Writer → Critic → Revise loop with max 3 + convergence | — |
| M5: Orchestrator-worker | Day 7 | Parallel workers, synthesis, failure handling | — |
| M6: Pydantic AI agent | Day 8 | Researcher in Pydantic AI, comparison table | — |
| M7: Memory | Day 9 | Short-term buffer + summarization + episodic pgvector | — |
| M8: Dashboard + E2E | Day 10 | Streamlit dashboard, full E2E, cost < $0.05/query | — |

### Post-Project Extensions (v2 ideas, not in scope)



## 19. Risk Register

| # | Risk | Likelihood | Impact | Mitigation |
|---|------|------------|--------|------------|
| R1 | **Infinite revision loop** — critic never approves, writer keeps revising | Medium | High (cost, latency) | Hard cap at 3 revisions + convergence detection (score not improving → stop) |
| R2 | **Router misclassification** — technical query routed to creative agent | Medium | Medium (quality degradation) | Fallback agent for low confidence; critic can reject and trigger re-routing |
| R3 | **Worker failure in parallel execution** — one worker errors, blocks synthesis | Medium | Medium | `asyncio.gather(return_exceptions=True)`; synthesizer proceeds with partial results + note |
| R4 | **LLM API rate limits** — OpenAI returns 429 during multi-agent runs | Medium | High (pipeline stalls) | tenacity retry with exponential backoff (stop_after_attempt=3, wait 2-30s) |
| R5 | **Cost overrun** — multi-agent pipeline costs more than expected | Low | Medium | Per-agent cost tracking; total cost logged and displayed; target < $0.05/query on gpt-4o-mini |
| R6 | **Redis checkpoint corruption** — checkpoint state inconsistent after crash | Low | High (graph can't resume) | LangGraph's RedisSaver handles serialization; test crash recovery in Phase 7 |
| R7 | **Context window overflow** — long conversation + research notes exceed token limit | Medium | High (LLM call fails) | Short-term memory summarization (compress after 10 turns); truncate research notes to ~12000 tokens |
| R8 | **LangGraph API breaking changes** — LangGraph 0.2 API changes in future versions | Low | Medium | Pin version in pyproject.toml; LangGraph is relatively stable; check migration guide on update |
| R9 | **Pydantic AI immaturity** — framework is pre-1.0, API may change | Medium | Low (only affects one agent) | Isolate Pydantic AI code in `researcher_pydantic.py`; raw function-calling version is the default |
| R10 | **Deadlock in supervisor pattern** — supervisor keeps routing to a failing worker | Low | High (pipeline hangs) | Supervisor has a max iteration count; after N retries, routes to fallback or terminates |


## 20. Appendix

### Appendix A: LangGraph Cheat Sheet

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

# 1. Define state
class MyState(TypedDict, total=False):
    query: str
    result: str

# 2. Define nodes (async functions that take state, return partial update)
async def my_node(state: MyState) -> dict:
    return {"result": f"processed: {state['query']}"}

# 3. Build graph
graph = StateGraph(MyState)
graph.add_node("my_node", my_node)
graph.add_edge(START, "my_node")
graph.add_edge("my_node", END)

# 4. Conditional edges
def route(state: MyState) -> str:
    if state["result"].startswith("error"):
        return "error_handler"
    return "success_handler"

graph.add_conditional_edges("my_node", route, {
    "error_handler": "error_handler",
    "success_handler": "success_handler",
})

# 5. Compile with checkpointer
from langgraph.checkpoint.redis import RedisSaver
compiled = graph.compile(checkpointer=RedisSaver(redis_client))

# 6. Invoke
result = await compiled.ainvoke(
    {"query": "hello"},
    config={"configurable": {"thread_id": "session-1"}},
)

# 7. Visualize
print(compiled.get_graph().draw_mermaid())
```

### Appendix B: Pydantic AI vs Raw Function Calling — Quick Reference

| Operation | Raw Function Calling | Pydantic AI |
|-----------|---------------------|-------------|
| Define agent | Manual prompt + tool schemas | `Agent(model, deps_type, output_type, system_prompt)` |
| Define tool | JSON schema dict | `@agent.tool` decorator |
| Call LLM | `client.chat.completions.create()` | `agent.run(query, deps=deps)` |
| Parse output | `json.loads()` + manual validation | Automatic Pydantic validation |
| Retry on parse error | Manual try/except + retry | Automatic |
| Inject dependencies | Pass as function args | `ctx.deps` (typed) |
| Stream tokens | `stream=True` + iterate | `agent.run_stream()` |

### Appendix C: Cost Model & Token Budget

```
gpt-4o-mini pricing:
  Input:  $0.15 per 1M tokens
  Output: $0.60 per 1M tokens

Typical pipeline (default pattern):
  Router:     ~200 input, ~50 output  = $0.000030 + $0.000030 = $0.000060
  Research:   ~500 input, ~300 output = $0.000075 + $0.000180 = $0.000255
  Writer:     ~800 input, ~400 output = $0.000120 + $0.000240 = $0.000360
  Critic:     ~600 input, ~200 output = $0.000090 + $0.000120 = $0.000210
  Synthesizer:~400 input, ~200 output = $0.000060 + $0.000120 = $0.000180
  ────────────────────────────────────────────────────────────────
  Total (1 round): ~$0.001065
  Total (3 rounds, worst case): ~$0.002130 + 2×(writer+critic) = ~$0.003540

  Well under the $0.05/query target.

Orchestrator-worker pattern (5 workers):
  Orchestrator: ~300 input, ~100 output = $0.000105
  5 Workers:    5 × (~400 input, ~300 output) = $0.001050
  Synthesizer:  ~1000 input, ~400 output = $0.000390
  ────────────────────────────────────────────────────────────────
  Total: ~$0.001545 (still well under $0.05)
```

### Appendix D: Sample Multi-Agent Trace

```json
{
  "session_id": "abc-123",
  "query": "Compare PostgreSQL and MongoDB for a startup",
  "pattern": "default",
  "query_type": "analytical",
  "router_confidence": 0.92,
  "revision_count": 2,
  "critic_score": 82,
  "critic_approved": true,
  "quality_warning": false,
  "cost_summary": {
    "total_cost_usd": 0.001834,
    "total_tokens": 4820,
    "total_calls": 7,
    "by_agent": {
      "router": {"calls": 1, "total_tokens": 280, "cost_usd": 0.000060, "latency_ms": 340},
      "research_analytical": {"calls": 1, "total_tokens": 850, "cost_usd": 0.000255, "latency_ms": 1200},
      "writer": {"calls": 2, "total_tokens": 2400, "cost_usd": 0.000720, "latency_ms": 2100},
      "critic": {"calls": 2, "total_tokens": 1600, "cost_usd": 0.000480, "latency_ms": 1800},
      "synthesizer": {"calls": 1, "total_tokens": 600, "cost_usd": 0.000319, "latency_ms": 900}
    }
  },
  "node_trace": [
    {"node": "router_node", "duration_ms": 340, "status": "success"},
    {"node": "research_analytical", "duration_ms": 1200, "status": "success"},
    {"node": "writer_node", "duration_ms": 1100, "status": "success", "revision": 1},
    {"node": "critic_node", "duration_ms": 900, "status": "success", "score": 68},
    {"node": "writer_node", "duration_ms": 1000, "status": "success", "revision": 2},
    {"node": "critic_node", "duration_ms": 900, "status": "success", "score": 82},
    {"node": "synthesizer_node", "duration_ms": 900, "status": "success"}
  ],
  "final_output": "PostgreSQL and MongoDB serve different use cases..."
}
```

### Appendix E: Relationship to Roadmap Projects

```
Tier 1: RAG & Retrieval
  P1: Advanced RAG ─────────────────────────┐
  P2: Secure RAG ──────────────────────────┐│
  P3: Advanced RAG Architectures ─────────┐││
                                           │││
Tier 2: LLM Orchestration & Agents         │││
  P4: Data Analysis Agent ──────────────┐  │││
    (tool use, ReAct, function calling) │  │││
  P5: Document Processing Pipeline ───┐ │  │││
    (pipeline, retry, structured out) │ │  │││
  P6: Multi-Agent Systems ◄───────────┤-┤──┘││
    (LangGraph, multi-agent, memory)   │ │   │
                                       │ │   │
Tier 3: AI Evaluation & Observability   │ │   │
  P7: LLM Evaluation Harness ──────────┤-┤───┘│
  P8: Prompt Versioning & A/B Testing ─┘ │    │
                                         │    │
Tier 4: AI Infrastructure & Production   │    │
  P9: LLM Serving Gateway ───────────────┤    │
  P10: PII Redaction Guardrails ─────────┘    │
                                              │
Tier 5: Capstone                              │
  P11: Customer Support AI System ◄───────────┘
    (consumes P6's multi-agent orchestration,
     cost tracking, memory management)
  P12: Technical Case Study
```


*End of Implementation Plan — Project 6: Multi-Agent Systems*