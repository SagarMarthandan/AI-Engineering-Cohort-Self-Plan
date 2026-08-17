# AI Engineering Roadmap — Data Engineer/Analyst → AI Engineer

> **Who this is for:** A data engineer / data analyst / analytics engineer who knows Python, SQL, dbt, Airflow, Docker, and data quality tooling — and wants to transition into AI engineering by building a portfolio that proves production AI capability.
>
> **Core thesis:** You already have 70% of the skills. The gap is LLM orchestration, retrieval engineering, AI evaluation, and production AI infrastructure. This roadmap bridges that gap with 12 sequenced projects across 5 tiers.

---

## Skill Mapping: What You Already Have → What You'll Add

| Your Current Skill | AI Engineering Application | Acquired By |
|---|---|---|
| dbt / SQL / data modeling | Metadata stores, evaluation datasets, structured LLM output schemas | All projects |
| Airflow / Dagster orchestration | ML pipeline orchestration, scheduled eval runs | P5, P7, P11 |
| Data quality (Soda, dbt tests) | LLM evaluation, hallucination detection, regression testing | P7, P8 |
| Snowflake / Postgres | pgvector, metadata stores, audit logging | P1, P2, P3, P9 |
| Kafka / streaming | Real-time inference, streaming RAG | P9, P11 |
| Docker / CI/CD | Model serving, deployment, MLOps | All projects |
| Python | Everything | All projects |
| Analytics / dashboards | LLM observability, cost tracking, eval dashboards | P7, P8, P9, P11 |

**What you need to learn (the 30% gap):**
- LLM orchestration (prompt engineering, function calling, agents, multi-agent systems)
- Retrieval engineering (chunking, embedding, re-ranking, hybrid search, GraphRAG, agentic RAG)
- AI evaluation (LLM-as-judge, faithfulness, groundedness metrics, context engineering)
- Production AI patterns (caching, routing, guardrails, cost control)

---

## Roadmap Structure

```
AI-Engineering Roadmap/
│
├── ROADMAP.md                              ← you are here
│
├── Tier 1 - RAG & Retrieval/               ← Months 1-3
│   ├── Project 1 - Advanced RAG/           ← Hybrid search, re-ranking, query transformation
│   ├── Project 2 - Secure RAG/             ← Tenant isolation, permissions, citations, audit
│   └── Project 3 - Advanced RAG Architectures/  ← GraphRAG, Agentic RAG, Multimodal RAG, decision framework
│
├── Tier 2 - LLM Orchestration & Agents/    ← Months 3-5
│   ├── Project 4 - Data Analysis Agent/    ← Tool-using agent: SQL generation, validation, execution
│   ├── Project 5 - Document Processing/    ← Classify → extract → validate → route pipeline
│   └── Project 6 - Multi-Agent Systems/    ← LangGraph, orchestration patterns, Pydantic AI, agent memory
│
├── Tier 3 - AI Evaluation & Observability/ ← Months 5-6.5
│   ├── Project 7 - LLM Evaluation Harness/ ← Faithfulness, relevance, citation accuracy, context engineering
│   └── Project 8 - Prompt A-B Testing/     ← Version prompts, A/B test, track metrics over time
│
├── Tier 4 - AI Infrastructure & Production/← Months 6.5-8
│   ├── Project 9 - LLM Serving Gateway/    ← Caching, model routing, cost tracking, rate limiting
│   └── Project 10 - PII Redaction/         ← NER + regex + LLM detection, compliance logging
│
└── Tier 5 - Capstone/                      ← Months 8-9.5
    ├── Project 11 - Customer Support AI/   ← Full system combining all skills
    └── Project 12 - Technical Case Study/  ← Architecture writeup, metrics, lessons learned
```

---

## The 12 Projects at a Glance

| # | Project | Tier | What It Proves | Key Skills |
|---|---------|------|----------------|------------|
| 1 | Advanced RAG with Hybrid Search + Re-ranking | RAG & Retrieval | Retrieval quality engineering — the hard part of RAG | BM25, vector search, cross-encoder re-ranking, HyDE, chunking strategies |
| 2 | Secure Customer RAG System | RAG & Retrieval | Enterprise-grade AI on sensitive data | Tenant isolation, RBAC, pgvector RLS, audit logs, citations |
| 3 | Advanced RAG Architectures | RAG & Retrieval | Choosing and building the right RAG architecture for the problem | GraphRAG (Neo4j), Agentic RAG, KAG/LightRAG, Multimodal RAG, decision framework |
| 4 | Tool-Using Data Analysis Agent | Agents | Agents that do real work, not just chat | Function calling, SQL generation, multi-step reasoning, error recovery |
| 5 | Multi-Document Processing Pipeline | Agents | LLM-powered ETL replacing brittle rule-based pipelines | Classification, structured extraction, Pydantic validation, Airflow |
| 6 | Multi-Agent Systems | Agents | Orchestrating specialized agents that collaborate | LangGraph, orchestrator-worker, routing, critic loops, Pydantic AI, agent memory |
| 7 | LLM Evaluation Harness | Evaluation | Systematic AI quality — "it works in demo" isn't enough | LLM-as-judge, faithfulness/relevance metrics, regression tests, context engineering |
| 8 | Prompt Versioning + A/B Testing | Evaluation | Prompts as engineering artifacts, not magic | Prompt versioning, A/B testing, statistical significance, metric tracking |
| 9 | LLM Serving Gateway | Infrastructure | Production AI at scale — cost, latency, reliability | Semantic caching, model routing, cost tracking, rate limiting, provider failover |
| 10 | PII Redaction Guardrails | Infrastructure | AI safety and compliance | NER (Presidio/spaCy), regex patterns, LLM-based detection, GDPR logging |
| 11 | Customer Support AI System | Capstone | Full AI product architecture | Combines RAG + agents + multi-agent + eval + guardrails + monitoring |
| 12 | Technical Case Study | Capstone | Communication of AI tradeoffs to stakeholders | Architecture docs, ADRs, metrics, failure analysis, decision records |

