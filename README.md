# AI Engineering Cohort — Self-Plan

A self-directed, project-based curriculum for transitioning from data engineering / analytics engineering into AI engineering. 12 sequenced projects across 5 tiers, ~9.5 months part-time.

> **Who this is for:** A data engineer / data analyst / analytics engineer who knows Python, SQL, dbt, Airflow, Docker, and data quality tooling — and wants to bridge the gap into LLM orchestration, retrieval engineering, AI evaluation, and production AI infrastructure.
>
> **Core thesis:** You already have 70% of the skills. The 30% gap is LLM orchestration, retrieval engineering, AI evaluation, and production AI infrastructure. This roadmap bridges that gap.

---

## Roadmap Structure

```
AI-Engineering Roadmap/
│
├── ROADMAP.md                              ← full curriculum + timeline
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

## Timeline

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

1. Read `ROADMAP.md` for the full curriculum, skill mapping, and timeline
2. Start with **Tier 1, Project 1** — read its `IMPLEMENTATION_PLAN.md`
3. Follow the phases in order; run the verification step at each phase boundary
4. Build each project as a portfolio piece: README, architecture diagrams, metrics, Docker deployment
5. Mark projects complete in the progress table in `ROADMAP.md`

---

## License

Personal study plan. Not affiliated with any institution.
