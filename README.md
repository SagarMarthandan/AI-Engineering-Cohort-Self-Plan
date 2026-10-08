# AI Engineering Cohort — Self-Plan

A self-directed, project-based curriculum for transitioning from data engineering / analytics engineering into AI engineering. The current schedule covers 60 study days across 12 weeks, with an optional 15-day City Disruption Memory capstone. The original 12 project specifications across five tiers remain longer-term references.

> **Who this is for:** A data engineer / data analyst / analytics engineer who knows Python, SQL, dbt, Airflow, Docker, and data quality tooling — and wants to bridge the gap into LLM orchestration, retrieval engineering, AI evaluation, and production AI infrastructure.
>
> **Core thesis:** You already have 70% of the skills. The 30% gap is LLM orchestration, retrieval engineering, AI evaluation, and production AI infrastructure. This roadmap bridges that gap.

---
## Current study schedule

Use [60-Day_AI_Engineering_Plan.md](60-Day_AI_Engineering_Plan.md) for the current integrated schedule: **60 study days, 12 weeks, Monday–Friday, six hours per day, 360 hours total**.

Printable version: [60-Day_AI_Engineering_Plan.pdf](60-Day_AI_Engineering_Plan.pdf), with one page per study week plus source links and completion checklists.

The plan combines Ed Donner's LLM Engineering course, selected AI Engineering from Scratch lessons, and all 12 self-track project topics. It includes daily build/proof tasks, Friday checkpoints, direct lesson links, and progress tracking. Reuse components across learning builds; the original implementation plans below retain their larger scope and timelines.

The 45-day plans and previous curriculum documents remain unchanged as references.

### Extra AI + data engineering capstone

The selected additional project is [City Disruption Memory](City_Disruption_Memory_Capstone.md): an AI-operated live Chicago data platform. Deploy Spark with Docker, provision local resources through Terraform, instrument infrastructure/jobs/data health with Prometheus and Grafana, and build an evidence-grounded AI assistant for configuration proposals, incident diagnosis, and approval-gated recovery. The city-data history supplies the real workload; text extraction is secondary.

The proposed extension is **D61–D75: three additional weeks, five days/week, six hours/day, 90 hours**. The core 60-day course and its existing capstone remain intact. Taking both means 75 study days and 450 hours. The local-first specification includes all 15 daily tasks, Terraform/Compose ownership, Spark benchmarks, controlled incident evaluation, operational safety, source licensing, provenance, and completion gates. The 90 hours are a target, not a guarantee; no platform has been implemented or deployed by these documents.

Printable build specification: [City_Disruption_Memory_Capstone.pdf](City_Disruption_Memory_Capstone.pdf).

---


## Roadmap Structure

```
AI-Engineering Roadmap/
│
├── ROADMAP.md                              ← full curriculum + timeline
├── 60-Day_AI_Engineering_Plan.md            ← current daily schedule
├── 60-Day_AI_Engineering_Plan.pdf           ← printable schedule + extension overview
├── City_Disruption_Memory_Capstone.md       ← extra AI + data engineering build spec
├── City_Disruption_Memory_Capstone.pdf      ← printable extra capstone
│
├── Tier 1 - RAG & Retrieval/               ← Months 1-3
│   ├── Project 1 - Advanced RAG/           ← Hybrid search, re-ranking, query transformation
│   ├── Project 2 - Secure RAG/             ← Tenant isolation, permissions, citations, audit
│   └── Project 3 - Advanced RAG Architectures/  ← GraphRAG, Agentic RAG, Multimodal RAG
│
├── Tier 2 - LLM Orchestration & Agents/    ← Months 3-5
│   ├── Project 4 - Data Analysis Agent/    ← Tool-using agent: SQL generation, validation, execution
│   ├── Project 5 - Document Processing/    ← Classify → extract → validate → route pipeline
│   └── Project 6 - Multi-Agent Systems/    ← LangGraph, orchestration patterns, agent memory
│
├── Tier 3 - AI Evaluation & Observability/ ← Months 5-6.5
│   ├── Project 7 - LLM Evaluation Harness/ ← Faithfulness, relevance, citation accuracy
│   └── Project 8 - Prompt A-B Testing/     ← Version prompts, A/B test, track metrics
│
├── Tier 4 - AI Infrastructure & Production/← Months 6.5-8
│   ├── Project 9 - LLM Serving Gateway/    ← Caching, model routing, cost tracking, rate limiting
│   └── Project 10 - PII Redaction/         ← NER + regex + LLM detection, compliance logging
│
└── Tier 5 - Capstone/                      ← Months 8-9.5
    ├── Project 11 - Customer Support AI/   ← Full system combining all skills
    └── Project 12 - Technical Case Study/  ← Architecture writeup, metrics, lessons learned
```