---

## Timeline

```
Month 1-3:    Tier 1 — RAG & Retrieval
              ├── P1: Advanced RAG                (2 weeks)
              ├── P2: Secure RAG                  (2 weeks)
              └── P3: Advanced RAG Architectures  (2 weeks)

Month 3-5:    Tier 2 — LLM Orchestration & Agents
              ├── P4: Data Analysis Agent         (2 weeks)
              ├── P5: Document Pipeline           (2 weeks)
              └── P6: Multi-Agent Systems         (2 weeks)

Month 5-6.5:  Tier 3 — AI Evaluation & Observability
              ├── P7: Eval Harness + Context Eng  (2.5 weeks)
              └── P8: Prompt A/B Testing          (1.5 weeks)

Month 6.5-8:  Tier 4 — AI Infrastructure & Production
              ├── P9: LLM Gateway                 (2 weeks)
              └── P10: PII Guardrails             (1.5 weeks)

Month 8-9.5:  Tier 5 — Capstone
              ├── P11: Support AI System          (3 weeks)
              └── P12: Case Study                 (1 week)
```

**Total: ~9.5 months part-time (evenings/weekends) or ~5 months full-time.**

---

## How to Use This Roadmap

### Rules of Engagement

1. **Build in order.** Each tier builds on the previous. RAG fundamentals before advanced RAG architectures; single agents before multi-agent; agents before evaluation; evaluation before infrastructure; everything before the capstone.

2. **Each project is a portfolio piece.** Don't just make it work — write a README, include architecture diagrams, deploy it (even locally via Docker), and document what you learned.

3. **Code quality matters.** You're a data engineer — show that you write clean, tested, well-structured code. pytest, type hints, docstrings, Docker. This is your unfair advantage over bootcamp AI engineers.

4. **Document metrics.** Every project should have a metrics section: retrieval accuracy, latency, cost per query, eval scores. Your analytics background makes this natural — use it.

5. **Git history is part of the portfolio.** Commit frequently with clear messages. A repo with 50 thoughtful commits tells a better story than one with 3 giant commits.

6. **Don't skip the evaluation tier.** Projects 7 and 8 are what separate "I played with the OpenAI API" from "I can deploy AI in production." If you skip evaluation, you're not an AI engineer — you're an API caller.

### Per-Project Workflow

```
1. Read the IMPLEMENTATION_PLAN.md for the project
2. Create a new Git repo for the project
3. Follow the phases in order
4. Run the verification step at each phase boundary
5. Write a README.md when done
6. Update this ROADMAP.md with ✅ when complete
```

### Progress Tracking

Mark projects as complete by editing this file:

| # | Project | Status | Repo URL | Completed |
|---|---------|--------|----------|-----------|
| 1 | Advanced RAG | ⬜ Not Started | | |
| 2 | Secure RAG | ⬜ Not Started | | |
| 3 | Advanced RAG Architectures | ⬜ Not Started | | |
| 4 | Data Analysis Agent | ⬜ Not Started | | |
| 5 | Document Processing Pipeline | ⬜ Not Started | | |
| 6 | Multi-Agent Systems | ⬜ Not Started | | |
| 7 | LLM Evaluation Harness | ⬜ Not Started | | |
| 8 | Prompt A/B Testing | ⬜ Not Started | | |
| 9 | LLM Serving Gateway | ⬜ Not Started | | |
| 10 | PII Redaction Guardrails | ⬜ Not Started | | |
| 11 | Customer Support AI | ⬜ Not Started | | |
| 12 | Technical Case Study | ⬜ Not Started | | |

---

## Companion Learning: Ed Donner's AI Engineer Core Track

This roadmap is self-directed and production-focused. To build foundational LLM skills with guided instruction, run **Ed Donner's "AI Engineer Core Track: LLM Engineering, RAG, QLoRA, Agents"** (Udemy, 33h, 4.7★, 327K students) alongside or before this roadmap.

### What Ed Donner covers that this roadmap does not

