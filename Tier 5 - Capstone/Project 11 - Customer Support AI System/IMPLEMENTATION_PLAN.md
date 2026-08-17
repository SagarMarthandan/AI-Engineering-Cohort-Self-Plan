# End-to-End Customer Support AI System — Detailed Implementation Plan

> **Elevator pitch:** A production-grade AI system that ingests customer support tickets, redacts PII, classifies urgency/category/sentiment, retrieves relevant solutions via hybrid RAG, drafts grounded responses with citations, auto-resolves high-confidence cases, routes complex cases to a multi-agent pipeline and human review queue, evaluates its own output, tracks per-ticket LLM costs, logs a full audit trail, and exposes everything on a monitoring dashboard — combining every skill from Projects 1–10 into one cohesive system.

> **This is the capstone.** It is not a new collection of techniques — it is the integration layer that proves you can assemble RAG, agents, evaluation, guardrails, cost control, and observability into a single production AI product. Every component maps to a skill acquired in a previous project. The architecture section makes those mappings explicit.

---

## Table of Contents

1. [Project Goals & Non-Goals](#1-project-goals--non-goals)
2. [Component-to-Project Mapping](#2-component-to-project-mapping)
3. [Architecture Overview](#3-architecture-overview)
4. [Data Flow: Single Ticket Lifecycle](#4-data-flow-single-ticket-lifecycle)
5. [Tech Stack](#5-tech-stack)
6. [Database Schema](#6-database-schema)
7. [Project Structure](#7-project-structure)
8. [Implementation Phases](#8-implementation-phases)
9. [Component Specifications](#9-component-specifications)
10. [Human Review Queue Design](#10-human-review-queue-design)
11. [Monitoring Dashboard Layout](#11-monitoring-dashboard-layout)
12. [Feedback Loop Architecture](#12-feedback-loop-architecture)
13. [Security & Safety Considerations](#13-security--safety-considerations)
14. [API Specification](#14-api-specification)
15. [Testing Strategy](#15-testing-strategy)
16. [Deployment](#16-deployment)
17. [Roadmap & Milestones](#17-roadmap--milestones)
18. [Risk Register](#18-risk-register)
19. [Appendix A: Quick Start](#appendix-a-quick-start)
20. [Appendix B: Key Design Decisions](#appendix-b-key-design-decisions)

---

## 1. Project Goals & Non-Goals

### Goals

| # | Goal | Success Metric |
|---|------|----------------|
| G1 | **End-to-end ticket processing** — from ingestion to resolution or human handoff | A ticket enters the system and reaches a terminal state (auto-resolved, human-approved, or human-rejected) with full audit trail |
| G2 | **PII redaction before LLM processing** — no customer PII reaches the LLM | 100% of tickets scanned by Presidio before any LLM call; PII entities replaced with reversible tokens |
| G3 | **Multi-signal classification** — urgency, category, sentiment in one structured LLM call | Every ticket has classification metadata stored; classification drives routing decisions |
| G4 | **Hybrid RAG with re-ranking** — retrieve from support docs, FAQs, and past resolved tickets | Top-k retrieval uses BM25 + vector search + cross-encoder re-ranking (from P1); GraphRAG for relationship-heavy queries (from P3) |
| G5 | **Confidence-gated auto-response** — high-confidence drafts auto-send; low-confidence route to humans | Confidence score computed from retrieval distance + classification certainty + LLM self-assessment; threshold is configurable |
| G6 | **Human review queue** — agents can review, edit, approve, or reject AI drafts | Streamlit dashboard with queue, draft editor, feedback capture, and approval workflow |
| G7 | **Evaluation harness** — automated eval on auto-responses (faithfulness, relevance, citation accuracy) | Eval suite runs on a schedule and on-demand; scores tracked over time (from P7) |
| G8 | **Cost tracking** — per-ticket LLM cost and per-tenant budget enforcement | Every LLM call logged with token counts and cost; budget exceeded → route to human (no LLM call) |
| G9 | **Full audit trail** — every action logged immutably | Ticket received → PII scanned → classified → RAG retrieved → response drafted → approved/rejected → sent; all timestamped |
| G10 | **Monitoring dashboard** — operational metrics at a glance | Streamlit dashboard: ticket volume, auto-resolution rate, avg response time, cost/ticket, eval scores, queue depth |
| G11 | **Feedback loop** — agent edits/rejections feed back into eval dataset | Rejected/edited drafts become labeled eval examples for continuous improvement |
| G12 | **Production-deployable** — containerized, documented, testable end-to-end | `docker compose up` starts the full system (API, Postgres, Redis, Streamlit dashboards); health checks pass |

### Non-Goals (explicitly out of scope)

- Real-time streaming responses (v1 processes tickets asynchronously)
- Multi-language ticket support (v1 is English-only)
- Fine-tuning classification or response models
- Email server integration (v1 uses API endpoint + optional IMAP poller stub)
- Voice/phone support channel
- Customer-facing chatbot (this is an agent-assist + auto-response system, not a chat UI)
- SLA management / escalation timers (v1 routes by confidence, not by SLA breach)
- Multi-region deployment (v1 is single-region Docker Compose)

---

## 2. Component-to-Project Mapping

This is the defining table of the capstone. Every component traces back to a skill demonstrated in a previous project. The "Integration Delta" column describes what is new beyond the original project.

| # | Component | Source Project | Skill Reused | Integration Delta (What's New Here) |
|---|-----------|---------------|--------------|-------------------------------------|
| 1 | Ticket ingestion | **P5** (Document Pipeline) | Parse → classify → extract → route pattern; async pipeline orchestration | Input is a support ticket (not a document); ingestion triggers the full processing pipeline; email parser stub alongside API endpoint |
| 2 | PII redaction | **P10** (PII Guardrails) | Presidio NER + regex + LLM-based detection; reversible tokenization; GDPR logging | PII redacted *before* LLM calls (not after); tokens reversed in the final response so the customer sees their real data; redaction events in audit trail |
| 3 | Ticket classification | **P5** (Document Pipeline) | LLM structured output classification; Pydantic validation | Three dimensions in one call (urgency + category + sentiment); classification drives routing and RAG query formulation |
| 4 | Knowledge base RAG | **P1** (Advanced RAG) + **P2** (Secure RAG) + **P3** (Adv RAG Architectures) | BM25 + vector hybrid search; cross-encoder re-ranking; tenant-aware permission filtering; citations; GraphRAG for relationship-heavy queries; agentic RAG for iterative retrieval on complex tickets | Knowledge base includes past resolved tickets (not just docs); retrieval query is formulated from ticket classification; permission filter applied if multi-tenant; architecture decision framework (from P3) selects vanilla vs GraphRAG vs agentic per ticket complexity |
| 5 | Response drafting | **P2** (Secure RAG) | Grounded answer generation with citation instructions; context-only prompt | Draft is a customer-facing response (not an internal answer); tone adapted to sentiment; citations included for human reviewer (stripped from auto-response) |
| 6 | Confidence scoring | **P7** (Eval Harness) + **P1** (RAG) | Retrieval distance as quality signal; LLM self-assessment | Composite score from retrieval distance + classification certainty + LLM confidence; threshold gates auto-response vs. human review |
| 7 | Human review queue | **P8** (Prompt A/B) + **P7** (Eval) | Streamlit dashboard patterns; feedback capture | Queue with priority sorting; inline draft editor; approve/reject/edit with structured feedback; feedback feeds eval dataset |
| 8 | Evaluation harness | **P7** (Eval Harness) | LLM-as-judge; faithfulness, relevance, citation accuracy metrics; regression testing; context engineering (window management, compression) | Runs on auto-responses in production (not just test data); scores tracked over time on monitoring dashboard; eval dataset grows from feedback loop |
| 9 | Cost tracking | **P9** (LLM Gateway) | Per-call token counting; cost calculation; budget enforcement | Per-ticket cost aggregation; per-tenant budget with hard cutoff (budget exceeded → human route, no LLM call); cost displayed on monitoring dashboard |
| 10 | Audit trail | **P2** (Secure RAG) | Append-only audit log with DB triggers; immutable records | Audit trail spans the full ticket lifecycle (not just queries); every pipeline stage logged as a separate event with stage-specific metadata |
| 11 | Monitoring dashboard | **P7** + **P8** + **P9** | Streamlit dashboard patterns; metric tracking over time; cost visualization | Unified operational dashboard combining metrics from all three prior dashboards; real-time queue depth; eval score trends |
| 12 | Feedback loop | **P7** (Eval Harness) + **P8** (Prompt A/B) | Eval dataset management; prompt improvement from data | Agent edits/rejections automatically become labeled eval examples; eval dataset versioned; prompt improvements validated against growing dataset |
| 13 | Multi-agent pipeline | **P6** (Multi-Agent Systems) | LangGraph orchestration; orchestrator-worker; critic/revision loop; agent memory | Complex tickets routed to a multi-agent pipeline (router → researcher → writer → critic); critic loop ensures response quality before auto-send; short-term memory for conversation context |
| 14 | RAG architecture selection | **P3** (Adv RAG Architectures) | GraphRAG; agentic RAG; decision framework | Decision framework from P3 selects RAG architecture per ticket: vanilla RAG for simple lookups, GraphRAG for relationship queries, agentic RAG for multi-step troubleshooting |
### Visual Mapping

```
Previous Projects (P1–P10)         Capstone System (P11)
────────────────────────           ────────────────────

P1 Advanced RAG ──────────────────►┐
  (hybrid search, re-ranking)      ├──► Knowledge Base RAG (#4)
                                   │
P2 Secure RAG ────────────────────►┤
  (permissions, citations, audit)  │    ├──► Response Drafting (#5)
                                   │    ├──► Audit Trail (#10)
                                   │    └──► Knowledge Base RAG (#4)
                                   │
P3 Advanced RAG Architectures ────►┤
  (GraphRAG, agentic RAG,          │    ├──► Knowledge Base RAG (#4)
   decision framework)             │    └──► RAG Architecture Selection (#14)
                                   │
P4 Data Analysis Agent ───────────►┤
  (tool use, error recovery)       │    └──► (pattern: structured LLM
                                   │         output + validation)
                                   │
P5 Document Pipeline ─────────────►┤
  (classify → extract → route)     │    ├──► Ticket Ingestion (#1)
                                   │    └──► Ticket Classification (#3)
                                   │
P6 Multi-Agent Systems ───────────►┤
  (LangGraph, orchestrator-worker, │    └──► Multi-Agent Pipeline (#13)
   critic loop, agent memory)      │
                                   │
P7 Eval Harness ──────────────────►┤
  (LLM-as-judge, metrics)          │    ├──► Confidence Scoring (#6)
                                   │    ├──► Evaluation Harness (#8)
                                   │    └──► Feedback Loop (#12)
                                   │
P8 Prompt A/B Testing ────────────►┤
  (prompt versioning, dashboards)  │    ├──► Human Review Queue (#7)
                                   │    └──► Feedback Loop (#12)
                                   │
P9 LLM Gateway ───────────────────►┤
  (caching, cost tracking, routing)│    └──► Cost Tracking (#9)
                                   │
P10 PII Guardrails ───────────────►┘
  (Presidio, GDPR, redaction)           └──► PII Redaction (#2)
```

---

## 3. Architecture Overview

### 3.1 System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              EXTERNAL INPUTS                                        │
│                                                                                     │
│  ┌──────────────────┐     ┌──────────────────┐          ┌────────────────────┐      │
│  │  Support Ticket   │     │  Email Poller     │          │  Knowledge Base    │      │
│  │  API (POST /tickets)│   │  (IMAP stub)      │          │  Ingestion API     │      │
│  │  (from helpdesk)  │     │  (optional)       │          │  (POST /kb/upload) │      │
│  └────────┬─────────┘     └────────┬─────────┘          └─────────┬──────────┘      │
└───────────┼────────────────────────┼──────────────────────────────┼─────────────────┘
            │                        │                              │
            ▼                        ▼                              ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              FASTAPI APPLICATION                                     │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │                           TICKET PROCESSING PIPELINE                          │  │
│  │                                                                              │  │
│  │  ┌─────────────┐   ┌──────────────┐   ┌───────────────┐   ┌──────────────┐  │  │
│  │  │  1. Ingest  │──►│  2. PII      │──►│  3. Classify  │──►│  4. RAG      │  │  │
│  │  │  (receive + │   │  Redact      │   │  (urgency,    │   │  Retrieve    │  │  │
│  │  │  validate)  │   │  (Presidio)  │   │   category,   │   │  (hybrid +   │  │  │
│  │  │             │   │              │   │   sentiment)  │   │   re-rank)   │  │  │
│  │  └─────────────┘   └──────────────┘   └───────────────┘   └──────┬───────┘  │  │
│  │                                                              │          │  │
│  │  ┌─────────────┐   ┌──────────────┐   ┌───────────────┐   ┌──────▼───────┐  │  │
│  │  │  8. Audit   │◄──│  7. Cost     │◄──│  6. Confidence│◄──│  5. Draft    │  │  │
│  │  │  Log        │   │  Track       │   │  Score +      │   │  Response    │  │  │
│  │  │  (every     │   │  (per-ticket │   │  Route        │   │  (LLM +      │  │  │
│  │  │  stage)     │   │   + per-     │   │  Decision     │   │   citations) │  │  │
│  │  │             │   │   tenant)    │   │               │   │              │  │  │
│  │  └─────────────┘   └──────────────┘   └───────┬───────┘   └──────────────┘  │  │
│  └──────────────────────────────────────────────┼──────────────────────────────┘  │
│                                                 │                                   │
│                                    ┌────────────┴────────────┐                      │
│                                    │   Routing Decision      │                      │
│                                    │                         │                      │
│                                    │  confidence ≥ threshold │                      │
│                                    │  AND budget not exceeded│                      │
│                                    │  AND urgency ≠ critical │                      │
│                                    │                         │                      │
│                                    │  YES → auto-respond     │                      │
│                                    │  NO  → human review     │                      │
│                                    └────┬───────────┬────────┘                      │
│                                         │           │                               │
│  ┌──────────────────────────────────────┼───────────┼───────────────────────────┐  │
│  │                           EVALUATION & FEEDBACK LAYER                        │  │
│  │                                      │           │                           │  │
│  │  ┌──────────────────┐     ┌──────────▼─┐   ┌────▼──────────┐                │  │
│  │  │  Eval Harness    │     │  Auto-     │   │  Human Review │                │  │
│  │  │  (faithfulness,  │     │  Response  │   │  Queue API    │                │  │
│  │  │   relevance,     │     │  Sender    │   │  (GET/POST    │                │  │
│  │  │   citation acc.) │     │            │   │   /review/*)  │                │  │
│  │  └────────┬─────────┘     └────────────┘   └───────┬───────┘                │  │
│  │           │                                         │                        │  │
│  │  ┌────────▼─────────────────────────────────────────▼──────┐                │  │
│  │  │              Feedback Collector                         │                │  │
│  │  │  (agent edits/rejections → eval dataset → prompt        │                │  │
│  │  │   improvement loop)                                     │                │  │
│  │  └─────────────────────────────────────────────────────────┘                │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────────────┐ │
│  │                           SHARED SERVICES                                    │ │
│  │                                                                              │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │ │
│  │  │ Redis Cache  │  │ LLM Client   │  │ Embedding    │  │ Audit Logger │    │ │
│  │  │ (classification│ │ (OpenAI /    │  │ Client       │  │ (append-only │    │ │
│  │  │  cache,       │  │  fallback)   │  │ (OpenAI /    │  │  DB writer)  │    │ │
│  │  │  response     │  │              │  │  local BGE)  │  │              │    │ │
│  │  │  cache)       │  │              │  │              │  │              │    │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘    │ │
│  └──────────────────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                    ┌────────────────────┼────────────────────┐
                    │                    │                    │
           ┌────────▼───────┐  ┌────────▼───────┐  ┌────────▼───────┐
           │  PostgreSQL    │  │  Streamlit:    │  │  Streamlit:    │
           │  + pgvector    │  │  Review Queue  │  │  Monitoring    │
           │                │  │  Dashboard     │  │  Dashboard     │
           │  Tables:       │  │  (port 8501)   │  │  (port 8502)   │
           │  - tickets     │  │                │  │                │
           │  - classifications│  │  Agent views   │  │  Ops views     │
           │  - kb_documents│  │  pending drafts│  │  metrics,      │
           │  - kb_chunks   │  │  edits, approves│  │  trends,       │
           │  - draft_      │  │  rejects,      │  │  costs,         │
           │    responses   │  │  gives feedback│  │  eval scores    │
           │  - review_queue│  │                │  │                │
           │  - audit_logs  │  │                │  │                │
           │  - cost_ledger │  │                │  │                │
           │  - eval_results│  │                │  │                │
           │  - feedback    │  │                │  │                │
           │  - tenants     │  │                │  │                │
           │  - agents      │  │                │  │                │
           └────────────────┘  └────────────────┘  └────────────────┘
```

### 3.2 Service Topology

```
                    ┌─────────────────────────────────┐
                    │         Docker Network          │
                    │      (support-ai_default)       │
                    │                                 │
  Port 8000 ───────►│  ┌──────────┐  ┌───────────┐   │
  (API)             │  │   api    │  │ postgres  │   │
                    │  │ (FastAPI)│  │ (pgvector)│   │
  Port 8501 ───────►│  └────┬─────┘  └─────▲─────┘   │
  (Review Queue)    │       │              │         │
                    │  ┌────▼─────┐  ┌─────┴─────┐   │
  Port 8502 ───────►│  │ streamlit│  │   redis   │   │
  (Monitoring)      │  │ review   │  │  (cache)  │   │
                    │  └──────────┘  └───────────┘   │
                    │  ┌──────────┐                  │
                    │  │ streamlit│                  │
                    │  │ monitor  │                  │
                    │  └──────────┘                  │
                    └─────────────────────────────────┘

  External calls:
    api ──► OpenAI API (embeddings + LLM)
    api ──► (optional) Ollama for local fallback
```

---

## 4. Data Flow: Single Ticket Lifecycle

This diagram traces a single ticket from arrival to terminal state. It shows every component, every decision point, and every audit event.

```
                         ┌──────────────────────┐
                         │  Ticket Arrives       │
                         │  POST /tickets        │
                         │  {subject, body,      │
                         │   customer_id,        │
                         │   tenant_id}          │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │  STAGE 1: INGEST     │   Audit: "ticket_received"
                         │  Validate schema     │
                         │  Store in tickets    │
                         │  table (status=new)  │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │  STAGE 2: PII REDACT │   Audit: "pii_scanned"
                         │  Presidio + regex    │   Store: pii_entities map
                         │  Detect: names,      │   (token → original value)
                         │   emails, phones,    │
                         │   addresses, IBAN    │
                         │  Replace with tokens │
                         │  [NAME_1], [EMAIL_1] │
                         └──────────┬───────────┘
                                    │
                          ┌─────────┴─────────┐
                          │ PII found?        │
                          │                   │
                          │ YES → redacted    │
                          │      text to LLM  │
                          │ NO  → original    │
                          │      text to LLM  │
                          └─────────┬─────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │  STAGE 3: CLASSIFY   │   Audit: "classified"
                         │  LLM (GPT-4o-mini)   │   Store: classifications
                         │  Structured output:  │
                         │  {                   │
                         │    urgency: "high",  │
                         │    category: "billing│
                         │    sentiment: "angry"│
                         │    confidence: 0.85  │
                         │  }                   │
                         │  Pydantic validation │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │  STAGE 4: RAG        │   Audit: "rag_retrieved"
                         │  RETRIEVE            │
                         │                      │
                         │  4a. Formulate query │
                         │      from ticket +   │
                         │      classification  │
                         │                      │
                         │  4b. BM25 search     │
                         │      (keyword)       │
                         │                      │
                         │  4c. Vector search   │
                         │      (pgvector)      │
                         │      + tenant filter  │
                         │                      │
                         │  4d. Merge +         │
                         │      cross-encoder   │
                         │      re-rank         │
                         │                      │
                         │  4e. Top-k chunks    │
                         │      with citations  │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │  STAGE 5: DRAFT      │   Audit: "response_drafted"
                         │  RESPONSE            │
                         │                      │
                         │  LLM generates       │
                         │  customer-facing     │
                         │  response using:     │
                         │  - redacted ticket   │
                         │  - retrieved context │
                         │  - classification    │
                         │    (tone from        │
                         │     sentiment)       │
                         │  - citation markers  │
                         │    [1], [2]          │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │  STAGE 6: CONFIDENCE │   Audit: "confidence_scored"
                         │  SCORE + ROUTE       │
                         │                      │
                         │  Composite score:    │
                         │  - retrieval dist    │
                         │    (avg of top-k)    │
                         │  - classification    │
                         │    confidence        │
                         │  - LLM self-assess   │
                         │  - context coverage  │
                         │                      │
                         │  score = w1*ret +    │
                         │   w2*cls + w3*self + │
                         │   w4*cov             │
                         └──────────┬───────────┘
                                    │
                         ┌──────────┴──────────┐
                         │  ROUTING DECISION    │
                         │                      │
                         │  Auto-respond if:    │
                         │  - score ≥ 0.75      │
                         │  - urgency ≠ critical│
                         │  - tenant budget OK  │
                         │  - category ≠        │
                         │    complaint (angry) │
                         │                      │
                         │  Else: human review  │
                         └───────┬──────┬───────┘
                                 │      │
                    AUTO ────────┘      └──────── HUMAN
                                 │                    │
                                 ▼                    ▼
                    ┌──────────────────┐  ┌──────────────────────┐
                    │  STAGE 7a: AUTO  │  │  STAGE 7b: ENQUEUE   │
                    │  RESPOND         │  │  FOR HUMAN REVIEW    │
                    │                  │  │                      │
                    │  Restore PII     │  │  Insert into         │
                    │  tokens → real   │  │  review_queue table  │
                    │  values in draft │  │  (status=pending,    │
                    │                  │  │   priority=urgency)  │
                    │  Strip citation  │  │                      │
                    │  markers from    │  │  Audit: "enqueued_   │
                    │  customer-facing │  │   for_review"        │
                    │  version         │  │                      │
                    │                  │  │  Agent sees in       │
                    │  Send response   │  │  Streamlit review    │
                    │  (mark ticket    │  │  queue dashboard     │
                    │   status=        │  │                      │
                    │   auto_resolved) │  │  Agent actions:      │
                    │                  │  │  - EDIT + APPROVE    │
                    │  Audit: "auto_   │  │  - APPROVE as-is     │
                    │   responded"     │  │  - REJECT + reason   │
                    └────────┬─────────┘  │                      │
                             │            │  Each action:         │
                             │            │  - feedback captured  │
                             │            │  - PII restored       │
                             │            │  - response sent      │
                             │            │  - ticket status      │
                             │            │    updated            │
                             │            │  - audit logged       │
                             │            │  - feedback → eval    │
                             │            │    dataset            │
                             │            └──────────┬───────────┘
                             │                       │
                             ▼                       ▼
                    ┌────────────────────────────────────────┐
                    │  STAGE 8: COST TRACKING                │
                    │  Sum all LLM call costs for this ticket│
                    │  (classification + RAG query embed +   │
                    │   response draft + eval)               │
                    │  Insert into cost_ledger                │
                    │  Update tenant budget balance           │
                    │  Audit: "cost_recorded"                │
                    └──────────────────────┬─────────────────┘
                                           │
                                           ▼
                    ┌────────────────────────────────────────┐
                    │  STAGE 9: EVALUATION (async,           │
                    │  for auto-responses only)              │
                    │                                        │
                    │  Run eval suite on the draft:          │
                    │  - faithfulness (is response grounded  │
                    │    in retrieved context?)              │
                    │  - relevance (does it address the      │
                    │    ticket?)                            │
                    │  - citation accuracy (do citations     │
                    │    point to correct sources?)          │
                    │  Store in eval_results                 │
                    │  Audit: "evaluated"                    │
                    └────────────────────────────────────────┘
```

### 4.1 Failure Modes & Fallbacks

| Stage | Failure Mode | Fallback | Audit Event |
|-------|-------------|----------|-------------|
| PII Redact | Presidio service error | Skip redaction, route ticket to human review (no LLM call), flag as "PII scan failed" | `pii_scan_failed` |
| Classify | LLM timeout / error | Retry 2× with backoff; if still failing, route to human with classification=null | `classification_failed` |
| RAG Retrieve | No results (empty KB) | Set confidence=0, route to human, draft = "We're reviewing your ticket and will respond shortly" | `rag_empty` |
| RAG Retrieve | Vector search timeout | Fall back to BM25-only search (skip vector); log degraded retrieval | `rag_degraded_bm25_only` |
| Draft Response | LLM timeout / error | Retry 2×; if still failing, route to human with draft=null | `draft_failed` |
| Confidence Score | All signals unavailable | Default to 0 (route to human) | `confidence_defaulted` |
| Auto-Respond | Send mechanism fails | Retry 3×; if still failing, enqueue for human review | `auto_send_failed` |
| Cost Track | Budget exceeded (checked pre-draft) | Skip draft LLM call, route directly to human, audit `budget_exceeded` | `budget_exceeded` |
| Eval Harness | LLM-as-judge error | Skip eval for this ticket, log warning; eval is non-blocking | `eval_skipped` |
| Database | Postgres connection lost | Circuit breaker opens; queue tickets in Redis; replay when DB recovers | `db_circuit_open` |

---

## 5. Tech Stack

| Layer | Technology | Rationale | From Project |
|-------|-----------|-----------|-------------|
| **Language** | Python 3.11+ | Ecosystem maturity, async support, type hints | All |
| **Web Framework** | FastAPI | Async, auto OpenAPI docs, Pydantic validation, background tasks | P1, P2, P9 |
| **Database** | PostgreSQL 16 + pgvector | Single system for relational data + vector search; SQL-level tenant filtering | P1, P2 |
| **Embedding Model** | `text-embedding-3-small` (OpenAI) | 1536-dim, $0.02/1M tokens, reliable. Fallback: `bge-large-en-v1.5` (local) | P1, P2, P3, P5 |
| **LLM** | `gpt-4o-mini` (OpenAI) | Fast, cheap ($0.15/1M input, $0.60/1M output), good at structured output. Fallback: Llama 3.1 8B via Ollama | All |
| **Re-ranker** | `cross-encoder/ms-marco-MiniLM-L-6-v2` (sentence-transformers) | Lightweight, fast, improves retrieval precision significantly | P1 |
| **Orchestration** | LangChain | Document loaders, text splitters, retrieval chains; industry standard | P1, P2, P3, P4, P5 |
| **Agent Orchestration** | LangGraph | Stateful multi-agent workflows; orchestrator-worker; critic loops; graph-based routing | P6 |
| **PII Detection** | Microsoft Presidio + spaCy `en_core_web_lg` | NER + regex + custom recognizers; GDPR-aligned; reversible tokenization | P10 |
| **Caching** | Redis 7 | Classification cache (same ticket text → same classification), response cache (semantic), circuit breaker state | P9 |
| **Dashboards** | Streamlit | Rapid Python-native dashboards; review queue + monitoring as separate apps | P7, P8, P9 |
| **Validation** | Pydantic v2 | Structured LLM output validation (classification, response, eval scores) | P5 |
| **Containerization** | Docker + Docker Compose | Reproducible multi-service deployment | All |
| **Testing** | pytest + pytest-asyncio + httpx | Async test support, FastAPI-native test client | All |
| **CI/CD** | GitHub Actions | Automated test + lint on every push | All |
| **Linting** | ruff | Fast, replaces flake8 + isort + black | All |
| **Structured Logging** | structlog | JSON logs for observability and audit correlation | P9 |

### Why These Choices (Capstone-Specific Rationale)

**pgvector over dedicated vector DB:** The knowledge base needs tenant filtering (from P2) and the ticket metadata, classifications, audit logs, and cost ledger all live in the same database. Using pgvector avoids a separate vector store and its sync problems. The knowledge base is modest in size (support docs + resolved tickets, not millions of chunks), so pgvector's HNSW index is performant.

**Redis for caching (from P9):** Classification is deterministic for identical ticket text — caching avoids redundant LLM calls. Semantic response cache catches near-duplicate tickets ("how do I reset my password" variants). Redis also holds circuit breaker state and rate limit counters.

**Streamlit for both dashboards:** The review queue needs an interactive UI (edit, approve, reject). The monitoring dashboard needs charts and metrics. Streamlit handles both without a separate frontend build. Two separate Streamlit apps (different ports) keep concerns isolated — agents use the review queue, ops uses the monitoring dashboard.

**GPT-4o-mini as primary LLM:** The capstone processes many tickets; cost matters. GPT-4o-mini is 10× cheaper than GPT-4o and handles classification and response drafting well. The cost tracking component (from P9) makes the cost tradeoff visible on the dashboard.

---

## 6. Database Schema

### 6.1 Extensions

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "vector";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";   -- BM25-like trigram search
```

### 6.2 Tables

```sql
-- =====================================================
-- TENANTS: multi-tenant support (from P2 pattern)
-- =====================================================
CREATE TABLE tenants (
    id              TEXT PRIMARY KEY,                  -- e.g. 'acme-corp'
    name            TEXT NOT NULL,
    monthly_budget_usd  NUMERIC(10,4) NOT NULL DEFAULT 100.00,
    current_spend_usd   NUMERIC(10,4) NOT NULL DEFAULT 0.00,
    budget_reset_day    INT NOT NULL DEFAULT 1,         -- day of month
    is_active      BOOLEAN NOT NULL DEFAULT TRUE,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- =====================================================
-- AGENTS: human support agents who use the review queue
-- =====================================================
CREATE TABLE agents (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    external_id     TEXT NOT NULL UNIQUE,               -- SSO/IdP subject
    email           TEXT NOT NULL UNIQUE,
    name            TEXT NOT NULL,
    tenant_id       TEXT NOT NULL REFERENCES tenants(id),
    role            TEXT NOT NULL DEFAULT 'agent',      -- 'agent', 'supervisor', 'admin'
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_agents_tenant ON agents(tenant_id);

-- =====================================================
-- TICKETS: the core entity — a customer support request
-- =====================================================
CREATE TABLE tickets (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       TEXT NOT NULL REFERENCES tenants(id),
    external_id     TEXT,                               -- ID from helpdesk system
    customer_email  TEXT,                               -- original (PII, stored encrypted)
    customer_name   TEXT,                               -- original (PII, stored encrypted)
    subject         TEXT NOT NULL,
    body            TEXT NOT NULL,                      -- original body (PII, stored encrypted)
    body_redacted   TEXT,                               -- PII-redacted version for LLM processing
    status          TEXT NOT NULL DEFAULT 'new',        -- new, processing, auto_resolved,
                                                       -- pending_review, human_resolved,
                                                       -- human_rejected, failed
    priority        INT NOT NULL DEFAULT 0,             -- for review queue sorting
    received_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tickets_tenant_status ON tickets(tenant_id, status);
CREATE INDEX idx_tickets_status_priority ON tickets(status, priority DESC, received_at);
CREATE INDEX idx_tickets_received ON tickets(received_at DESC);

-- =====================================================
-- CLASSIFICATIONS: LLM classification result per ticket
-- =====================================================
CREATE TABLE classifications (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    ticket_id       UUID NOT NULL REFERENCES tickets(id) ON DELETE CASCADE,
    tenant_id       TEXT NOT NULL,                      -- denormalized for filtering
    urgency         TEXT NOT NULL,                      -- 'critical', 'high', 'medium', 'low'
    category        TEXT NOT NULL,                      -- 'billing', 'technical', 'general', 'complaint'
    sentiment       TEXT NOT NULL,                      -- 'positive', 'neutral', 'negative', 'angry'
    confidence      NUMERIC(3,2) NOT NULL,              -- 0.00–1.00, LLM self-reported
    model_used      TEXT NOT NULL,                      -- e.g. 'gpt-4o-mini'
    tokens_used     INT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT valid_urgency CHECK (urgency IN ('critical','high','medium','low')),
    CONSTRAINT valid_category CHECK (category IN ('billing','technical','general','complaint')),
    CONSTRAINT valid_sentiment CHECK (sentiment IN ('positive','neutral','negative','angry')),
    CONSTRAINT valid_confidence CHECK (confidence >= 0.0 AND confidence <= 1.0)
);

CREATE INDEX idx_classifications_ticket ON classifications(ticket_id);
CREATE INDEX idx_classifications_tenant_urgency ON classifications(tenant_id, urgency);

-- =====================================================
-- KNOWLEDGE BASE DOCUMENTS: support docs, FAQs, runbooks
-- =====================================================
CREATE TABLE kb_documents (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       TEXT NOT NULL REFERENCES tenants(id),
    title           TEXT NOT NULL,
    source_type     TEXT NOT NULL,                      -- 'faq', 'doc', 'runbook', 'resolved_ticket'
    file_path       TEXT,
    file_hash       TEXT,                               -- SHA-256 for dedup
    chunk_count     INT NOT NULL DEFAULT 0,
    allowed_roles   TEXT[] NOT NULL DEFAULT '{}',       -- from P2: role-based access
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    UNIQUE(tenant_id, file_hash)
);

CREATE INDEX idx_kb_docs_tenant ON kb_documents(tenant_id);
CREATE INDEX idx_kb_docs_type ON kb_documents(source_type);

-- =====================================================
-- KB CHUNKS: vector-indexed text segments for RAG
-- =====================================================
CREATE TABLE kb_chunks (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    doc_id          UUID NOT NULL REFERENCES kb_documents(id) ON DELETE CASCADE,
    tenant_id       TEXT NOT NULL,                      -- denormalized for fast filtering
    chunk_idx       INT NOT NULL,
    content         TEXT NOT NULL,
    embedding       vector(1536) NOT NULL,
    allowed_roles   TEXT[] NOT NULL DEFAULT '{}',
    -- For resolved_ticket source: link back to original ticket
    source_ticket_id UUID REFERENCES tickets(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    UNIQUE(doc_id, chunk_idx)
);

-- HNSW index for fast approximate nearest neighbor search
CREATE INDEX idx_kb_chunks_embedding ON kb_chunks
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);

-- B-tree for tenant pre-filter
CREATE INDEX idx_kb_chunks_tenant ON kb_chunks(tenant_id);

-- GIN for role intersection (from P2)
CREATE INDEX idx_kb_chunks_roles ON kb_chunks USING GIN(allowed_roles);

-- Trigram index for BM25-like keyword search (from P1)
CREATE INDEX idx_kb_chunks_content_trgm ON kb_chunks USING GIN(content gin_trgm_ops);

-- =====================================================
-- DRAFT RESPONSES: LLM-generated response per ticket
-- =====================================================
CREATE TABLE draft_responses (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    ticket_id       UUID NOT NULL REFERENCES tickets(id) ON DELETE CASCADE,
    tenant_id       TEXT NOT NULL,
    draft_text      TEXT NOT NULL,                      -- with citation markers [1], [2]
    draft_text_clean TEXT,                              -- citations stripped (customer-facing)
    final_text      TEXT,                               -- after human edit (if any)
    retrieved_chunk_ids UUID[] NOT NULL,                -- which KB chunks were used
    citations      JSONB NOT NULL DEFAULT '[]',         -- structured citation array
    confidence_score NUMERIC(3,2) NOT NULL,             -- composite confidence
    route_decision  TEXT NOT NULL,                      -- 'auto' or 'human'
    model_used     TEXT NOT NULL,
    tokens_used    INT NOT NULL,
    cost_usd       NUMERIC(10,6) NOT NULL,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT valid_route CHECK (route_decision IN ('auto', 'human'))
);

CREATE INDEX idx_drafts_ticket ON draft_responses(ticket_id);
CREATE INDEX idx_drafts_tenant_route ON draft_responses(tenant_id, route_decision);

-- =====================================================
-- REVIEW QUEUE: tickets awaiting human review
-- =====================================================
CREATE TABLE review_queue (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    ticket_id       UUID NOT NULL REFERENCES tickets(id) ON DELETE CASCADE,
    tenant_id       TEXT NOT NULL,
    draft_id        UUID REFERENCES draft_responses(id),
    status          TEXT NOT NULL DEFAULT 'pending',    -- pending, in_review, approved,
                                                       -- rejected, edited
    priority        INT NOT NULL DEFAULT 0,             -- derived from urgency
    assigned_to     UUID REFERENCES agents(id),         -- agent who claimed it
    agent_feedback  TEXT,                               -- free-text feedback
    feedback_tags   TEXT[] NOT NULL DEFAULT '{}',       -- structured: 'wrong_category',
                                                       -- 'bad_tone', 'incorrect_info', etc.
    edit_diff       JSONB,                              -- diff between draft and final
    queued_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    resolved_at     TIMESTAMPTZ,

    CONSTRAINT valid_review_status CHECK (status IN ('pending','in_review','approved','rejected','edited'))
);

CREATE INDEX idx_review_tenant_status ON review_queue(tenant_id, status, priority DESC, queued_at);
CREATE INDEX idx_review_assigned ON review_queue(assigned_to) WHERE status = 'in_review';

-- =====================================================
-- COST LEDGER: every LLM call's cost, per ticket + per tenant
-- =====================================================
CREATE TABLE cost_ledger (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       TEXT NOT NULL REFERENCES tenants(id),
    ticket_id       UUID REFERENCES tickets(id),
    stage           TEXT NOT NULL,                      -- 'classify', 'embed_query',
                                                       -- 'draft', 'rerank', 'eval'
    model_used      TEXT NOT NULL,
    input_tokens    INT NOT NULL,
    output_tokens   INT NOT NULL,
    cost_usd        NUMERIC(10,6) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_cost_tenant_time ON cost_ledger(tenant_id, created_at DESC);
CREATE INDEX idx_cost_ticket ON cost_ledger(ticket_id);
CREATE INDEX idx_cost_stage ON cost_ledger(stage);

-- =====================================================
-- AUDIT LOGS: immutable record of every pipeline event
-- (from P2: append-only with triggers)
-- =====================================================
CREATE TABLE audit_logs (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    timestamp       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    tenant_id       TEXT NOT NULL,
    ticket_id       UUID REFERENCES tickets(id),
    agent_id        UUID REFERENCES agents(id),
    stage           TEXT NOT NULL,                      -- 'ticket_received', 'pii_scanned',
                                                       -- 'classified', 'rag_retrieved',
                                                       -- 'response_drafted', 'confidence_scored',
                                                       -- 'auto_responded', 'enqueued_for_review',
                                                       -- 'reviewed', 'cost_recorded',
                                                       -- 'evaluated', 'budget_exceeded', etc.
    event_data      JSONB NOT NULL DEFAULT '{}',        -- stage-specific metadata
    success         BOOLEAN NOT NULL DEFAULT TRUE,
    error_message   TEXT
);

CREATE INDEX idx_audit_tenant_time ON audit_logs(tenant_id, timestamp DESC);
CREATE INDEX idx_audit_ticket ON audit_logs(ticket_id, timestamp);
CREATE INDEX idx_audit_stage ON audit_logs(stage);

-- Trigger: prevent UPDATE and DELETE on audit_logs (from P2)
CREATE OR REPLACE FUNCTION prevent_audit_modification()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'audit_logs is append-only; UPDATE and DELETE are forbidden';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER no_audit_update
    BEFORE UPDATE ON audit_logs
    FOR EACH ROW EXECUTE FUNCTION prevent_audit_modification();

CREATE TRIGGER no_audit_delete
    BEFORE DELETE ON audit_logs
    FOR EACH ROW EXECUTE FUNCTION prevent_audit_modification();

-- =====================================================
-- EVAL RESULTS: scores from the evaluation harness (from P7)
-- =====================================================
CREATE TABLE eval_results (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       TEXT NOT NULL,
    ticket_id       UUID REFERENCES tickets(id),
    draft_id        UUID REFERENCES draft_responses(id),
    eval_run_id     TEXT NOT NULL,                      -- batch ID for grouped runs
    faithfulness    NUMERIC(3,2),                       -- 0.00–1.00
    relevance       NUMERIC(3,2),
    citation_accuracy NUMERIC(3,2),
    overall_score   NUMERIC(3,2),
    eval_model_used TEXT NOT NULL,                      -- LLM-as-judge model
    eval_tokens     INT NOT NULL,
    eval_cost_usd   NUMERIC(10,6) NOT NULL,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_eval_tenant_time ON eval_results(tenant_id, created_at DESC);
CREATE INDEX idx_eval_ticket ON eval_results(ticket_id);

-- =====================================================
-- FEEDBACK: agent feedback on AI drafts (for feedback loop)
-- =====================================================
CREATE TABLE feedback (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       TEXT NOT NULL,
    ticket_id       UUID NOT NULL REFERENCES tickets(id),
    draft_id        UUID NOT NULL REFERENCES draft_responses(id),
    agent_id        UUID NOT NULL REFERENCES agents(id),
    action          TEXT NOT NULL,                      -- 'approved', 'rejected', 'edited'
    feedback_tags   TEXT[] NOT NULL DEFAULT '{}',       -- 'wrong_category', 'bad_tone', etc.
    feedback_text   TEXT,
    original_draft  TEXT NOT NULL,                      -- for eval dataset
    final_response  TEXT,                               -- agent's version (if edited)
    added_to_eval_dataset BOOLEAN NOT NULL DEFAULT FALSE,
    eval_dataset_version TEXT,                          -- which dataset version it was added to
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT valid_feedback_action CHECK (action IN ('approved','rejected','edited'))
);

CREATE INDEX idx_feedback_tenant ON feedback(tenant_id, created_at DESC);
CREATE INDEX idx_feedback_added_eval ON feedback(added_to_eval_dataset) WHERE added_to_eval_dataset = FALSE;

-- =====================================================
-- PII ENTITIES: reversible mapping for redaction (from P10)
-- =====================================================
CREATE TABLE pii_entities (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    ticket_id       UUID NOT NULL REFERENCES tickets(id) ON DELETE CASCADE,
    tenant_id       TEXT NOT NULL,
    entity_type     TEXT NOT NULL,                      -- 'PERSON', 'EMAIL', 'PHONE', 'IBAN', etc.
    token           TEXT NOT NULL,                      -- e.g. '[NAME_1]'
    original_value  TEXT NOT NULL,                      -- encrypted at application level
    start_pos       INT,
    end_pos         INT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    UNIQUE(ticket_id, token)
);

CREATE INDEX idx_pii_ticket ON pii_entities(ticket_id);
```

### 6.3 Row-Level Security (Defense in Depth — from P2)

```sql
ALTER TABLE tickets ENABLE ROW LEVEL SECURITY;
ALTER TABLE kb_chunks ENABLE ROW LEVEL SECURITY;
ALTER TABLE kb_documents ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_logs ENABLE ROW LEVEL SECURITY;
ALTER TABLE review_queue ENABLE ROW LEVEL SECURITY;
ALTER TABLE cost_ledger ENABLE ROW LEVEL SECURITY;
ALTER TABLE draft_responses ENABLE ROW LEVEL SECURITY;
ALTER TABLE eval_results ENABLE ROW LEVEL SECURITY;
ALTER TABLE feedback ENABLE ROW LEVEL SECURITY;
ALTER TABLE pii_entities ENABLE ROW LEVEL SECURITY;

-- Policy: all tables scoped by tenant_id
CREATE POLICY tenant_isolation_tickets ON tickets
    USING (tenant_id = current_setting('app.current_tenant_id'));
CREATE POLICY tenant_isolation_kb_chunks ON kb_chunks
    USING (tenant_id = current_setting('app.current_tenant_id'));
CREATE POLICY tenant_isolation_kb_docs ON kb_documents
    USING (tenant_id = current_setting('app.current_tenant_id'));
CREATE POLICY tenant_isolation_audit ON audit_logs
    USING (tenant_id = current_setting('app.current_tenant_id'));
CREATE POLICY tenant_isolation_review ON review_queue
    USING (tenant_id = current_setting('app.current_tenant_id'));
CREATE POLICY tenant_isolation_cost ON cost_ledger
    USING (tenant_id = current_setting('app.current_tenant_id'));
CREATE POLICY tenant_isolation_drafts ON draft_responses
    USING (tenant_id = current_setting('app.current_tenant_id'));
CREATE POLICY tenant_isolation_eval ON eval_results
    USING (tenant_id = current_setting('app.current_tenant_id'));
CREATE POLICY tenant_isolation_feedback ON feedback
    USING (tenant_id = current_setting('app.current_tenant_id'));
CREATE POLICY tenant_isolation_pii ON pii_entities
    USING (tenant_id = current_setting('app.current_tenant_id'));
```

Application sets tenant context per request:
```python
await db.execute(f"SET LOCAL app.current_tenant_id = '{tenant_id}'")
```

---

## 7. Project Structure

```
support-ai/
├── docker-compose.yml
├── .env.example
├── pyproject.toml
├── README.md
├── IMPLEMENTATION_PLAN.md              ← this file
├── agent.md                      # Herdr multi-agent orchestration guide
│
├── src/
│   ├── __init__.py
│   ├── main.py                         # FastAPI app factory + lifespan + background task runner
│   ├── config.py                       # Pydantic Settings (env vars)
│   │
│   ├── db/
│   │   ├── __init__.py
│   │   ├── connection.py               # asyncpg pool + pgvector setup
│   │   ├── schema.sql                  # full DDL (tables, indexes, RLS, triggers)
│   │   └── migrations/
│   │       ├── 001_initial.sql
│   │       ├── 002_rls.sql
│   │       └── 003_indexes.sql
│   │
│   ├── ingest/
│   │   ├── __init__.py
│   │   ├── ticket_receiver.py          # POST /tickets handler + validation
│   │   ├── email_parser.py             # IMAP poller stub (optional)
│   │   └── kb_loader.py                # KB document ingestion (parse → chunk → embed → store)
│   │
│   ├── pii/
│   │   ├── __init__.py
│   │   ├── redactor.py                 # Presidio-based PII detection + tokenization
│   │   ├── restorer.py                 # Reverse tokens → original values in final response
│   │   ├── recognizers.py              # Custom Presidio recognizers (IBAN, German phone, etc.)
│   │   └── encryption.py               # Encrypt original PII values at rest
│   │
│   ├── classify/
│   │   ├── __init__.py
│   │   ├── classifier.py               # LLM classification (urgency + category + sentiment)
│   │   ├── models.py                   # Pydantic models for structured output
│   │   └── cache.py                    # Redis classification cache
│   │
│   ├── rag/
│   │   ├── __init__.py
│   │   ├── query_formulator.py         # Build RAG query from ticket + classification
│   │   ├── bm25_search.py              # Trigram/keyword search
│   │   ├── vector_search.py            # pgvector cosine similarity + tenant filter
│   │   ├── reranker.py                 # Cross-encoder re-ranking
│   │   ├── permission_filter.py        # Role-based post-filter (from P2)
│   │   └── service.py                  # Orchestrates: formulate → BM25 + vector → merge → rerank → filter
│   │
│   ├── draft/
│   │   ├── __init__.py
│   │   ├── prompt_builder.py           # Context + ticket + sentiment → response prompt
│   │   ├── llm_client.py               # LLM interface (OpenAI + Ollama fallback)
│   │   ├── citation_extractor.py       # Parse [1], [2] → structured citations
│   │   └── service.py                  # Orchestrates: build prompt → LLM → citations
│   │
│   ├── confidence/
│   │   ├── __init__.py
│   │   ├── scorer.py                   # Composite confidence: retrieval + classification + self-assess
│   │   └── router.py                   # Routing decision: auto-respond vs. human review
│   │
│   ├── review/
│   │   ├── __init__.py
│   │   ├── queue_manager.py            # Enqueue, claim, release, resolve
│   │   └── feedback_collector.py       # Capture agent feedback → feedback table + eval dataset
│   │
│   ├── eval/
│   │   ├── __init__.py
│   │   ├── harness.py                  # Run eval suite on auto-responses
│   │   ├── metrics.py                  # Faithfulness, relevance, citation accuracy
│   │   ├── judge.py                    # LLM-as-judge client
│   │   └── dataset.py                  # Eval dataset management (versioned, grows from feedback)
│   │
│   ├── cost/
│   │   ├── __init__.py
│   │   ├── tracker.py                  # Per-call token counting + cost calculation
│   │   └── budget.py                   # Per-tenant budget enforcement + reset
│   │
│   ├── audit/
│   │   ├── __init__.py
│   │   └── logger.py                   # Append-only audit log writer (from P2)
│   │
│   ├── pipeline/
│   │   ├── __init__.py
│   │   ├── orchestrator.py             # Full ticket processing pipeline (stages 1–9)
│   │   └── failure_handlers.py         # Fallback logic for each stage
│   │
│   ├── cache/
│   │   ├── __init__.py
│   │   └── redis_client.py             # Redis connection + helpers
│   │
│   └── api/
│       ├── __init__.py
│       ├── routes/
│       │   ├── __init__.py
│       │   ├── tickets.py              # POST /tickets, GET /tickets/{id}
│       │   ├── kb.py                   # POST /kb/upload, GET /kb/documents
│       │   ├── review.py               # GET /review/queue, POST /review/{id}/{action}
│       │   ├── metrics.py              # GET /metrics (for monitoring dashboard)
│       │   ├── eval.py                 # POST /eval/run, GET /eval/results
│       │   └── health.py               # GET /health, GET /ready
│       └── dependencies.py             # FastAPI DI: get current agent, tenant context
│
├── dashboards/
│   ├── review_queue/
│   │   ├── app.py                      # Streamlit: human review queue
│   │   ├── components.py               # Reusable UI components
│   │   └── api_client.py               # HTTP client for API calls
│   └── monitoring/
│       ├── app.py                      # Streamlit: monitoring dashboard
│       ├── charts.py                   # Chart definitions
│       └── api_client.py               # HTTP client for metrics endpoint
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py                     # Fixtures: test DB, test client, seed data, mock LLM
│   ├── test_pii.py                     # PII detection, redaction, restoration
│   ├── test_classify.py                # Classification structured output, validation
│   ├── test_rag.py                     # Hybrid search, re-ranking, tenant filtering
│   ├── test_draft.py                   # Response drafting, citation extraction
│   ├── test_confidence.py              # Confidence scoring, routing thresholds
│   ├── test_review.py                  # Queue operations, feedback capture
│   ├── test_eval.py                    # Eval metrics, LLM-as-judge
│   ├── test_cost.py                    # Cost tracking, budget enforcement
│   ├── test_audit.py                   # Audit immutability, completeness
│   ├── test_pipeline.py                # End-to-end pipeline (all stages)
│   ├── test_security.py                # Cross-tenant leakage, PII leakage to LLM
│   └── test_api.py                     # API endpoint tests
│
├── scripts/
│   ├── init_db.py                      # Run schema.sql against fresh database
│   ├── seed_test_data.py               # Insert tenants, agents, KB docs, sample tickets
│   ├── seed_kb.py                      # Load sample support docs + FAQs into KB
│   ├── run_eval.py                     # Trigger eval suite manually
│   └── generate_agent_token.py         # Create JWT for review queue auth
│
├── docker/
│   ├── Dockerfile.api                  # API service image
│   ├── Dockerfile.review               # Streamlit review queue image
│   ├── Dockerfile.monitor              # Streamlit monitoring dashboard image
│   └── postgres/
│       └── init.sql                    # pgvector + pg_trgm extensions on first boot
│
└── .github/
    └── workflows/
        └── ci.yml                      # GitHub Actions: lint + test on push
```

---

## 8. Implementation Phases

> **Timeline:** 15 working days (~3 weeks). Each phase has a verification gate that must pass before proceeding.

### Phase 1: Foundation & Infrastructure (Days 1–2)

**Goal:** Running Postgres + pgvector + Redis + FastAPI skeleton with auth and DB schema.

| Step | Task | Deliverable |
|------|------|-------------|
| 1.1 | Write `docker-compose.yml` with Postgres+pgvector, Redis, API, two Streamlit services | `docker compose up` starts all services |
| 1.2 | Write `docker/postgres/init.sql` (extensions) + `src/db/schema.sql` (all tables, indexes, RLS, triggers) | DB initializes on first boot with full schema |
| 1.3 | Write `src/config.py` (Pydantic Settings for all env vars) | Env-driven config |
| 1.4 | Write `src/db/connection.py` (asyncpg pool + RLS context setter) | DB pool works, tenant context settable |
| 1.5 | Write `src/cache/redis_client.py` | Redis connection works |
| 1.6 | Write `src/api/dependencies.py` (agent auth, tenant extraction) | JWT decode, agent lookup, tenant context |
| 1.7 | Write `src/main.py` (FastAPI app factory, lifespan, health routes) | App starts, `/health` responds |
| 1.8 | Write `scripts/init_db.py` + `scripts/seed_test_data.py` | Can initialize and seed DB |
| 1.9 | Write `tests/conftest.py` (test DB, client, mock LLM fixtures) | Test infrastructure ready |

**Verification:** `docker compose up` → `curl localhost:8000/health` → 200 OK. `pytest tests/test_audit.py -k immutability` passes (audit triggers work). DB has all tables with RLS enabled.

---

### Phase 2: Ticket Ingestion & PII Redaction (Days 3–4)

**Goal:** Tickets arrive via API, PII is detected and redacted before any LLM call.

| Step | Task | Deliverable |
|------|------|-------------|
| 2.1 | Write `src/ingest/ticket_receiver.py` (POST /tickets validation + storage) | Tickets stored in DB with encrypted PII fields |
| 2.2 | Write `src/pii/recognizers.py` (custom Presidio recognizers: IBAN, German phone, address) | Custom entity types detected |
| 2.3 | Write `src/pii/redactor.py` (Presidio analyze → tokenize → replace) | PII replaced with `[NAME_1]`, `[EMAIL_1]` tokens in `body_redacted` |
| 2.4 | Write `src/pii/restorer.py` (reverse tokens in final response) | Tokens restored to original values |
| 2.5 | Write `src/pii/encryption.py` (encrypt original PII values at rest) | Original PII encrypted in `pii_entities` table |
| 2.6 | Write `src/audit/logger.py` (append-only audit writer) | Audit events logged for every stage |
| 2.7 | Write `src/api/routes/tickets.py` (POST /tickets, GET /tickets/{id}) | API endpoints work |
| 2.8 | Write `tests/test_pii.py` (detection, redaction, restoration, no-leak) | PII tests pass |
| 2.9 | Write `tests/test_audit.py` (immutability, completeness) | Audit tests pass |

**Verification:** POST a ticket with PII ("Hi, I'm Jane Doe, jane@example.com, my IBAN is DE89...") → check `body_redacted` has `[NAME_1]`, `[EMAIL_1]`, `[IBAN_1]` → check `pii_entities` table has encrypted originals → check audit log has `ticket_received` and `pii_scanned` events.

---

### Phase 3: Knowledge Base RAG (Days 5–6)

**Goal:** KB documents ingested, hybrid search + re-ranking working with tenant filtering.

| Step | Task | Deliverable |
|------|------|-------------|
| 3.1 | Write `src/ingest/kb_loader.py` (parse → chunk → embed → store with metadata) | KB docs ingested into `kb_chunks` with embeddings |
| 3.2 | Write `src/api/routes/kb.py` (POST /kb/upload, GET /kb/documents) | KB upload endpoint works |
| 3.3 | Write `src/rag/query_formulator.py` (ticket + classification → RAG query) | Query formulated from ticket context |
| 3.4 | Write `src/rag/bm25_search.py` (trigram keyword search) | BM25 candidates retrieved |
| 3.5 | Write `src/rag/vector_search.py` (pgvector + tenant pre-filter) | Vector candidates retrieved with tenant isolation |
| 3.6 | Write `src/rag/reranker.py` (cross-encoder re-ranking) | Candidates re-ranked by relevance |
| 3.7 | Write `src/rag/permission_filter.py` (role post-filter from P2) | Role-based filtering applied |
| 3.8 | Write `src/rag/service.py` (orchestrator: formulate → search → merge → rerank → filter) | Full RAG pipeline works |
| 3.9 | Write `tests/test_rag.py` (hybrid search, re-ranking, tenant filter, role filter) | RAG tests pass |
| 3.10 | Write `tests/test_security.py` (cross-tenant KB leakage) | Tenant isolation holds |

**Verification:** Upload KB docs for tenant A and B → query as tenant A → only tenant A's chunks returned. Verify re-ranking improves top-k precision vs. vector-only search (compare retrieval distances before/after).

---

### Phase 4: Classification & Response Drafting (Days 7–8)

**Goal:** LLM classifies tickets and drafts grounded responses with citations.

| Step | Task | Deliverable |
|------|------|-------------|
| 4.1 | Write `src/classify/models.py` (Pydantic models for structured output) | Classification schema defined |
| 4.2 | Write `src/classify/classifier.py` (LLM call with structured output) | Classification works (urgency + category + sentiment) |
| 4.3 | Write `src/classify/cache.py` (Redis cache: same text → same classification) | Cache hits avoid redundant LLM calls |
| 4.4 | Write `src/draft/prompt_builder.py` (context + ticket + sentiment → prompt) | Prompt adapts tone to sentiment |
| 4.5 | Write `src/draft/llm_client.py` (OpenAI + Ollama fallback) | LLM calls work with fallback |
| 4.6 | Write `src/draft/citation_extractor.py` (parse [1], [2] → structured) | Citations extracted and validated |
| 4.7 | Write `src/draft/service.py` (orchestrator: prompt → LLM → citations) | Full draft pipeline works |
| 4.8 | Write `tests/test_classify.py` (structured output, validation, cache) | Classification tests pass |
| 4.9 | Write `tests/test_draft.py` (response quality, citation accuracy) | Draft tests pass |

**Verification:** Submit a billing ticket → classification returns `{urgency: "high", category: "billing", sentiment: "negative"}` → RAG retrieves billing FAQ chunks → draft response includes citation [1] pointing to the correct KB chunk. Verify redacted PII tokens appear in the draft (not real PII).

---

### Phase 5: Confidence Scoring, Routing & Cost Tracking (Day 9)

**Goal:** Composite confidence score drives routing; cost tracked per ticket and per tenant.

| Step | Task | Deliverable |
|------|------|-------------|
| 5.1 | Write `src/confidence/scorer.py` (composite: retrieval + classification + self-assess + coverage) | Confidence score computed |
| 5.2 | Write `src/confidence/router.py` (auto-respond vs. human review decision) | Routing decision made with configurable thresholds |
| 5.3 | Write `src/cost/tracker.py` (per-call token counting + cost calculation) | Every LLM call cost logged in `cost_ledger` |
| 5.4 | Write `src/cost/budget.py` (per-tenant budget check + monthly reset) | Budget exceeded → route to human, no LLM call |
| 5.5 | Wire cost tracking into all LLM-calling components (classify, draft, eval) | All LLM calls tracked |
| 5.6 | Write `tests/test_confidence.py` (threshold boundaries, routing logic) | Confidence tests pass |
| 5.7 | Write `tests/test_cost.py` (cost calculation, budget enforcement, reset) | Cost tests pass |

**Verification:** Submit 5 tickets with varying complexity → verify high-confidence tickets route to auto, low-confidence route to human. Set tenant budget to $0.01 → submit ticket → verify it routes to human without LLM draft call. Check `cost_ledger` has rows for each LLM call.

---

### Phase 6: Human Review Queue (Days 10–11)

**Goal:** Streamlit dashboard where agents review, edit, approve, or reject drafts with feedback.

| Step | Task | Deliverable |
|------|------|-------------|
| 6.1 | Write `src/review/queue_manager.py` (enqueue, claim, release, resolve) | Queue operations work |
| 6.2 | Write `src/api/routes/review.py` (GET /review/queue, POST /review/{id}/{action}) | Review API endpoints work |
| 6.3 | Write `src/review/feedback_collector.py` (capture feedback → feedback table) | Feedback stored with tags + text + diff |
| 6.4 | Write `dashboards/review_queue/app.py` (Streamlit: queue view, draft editor, feedback form) | Agents can view, edit, approve, reject |
| 6.5 | Write `dashboards/review_queue/components.py` (reusable: ticket card, draft editor, feedback widget) | UI components work |
| 6.6 | Write `dashboards/review_queue/api_client.py` (HTTP client for API) | Dashboard communicates with API |
| 6.7 | Write `tests/test_review.py` (queue operations, feedback capture, status transitions) | Review tests pass |

**Verification:** Submit a low-confidence ticket → it appears in the review queue dashboard → agent edits the draft → approves → ticket status changes to `human_resolved` → feedback row created in `feedback` table → audit log has `reviewed` event. Verify PII tokens are restored in the final response sent to the customer.

---

### Phase 7: Evaluation Harness & Feedback Loop (Days 12–13)

**Goal:** Eval suite runs on auto-responses; agent feedback feeds back into eval dataset.

| Step | Task | Deliverable |
|------|------|-------------|
| 7.1 | Write `src/eval/metrics.py` (faithfulness, relevance, citation accuracy) | Eval metrics computed |
| 7.2 | Write `src/eval/judge.py` (LLM-as-judge client) | Judge LLM scores responses |
| 7.3 | Write `src/eval/harness.py` (run eval suite on auto-responses, store results) | Eval results stored in `eval_results` |
| 7.4 | Write `src/eval/dataset.py` (versioned eval dataset, grows from feedback) | Feedback rows become eval examples |
| 7.5 | Wire feedback collector to auto-add rejected/edited drafts to eval dataset | Feedback loop closed |
| 7.6 | Write `src/api/routes/eval.py` (POST /eval/run, GET /eval/results) | Eval API endpoints work |
| 7.7 | Write `tests/test_eval.py` (metric computation, dataset growth, regression) | Eval tests pass |

**Verification:** Submit 3 auto-resolved tickets → run eval suite → check `eval_results` has faithfulness/relevance/citation_accuracy scores. Reject a draft in the review queue → check `feedback` row has `added_to_eval_dataset=TRUE` → verify eval dataset grows by one example. Run eval again → verify the new example is included.

---

### Phase 8: Monitoring Dashboard & Pipeline Integration (Day 14)

**Goal:** Monitoring dashboard shows all operational metrics; full pipeline runs end-to-end.

| Step | Task | Deliverable |
|------|------|-------------|
| 8.1 | Write `src/api/routes/metrics.py` (GET /metrics: aggregated stats for dashboard) | Metrics endpoint returns all dashboard data |
| 8.2 | Write `dashboards/monitoring/app.py` (Streamlit: all metrics + charts) | Dashboard renders with live data |
| 8.3 | Write `dashboards/monitoring/charts.py` (volume, auto-resolution rate, cost, eval trends, queue depth) | All charts render |
| 8.4 | Write `src/pipeline/orchestrator.py` (full pipeline: stages 1–9 with failure handling) | Full ticket processing pipeline works |
| 8.5 | Write `src/pipeline/failure_handlers.py` (fallback logic for each stage) | Failures handled gracefully |
| 8.6 | Write `tests/test_pipeline.py` (end-to-end: ingest → classify → RAG → draft → route → resolve) | Pipeline tests pass |

**Verification:** Submit 10 tickets with varying complexity → some auto-resolve, some go to review queue → approve/reject in review queue → run eval → open monitoring dashboard → verify all metrics display correctly (ticket volume=10, auto-resolution rate, avg response time, cost/ticket, eval scores, queue depth).

---

### Phase 9: Hardening, Testing & Documentation (Day 15)

**Goal:** Production-ready, fully tested, documented.

| Step | Task | Deliverable |
|------|------|-------------|
| 9.1 | Write `.github/workflows/ci.yml` (ruff + pytest on push) | CI pipeline works |
| 9.2 | Run full test suite end-to-end | All tests green |
| 9.3 | Security review: re-run `test_security.py` + manual PII leakage check | No leaks found |
| 9.4 | Add structured logging (structlog) across all components | JSON logs with correlation IDs |
| 9.5 | Add rate limiting on API endpoints | Abuse prevention |
| 9.6 | Write `README.md` (setup, architecture, usage, metrics) | Someone else can run it |
| 9.7 | Add `.env.example` with all env vars | Configuration documented |
| 9.8 | Performance check: process 50 tickets, measure throughput + cost | Baseline metrics recorded |

**Verification:** Fresh `docker compose up` → run `scripts/seed_test_data.py` + `scripts/seed_kb.py` → submit tickets via API → some auto-resolve, some go to review queue → approve/reject in review queue → run eval → monitoring dashboard shows all metrics → all tests green → CI passes.

---

## 9. Component Specifications

### 9.1 PII Redactor (`src/pii/redactor.py`)

```python
"""
PII redaction using Microsoft Presidio. Runs BEFORE any LLM call.

Flow:
  1. Analyze text with Presidio (NER + regex + custom recognizers)
  2. For each detected entity, generate a reversible token: [TYPE_N]
  3. Replace entity text with token in the redacted text
  4. Store token → encrypted_original mapping in pii_entities table
  5. Return redacted text + entity map

The original PII never leaves the application. The LLM only sees tokens.
The restorer reverses tokens in the final response before sending to customer.
"""

from presidio_analyzer import AnalyzerEngine
from presidio_analyzer.predefined_recognizers import EmailRecognizer, PhoneRecognizer
from src.pii.recognizers import IBANRecognizer, GermanPhoneRecognizer
from src.pii.encryption import encrypt_value

class PIIRedactor:
    def __init__(self):
        self.analyzer = AnalyzerEngine()
        # Register custom recognizers (from P10)
        self.analyzer.registry.add_recognizer(IBANRecognizer())
        self.analyzer.registry.add_recognizer(GermanPhoneRecognizer())

    async def redact(self, text: str, ticket_id: UUID, tenant_id: str) -> RedactionResult:
        # 1. Analyze
        results = self.analyzer.analyze(
            text=text,
            entities=["PERSON", "EMAIL_ADDRESS", "PHONE_NUMBER",
                      "IBAN", "LOCATION", "US_SSN"],
            language="en",
        )

        # 2. Sort by start position (descending) so replacements don't shift offsets
        results = sorted(results, key=lambda r: r.start, reverse=True)

        # 3. Tokenize + replace
        redacted_text = text
        entity_map = []
        counters: dict[str, int] = {}

        for result in results:
            entity_type = result.entity_type
            counters[entity_type] = counters.get(entity_type, 0) + 1
            token = f"[{entity_type}_{counters[entity_type]}]"
            original_value = text[result.start:result.end]

            # Replace in redacted text
            redacted_text = (
                redacted_text[:result.start]
                + token
                + redacted_text[result.end:]
            )

            # Store encrypted original
            entity_map.append({
                "token": token,
                "entity_type": entity_type,
                "original_value": encrypt_value(original_value),
                "start_pos": result.start,
                "end_pos": result.end,
            })

        # 4. Persist to pii_entities table
        await self._store_entities(ticket_id, tenant_id, entity_map)

        # 5. Audit log
        await audit_logger.log(
            tenant_id=tenant_id,
            ticket_id=ticket_id,
            stage="pii_scanned",
            event_data={
                "entities_found": len(results),
                "entity_types": list(counters.keys()),
            },
        )

        return RedactionResult(
            redacted_text=redacted_text,
            entity_count=len(results),
            entity_types=list(counters.keys()),
        )
```

### 9.2 PII Restorer (`src/pii/restorer.py`)

```python
"""
Reverses PII tokenization in the final response before sending to customer.

The LLM draft contains tokens like [PERSON_1], [EMAIL_1].
This module replaces them with the original values from pii_entities table.

This runs AFTER the LLM generates the draft and BEFORE the response is sent.
For human-reviewed drafts, this runs after the agent approves/edits.
"""

async def restore_pii(text: str, ticket_id: UUID, tenant_id: str) -> str:
    # Load all PII entities for this ticket
    entities = await db.fetch(
        "SELECT token, original_value FROM pii_entities WHERE ticket_id = $1",
        ticket_id,
    )

    restored = text
    for entity in entities:
        original = decrypt_value(entity["original_value"])
        restored = restored.replace(entity["token"], original)

    return restored
```

### 9.3 Ticket Classifier (`src/classify/classifier.py`)

```python
"""
Classifies a ticket into urgency, category, and sentiment in a single LLM call.

Uses structured output (JSON mode) with Pydantic validation.
If the LLM output doesn't match the schema, retry with error feedback.

Classification cache: if the same ticket body (hashed) has been classified
before, return the cached result without an LLM call (from P9 caching pattern).
"""

CLASSIFY_PROMPT = """\
You are a customer support ticket classifier. Analyze the following support \
ticket and classify it.

Ticket subject: {subject}
Ticket body: {body}

Respond with a JSON object containing exactly these fields:
{{
  "urgency": "critical" | "high" | "medium" | "low",
  "category": "billing" | "technical" | "general" | "complaint",
  "sentiment": "positive" | "neutral" | "negative" | "angry",
  "confidence": 0.00 to 1.00,
  "reasoning": "one sentence explaining your classification"
}}

Classification guidelines:
- urgency=critical: system down, data loss, security breach, payment failure for critical service
- urgency=high: feature broken, can't work, billing error causing disruption
- urgency=medium: how-to question, feature request, non-blocking issue
- urgency=low: general inquiry, feedback, cosmetic issue
- category=complaint: customer is explicitly unhappy with the company/service (not just frustrated)
- sentiment=angry: hostile language, threats, demands for escalation
"""

async def classify_ticket(
    subject: str,
    body: str,  # already PII-redacted
    ticket_id: UUID,
    tenant_id: str,
) -> ClassificationResult:
    # 1. Check cache (from P9 pattern)
    cache_key = f"classify:{hashlib.sha256(body.encode()).hexdigest()}"
    cached = await redis.get(cache_key)
    if cached:
        return ClassificationResult.parse_raw(cached)

    # 2. Call LLM with structured output
    prompt = CLASSIFY_PROMPT.format(subject=subject, body=body)
    response = await llm_client.generate_json(
        prompt=prompt,
        response_model=ClassificationResult,  # Pydantic schema for JSON mode
    )

    # 3. Validate (Pydantic will raise if schema mismatch)
    result = ClassificationResult(
        urgency=response.urgency,
        category=response.category,
        sentiment=response.sentiment,
        confidence=response.confidence,
        model_used=llm_client.model_name,
        tokens_used=response.total_tokens,
    )

    # 4. Cache for 24h
    await redis.setex(cache_key, 86400, result.json())

    # 5. Store in classifications table
    await db.execute(
        """INSERT INTO classifications
           (ticket_id, tenant_id, urgency, category, sentiment, confidence,
            model_used, tokens_used)
           VALUES ($1,$2,$3,$4,$5,$6,$7,$8)""",
        ticket_id, tenant_id, result.urgency, result.category,
        result.sentiment, result.confidence, result.model_used, result.tokens_used,
    )

    # 6. Cost tracking (from P9)
    await cost_tracker.record(
        tenant_id=tenant_id, ticket_id=ticket_id, stage="classify",
        model=result.model_used, input_tokens=response.prompt_tokens,
        output_tokens=response.completion_tokens,
    )

    # 7. Audit
    await audit_logger.log(
        tenant_id=tenant_id, ticket_id=ticket_id, stage="classified",
        event_data={"urgency": result.urgency, "category": result.category,
                     "sentiment": result.sentiment, "confidence": float(result.confidence)},
    )

    return result
```

### 9.4 RAG Service (`src/rag/service.py`)

```python
"""
Hybrid RAG retrieval: BM25 + vector search + cross-encoder re-ranking.

Combines:
  - P1: hybrid search (BM25 + vector) and cross-encoder re-ranking
  - P2: tenant pre-filter and role-based post-filter

The query is formulated from the ticket subject + body + classification,
not just the raw ticket text. Classification helps disambiguate
("billing" ticket about "charges" → retrieve billing FAQ, not technical docs).
"""

async def retrieve(
    ticket: Ticket,
    classification: ClassificationResult,
    user_roles: list[str],
    top_k: int = 5,
    oversample: int = 3,
) -> list[RetrievedChunk]:
    # 1. Formulate RAG query from ticket + classification
    query = query_formulator.build(ticket, classification)
    # e.g. "billing charges refund policy — customer is angry about unexpected charge"

    # 2. Parallel: BM25 + vector search
    bm25_candidates = await bm25_search.search(
        query=query, tenant_id=ticket.tenant_id, limit=top_k * oversample * 2
    )
    vector_candidates = await vector_search.search(
        query=query, tenant_id=ticket.tenant_id, limit=top_k * oversample * 2
    )

    # 3. Merge (RRF — Reciprocal Rank Fusion, from P1)
    merged = reciprocal_rank_fusion(bm25_candidates, vector_candidates)

    # 4. Cross-encoder re-ranking (from P1)
    reranked = await reranker.rerank(query, merged, top_k * oversample)

    # 5. Role-based post-filter (from P2)
    user_role_set = set(user_roles)
    filtered = [c for c in reranked if set(c.allowed_roles) & user_role_set]

    # 6. Take top_k
    final = filtered[:top_k]

    # 7. Audit
    await audit_logger.log(
        tenant_id=ticket.tenant_id, ticket_id=ticket.id, stage="rag_retrieved",
        event_data={
            "query": query,
            "bm25_count": len(bm25_candidates),
            "vector_count": len(vector_candidates),
            "merged_count": len(merged),
            "reranked_count": len(reranked),
            "final_count": len(final),
            "chunk_ids": [str(c.id) for c in final],
            "avg_distance": float(np.mean([c.distance for c in final])) if final else 1.0,
        },
    )

    return final
```

### 9.5 Confidence Scorer & Router (`src/confidence/scorer.py` + `router.py`)

```python
"""
Composite confidence score determines whether a ticket is auto-resolved
or routed to the human review queue.

Score components (weighted):
  - retrieval_distance (w1=0.35): avg cosine distance of top-k chunks.
    Lower distance → higher confidence. Normalized: 1 - avg_distance.
  - classification_confidence (w2=0.20): LLM self-reported confidence.
  - llm_self_assessment (w3=0.25): LLM rates its own draft quality (0–1).
  - context_coverage (w4=0.20): fraction of ticket questions addressed
    by retrieved context (heuristic: retrieved_chunks > 0 and top chunk
    distance < 0.3).

Hard rules that override score (always route to human):
  - urgency == 'critical'
  - sentiment == 'angry'
  - category == 'complaint'
  - tenant budget exceeded
  - PII scan failed
  - any pipeline stage failed
"""

WEIGHTS = {"retrieval": 0.35, "classification": 0.20, "self_assess": 0.25, "coverage": 0.20}
AUTO_THRESHOLD = 0.75

async def compute_confidence(
    retrieved_chunks: list[RetrievedChunk],
    classification: ClassificationResult,
    draft: DraftResult,
) -> float:
    # 1. Retrieval distance score
    if retrieved_chunks:
        avg_dist = np.mean([c.distance for c in retrieved_chunks])
        retrieval_score = max(0.0, 1.0 - avg_dist)
    else:
        retrieval_score = 0.0

    # 2. Classification confidence
    cls_score = float(classification.confidence)

    # 3. LLM self-assessment (asked in draft prompt: "rate your confidence 0-1")
    self_score = draft.self_confidence or 0.0

    # 4. Context coverage
    if retrieved_chunks and retrieved_chunks[0].distance < 0.3:
        coverage_score = 1.0
    elif retrieved_chunks:
        coverage_score = 0.5
    else:
        coverage_score = 0.0

    # 5. Weighted composite
    score = (
        WEIGHTS["retrieval"] * retrieval_score
        + WEIGHTS["classification"] * cls_score
        + WEIGHTS["self_assess"] * self_score
        + WEIGHTS["coverage"] * coverage_score
    )

    return round(score, 2)


def should_auto_respond(
    confidence: float,
    classification: ClassificationResult,
    tenant_budget_ok: bool,
    pii_scan_ok: bool,
    pipeline_ok: bool,
) -> RoutingDecision:
    # Hard rules: always human
    if not pipeline_ok:
        return RoutingDecision(route="human", reason="pipeline_failure")
    if not pii_scan_ok:
        return RoutingDecision(route="human", reason="pii_scan_failed")
    if not tenant_budget_ok:
        return RoutingDecision(route="human", reason="budget_exceeded")
    if classification.urgency == "critical":
        return RoutingDecision(route="human", reason="critical_urgency")
    if classification.sentiment == "angry":
        return RoutingDecision(route="human", reason="angry_sentiment")
    if classification.category == "complaint":
        return RoutingDecision(route="human", reason="complaint_category")

    # Confidence threshold
    if confidence >= AUTO_THRESHOLD:
        return RoutingDecision(route="auto", reason="confidence_above_threshold")
    else:
        return RoutingDecision(route="human", reason="confidence_below_threshold")
```

### 9.6 Pipeline Orchestrator (`src/pipeline/orchestrator.py`)

```python
"""
Orchestrates the full ticket processing pipeline: stages 1–9.

Each stage is wrapped in a try/except with a failure handler.
If a stage fails, the failure handler decides:
  - Retry (with backoff)
  - Fallback (degraded mode)
  - Route to human (skip remaining LLM stages)

The orchestrator is async and runs as a FastAPI background task.
Multiple tickets can be processed concurrently.
"""

async def process_ticket(ticket_id: UUID, tenant_id: str):
    audit = AuditContext(ticket_id=ticket_id, tenant_id=tenant_id)
    stage_status = {}

    try:
        # STAGE 1: Already ingested (ticket exists in DB)
        ticket = await get_ticket(ticket_id, tenant_id)
        await audit.log("ticket_received", {"external_id": ticket.external_id})
        stage_status["ingest"] = True

        # STAGE 2: PII Redaction
        try:
            redaction = await pii_redactor.redact(
                ticket.body, ticket_id, tenant_id
            )
            await update_ticket(ticket_id, body_redacted=redaction.redacted_text)
            stage_status["pii"] = True
        except Exception as e:
            stage_status["pii"] = False
            await failure_handlers.handle_pii_failure(ticket, audit, e)
            return  # route to human, no LLM calls

        # STAGE 3: Classification
        try:
            classification = await classifier.classify_ticket(
                ticket.subject, ticket.body_redacted, ticket_id, tenant_id
            )
            stage_status["classify"] = True
        except Exception as e:
            classification = None
            stage_status["classify"] = False
            await failure_handlers.handle_classification_failure(ticket, audit, e)

        # STAGE 4: RAG Retrieval
        try:
            retrieved = await rag_service.retrieve(
                ticket, classification, user_roles=["agent"],  # system roles
            )
            stage_status["rag"] = True
        except Exception as e:
            retrieved = []
            stage_status["rag"] = False
            await failure_handlers.handle_rag_failure(ticket, audit, e)

        # BUDGET CHECK (before draft — from P9)
        budget_ok = await budget_checker.check(tenant_id)
        if not budget_ok:
            await audit.log("budget_exceeded", {})
            await enqueue_for_review(ticket, classification, draft=None,
                                      reason="budget_exceeded")
            return

        # STAGE 5: Draft Response
        try:
            draft = await draft_service.generate(
                ticket, classification, retrieved
            )
            stage_status["draft"] = True
        except Exception as e:
            draft = None
            stage_status["draft"] = False
            await failure_handlers.handle_draft_failure(ticket, audit, e)
            await enqueue_for_review(ticket, classification, draft=None,
                                      reason="draft_failed")
            return

        # STAGE 6: Confidence + Routing
        confidence = await confidence_scorer.compute_confidence(
            retrieved, classification, draft
        )
        routing = should_auto_respond(
            confidence=confidence,
            classification=classification,
            tenant_budget_ok=budget_ok,
            pii_scan_ok=stage_status.get("pii", False),
            pipeline_ok=all(stage_status.values()),
        )
        await audit.log("confidence_scored", {
            "score": confidence, "route": routing.route, "reason": routing.reason,
        })

        # STAGE 7: Route
        if routing.route == "auto":
            # Restore PII in draft
            final_text = await pii_restorer.restore(
                draft.draft_text_clean, ticket_id, tenant_id
            )
            await send_response(ticket, final_text)
            await update_ticket(ticket_id, status="auto_resolved", resolved_at=now())
            await audit.log("auto_responded", {"confidence": confidence})

            # STAGE 9: Eval (async, non-blocking)
            asyncio.create_task(
                eval_harness.evaluate_auto_response(
                    ticket, draft, tenant_id
                )
            )
        else:
            await enqueue_for_review(
                ticket, classification, draft, reason=routing.reason
            )
            await audit.log("enqueued_for_review", {
                "reason": routing.reason, "confidence": confidence,
            })

        # STAGE 8: Cost aggregation
        total_cost = await cost_tracker.aggregate_ticket(ticket_id, tenant_id)
        await audit.log("cost_recorded", {"total_cost_usd": float(total_cost)})

    except Exception as e:
        # Unhandled error: mark ticket as failed, audit log
        await update_ticket(ticket_id, status="failed")
        await audit.log("pipeline_error", {"error": str(e)}, success=False)
        logger.exception("Pipeline error", ticket_id=str(ticket_id))
```

### 9.7 Cost Tracker (`src/cost/tracker.py`)

```python
"""
Per-call cost tracking and per-tenant budget enforcement (from P9).

Every LLM call (classify, embed, draft, eval) records its cost.
The budget checker is called BEFORE the draft LLM call to prevent
spending beyond the tenant's monthly budget.

Cost model (GPT-4o-mini):
  - Input: $0.15 / 1M tokens
  - Output: $0.60 / 1M tokens
  - Embedding (text-embedding-3-small): $0.02 / 1M tokens
"""

PRICING = {
    "gpt-4o-mini": {"input": 0.15e-6, "output": 0.60e-6},
    "text-embedding-3-small": {"input": 0.02e-6, "output": 0.0},
    "cross-encoder/ms-marco-MiniLM-L-6-v2": {"input": 0.0, "output": 0.0},  # local
}

async def record(
    tenant_id: str,
    ticket_id: UUID,
    stage: str,
    model: str,
    input_tokens: int,
    output_tokens: int,
):
    pricing = PRICING.get(model, {"input": 0, "output": 0})
    cost = (input_tokens * pricing["input"]) + (output_tokens * pricing["output"])

    await db.execute(
        """INSERT INTO cost_ledger
           (tenant_id, ticket_id, stage, model_used, input_tokens,
            output_tokens, cost_usd)
           VALUES ($1,$2,$3,$4,$5,$6,$7)""",
        tenant_id, ticket_id, stage, model,
        input_tokens, output_tokens, cost,
    )

    # Update tenant spend
    await db.execute(
        "UPDATE tenants SET current_spend_usd = current_spend_usd + $1 WHERE id = $2",
        cost, tenant_id,
    )

    return cost


async def check_budget(tenant_id: str) -> bool:
    row = await db.fetchrow(
        "SELECT current_spend_usd, monthly_budget_usd FROM tenants WHERE id = $1",
        tenant_id,
    )
    return row["current_spend_usd"] < row["monthly_budget_usd"]


async def aggregate_ticket(ticket_id: UUID, tenant_id: str) -> Decimal:
    row = await db.fetchrow(
        "SELECT COALESCE(SUM(cost_usd), 0) as total FROM cost_ledger WHERE ticket_id = $1",
        ticket_id,
    )
    return row["total"]
```

### 9.8 Evaluation Harness (`src/eval/harness.py`)

```python
"""
Evaluation harness for auto-responses (from P7).

Runs three LLM-as-judge metrics:
  1. Faithfulness: Is the response grounded in retrieved context? (no hallucination)
  2. Relevance: Does the response address the customer's actual question?
  3. Citation accuracy: Do citations point to the correct source chunks?

Runs asynchronously after auto-response is sent. Non-blocking — if eval
fails, the ticket is still resolved. Results stored in eval_results and
displayed on the monitoring dashboard.

The eval dataset grows from the feedback loop: rejected/edited drafts
become labeled examples. Over time, the eval suite tests against
real agent-corrected responses, not just synthetic test cases.
"""

async def evaluate_auto_response(
    ticket: Ticket,
    draft: DraftResult,
    tenant_id: str,
):
    try:
        # 1. Retrieve the context that was used for the draft
        retrieved_chunks = await get_chunks(draft.retrieved_chunk_ids)

        # 2. Run three judge metrics in parallel
        faithfulness, relevance, citation_acc = await asyncio.gather(
            judge.faithfulness(draft.draft_text, retrieved_chunks, ticket.body_redacted),
            judge.relevance(draft.draft_text, ticket.subject, ticket.body_redacted),
            judge.citation_accuracy(draft.citations, retrieved_chunks),
        )

        overall = (faithfulness + relevance + citation_acc) / 3

        # 3. Store results
        eval_id = uuid4()
        await db.execute(
            """INSERT INTO eval_results
               (id, tenant_id, ticket_id, draft_id, eval_run_id,
                faithfulness, relevance, citation_accuracy, overall_score,
                eval_model_used, eval_tokens, eval_cost_usd)
               VALUES ($1,$2,$3,$4,$5,$6,$7,$8,$9,$10,$11,$12)""",
            eval_id, tenant_id, ticket.id, draft.id, str(uuid4()),
            faithfulness, relevance, citation_acc, overall,
            judge.model_name, judge.last_token_count, judge.last_cost,
        )

        # 4. Cost tracking for eval
        await cost_tracker.record(
            tenant_id=tenant_id, ticket_id=ticket.id, stage="eval",
            model=judge.model_name, input_tokens=judge.last_input_tokens,
            output_tokens=judge.last_output_tokens,
        )

        # 5. Audit
        await audit_logger.log(
            tenant_id=tenant_id, ticket_id=ticket.id, stage="evaluated",
            event_data={
                "faithfulness": faithfulness, "relevance": relevance,
                "citation_accuracy": citation_acc, "overall": overall,
            },
        )

    except Exception as e:
        # Eval is non-blocking — log and move on
        await audit_logger.log(
            tenant_id=tenant_id, ticket_id=ticket.id, stage="eval_skipped",
            event_data={"error": str(e)}, success=False,
        )
        logger.warning("Eval skipped", ticket_id=str(ticket.id), error=str(e))
```

### 9.9 Feedback Collector (`src/review/feedback_collector.py`)

```python
"""
Captures agent feedback on AI drafts and feeds it into the eval dataset.

When an agent edits or rejects a draft, the feedback is:
  1. Stored in the feedback table (with original draft + final response + tags)
  2. Added to the eval dataset as a new labeled example
  3. The eval dataset is versioned (each addition increments the version)

This creates a continuous improvement loop:
  - Agent rejects draft with tag "incorrect_info"
  - Feedback row created with original draft + agent's correction
  - Eval dataset grows with this example
  - Next eval run tests against this example
  - If the system still fails → prompt/template improvement needed
  - Prompt improvement → re-run eval → verify improvement (from P8 pattern)
"""

async def capture_feedback(
    review_id: UUID,
    ticket_id: UUID,
    draft_id: UUID,
    agent_id: UUID,
    tenant_id: str,
    action: str,               # 'approved', 'rejected', 'edited'
    feedback_tags: list[str],
    feedback_text: str | None,
    final_response: str | None,  # agent's version if edited
) -> Feedback:
    # 1. Get original draft
    draft = await get_draft(draft_id)

    # 2. Store feedback
    feedback = await db.fetchrow(
        """INSERT INTO feedback
           (tenant_id, ticket_id, draft_id, agent_id, action,
            feedback_tags, feedback_text, original_draft, final_response)
           VALUES ($1,$2,$3,$4,$5,$6,$7,$8,$9)
           RETURNING id""",
        tenant_id, ticket_id, draft_id, agent_id, action,
        feedback_tags, feedback_text, draft.draft_text, final_response,
    )

    # 3. Add to eval dataset (for rejected and edited drafts)
    if action in ("rejected", "edited"):
        dataset_version = await eval_dataset.add_example(
            ticket_id=ticket_id,
            tenant_id=tenant_id,
            original_draft=draft.draft_text,
            corrected_response=final_response or "",
            feedback_tags=feedback_tags,
            source="agent_feedback",
        )
        await db.execute(
            "UPDATE feedback SET added_to_eval_dataset=TRUE, eval_dataset_version=$1 WHERE id=$2",
            dataset_version, feedback["id"],
        )

    # 4. Compute edit diff (if edited)
    if action == "edited" and final_response:
        diff = compute_diff(draft.draft_text, final_response)
        await db.execute(
            "UPDATE review_queue SET edit_diff=$1 WHERE id=$2",
            json.dumps(diff), review_id,
        )

    # 5. Audit
    await audit_logger.log(
        tenant_id=tenant_id, ticket_id=ticket_id, agent_id=agent_id,
        stage="reviewed",
        event_data={
            "action": action, "feedback_tags": feedback_tags,
            "added_to_eval": action in ("rejected", "edited"),
        },
    )

    return Feedback(id=feedback["id"])
```

---

## 10. Human Review Queue Design

### 10.1 Queue Data Model

```
review_queue table:
  ┌────────────┬────────────┬──────────┬──────────┬───────────┬───────────┐
  │ ticket_id  │ draft_id   │ status   │ priority │ assigned_ │ queued_at │
  │            │            │          │          │ to        │           │
  ├────────────┼────────────┼──────────┼──────────┼───────────┼───────────┤
  │ uuid-001   │ uuid-draft1│ pending  │ 3 (high) │ null      │ 10:01 AM  │
  │ uuid-002   │ uuid-draft2│ in_review│ 2 (med)  │ agent-05  │ 10:03 AM  │
  │ uuid-003   │ uuid-draft3│ pending  │ 4 (crit) │ null      │ 10:05 AM  │
  │ uuid-004   │ null       │ pending  │ 1 (low)  │ null      │ 10:07 AM  │
  └────────────┴────────────┴──────────┴──────────┴───────────┴───────────┘

Priority mapping:
  critical urgency → priority 4
  high urgency    → priority 3
  medium urgency  → priority 2
  low urgency     → priority 1

Sort order: priority DESC, queued_at ASC (FIFO within same priority)
```

### 10.2 Review Queue Dashboard Layout (Streamlit, port 8501)

```
┌─────────────────────────────────────────────────────────────────────────┐
│  🎫 Customer Support AI — Review Queue          [Agent: jane@acme.com]  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────┐  ┌──────────────────────────────────────────────┐  │
│  │                 │  │                                              │  │
│  │  QUEUE LIST     │  │  TICKET DETAIL + DRAFT EDITOR                │  │
│  │                 │  │                                              │  │
│  │  ▸ #T-001       │  │  Subject: "Can't reset my password"         │  │
│  │    HIGH · bill  │  │  From: [NAME_1] <[EMAIL_1]>                 │  │
│  │    angry        │  │  Received: 2025-01-15 10:01 AM              │  │
│  │                 │  │                                              │  │
│  │  ▸ #T-003       │  │  ── Classification ──                       │  │
│  │    CRIT · tech  │  │  Urgency: HIGH    Category: BILLING         │  │
│  │    negative     │  │  Sentiment: ANGRY  Confidence: 0.42         │  │
│  │                 │  │                                              │  │
│  │  ▸ #T-004       │  │  ── Retrieved Knowledge ──                  │  │
│  │    LOW · general│  │  [1] FAQ: Password Reset Procedure          │  │
│  │    neutral      │  │  [2] Doc: Account Security Best Practices   │  │
│  │                 │  │                                              │  │
│  │                 │  │  ── AI Draft (editable) ──                  │  │
│  │                 │  │  ┌──────────────────────────────────────┐  │  │
│  │                 │  │  │ Hi [NAME_1],                         │  │  │
│  │                 │  │  │                                      │  │  │
│  │                 │  │  │ I understand you're having trouble   │  │  │
│  │                 │  │  │ resetting your password. To reset    │  │  │
│  │                 │  │  │ your password, please go to the      │  │  │
│  │                 │  │  │ login page and click "Forgot         │  │  │
│  │                 │  │  │ Password" [1]. You'll receive an     │  │  │
│  │                 │  │  │ email at [EMAIL_1] with a reset      │  │  │
│  │                 │  │  │ link...                              │  │  │
│  │                 │  │  │                                      │  │  │
│  │                 │  │  │ [EDIT HERE — agent can modify text]  │  │  │
│  │                 │  │  └──────────────────────────────────────┘  │  │
│  │                 │  │                                              │  │
│  │                 │  │  ── Feedback (required on reject/edit) ──  │  │
│  │                 │  │  Tags: ☐ wrong_category  ☐ bad_tone        │  │
│  │                 │  │        ☑ incorrect_info  ☐ missing_info    │  │
│  │                 │  │        ☐ other                               │  │
│  │                 │  │  Notes: [text area for free-form feedback] │  │
│  │                 │  │                                              │  │
│  │                 │  │  [APPROVE AS-IS]  [EDIT & APPROVE]  [REJECT]│  │
│  │                 │  │                                              │  │
│  └─────────────────┘  └──────────────────────────────────────────────┘  │
│                                                                         │
│  Queue stats: 3 pending · 1 in review · 12 resolved today               │
└─────────────────────────────────────────────────────────────────────────┘
```

### 10.3 Agent Workflow

```
1. Agent opens review queue dashboard (port 8501)
2. Agent sees sorted queue (priority DESC, queued_at ASC)
3. Agent clicks a ticket → ticket detail + AI draft loads
4. Agent reviews:
   a. Ticket metadata (subject, customer, classification)
   b. Retrieved knowledge chunks (with source links)
   c. AI draft response (with citation markers visible to agent)
5. Agent takes one of three actions:

   ┌──────────────────┬──────────────────────────────────────────────────┐
   │ Action           │ System Behavior                                  │
   ├──────────────────┼──────────────────────────────────────────────────┤
   │ APPROVE AS-IS    │ - PII tokens restored in draft                   │
   │                  │ - Response sent to customer                      │
   │                  │ - Ticket status → human_resolved                 │
   │                  │ - Feedback row: action=approved                 │
   │                  │ - Audit: "reviewed" (action=approved)            │
   │                  │ - NOT added to eval dataset (no correction)      │
   ├──────────────────┼──────────────────────────────────────────────────┤
   │ EDIT & APPROVE   │ - Agent edits draft text in editor               │
   │                  │ - PII tokens restored in edited version          │
   │                  │ - Edited response sent to customer               │
   │                  │ - Ticket status → human_resolved                 │
   │                  │ - Feedback row: action=edited, edit_diff stored  │
   │                  │ - Added to eval dataset (original vs corrected)  │
   │                  │ - Audit: "reviewed" (action=edited)              │
   ├──────────────────┼──────────────────────────────────────────────────┤
   │ REJECT           │ - Feedback required (tags + text)                │
   │                  │ - No response sent to customer                   │
   │                  │ - Ticket status → human_rejected                 │
   │                  │ - Ticket re-queued for manual response OR        │
   │                  │   supervisor notified                            │
   │                  │ - Feedback row: action=rejected                 │
   │                  │ - Added to eval dataset (original + feedback)    │
   │                  │ - Audit: "reviewed" (action=rejected)            │
   └──────────────────┴──────────────────────────────────────────────────┘
6. Queue auto-refreshes (next pending ticket loads)
```

### 10.4 Queue API Endpoints

```
GET /review/queue?status=pending&limit=20
  → List of pending review items sorted by priority DESC, queued_at ASC
  → Each item: {review_id, ticket_id, subject, urgency, category, sentiment,
                confidence, queued_at, draft_preview}

GET /review/{review_id}
  → Full detail: ticket body (redacted), classification, retrieved chunks,
    draft text with citations, PII entity map (for agent context)

POST /review/{review_id}/claim
  → Agent claims the ticket (status → in_review, assigned_to → agent)
  → Prevents two agents working on the same ticket

POST /review/{review_id}/release
  → Agent releases claim (status → pending, assigned_to → null)

POST /review/{review_id}/approve
  → Body: {feedback_tags?: [], feedback_text?: ""}
  → Approves draft as-is, sends response, closes ticket

POST /review/{review_id}/edit
  → Body: {edited_text: "...", feedback_tags: [], feedback_text: "..."}
  → Approves edited draft, sends response, closes ticket, stores diff

POST /review/{review_id}/reject
  → Body: {feedback_tags: ["required"], feedback_text: "required"}
  → Rejects draft, ticket re-queued or escalated, feedback captured
```

### 10.5 Concurrency & Claim Mechanism

```
Problem: Two agents open the same ticket simultaneously.

Solution: Claim mechanism with optimistic locking.

1. Agent A clicks ticket → POST /review/{id}/claim
   - If status == 'pending': set status='in_review', assigned_to=agent_A
   - Return 200 OK

2. Agent B clicks same ticket → POST /review/{id}/claim
   - If status == 'in_review': return 409 Conflict
   - "This ticket is being reviewed by another agent"

3. Agent A finishes (approve/edit/reject) → status changes to terminal

4. If Agent A goes idle (no action for 10 min):
   - Background task: release claim, status → pending
   - Ticket reappears in queue
```

---

## 11. Monitoring Dashboard Layout

### 11.1 Dashboard Sections (Streamlit, port 8502)

```
┌─────────────────────────────────────────────────────────────────────────┐
│  📊 Customer Support AI — Monitoring Dashboard                         │
│  Tenant: [acme-corp ▼]   Date range: [Last 7 days ▼]   [Refresh]      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─── KPI CARDS (top row) ─────────────────────────────────────────────┐ │
│  │                                                                     │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐│ │
│  │  │  Ticket  │  │   Auto   │  │   Avg    │  │   Cost   │  │ Queue  ││ │
│  │  │  Volume  │  │ Resolution│  │ Response │  │   per   │  │ Depth  ││ │
│  │  │          │  │   Rate   │  │   Time   │  │  Ticket  │  │        ││ │
│  │  │   847    │  │  62.3%   │  │  4.2 min │  │ $0.0084  │  │   12   ││ │
│  │  │  ↑ 15%   │  │  ↑ 3.1%  │  │  ↓ 0.8m  │  │  ↓ 12%   │  │ ↓ 4   ││ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └────────┘│ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌─── TICKET VOLUME (time series) ─────────────────────────────────────┐ │
│  │                                                                     │ │
│  │  120 ┤                                    ╭──                        │ │
│  │  100 ┤                              ╭───╯                           │ │
│  │   80 ┤                        ╭───╯                               │ │
│  │   60 ┤                  ╭───╯                                     │ │
│  │   40 ┤            ╭───╯                                           │ │
│  │   20 ┤      ╭───╯                                                 │ │
│  │   0  └─────╯                                                       │ │
│  │       Mon  Tue  Wed  Thu  Fri  Sat  Sun                            │ │
│  │       ▓ Auto-resolved  ░ Human-resolved  ░ Pending                 │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌─── AUTO-RESOLUTION RATE (trend) ───────────────────────────────────┐ │
│  │                                                                     │ │
│  │  70% ┤                                    ╭──╮                      │ │
│  │  65% ┤                              ╭──╯     ╰──╮                   │ │
│  │  60% ┤                        ╭──╯              ╰──                │ │
│  │  55% ┤                  ╭──╯                                      │ │
│  │  50% ┤            ╭──╯                                            │ │
│  │       Mon  Tue  Wed  Thu  Fri  Sat  Sun                            │ │
│  │       Target line at 65% (dashed)                                  │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌─── EVAL SCORES (trend) ────────────┐  ┌─── COST BREAKDOWN ──────────┐ │
│  │                                    │  │                              │ │
│  │  1.0 ┤  ╭──╮                       │  │  ▓▓▓▓▓ Classification 42%  │ │
│  │  0.8 ┤╯     ╰──╮                    │  │  ▓▓▓▓ Draft Response 31%   │ │
│  │  0.6 ┤          ╰──                 │  │  ▓▓▓ RAG Embedding 15%     │ │
│  │  0.4 ┤                             │  │  ▓▓ Evaluation 8%           │ │
│  │      ┤─── Faithfulness              │  │  ▓ Re-ranking 4%           │ │
│  │      ┤─── Relevance                 │  │                              │ │
│  │      ┤─── Citation Accuracy         │  │  Total: $7.12 this week    │ │
│  │      Mon  Tue  Wed  Thu  Fri       │  │  Budget: $100.00/month      │ │
│  │                                    │  │  Remaining: $92.88          │ │
│  └────────────────────────────────────┘  └──────────────────────────────┘ │
│                                                                         │
│  ┌─── REVIEW QUEUE DEPTH (time series) ────────────────────────────────┐ │
│  │                                                                     │ │
│  │  20 ┤     ╭──╮                                                      │ │
│  │  15 ┤   ╯     ╰──                                                   │ │
│  │  10 ┤──╯          ╰──╮                                              │ │
│  │   5 ┤                  ╰──                                          │ │
│  │   0 ┤                                                               │ │
│  │       Mon  Tue  Wed  Thu  Fri  Sat  Sun                            │ │
│  │       Shows backlog trend — rising queue = system needs attention   │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌─── CLASSIFICATION DISTRIBUTION ─────────────────────────────────────┐ │
│  │                                                                     │ │
│  │  By Urgency:    ▓▓▓▓▓ Medium 45%  ▓▓▓ High 30%  ▓▓ Low 20%  ▓ Crit 5% │
│  │  By Category:   ▓▓▓▓ Technical 40%  ▓▓▓ Billing 30%  ▓▓ General 20%  │ │
│  │                 ▓ Complaint 10%                                    │ │
│  │  By Sentiment:  ▓▓▓▓ Neutral 50%  ▓▓ Negative 25%  ▓ Positive 20%   │ │
│  │                 ▓ Angry 5%                                         │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌─── RECENT AUTO-RESPONSES (table) ───────────────────────────────────┐ │
│  │                                                                     │ │
│  │  Ticket  │ Category │ Confidence │ Eval Score │ Cost    │ Status    │ │
│  │  ────────┼──────────┼────────────┼────────────┼─────────┼────────── │ │
│  │  T-847   │ billing  │ 0.89       │ 0.92       │ $0.006  │ resolved  │ │
│  │  T-846   │ technical│ 0.81       │ 0.85       │ $0.009  │ resolved  │ │
│  │  T-845   │ general  │ 0.78       │ 0.88       │ $0.005  │ resolved  │ │
│  │  T-844   │ billing  │ 0.76       │ 0.71       │ $0.007  │ resolved  │ │
│  │  ...     │ ...      │ ...        │ ...        │ ...     │ ...       │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌─── FEEDBACK SUMMARY ────────────────────────────────────────────────┐ │
│  │                                                                     │ │
│  │  Total feedback this week: 23                                       │ │
│  │  Approved as-is: 8 (35%)                                            │ │
│  │  Edited: 10 (43%)                                                   │ │
│  │  Rejected: 5 (22%)                                                  │ │
│  │                                                                     │ │
│  │  Top feedback tags:                                                 │ │
│  │    ▓▓▓▓▓ incorrect_info (8)                                         │ │
│  │    ▓▓▓ bad_tone (4)                                                 │ │
│  │    ▓▓ missing_info (3)                                              │ │
│  │    ▓ wrong_category (2)                                             │ │
│  │                                                                     │ │
│  │  Eval dataset size: 156 examples (↑ 23 this week)                  │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 11.2 Metrics Endpoint

```python
"""
GET /metrics?tenant_id=X&days=7

Returns all aggregated data needed by the monitoring dashboard.
Single endpoint to minimize API calls from Streamlit.
"""

async def get_metrics(tenant_id: str, days: int = 7) -> dict:
    return {
        "kpi_cards": {
            "ticket_volume": await count_tickets(tenant_id, days),
            "auto_resolution_rate": await auto_resolution_rate(tenant_id, days),
            "avg_response_time_min": await avg_response_time(tenant_id, days),
            "avg_cost_per_ticket": await avg_cost_per_ticket(tenant_id, days),
            "queue_depth": await queue_depth(tenant_id),
        },
        "time_series": {
            "ticket_volume": await daily_ticket_counts(tenant_id, days),
            "auto_resolution_rate": await daily_auto_resolution(tenant_id, days),
            "queue_depth": await hourly_queue_depth(tenant_id, days),
            "eval_scores": await daily_eval_scores(tenant_id, days),
        },
        "distributions": {
            "urgency": await urgency_distribution(tenant_id, days),
            "category": await category_distribution(tenant_id, days),
            "sentiment": await sentiment_distribution(tenant_id, days),
        },
        "cost": {
            "total_usd": await total_cost(tenant_id, days),
            "by_stage": await cost_by_stage(tenant_id, days),
            "budget_remaining": await budget_remaining(tenant_id),
        },
        "feedback": {
            "total": await feedback_count(tenant_id, days),
            "by_action": await feedback_by_action(tenant_id, days),
            "top_tags": await top_feedback_tags(tenant_id, days),
            "eval_dataset_size": await eval_dataset_size(tenant_id),
        },
        "recent_auto_responses": await recent_auto_responses(tenant_id, limit=20),
    }
```

---

## 12. Feedback Loop Architecture

### 12.1 The Continuous Improvement Loop

```
                         ┌──────────────────────────┐
                         │   AI Drafts Response      │
                         │   (auto or review queue)  │
                         └────────────┬─────────────┘
                                      │
                                      ▼
                         ┌──────────────────────────┐
                         │   Agent Interacts         │
                         │   in Review Queue         │
                         │                          │
                         │   ┌─────────────────┐    │
                         │   │ APPROVE as-is   │    │
                         │   │ → positive signal│   │
                         │   │ → not added to  │    │
                         │   │   eval dataset  │    │
                         │   └─────────────────┘    │
                         │                          │
                         │   ┌─────────────────┐    │
                         │   │ EDIT & APPROVE  │    │
                         │   │ → negative signal│   │
                         │   │ → added to eval │    │
                         │   │   dataset with  │    │
                         │   │   correction    │    │
                         │   └─────────────────┘    │
                         │                          │
                         │   ┌─────────────────┐    │
                         │   │ REJECT          │    │
                         │   │ → strong neg    │    │
                         │   │ → added to eval │    │
                         │   │   dataset with  │    │
                         │   │   feedback tags │    │
                         │   └─────────────────┘    │
                         └────────────┬─────────────┘
                                      │
                                      ▼
                         ┌──────────────────────────┐
                         │   Feedback Collector      │
                         │                          │
                         │   Stores in feedback     │
                         │   table:                 │
                         │   - original_draft       │
                         │   - final_response       │
                         │   - feedback_tags        │
                         │   - feedback_text        │
                         │                          │
                         │   Adds to eval dataset:  │
                         │   - versioned            │
                         │   - grows over time      │
                         └────────────┬─────────────┘
                                      │
                                      ▼
                         ┌──────────────────────────┐
                         │   Eval Dataset            │
                         │   (versioned, growing)   │
                         │                          │
                         │   Contains:              │
                         │   - synthetic test cases │
                         │     (initial seed)       │
                         │   - real agent-corrected │
                         │     examples             │
                         │   - rejection cases with │
                         │     failure tags         │
                         └────────────┬─────────────┘
                                      │
                                      ▼
                         ┌──────────────────────────┐
                         │   Eval Harness Runs       │
                         │   (scheduled + on-demand) │
                         │                          │
                         │   Tests current system    │
                         │   against growing dataset │
                         │                          │
                         │   Metrics:               │
                         │   - faithfulness         │
                         │   - relevance            │
                         │   - citation accuracy    │
                         └────────────┬─────────────┘
                                      │
                                      ▼
                         ┌──────────────────────────┐
                         │   Scores Tracked Over    │
                         │   Time on Dashboard      │
                         │                          │
                         │   If scores decline:     │
                         │   → investigate which    │
                         │     feedback tags are    │
                         │     most common          │
                         │   → identify systemic    │
                         │     failure patterns     │
                         └────────────┬─────────────┘
                                      │
                                      ▼
                         ┌──────────────────────────┐
                         │   Prompt / Template       │
                         │   Improvement             │
                         │   (from P8 pattern)       │
                         │                          │
                         │   Based on feedback tags: │
                         │   - "incorrect_info" →    │
                         │     improve RAG retrieval │
                         │     or prompt grounding   │
                         │   - "bad_tone" →          │
                         │     adjust sentiment-     │
                         │     aware prompt template │
                         │   - "missing_info" →      │
                         │     increase top_k or     │
                         │     add KB docs           │
                         │   - "wrong_category" →    │
                         │     refine classification │
                         │     prompt               │
                         └────────────┬─────────────┘
                                      │
                                      ▼
                         ┌──────────────────────────┐
                         │   Re-run Eval Suite       │
                         │   Against Updated System  │
                         │                          │
                         │   Verify improvement:     │
                         │   - scores should go up   │
                         │   - specific failure      │
                         │     cases should pass     │
                         │                          │
                         │   If not improved:        │
                         │   → revert prompt change  │
                         │   → try different fix     │
                         └────────────┬─────────────┘
                                      │
                                      └───────────────┐
                                      │               │
                                      ▼               │
                         ┌──────────────────────────┐│
                         │   Improved System         ││
                         │   Produces Better Drafts  ││
                         │   → Fewer rejections      ││
                         │   → Higher auto-rate      ││
                         │   → Lower cost/ticket     ││
                         └────────────┬─────────────┘│
                                      │               │
                                      └───────────────┘
                                      (loop continues)
```

### 12.2 Feedback Tags → Systemic Improvement Mapping

| Feedback Tag | Root Cause Hypothesis | Improvement Action | Eval Metric to Watch |
---|---|---|---|
| `incorrect_info` | RAG retrieved wrong chunks OR LLM hallucinated | Improve retrieval (chunk size, re-ranking model) OR strengthen grounding prompt | Faithfulness |
| `bad_tone` | Sentiment-aware prompt didn't match customer emotion | Adjust tone instructions in prompt template per sentiment | Relevance |
| `missing_info` | RAG didn't find relevant docs OR top_k too low | Increase top_k, add more KB docs, improve query formulation | Citation accuracy |
| `wrong_category` | Classification LLM misclassified | Refine classification prompt with more examples; add few-shot | Classification accuracy (custom) |
| `too_long` | Response draft is verbose | Add length constraint to prompt | Relevance (partial) |
| `too_generic` | Response lacks specificity | Improve RAG retrieval precision; add ticket-specific context to prompt | Faithfulness |

### 12.3 Eval Dataset Versioning

```python
"""
The eval dataset is versioned. Each time a feedback example is added,
the version increments. This allows:
  - Tracking eval score trends against a stable dataset version
  - Comparing system performance across dataset versions
  - Rolling back to a previous dataset if new examples are noisy

Dataset format: JSONL file stored in /data/eval_datasets/
  eval_dataset_v001.jsonl  (initial seed: 20 synthetic examples)
  eval_dataset_v002.jsonl  (+ 5 examples from agent feedback)
  eval_dataset_v003.jsonl  (+ 12 examples from agent feedback)
  ...

Each example:
{
  "id": "uuid",
  "ticket_subject": "...",
  "ticket_body_redacted": "...",
  "classification": {"urgency": "...", "category": "...", "sentiment": "..."},
  "expected_response": "...",  // agent's corrected version
  "feedback_tags": ["incorrect_info"],
  "source": "agent_feedback" | "synthetic",
  "added_at": "2025-01-15T10:00:00Z"
}
"""

class EvalDataset:
    def __init__(self, base_path: str = "data/eval_datasets/"):
        self.base_path = base_path

    async def add_example(self, ticket_id, tenant_id, original_draft,
                          corrected_response, feedback_tags, source) -> str:
        # Load current version
        current_version = await self._get_latest_version()
        current_examples = await self._load(current_version)

        # Add new example
        example = {
            "id": str(uuid4()),
            "ticket_id": str(ticket_id),
            "tenant_id": tenant_id,
            "original_draft": original_draft,
            "expected_response": corrected_response,
            "feedback_tags": feedback_tags,
            "source": source,
            "added_at": datetime.utcnow().isoformat(),
        }
        current_examples.append(example)

        # Save as new version
        new_version = self._increment_version(current_version)
        await self._save(new_version, current_examples)

        return new_version

    async def load_latest(self) -> list[dict]:
        version = await self._get_latest_version()
        return await self._load(version)
```

---

## 13. Security & Safety Considerations

### 13.1 Threat Model

| Threat | Mitigation | Layer | From Project |
---|---|---|---|
| **PII leakage to LLM** (customer PII sent to OpenAI) | Presidio redaction before any LLM call; PII stored encrypted at rest; tokens used in all LLM prompts | App | P10 |
| **Cross-tenant data leakage** (tenant A sees tenant B's tickets/KB) | SQL `WHERE tenant_id = ?` pre-filter + PostgreSQL RLS policies + test suite with leakage tests | App + DB | P2 |
| **Audit log tampering** | DB triggers block UPDATE/DELETE on `audit_logs`; only INSERT allowed | DB | P2 |
| **Prompt injection** (customer crafts ticket to manipulate LLM) | Customer input is in the "ticket body" field only; system prompt is immutable; output validation for injection patterns | App | P2 |
| **Budget bypass** (tenant exceeds LLM budget) | Budget checked before draft LLM call; if exceeded → route to human (no LLM call); budget enforced in `budget.py` | App | P9 |
| **Auto-response sends wrong response** (high confidence but wrong answer) | Confidence threshold (0.75); hard rules for critical/angry/complaint; eval suite catches faithfulness issues; all auto-responses logged for audit | App + Eval | P7 |
| **PII restoration failure** (tokens not replaced in final response) | Restorer runs on every response before sending; test suite verifies no tokens in sent responses; fallback: if restoration fails, route to human | App | P10 |
| **Cost tracking drift** (actual OpenAI cost ≠ tracked cost) | Token counts from API response used directly (not estimated); monthly reconciliation against OpenAI billing | App | P9 |
| **Review queue race condition** (two agents approve same ticket) | Claim mechanism with optimistic locking; 10-min idle timeout releases claims | App | — |
| **Eval dataset poisoning** (bad feedback examples degrade eval) | Supervisor can flag feedback as invalid; eval dataset versioned (can roll back); eval scores tracked over time to detect anomalies | App | P7 |

### 13.2 PII Safety Flow (Critical Path)

```
Ticket arrives with PII
       │
       ▼
┌──────────────────┐
│ Presidio scans    │     FAILS?
│ ────► Route to human (no LLM call)
│ body + subject    │            flag "PII scan failed"
└────────┬─────────┘
         │ SUCCESS
         ▼
┌──────────────────┐
│ PII replaced with │
│ tokens in         │
│ body_redacted     │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Original PII      │
│ encrypted in      │
│ pii_entities      │
│ table             │
└────────┬─────────┘
         │
         ▼
    All LLM calls use body_redacted (tokens only, no real PII)
         │
         ▼
┌──────────────────┐
│ LLM draft contains│
│ tokens: [NAME_1]  │
│ [EMAIL_1] etc.    │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Restorer replaces │     FAILS?
│ ──► Route to human (don't send)
│ tokens with       │            flag "PII restoration failed"
│ original values   │
└────────┬─────────┘
         │ SUCCESS
         ▼
┌──────────────────┐
│ Final response    │
│ sent to customer  │
│ with real PII     │
│ (their own data)  │
└──────────────────┘

INVARIANT: No real customer PII ever reaches an LLM API call.
INVARIANT: No PII tokens ever reach the customer in a response.
```

### 13.3 Defense in Depth

```
Layer 1: Application   → PII redaction before LLM; tenant pre-filter; budget check
Layer 2: Database      → RLS policies (tenant isolation even with direct DB access)
Layer 3: Audit         → Immutable audit trail proves every action (compliance)
Layer 4: Evaluation    → Eval suite catches quality regressions before they scale
Layer 5: Human Review  → Complex cases reviewed by humans; feedback loop improves system
```

### 13.4 GDPR Considerations (German Market)

| Concern | Mitigation |
---|---|
| **Customer PII in cloud LLM** (OpenAI) | PII redacted before LLM call; only tokens sent; OpenAI processes tokenized text, not real PII |
| **Right to erasure** | `pii_entities` table can be purged per ticket; ticket body encrypted at rest with per-tenant key; audit logs retain action metadata but can be pseudonymized |
| **Data residency** | OpenAI API calls go to EU endpoint (`openai.eu` if configured); all data at rest in local Postgres |
| **Audit trail for compliance** | Every action logged with timestamp, actor, and stage-specific metadata; immutable (triggers prevent modification) |
| **PII minimization** | Only PII necessary for ticket resolution is stored; original values encrypted; tokens used for processing |

---

## 14. API Specification

### 14.1 `POST /tickets`

Submit a new support ticket for processing.

```
Headers:
  Authorization: Bearer <JWT>
  X-Tenant-ID: acme-corp

Body:
{
  "external_id": "ZD-12345",           // optional, from helpdesk
  "customer_email": "jane@example.com", // PII, will be encrypted
  "customer_name": "Jane Doe",          // PII, will be encrypted
  "subject": "Can't reset my password",
  "body": "Hi, I'm Jane Doe. I've been trying to reset my password for 3 days..."
}

Response 202 (Accepted):
{
  "ticket_id": "uuid",
  "status": "new",
  "message": "Ticket received and queued for processing"
}

Response 422: Validation error
```

### 14.2 `GET /tickets/{ticket_id}`

Check ticket status and processing details.

```
Response 200:
{
  "ticket_id": "uuid",
  "external_id": "ZD-12345",
  "subject": "Can't reset my password",
  "status": "auto_resolved",            // or: new, processing, pending_review,
                                        //     human_resolved, human_rejected, failed
  "classification": {
    "urgency": "high",
    "category": "technical",
    "sentiment": "negative",
    "confidence": 0.85
  },
  "draft": {
    "confidence_score": 0.82,
    "route_decision": "auto",
    "citations": [...]
  },
  "cost_usd": 0.0084,
  "received_at": "2025-01-15T10:01:00Z",
  "resolved_at": "2025-01-15T10:05:30Z"
}
```

### 14.3 `POST /kb/upload`

Upload a document to the knowledge base. Requires `admin` or `editor` role.

```
Body (multipart/form-data):
  file: <binary file>
  source_type: "faq"                    // faq, doc, runbook
  allowed_roles: "agent,supervisor"     // comma-separated

Response 201:
{
  "doc_id": "uuid",
  "chunk_count": 42,
  "title": "Password Reset FAQ"
}
```

### 14.4 `GET /review/queue`

List tickets in the review queue.

```
Query params:
  ?status=pending&limit=20&page=1

Response 200:
{
  "items": [
    {
      "review_id": "uuid",
      "ticket_id": "uuid",
      "subject": "Can't reset my password",
      "urgency": "high",
      "category": "technical",
      "sentiment": "negative",
      "confidence_score": 0.42,
      "queued_at": "2025-01-15T10:03:00Z",
      "draft_preview": "Hi [NAME_1], I understand you're having..."
    }
  ],
  "total": 12,
  "page": 1
}
```

### 14.5 `POST /review/{review_id}/approve`

```
Body:
{
  "feedback_tags": [],           // optional for approve
  "feedback_text": ""            // optional for approve
}

Response 200:
{
  "ticket_id": "uuid",
  "status": "human_resolved",
  "response_sent": true
}
```

### 14.6 `POST /review/{review_id}/edit`

```
Body:
{
  "edited_text": "Hi Jane, I understand you're having trouble...",
  "feedback_tags": ["bad_tone"],
  "feedback_text": "Original was too formal for an angry customer"
}

Response 200:
{
  "ticket_id": "uuid",
  "status": "human_resolved",
  "response_sent": true,
  "feedback_id": "uuid",
  "added_to_eval_dataset": true,
  "eval_dataset_version": "v003"
}
```

### 14.7 `POST /review/{review_id}/reject`

```
Body:
{
  "feedback_tags": ["incorrect_info", "missing_info"],   // required
  "feedback_text": "The response references a feature we don't have"  // required
}

Response 200:
{
  "ticket_id": "uuid",
  "status": "human_rejected",
  "feedback_id": "uuid",
  "added_to_eval_dataset": true,
  "eval_dataset_version": "v003"
}
```

### 14.8 `GET /metrics`

Aggregated metrics for the monitoring dashboard.

```
Query params:
  ?tenant_id=acme-corp&days=7

Response 200:
{
  "kpi_cards": {
    "ticket_volume": 847,
    "auto_resolution_rate": 0.623,
    "avg_response_time_min": 4.2,
    "avg_cost_per_ticket": 0.0084,
    "queue_depth": 12
  },
  "time_series": { ... },
  "distributions": { ... },
  "cost": { ... },
  "feedback": { ... },
  "recent_auto_responses": [ ... ]
}
```

### 14.9 `POST /eval/run`

Trigger an evaluation run (manual or scheduled).

```
Body:
{
  "tenant_id": "acme-corp",
  "dataset_version": "latest",       // or specific version
  "limit": 50                        // max examples to eval
}

Response 202:
{
  "eval_run_id": "uuid",
  "message": "Evaluation started",
  "dataset_version": "v003",
  "example_count": 156
}
```

### 14.10 `GET /health`

```
Response 200:
{
  "status": "healthy",
  "database": "connected",
  "redis": "connected",
  "openai": "reachable",
  "version": "1.0.0"
}
```

---
## 15. Testing Strategy

### 15.1 Test Categories

| Category | What It Proves | Priority | From Project |
|---|---|---|---|
| **PII safety tests** | No real PII reaches LLM; no tokens reach customer | P0 — must pass before any merge | P10 |
| **Security tests** | No cross-tenant leakage; RLS enforcement | P0 | P2 |
| **Pipeline integration tests** | Full flow: ingest → classify → RAG → draft → route → resolve | P0 | P5 |
| **Confidence/routing tests** | Threshold boundaries; hard rules override score | P0 | — |
| **Cost/budget tests** | Cost calculation accuracy; budget enforcement | P1 | P9 |
| **Eval tests** | Metric computation; dataset growth from feedback | P1 | P7 |
| **Review queue tests** | Queue operations; feedback capture; claim mechanism | P1 | — |
| **Unit tests** | Individual components work in isolation | P2 | All |
| **API tests** | HTTP endpoints return correct status codes + payloads | P2 | All |

### 15.2 Critical PII Safety Tests (`tests/test_pii.py`)

```python
"""
These tests are the backbone of the PII safety claim.
If ANY of these fail, the system is not safe to deploy.
"""

async def test_no_real_pii_in_redacted_text():
    """After redaction, body_redacted must contain zero real PII."""
    ticket_body = "Hi, I'm Jane Doe, email jane@example.com, phone +49 170 1234567"
    result = await redactor.redact(ticket_body, ticket_id, tenant_id)
    assert "Jane Doe" not in result.redacted_text
    assert "jane@example.com" not in result.redacted_text
    assert "+49 170 1234567" not in result.redacted_text
    assert "[PERSON_1]" in result.redacted_text
    assert "[EMAIL_1]" in result.redacted_text
    assert "[PHONE_1]" in result.redacted_text


async def test_pii_restoration_replaces_all_tokens():
    """Restorer must replace every token with its original value."""
    draft = "Hi [PERSON_1], we'll email you at [EMAIL_1] within 24 hours."
    restored = await restorer.restore(draft, ticket_id, tenant_id)
    assert "[PERSON_1]" not in restored
    assert "[EMAIL_1]" not in restored
    assert "Jane Doe" in restored
    assert "jane@example.com" in restored


async def test_no_tokens_in_sent_response(mock_send):
    """End-to-end: auto-response must not contain any PII tokens."""
    ticket = await submit_ticket(body="Hi, I'm Jane Doe...")
    await process_ticket(ticket.id, tenant_id)
    sent_text = mock_send.last_sent_text
    # No tokens should reach the customer
    assert "[PERSON_" not in sent_text
    assert "[EMAIL_" not in sent_text
    assert "[PHONE_" not in sent_text
    # Real PII should be present (it's the customer's own data)
    assert "Jane Doe" in sent_text


async def test_pii_scan_failure_routes_to_human():
    """If Presidio fails, ticket must route to human review (no LLM call)."""
    with mock_presidio_failure():
        ticket = await submit_ticket(body="...")
        await process_ticket(ticket.id, tenant_id)
        assert ticket.status == "pending_review"
        # Verify no LLM calls were made
        assert mock_llm.call_count == 0
```

### 15.3 Critical Security Tests (`tests/test_security.py`)

```python
async def test_tenant_a_cannot_see_tenant_b_tickets():
    """Tenant A's review queue must not contain tenant B's tickets."""
    ticket_b = await submit_ticket(tenant_id="tenant_b", body="...")
    await process_ticket(ticket_b.id, "tenant_b")
    queue_a = await get_review_queue(tenant_id="tenant_a")
    assert all(item["ticket_id"] != str(ticket_b.id) for item in queue_a)


async def test_tenant_a_cannot_retrieve_tenant_b_kb():
    """RAG retrieval for tenant A must not return tenant B's KB chunks."""
    await upload_kb_doc(tenant_id="tenant_b", content="Tenant B secret info")
    results = await rag_service.retrieve(
        ticket=make_ticket(tenant_id="tenant_a"),
        classification=make_classification(),
        user_roles=["agent"],
    )
    assert all(r.tenant_id == "tenant_a" for r in results)


async def test_rls_blocks_cross_tenant_db():
    """Even with direct DB access, RLS prevents cross-tenant reads."""
    conn = await get_db_connection(tenant="tenant_a")
    tickets = await conn.fetch("SELECT * FROM tickets")
    assert all(t["tenant_id"] == "tenant_a" for t in tickets)


async def test_audit_log_immutable():
    """Audit logs cannot be UPDATEd or DELETEd."""
    with pytest.raises(Exception):
        await db.execute("UPDATE audit_logs SET stage = 'tampered'")
    with pytest.raises(Exception):
        await db.execute("DELETE FROM audit_logs")
```

### 15.4 Pipeline Integration Test (`tests/test_pipeline.py`)

```python
async def test_full_pipeline_auto_resolve():
    """High-confidence ticket: ingest → classify → RAG → draft → auto-resolve."""
    # Seed KB with relevant FAQ
    await upload_kb_doc(tenant_id="acme", content="Password reset: click 'Forgot Password'...")

    # Submit ticket
    ticket = await submit_ticket(
        tenant_id="acme",
        subject="Can't reset my password",
        body="I forgot my password and can't log in. Please help.",
    )

    # Process
    await process_ticket(ticket.id, "acme")

    # Verify auto-resolved
    updated = await get_ticket(ticket.id, "acme")
    assert updated.status == "auto_resolved"
    assert updated.resolved_at is not None

    # Verify classification
    classification = await get_classification(ticket.id)
    assert classification.urgency in ("high", "medium")
    assert classification.category == "technical"

    # Verify draft exists with citations
    draft = await get_draft(ticket.id)
    assert len(draft.citations) > 0
    assert draft.route_decision == "auto"
    assert draft.confidence_score >= 0.75

    # Verify cost tracked
    cost = await get_ticket_cost(ticket.id)
    assert cost > 0

    # Verify audit trail
    audit_events = await get_audit_events(ticket.id)
    stages = [e["stage"] for e in audit_events]
    assert "ticket_received" in stages
    assert "pii_scanned" in stages
    assert "classified" in stages
    assert "rag_retrieved" in stages
    assert "response_drafted" in stages
    assert "confidence_scored" in stages
    assert "auto_responded" in stages
    assert "cost_recorded" in stages


async def test_full_pipeline_human_review():
    """Low-confidence ticket: routes to human review queue."""
    ticket = await submit_ticket(
        tenant_id="acme",
        subject="Complex billing dispute",
        body="I was charged 3 times for the same invoice and the amounts don't match my contract.",
    )
    await process_ticket(ticket.id, "acme")

    updated = await get_ticket(ticket.id, "acme")
    assert updated.status == "pending_review"

    queue_item = await get_review_item(ticket.id)
    assert queue_item is not None
    assert queue_item.status == "pending"


async def test_critical_urgency_always_routes_to_human():
    """Critical urgency overrides confidence — always human review."""
    ticket = await submit_ticket(
        tenant_id="acme",
        subject="PRODUCTION SYSTEM DOWN",
        body="Our entire production environment is down since 2 hours.",
    )
    await process_ticket(ticket.id, "acme")

    updated = await get_ticket(ticket.id, "acme")
    assert updated.status == "pending_review"  # never auto_resolved


async def test_budget_exceeded_routes_to_human():
    """When tenant budget is exceeded, ticket routes to human without LLM draft."""
    # Set budget to near-zero
    await set_tenant_budget("acme", monthly_budget=0.001)

    ticket = await submit_ticket(
        tenant_id="acme", subject="...", body="..."
    )
    await process_ticket(ticket.id, "acme")

    updated = await get_ticket(ticket.id, "acme")
    assert updated.status == "pending_review"

    # Verify no draft LLM call was made (only classification)
    costs = await get_cost_ledger(ticket.id)
    stages = [c["stage"] for c in costs]
    assert "draft" not in stages
```

### 15.5 Feedback Loop Test (`tests/test_eval.py`)

```python
async def test_rejected_draft_added_to_eval_dataset():
    """Rejecting a draft adds it to the eval dataset."""
    # Submit and process a ticket that goes to review
    ticket = await submit_ticket(tenant_id="acme", subject="...", body="...")
    await process_ticket(ticket.id, "acme")
    review_item = await get_review_item(ticket.id)

    # Reject it with feedback
    await reject_draft(
        review_id=review_item.id,
        feedback_tags=["incorrect_info"],
        feedback_text="Response references wrong feature",
    )

    # Verify feedback stored
    feedback = await get_feedback(ticket.id)
    assert feedback.action == "rejected"
    assert "incorrect_info" in feedback.feedback_tags
    assert feedback.added_to_eval_dataset == True

    # Verify eval dataset grew
    dataset = await eval_dataset.load_latest()
    assert any(ex["ticket_id"] == str(ticket.id) for ex in dataset)


async def test_approved_draft_not_added_to_eval_dataset():
    """Approving a draft as-is does NOT add to eval dataset (no correction)."""
    ticket = await submit_ticket(tenant_id="acme", subject="...", body="...")
    await process_ticket(ticket.id, "acme")
    review_item = await get_review_item(ticket.id)

    await approve_draft(review_id=review_item.id)

    feedback = await get_feedback(ticket.id)
    assert feedback.action == "approved"
    assert feedback.added_to_eval_dataset == False
```

### 15.6 Test Fixtures (`tests/conftest.py`)

```python
"""
Test setup:
  - Spin up test Postgres+pgvector+Redis (docker or testcontainers)
  - Run schema.sql
  - Seed two tenants (acme, globex) with agents and KB docs
  - Mock LLM client (returns deterministic responses for reproducible tests)
  - Mock OpenAI embedding client (returns deterministic vectors)
  - Provide FastAPI TestClient with overridden DB
"""

@pytest.fixture
async def db():
    """Fresh database for each test module."""
    conn = await asyncpg.connect(TEST_DB_URL)
    await conn.execute(open("src/db/schema.sql").read())
    yield conn
    await conn.close()

@pytest.fixture
def mock_llm():
    """Mock LLM that returns deterministic responses."""
    llm = MockLLMClient()
    llm.classification_response = {
        "urgency": "medium", "category": "technical",
        "sentiment": "neutral", "confidence": 0.82,
    }
    llm.draft_response = "Hi [NAME_1], to reset your password [1]..."
    return llm

@pytest.fixture
def acme_token():
    return generate_jwt(tenant_id="acme", agent_id="agent-001", role="agent")

@pytest.fixture
def globex_token():
    return generate_jwt(tenant_id="globex", agent_id="agent-002", role="agent")
```

---

## 16. Deployment

### 16.1 `docker-compose.yml`

```yaml
version: "3.9"

services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: support_ai
      POSTGRES_USER: support_user
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-changeme}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./docker/postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U support_user -d support_ai"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  api:
    build:
      context: .
      dockerfile: docker/Dockerfile.api
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgres://support_user:${POSTGRES_PASSWORD:-changeme}@postgres:5432/support_ai
      REDIS_URL: redis://redis:6379/0
      JWT_SECRET: ${JWT_SECRET:-dev-secret-change-in-prod}
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      EMBEDDING_MODEL: text-embedding-3-small
      LLM_MODEL: gpt-4o-mini
      RERANKER_MODEL: cross-encoder/ms-marco-MiniLM-L-6-v2
      AUTO_RESPONSE_THRESHOLD: "0.75"
      LOG_LEVEL: info
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 10s
      timeout: 5s
      retries: 3

  review_queue:
    build:
      context: .
      dockerfile: docker/Dockerfile.review
    ports:
      - "8501:8501"
    environment:
      API_URL: http://api:8000
      JWT_SECRET: ${JWT_SECRET:-dev-secret-change-in-prod}
    depends_on:
      api:
        condition: service_healthy

  monitoring:
    build:
      context: .
      dockerfile: docker/Dockerfile.monitor
    ports:
      - "8502:8502"
    environment:
      API_URL: http://api:8000
      JWT_SECRET: ${JWT_SECRET:-dev-secret-change-in-prod}
    depends_on:
      api:
        condition: service_healthy

volumes:
  postgres_data:
  redis_data:
```

### 16.2 `docker/Dockerfile.api`

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# System deps for PDF parsing + Presidio + spaCy
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential libpq-dev curl poppler-utils \
    && rm -rf /var/lib/apt/lists/*

# Python deps
COPY pyproject.toml .
RUN pip install --no-cache-dir -e .

# Download spaCy model for Presidio
RUN python -m spacy download en_core_web_lg

# Download re-ranker model (cached in image)
RUN python -c "from sentence_transformers import CrossEncoder; CrossEncoder('cross-encoder/ms-marco-MiniLM-L-6-v2')"

# Copy source
COPY src/ ./src/
COPY scripts/ ./scripts/
COPY data/ ./data/

EXPOSE 8000

CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 16.3 `docker/Dockerfile.review`

```dockerfile
FROM python:3.12-slim

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential libpq-dev curl \
    && rm -rf /var/lib/apt/lists/*

COPY pyproject.toml .
RUN pip install --no-cache-dir -e .

COPY dashboards/review_queue/ ./dashboards/review_queue/
COPY src/ ./src/

EXPOSE 8501

CMD ["streamlit", "run", "dashboards/review_queue/app.py", \
     "--server.port=8501", "--server.address=0.0.0.0"]
```

### 16.4 `docker/Dockerfile.monitor`

```dockerfile
FROM python:3.12-slim

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential libpq-dev curl \
    && rm -rf /var/lib/apt/lists/*

COPY pyproject.toml .
RUN pip install --no-cache-dir -e .

COPY dashboards/monitoring/ ./dashboards/monitoring/
COPY src/ ./src/

EXPOSE 8502

CMD ["streamlit", "run", "dashboards/monitoring/app.py", \
     "--server.port=8502", "--server.address=0.0.0.0"]
```

### 16.5 `docker/postgres/init.sql`

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "vector";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";
-- Schema tables created by scripts/init_db.py on first run
```

### 16.6 `.env.example`

```env
# Database
POSTGRES_PASSWORD=changeme-in-production

# Redis
REDIS_URL=redis://redis:6379/0

# JWT
JWT_SECRET=your-256-bit-secret
JWT_ALGORITHM=HS256
JWT_EXPIRY_HOURS=24

# OpenAI
OPENAI_API_KEY=sk-...
EMBEDDING_MODEL=text-embedding-3-small
LLM_MODEL=gpt-4o-mini

# Re-ranker (local, no API key needed)
RERANKER_MODEL=cross-encoder/ms-marco-MiniLM-L-6-v2

# Routing
AUTO_RESPONSE_THRESHOLD=0.75

# PII Encryption
PII_ENCRYPTION_KEY=base64-encoded-32-byte-key

# App
LOG_LEVEL=info
MAX_TICKET_BODY_LENGTH=10000

# Local models (alternative to OpenAI)
# USE_LOCAL_EMBEDDINGS=true
# USE_LOCAL_LLM=true
# OLLAMA_HOST=http://localhost:11434

# Eval
EVAL_JUDGE_MODEL=gpt-4o-mini
EVAL_SCHEDULE_HOURS=6
```

### 16.7 `.github/workflows/ci.yml`

```yaml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: pgvector/pgvector:pg16
        env:
          POSTGRES_DB: support_ai_test
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_pass
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 5s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 5s
          --health-timeout 3s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - name: Install deps
        run: |
          pip install -e .
          python -m spacy download en_core_web_lg
      - name: Lint
        run: ruff check src/ tests/
      - name: Test
        env:
          DATABASE_URL: postgres://test_user:test_pass@localhost:5432/support_ai_test
          REDIS_URL: redis://localhost:6379/0
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
          PII_ENCRYPTION_KEY: ${{ secrets.PII_ENCRYPTION_KEY }}
          JWT_SECRET: test-secret
        run: pytest tests/ -v --tb=short
```

### 16.8 Production Considerations

| Concern | Recommendation |
|---|---|
| **TLS** | Terminate TLS at reverse proxy (nginx/traefik); API and Streamlit listen on HTTP |
| **DB backups** | `pg_dump` cron job; pgvector indexes included in dump; encrypted PII values in backup |
| **Secrets** | Docker secrets or vault; PII encryption key must be managed separately (if lost, PII is unrecoverable) |
| **DB user privileges** | App uses limited-privilege user (SELECT/INSERT on most tables; INSERT-only on audit_logs) |
| **Rate limiting** | nginx `limit_req` or slowapi (e.g. 100 tickets/min per tenant, 30 review actions/min per agent) |
| **Monitoring** | Structured JSON logs → Loki/ELK; `/health` for k8s probes; Streamlit monitoring dashboard for ops |
| **PII key rotation** | Rotate PII encryption key periodically; re-encrypt `pii_entities` table during maintenance window |
| **Eval scheduling** | Cron or APScheduler: run eval suite every 6 hours; alert if scores drop below threshold |
| **KB updates** | Re-embed KB docs when updated; version KB docs; old chunks deactivated (not deleted) for audit |
| **Scale** | For high volume: separate worker process for pipeline (Celery/RQ); API only handles ingestion + review |

---

## 17. Roadmap & Milestones

### 17.1 Phase Timeline

```
Week 1 (Days 1–5)
├── Phase 1: Foundation & Infrastructure    [██░░░] Days 1–2
├── Phase 2: Ingestion & PII Redaction      [░██░░] Days 3–4
└── Phase 3: Knowledge Base RAG             [░░░██] Days 5–6

Week 2 (Days 6–10)
├── Phase 4: Classification & Drafting      [██░░░] Days 7–8
├── Phase 5: Confidence & Cost              [░░██░] Day 9
└── Phase 6: Human Review Queue             [░░░██] Days 10–11

Week 3 (Days 11–15)
├── Phase 7: Eval Harness & Feedback Loop   [██░░░] Days 12–13
├── Phase 8: Monitoring & Pipeline Integ.   [░░██░] Day 14
└── Phase 9: Hardening & Documentation      [░░░░█] Day 15
```

### 17.2 Milestone Summary

| Milestone | Day | Deliverable | Verification |
|---|---|---|---|
| M1: Infrastructure running | Day 2 | Docker Compose up (5 services), DB schema deployed, health check passes | `curl /health` → 200 |
| M2: PII-safe ingestion | Day 4 | Tickets ingested with PII redaction; audit trail working | POST ticket with PII → `body_redacted` has tokens; audit log has events |
| M3: RAG retrieval working | Day 6 | KB docs ingested; hybrid search + re-ranking + tenant filter | Upload KB → query → relevant chunks retrieved; tenant isolation holds |
| M4: Classification + drafting | Day 8 | LLM classifies tickets and drafts grounded responses with citations | Submit ticket → classification stored → draft with [1] citation |
| M5: Confidence routing + cost | Day 9 | Composite confidence gates auto vs. human; cost tracked per ticket/tenant | High-confidence → auto; low → human; budget exceeded → human (no LLM) |
| M6: Human review queue | Day 11 | Streamlit dashboard: agents review, edit, approve, reject with feedback | Agent edits draft → approves → ticket resolved → feedback captured |
| M7: Eval + feedback loop | Day 13 | Eval suite runs on auto-responses; feedback grows eval dataset | Reject draft → eval dataset grows → eval run includes new example |
| M8: Monitoring dashboard | Day 14 | All operational metrics displayed; full pipeline runs end-to-end | Submit 10 tickets → dashboard shows volume, auto-rate, cost, eval, queue |
| M9: Production-ready | Day 15 | Full test suite green, CI passes, documented, deployable | Fresh `docker compose up` → seed → submit → process → dashboard → all works |

### 17.3 Post-v1 Roadmap

```
v1 (this plan)
  └── Full system: ingestion → PII → classify → RAG → draft → route → review → eval → monitor

v2 (future)
  ├── Email/IMAP integration (real email polling)
  ├── Streaming responses (SSE for real-time draft generation)
  ├── Multi-language support (German + English)
  ├── Customer-facing chat widget (not just agent-assist)
  ├── SLA management with escalation timers
  ├── Fine-tuned classification model (replace LLM for speed)
  ├── Semantic deduplication (detect duplicate tickets, link them)
  ├── Proactive suggestions (suggest KB articles to customer before they submit)
  └── Kubernetes deployment with auto-scaling

v3 (future)
  ├── Voice channel (speech-to-text → pipeline → text-to-speech)
  ├── Multi-modal tickets (screenshots, attachments)
  ├── Agent copilot (suggest responses in real-time as agent types)
  └── A/B testing response templates (from P8)
```

---

## 18. Risk Register

| Risk | Likelihood | Impact | Mitigation | From Project |
|---|---|---|---|---|
| **PII leakage to LLM** (redaction misses an entity type) | Medium | Critical | Presidio + custom recognizers + regex + test suite with diverse PII samples; fail-closed (if scan fails, route to human) | P10 |
| **PII restoration failure** (tokens in sent response) | Low | Critical | Restorer tests on every response; fallback: if restoration fails, don't send, route to human; test suite verifies no tokens in sent responses | P10 |
| **LLM hallucination in auto-response** (confident but wrong) | Medium | High | Confidence threshold (0.75); hard rules for critical/angry/complaint; eval suite catches faithfulness issues; audit trail for every auto-response | P7 |
| **Cross-tenant data leakage** | Low | Critical | SQL pre-filter + RLS + test suite with explicit leakage tests (from P2 pattern) | P2 |
| **Cost overrun** (tenant exceeds budget significantly) | Medium | Medium | Budget checked before draft LLM call; per-tenant monthly cap; dashboard shows spend vs budget; alert at 80% | P9 |
| **RAG retrieval quality degrades** (KB grows, irrelevant chunks surface) | Medium | Medium | Cross-encoder re-ranking; periodic eval of retrieval quality; KB doc deactivation for outdated content | P1 |
| **Review queue backlog grows** (more tickets than agents can review) | Medium | Medium | Dashboard shows queue depth trend; supervisor can re-prioritize; consider lowering auto-response threshold if eval scores are high | — |
| **Eval dataset poisoning** (bad feedback examples degrade eval) | Low | Medium | Dataset versioned (can roll back); supervisor can flag invalid feedback; eval scores tracked over time to detect anomalies | P7 |
| **LLM API downtime** (OpenAI unavailable) | Low | High | Retry with backoff; fallback to Ollama (local Llama 3.1); if both fail, route all tickets to human review | P9 |
| **Presidio model limitations** (misses non-standard PII formats) | Medium | Medium | Custom recognizers for domain-specific PII (IBAN, German phone); regex patterns as backstop; periodic review of missed entities | P10 |
| **Classification inaccuracy** (wrong urgency/category) | Medium | Medium | Pydantic validation; classification confidence in composite score; wrong_category feedback tag → prompt improvement | P5 |
| **Postgres performance at scale** (>100k tickets, >50k KB chunks) | Medium | Low | HNSW index for vector search; partition tickets by month; archive old tickets; consider Qdrant for very large KB | P1 |
| **Streamlit dashboard performance** (slow with large datasets) | Medium | Low | Metrics endpoint pre-aggregates data; Streamlit caches with `@st.cache_data`; limit table display to recent N rows | — |
| **Audit log grows unbounded** | High | Low | Partition by month; archive old partitions; retention policy (2 years active, then cold storage) | P2 |
| **Prompt injection via ticket body** | Low | High | Customer input only in "ticket body" field; system prompt immutable; output validation for injection patterns; route suspicious tickets to human | P2 |

---

## Appendix A: Quick Start

```bash
# 1. Clone
git clone <repo-url> support-ai
cd support-ai

# 2. Configure
cp .env.example .env
# Edit .env: set OPENAI_API_KEY, JWT_SECRET, POSTGRES_PASSWORD, PII_ENCRYPTION_KEY

# 3. Start all services
docker compose up -d

# 4. Wait for health
curl http://localhost:8000/health
# → {"status": "healthy", "database": "connected", "redis": "connected", ...}

# 5. Initialize database
docker compose exec api python scripts/init_db.py

# 6. Seed test data (tenants, agents, KB docs)
docker compose exec api python scripts/seed_test_data.py
docker compose exec api python scripts/seed_kb.py

# 7. Generate an agent JWT
python scripts/generate_agent_token.py --tenant acme --agent agent-001 --role agent

# 8. Upload a KB document
curl -X POST http://localhost:8000/kb/upload \
  -H "Authorization: Bearer <TOKEN>" \
  -F "file=@./faq.pdf" \
  -F "source_type=faq" \
  -F "allowed_roles=agent,supervisor"

# 9. Submit a ticket
curl -X POST http://localhost:8000/tickets \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "subject": "Cannot reset password",
    "body": "Hi, I am Jane Doe (jane@example.com). I cannot reset my password.",
    "customer_email": "jane@example.com",
    "customer_name": "Jane Doe"
  }'

# 10. Check ticket status
curl http://localhost:8000/tickets/<ticket_id> \
  -H "Authorization: Bearer <TOKEN>"

# 11. Open review queue dashboard (if ticket was routed to human)
open http://localhost:8501

# 12. Open monitoring dashboard
open http://localhost:8502

# 13. Run eval suite
curl -X POST http://localhost:8000/eval/run \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"tenant_id": "acme", "dataset_version": "latest"}'
```

---

## Appendix B: Key Design Decisions

| Decision | Chosen | Rejected Alternative | Why |
|---|---|---|---|
| PII strategy | Redact before LLM, restore after | LLM with PII + post-filter | Redaction-before-LLM guarantees no PII reaches external API; restoration is a simple string replace; fail-closed if either step fails (from P10) |
| Vector DB | pgvector (Postgres) | Pinecone, Qdrant | SQL-level tenant filtering (from P2); single DB for tickets + KB + audit + cost; modest KB size doesn't need dedicated vector DB |
| Retrieval | Hybrid BM25 + vector + cross-encoder re-rank | Vector-only | Hybrid search + re-ranking significantly improves precision (from P1); BM25 catches keyword matches that embeddings miss |
| Classification | Single LLM call for urgency + category + sentiment | Three separate calls | One call is 3× cheaper and faster; GPT-4o-mini handles multi-field structured output well; single call ensures consistency between fields |
| Confidence | Composite (retrieval + classification + self-assess + coverage) | Single signal (retrieval distance only) | Composite score is more robust; no single signal is reliable alone; weights are configurable for tuning |
| Routing hard rules | Critical/angry/complaint always human | Pure confidence threshold | Some tickets should never be auto-resolved regardless of confidence; angry customers need empathy; complaints need accountability |
| Review queue UI | Streamlit | React/Vue custom frontend | Streamlit is Python-native, fast to build, good enough for agent workflow; no separate frontend build/deploy; two Streamlit apps (review + monitor) keep concerns isolated |
| Eval dataset | Versioned JSONL files, grows from feedback | Static test set only | Growing dataset tests against real failure modes; versioning allows score comparison across time; from P7 pattern |
| Feedback → eval | Auto-add rejected/edited drafts to eval dataset | Manual curation only | Automation ensures the loop is closed; agent corrections are the highest-quality eval examples; manual curation is a bottleneck |
| Cost tracking | Per-call token counting from API response | Estimated tokens | API response provides exact token counts; estimation drift would make budget enforcement unreliable (from P9) |
| Budget enforcement | Check before draft LLM call | Check after (refund) | Pre-check prevents spending beyond budget; post-check would allow overage; "refund" is complex and unreliable |
| Audit trail | DB table with triggers (append-only) | External log file / SIEM | DB table is queryable via API; triggers guarantee immutability; SIEM integration is a v2 concern (from P2) |
| LLM | GPT-4o-mini | GPT-4o, Claude, local Llama | GPT-4o-mini is 10× cheaper than GPT-4o, handles classification + drafting well; cost tracking makes the tradeoff visible; Ollama fallback for on-prem (from P9) |
| Caching | Redis for classification + semantic cache | No cache | Classification is deterministic for same text; semantic cache catches near-duplicate tickets; reduces cost and latency (from P9) |
| Pipeline execution | FastAPI background tasks | Celery/RQ worker | Background tasks are simpler for v1; sufficient for moderate volume; Celery is a v2 scaling concern |
| PII storage | Encrypted in DB with per-tenant key | Not stored (redact and discard) | Need originals for response restoration; encryption at rest with managed key; pii_entities table scoped by tenant with RLS |
| Re-ranker | cross-encoder/ms-marco-MiniLM-L-6-v2 | Cohere Rerank API | Local model is free, fast enough for v1 volume; no additional API dependency; Cohere is a v2 option for higher quality |

---

> **This capstone demonstrates the full AI engineering lifecycle:** ingestion → processing → inference → evaluation → monitoring → feedback → improvement. It combines RAG (P1, P2, P3), pipeline orchestration (P5), evaluation (P7), prompt improvement (P8), cost control (P9), PII safety (P10), and multi-agent systems (P6) into one production system. The component-to-project mapping in Section 2 makes the skill integration explicit. This is the portfolio piece that proves you can build production AI systems, not just call APIs.