Each project folder contains an `IMPLEMENTATION_PLAN.md` — a self-contained spec with architecture diagrams, database schemas, Python pseudocode, phased day-by-day plans, risk registers, and deployment configs.

---

## The 12 Projects

| # | Project | Tier | What It Proves | Key Skills |
|---|---------|------|----------------|------------|
| 1 | Advanced RAG with Hybrid Search + Re-ranking | RAG & Retrieval | Retrieval quality engineering | BM25, vector search, cross-encoder re-ranking, HyDE, chunking |
| 2 | Secure Customer RAG System | RAG & Retrieval | Enterprise-grade AI on sensitive data | Tenant isolation, RBAC, pgvector RLS, audit logs, citations |
| 3 | Advanced RAG Architectures | RAG & Retrieval | Choosing the right RAG architecture | GraphRAG (Neo4j), Agentic RAG, KAG/LightRAG, Multimodal RAG |
| 4 | Tool-Using Data Analysis Agent | Agents | Agents that do real work | Function calling, SQL generation, multi-step reasoning |
| 5 | Multi-Document Processing Pipeline | Agents | LLM-powered ETL | Classification, structured extraction, Pydantic validation |
| 6 | Multi-Agent Systems | Agents | Orchestrating collaborating agents | LangGraph, orchestrator-worker, critic loops, agent memory |
| 7 | LLM Evaluation Harness | Evaluation | Systematic AI quality | LLM-as-judge, faithfulness/relevance metrics, regression tests |
| 8 | Prompt Versioning + A/B Testing | Evaluation | Prompts as engineering artifacts | Prompt versioning, A/B testing, statistical significance |
| 9 | LLM Serving Gateway | Infrastructure | Production AI at scale | Semantic caching, model routing, cost tracking, failover |
| 10 | PII Redaction Guardrails | Infrastructure | AI safety and compliance | NER (Presidio/spaCy), regex, LLM detection, GDPR logging |
| 11 | Customer Support AI System | Capstone | Full AI product architecture | RAG + agents + multi-agent + eval + guardrails + monitoring |
| 12 | Technical Case Study | Capstone | Communicating AI tradeoffs | Architecture docs, ADRs, metrics, failure analysis |

---

## Full-Spec Reference Timeline

The timeline below belongs to the original project specifications, not the current 60-day learning-build schedule. Use the current study schedule above for daily work.

```
Month 1-3:    Tier 1 — RAG & Retrieval              (P1, P2, P3)
Month 3-5:    Tier 2 — LLM Orchestration & Agents    (P4, P5, P6)
Month 5-6.5:  Tier 3 — AI Evaluation & Observability (P7, P8)
Month 6.5-8:  Tier 4 — AI Infrastructure & Production(P9, P10)
Month 8-9.5:  Tier 5 — Capstone                      (P11, P12)
```

**Total: ~9.5 months part-time (evenings/weekends) or ~5 months full-time.**

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Language | Python 3.11+ |
| Web Framework | FastAPI |
| Vector DB | pgvector / Qdrant |
| Graph DB | Neo4j |
| LLM | GPT-4o-mini / Llama 3.1 (Ollama) |
| Orchestration | LangChain / LangGraph |
| Agent Framework | Pydantic AI / raw function calling |
| Evaluation | Custom + LangSmith (optional) |
| Caching | Redis |
| PII Detection | Microsoft Presidio + spaCy |
| Container | Docker + Docker Compose |
| Testing | pytest + pytest-asyncio |
| Database | PostgreSQL 16 |
| CI/CD | GitHub Actions |

---

## How to Use

1. Start with `60-Day_AI_Engineering_Plan.md` and follow D01–D60, including each day's proof and Friday checkpoint
2. Read the linked `IMPLEMENTATION_PLAN.md` files for design detail; the learning-build scope is defined in the current schedule
3. Save runnable examples, measured results, and project run instructions as you progress
4. After the core course, follow `City_Disruption_Memory_Capstone.md` for the optional D61–D75 extension
5. Mark an original project complete in `ROADMAP.md` only after meeting its original full-spec requirements

### Historical reference documents

These documents retain their earlier schedules and mappings; they do not override the current plan.

| Document | Reference purpose |
|---|---|
| [45-day overview](45-Day_AI_Engineering_Study_Plan.pdf) | Earlier 8.5-hour/day schedule |
| [Detailed 45-day plan](45-Day_AI_Engineering_Study_Plan_Detailed.pdf) | Earlier daily activities and resource allocations |
| [Ed Donner / From-Scratch mapping](Ed_Donners_vs_AI_Engineering_from_Scratch_Mapping.pdf) | Earlier topic-by-topic source comparison |
| [Eight-week study plan](ai_engineering_study_plan.md) | Earlier theory, exercises, and reading list |
| [Extensive interleaved plan](EXTENSIVE_PLAN.md) | Earlier course-to-project mapping |

---

## License

Personal study plan. Not affiliated with any institution.