| Topic | Ed Donner | This Roadmap |
|-------|-----------|-------------|
| QLoRA fine-tuning | ✅ Full capstone (open-source model competing with frontier) | ❌ Not covered |
| Local/open-source models (Ollama, Gemma, Phi-3, GPT-OSS) | ✅ From Day 1 | ⚠️ Ollama as fallback only |
| Guided video instruction (33h) | ✅ | ❌ Self-directed |
| Community support (327K students, Q&A) | ✅ | ❌ |
| Gradio UIs + Modal.com cloud deployment | ✅ | ❌ (FastAPI + Docker instead) |
| Multimodal: audio → meeting minutes | ✅ | ⚠️ Multimodal RAG in P3 (images/tables, not audio) |
| Python → C++ code optimization | ✅ | ❌ |

### Recommended interleaving

| Ed Donner weeks | Do this roadmap tier after |
|----------------|---------------------------|
| Weeks 1-2 (fundamentals, local models, API calls) | — (just learn) |
| Weeks 3-5 (RAG, embeddings, knowledge worker) | Tier 1 (P1-P3) |
| Weeks 6-7 (agents, function calling) | Tier 2 (P4-P6) |
| Week 8 (fine-tuning, QLoRA, multi-agent capstone) | — (learn QLoRA, then do Tier 3+4) |
| After Ed Donner | Tier 3 + Tier 4 + Tier 5 (P7-P12) |

> **Alternative**: Run Ed Donner completely first (8 weeks), then this roadmap (9.5 months). Cleaner separation but longer total timeline.

---

## Why These Projects (Not the SaaS List)

The original project list (multi-tenant SaaS, SSO, webhooks, billing) was **SaaS platform engineering** — 10 of 12 projects had zero AI content. Building SSO and webhook engines makes you look like a backend engineer, not an AI engineer.

This roadmap is curated specifically for the **data → AI transition**:

- **Every project has AI/LLM content** — no filler
- **Leverages your existing skills** — data pipelines, quality testing, SQL, orchestration
- **Fills the actual gap** — LLM orchestration, retrieval (including GraphRAG and agentic RAG), multi-agent systems, evaluation, context engineering, production serving
- **Sequenced for learning** — each project builds on the last
- **Enterprise-relevant for Germany** — GDPR compliance, audit trails, data privacy

---

## Core Advice

1. **Your data background is your unfair advantage.** Most AI engineers are weak on data pipelines, quality testing, and observability. You're strong on all three. Lean into it.

2. **Evaluation is the differentiator.** Everyone can call the OpenAI API. Almost nobody builds proper evaluation harnesses. If your portfolio shows systematic eval, you stand out immediately.

3. **Show the full loop.** For each project: ingestion → processing → inference → evaluation → monitoring. That's the AI engineering lifecycle.

4. **Depth beats breadth.** 4 strong, complete, deployed projects with metrics and documentation beat 12 half-finished ones.

5. **For the German market:** Enterprise AI roles care heavily about GDPR, auditability, and reliability. Projects 2, 7, and 10 will resonate strongly with German enterprise employers.

6. **Don't build SaaS plumbing.** SSO, webhooks, billing — those don't prove AI capability. Every hour on SSO is an hour not spent on retrieval quality or agent design.

7. **Architecture decisions matter.** Project 3's decision framework and Project 12's ADRs teach you to justify *why* you chose GraphRAG over vanilla RAG, or multi-agent over single-agent. This is what senior AI engineers do.

---

## Tech Stack Summary

| Component | Technology | Used In |
|-----------|-----------|---------|
| Language | Python 3.11+ | All |
| Web Framework | FastAPI | P1, P2, P3, P4, P5, P6, P9, P10, P11 |
| Vector DB | pgvector / Qdrant | P1, P2, P3, P11 |
| Graph DB | Neo4j | P3, P11 |
| Embeddings | OpenAI text-embedding-3-small / BGE | P1, P2, P3, P5, P11 |
| LLM | GPT-4o-mini / Llama 3.1 (Ollama) | All |
| Orchestration | LangChain / LangGraph | P1, P2, P3, P4, P5, P6, P11 |
| Agent Framework | Pydantic AI / raw function calling | P4, P5, P6, P11 |
| Evaluation | Custom + LangSmith (optional) | P7, P8, P11 |
| Caching | Redis | P9, P11 |
| PII Detection | Microsoft Presidio + spaCy | P10, P11 |
| Container | Docker + Docker Compose | All |
| Testing | pytest + pytest-asyncio | All |
| Dashboards | Streamlit | P7, P8, P9, P11 |
| Database | PostgreSQL 16 | P2, P3, P4, P5, P7, P8, P9, P11 |
| CI/CD | GitHub Actions | All |

---

## Next Steps

1. Start with **Tier 1, Project 1: Advanced RAG**
2. Read its `IMPLEMENTATION_PLAN.md`
3. Create a Git repo: `git init advanced-rag`
4. Follow the phases day by day
5. When done, mark it complete in the progress table above
6. Move to Project 2

**You're closer to AI engineering than you think. Start building.**
