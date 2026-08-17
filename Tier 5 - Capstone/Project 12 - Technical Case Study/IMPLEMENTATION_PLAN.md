# Technical Case Study & Runbook — Detailed Implementation Plan

> **Elevator pitch:** A comprehensive technical writeup of the Customer Support AI System (Project 11). This is not a code project — it's a communication artifact that demonstrates the ability to explain AI architecture decisions, tradeoffs, failures, and metrics to both technical and non-technical stakeholders. This is what you bring to interviews and attach to job applications. It is the single document that differentiates a senior AI engineer from a junior one.

---

## Table of Contents

1. [Project Goals & Non-Goals](#1-project-goals--non-goals)
2. [Architecture Overview](#2-architecture-overview)
3. [Tech Stack](#3-tech-stack)
4. [Data Schema for Metrics](#4-data-schema-for-metrics)
5. [Project Structure](#5-project-structure)
6. [Implementation Phases](#6-implementation-phases)
7. [Component Specifications](#7-component-specifications)
8. [Security & Safety Considerations](#8-security--safety-considerations)
9. [Document Interface & Consumption Guide](#9-document-interface--consumption-guide)
10. [Testing & Validation Strategy](#10-testing--validation-strategy)
11. [Deployment & Distribution](#11-deployment--distribution)
12. [Roadmap & Milestones](#12-roadmap--milestones)
13. [Risk Register](#13-risk-register)
14. [Appendix A: Quick Start](#appendix-a-quick-start)
15. [Appendix B: ADR Template](#appendix-b-adr-template)
16. [Appendix C: Metrics Presentation Format](#appendix-c-metrics-presentation-format)
17. [Appendix D: Presentation Deck Outline](#appendix-d-presentation-deck-outline)
18. [Appendix E: Operational Runbook Template](#appendix-e-operational-runbook-template)
19. [Appendix F: Key Design Decisions](#appendix-f-key-design-decisions)

---

## 1. Project Goals & Non-Goals

### Goals

| # | Goal | Success Metric |
|---|------|----------------|
| G1 | **Executive Summary** — a 1-page non-technical overview that a VP or hiring manager can read in 3 minutes and understand what was built, why it matters, and what the business value is | Readable by a non-technical stakeholder; includes key metrics and business impact in plain language |
| G2 | **Architecture Overview** — a 2–3 page technical description with system diagram, component descriptions, data flow, and technology rationale | A senior engineer can reconstruct the system's shape from this section alone |
| G3 | **Design Decisions as ADRs** — 5 Architecture Decision Records, each with context, alternatives considered, tradeoffs, and consequences | Each ADR follows the standard template; a reader can understand *why* each decision was made, not just *what* was decided |
| G4 | **Metrics & Results** — quantitative results presented as tables and described charts covering retrieval quality, answer quality, system performance, cost, and business impact | Every metric has a baseline, a target, and an actual; charts are described with enough detail to recreate |
| G5 | **Failure Modes & Incidents** — honest documentation of what broke, how it was detected, how it was fixed, and what was learned | At least 4 concrete incidents with timeline, root cause, fix, and preventive measure |
| G6 | **Lessons Learned** — reflective section on what you'd do differently, what surprised you, and advice for others | Demonstrates engineering maturity; goes beyond "it worked great" |
| G7 | **Operational Runbook** — deploy, monitor, debug, and on-call procedures | A new team member could follow the runbook to operate the system |
| G8 | **Future Work** — what's next and what you'd build with more time | Shows vision and awareness of the system's current limitations |
| G9 | **ADR Directory** — individual decision records as separate files, importable and citable | 5 ADR files in `docs/adr/` following the MADR template |
| G10 | **Metrics Dashboard Export** — chart descriptions and/or screenshots of the metrics dashboard | A `docs/metrics/` directory with chart descriptions and export instructions |
| G11 | **Presentation Deck Outline** — slide-by-slide outline for a 15-minute talk based on the case study | 15–20 slides outlined in `docs/presentation/DECK_OUTLINE.md` |
| G12 | **Interview-ready** — the case study is structured to serve as a talking document in interviews | Each section maps to common interview questions (see Appendix A) |

### Non-Goals (explicitly out of scope)

- Writing new code — this project documents Project 11; it does not modify or extend it
- Running new experiments — all metrics come from Project 11's evaluation runs; this project organizes and presents them
- Building a web frontend for the case study — the primary deliverable is a Markdown document
- Creating a published blog post — the case study is a portfolio artifact, not a marketing piece (though it can be adapted into one)
- Covering projects 1–10 — this case study focuses exclusively on the Customer Support AI System (Project 11), which is the capstone that combines all prior skills
- Producing a video presentation — the deck outline is slide-by-slide text; recording is optional and out of scope

---

## 2. Architecture Overview

This project produces a **documentation artifact**, not a software system. The "architecture" below describes the structure of the case study document and its supporting deliverables.

```
                            ┌─────────────────────────────────────────────────┐
                            │            Technical Case Study (P12)            │
                            │      "A communication artifact, not code"        │
                            └────────────────────┬────────────────────────────┘
                                                 │
              ┌──────────────────────────────────┼──────────────────────────────────┐
              │                                  │                                  │
    ┌─────────▼─────────┐          ┌────────────▼────────────┐          ┌──────────▼──────────┐
    │   CASE_STUDY.md   │          │    docs/adr/            │          │  docs/metrics/      │
    │   (primary        │          │    (ADR directory)      │          │  (dashboard export) │
    │    deliverable)   │          │                         │          │                     │
    │                   │          │  ADR-001-pgvector.md     │          │  chart-descriptions │
    │  1. Exec Summary  │          │  ADR-002-hybrid-search  │          │  dashboard-export   │
    │  2. Architecture  │◄─────────┤  ADR-003-hitl.md        │◄─────────┤  metrics-tables     │
    │  3. Design Dec.   │  cites   │  ADR-004-pii-dual.md    │  data    │                     │
    │  4. Metrics       │◄─────────┤  ADR-005-model-rout.md  │          └─────────────────────┘
    │  5. Failures      │  cites   │  ADR-000-template.md    │
    │  6. Lessons       │          └─────────────────────────┘
    │  7. Runbook       │                    │
    │  8. Future Work   │                    │ cites
    └─────────┬─────────┘                    │
              │                              │
    ┌─────────▼─────────┐          ┌────────▼────────────┐
    │  docs/presentation│          │  docs/runbook/      │
    │  /DECK_OUTLINE.md │          │  RUNBOOK.md         │
    │                   │          │                     │
    │  Slide-by-slide   │          │  Deploy, monitor,   │
    │  outline for a    │          │  debug, on-call     │
    │  15-min talk      │          │  procedures         │
    └───────────────────┘          └─────────────────────┘
```

### Document Flow (How a Reader Consumes the Case Study)

```
1. Hiring manager / VP reads Executive Summary (1 page, 3 min)
   → Decides: "interesting, let me have an engineer look"

2. Senior engineer reads Architecture Overview + Design Decisions (5-7 pages, 20 min)
   → Evaluates: "did they make sound architectural choices? do they understand tradeoffs?"

3. Data/AI engineer reads Metrics & Results + Failure Modes (4-5 pages, 15 min)
   → Evaluates: "do they measure the right things? do they understand failure?"

4. DevOps/SRE reads Operational Runbook (2 pages, 10 min)
   → Evaluates: "can they operate this in production? do they think about on-call?"

5. Interview conversation uses the case study as a talking document
   → Each section becomes a discussion prompt
   → ADRs become "tell me about a time you had to choose between X and Y" stories
```

### Content Sourcing Flow

```
Project 11 (Customer Support AI System)
  │
  ├── source code & architecture ──→ Architecture Overview section
  ├── design decisions & tradeoffs ─→ ADR-001 through ADR-005
  ├── evaluation harness results ───→ Metrics & Results section
  │   ├── retrieval eval (recall@5, MRR, nDCG)
  │   ├── answer eval (faithfulness, relevance, citation accuracy)
  │   └── system eval (latency, throughput, cost)
  ├── production logs & incidents ──→ Failure Modes & Incidents section
  ├── monitoring dashboards ────────→ Metrics Dashboard Export
  ├── deployment configs ───────────→ Operational Runbook
  └── personal notes & reflections ─→ Lessons Learned + Future Work
```

---

## 3. Tech Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **Primary format** | Markdown (GitHub-flavored) | Universal readability, renders on GitHub/GitLab/any browser, version-controllable, copy-pasteable into any CMS |
| **Diagrams** | ASCII art (in-document) + Mermaid (supplementary) | ASCII renders everywhere without plugins; Mermaid for more complex flows where ASCII becomes unreadable |
| **ADR format** | MADR (Markdown Architecture Decision Records) | Industry-standard, lightweight, structured; each ADR is a standalone file that can be cited independently |
| **Metrics tables** | GitHub-flavored Markdown tables | Render natively on GitHub; no external dependencies |
| **Chart descriptions** | Structured text specs (chart type, axes, data, interpretation) | Allows recreating charts in any tool (Excel, Python/matplotlib, Google Sheets); doesn't lock to a specific visualization tool |
| **Chart rendering** (optional) | Python + matplotlib/plotly (for actual screenshots) | If you choose to generate actual chart images, use the same Python stack from Project 11's eval harness |
| **Presentation deck** | Markdown outline (tool-agnostic) | Outline can be adapted to Google Slides, PowerPoint, Keynote, reveal.js, or Slidev |
| **Dashboard export** | Streamlit screenshots (from P11) or Grafana export PNGs | Reuses the observability dashboards already built in Project 11 |
| **PDF export** | `pandoc` or browser "Print to PDF" | For attaching to job applications where Markdown isn't accepted |
| **Version control** | Git | The case study is a versioned artifact; commit history shows iteration and care |
| **Spell/grammar check** | `cspell` + manual review | Professional quality; a typo in a case study undermines credibility |

### Why Markdown over Google Docs / Notion / Confluence?

1. **Version control** — every revision is a Git commit; the diff *is* the review trail
2. **Portability** — renders anywhere; no "you need access to this Notion workspace" friction
3. **Interview-friendly** — can be shared as a GitHub link, which also showcases your Git hygiene
4. **ADR ecosystem** — MADR and most ADR tools are Markdown-native
5. **Adaptability** — `pandoc` can convert to PDF, DOCX, HTML, or slides in one command

---

## 4. Data Schema for Metrics

This project does not create a database, but it organizes metrics data into structured tables. The schema below defines the shape of metrics data that the case study presents.

### 4.1 Retrieval Quality Metrics

```
retrieval_metrics
├── strategy:           text   -- 'vector_only', 'bm25_only', 'hybrid', 'hybrid_reranked'
├── query_set:          text   -- 'faq', 'policy', 'troubleshooting', 'mixed'
├── recall_at_5:        float  -- fraction of relevant docs in top-5
├── recall_at_10:       float
├── mrr:                float  -- mean reciprocal rank
├── ndcg_at_5:          float  -- normalized discounted cumulative gain
├── ndcg_at_10:         float
├── latency_ms_p50:     int
├── latency_ms_p90:     int
└── notes:              text   -- e.g. "hybrid_reranked best overall but 2x latency"
```

### 4.2 Answer Quality Metrics

```
answer_metrics
├── model:              text   -- 'gpt-4o-mini', 'gpt-4o', 'llama-3.1-8b'
├── prompt_version:     text   -- 'v1.0', 'v1.1', 'v2.0'
├── faithfulness_score: float  -- 0.0-1.0, LLM-as-judge
├── relevance_score:    float  -- 0.0-1.0, LLM-as-judge
├── citation_accuracy:  float  -- fraction of citations that are correct
├── citation_coverage:  float  -- fraction of claims that have citations
├── human_eval_score:   float  -- 1-5 scale, manual sample
└── sample_size:        int    -- number of evaluated responses
```

### 4.3 System Performance Metrics

```
system_metrics
├── endpoint:           text   -- '/query', '/classify', '/escalate'
├── latency_p50_ms:     int
├── latency_p90_ms:     int
├── latency_p99_ms:     int
├── throughput_rps:     float  -- requests per second sustained
├── error_rate:         float  -- 0.0-1.0
├── cache_hit_rate:     float  -- 0.0-1.0
└── period:             text   -- '2025-W12', '2025-03', etc.
```

### 4.4 Cost Metrics

```
cost_metrics
├── period:             text
├── total_tokens:       int
├── input_tokens:       int
├── output_tokens:      int
├── llm_spend_usd:      float
├── embedding_spend_usd: float
├── cost_per_ticket:    float  -- total_spend / tickets_resolved
├── cache_savings_usd:  float  -- estimated savings from cache hits
└── model_breakdown:    json   -- {model: spend} for routing analysis
```

### 4.5 Business Impact Metrics

```
business_metrics
├── period:                  text
├── total_tickets:           int
├── auto_resolved:           int   -- resolved without human agent
├── auto_resolution_rate:    float -- auto_resolved / total_tickets
├── avg_time_to_resolution:  float -- hours, all tickets
├── avg_auto_resolution_time: float -- hours, auto-resolved only
├── human_agent_hours_saved: float -- estimated hours saved
├── csat_score:              float -- customer satisfaction 1-5
└── escalation_rate:         float -- fraction escalated to human
```

---

## 5. Project Structure

```
project-10-technical-case-study/
│
├── CASE_STUDY.md                          ← PRIMARY DELIVERABLE (15-20 pages)
│
├── docs/
│   ├── adr/
│   │   ├── ADR-000-template.md            ← MADR template (for reference)
│   │   ├── ADR-001-pgvector-over-qdrant.md
│   │   ├── ADR-002-hybrid-search-over-pure-vector.md
│   │   ├── ADR-003-human-in-the-loop-low-confidence.md
│   │   ├── ADR-004-presidio-llm-dual-pii-detection.md
│   │   └── ADR-005-gpt-4o-mini-with-4o-escalation.md
│   │
│   ├── metrics/
│   │   ├── CHART_DESCRIPTIONS.md          ← Structured descriptions of every chart
│   │   ├── retrieval-quality.md           ← Recall@5, MRR, nDCG tables + chart specs
│   │   ├── answer-quality.md              ← Faithfulness, relevance, citation tables
│   │   ├── system-performance.md          ← Latency percentiles, throughput tables
│   │   ├── cost-analysis.md               ← Cost per ticket, spend breakdown tables
│   │   ├── business-impact.md             ← Auto-resolution, time saved tables
│   │   └── exports/                       ← (optional) PNG screenshots of dashboards
│   │       ├── retrieval-comparison.png
│   │       ├── latency-percentiles.png
│   │       ├── cost-trend.png
│   │       └── resolution-funnel.png
│   │
│   ├── presentation/
│   │   └── DECK_OUTLINE.md                ← Slide-by-slide outline for 15-min talk
│   │
│   └── runbook/
│       └── RUNBOOK.md                     ← Operational runbook (also embedded in CASE_STUDY.md)
│
├── README.md                              ← Project overview + how to read the case study
├── agent.md                      # Herdr multi-agent orchestration guide
├── .gitignore
└── LICENSE                                ← MIT (portfolio artifact)
```

### File Responsibilities

| File | Purpose | Audience | Length |
|------|---------|----------|--------|
| `CASE_STUDY.md` | The comprehensive writeup — all 8 sections in one document | All stakeholders | 15–20 pages |
| `docs/adr/ADR-00X-*.md` | Individual decision records, citable and standalone | Senior engineers, architects | 1–2 pages each |
| `docs/metrics/CHART_DESCRIPTIONS.md` | Structured specs for every chart (type, axes, data, interpretation) | Anyone recreating the charts | 3–4 pages |
| `docs/metrics/*.md` | Per-category metric tables and chart specs | Data-literate readers | 1–2 pages each |
| `docs/presentation/DECK_OUTLINE.md` | Slide-by-slide outline for a 15-minute presentation | Speaker / audience | 5–6 pages |
| `docs/runbook/RUNBOOK.md` | Operational procedures (also embedded in CASE_STUDY.md §7) | SRE / on-call engineers | 2 pages |
| `README.md` | How to navigate the case study; reading guide | First-time visitors | 1 page |

---

## 6. Implementation Phases

> **Timeline:** 5 working days (1 week). This is a writing project — the "implementation" is drafting, revising, and polishing documentation. Each phase has a clear deliverable and a verification step (self-review checklist).

---

### Phase 1 (Day 1): Sourcing, Structure & Executive Summary

**Objective:** Gather all raw material from Project 11, establish the document skeleton, and write the Executive Summary.

#### 1.1 Source Material Audit (Morning)

Collect from Project 11:

| Source | What to Extract | Target Section |
|--------|----------------|----------------|
| Project 11 README.md | System overview, tech stack, architecture | §2 Architecture Overview |
| Project 11 source code | Component structure, key classes, data flow | §2 Architecture Overview |
| Project 11 design notes / commit messages | Decision context, alternatives considered | §3 Design Decisions (ADRs) |
| Project 11 evaluation results (JSON/CSV) | Retrieval metrics, answer quality metrics | §4 Metrics & Results |
| Project 11 monitoring dashboards (Streamlit/Grafana) | Latency, throughput, cost, cache hit rate | §4 Metrics & Results |
| Project 11 incident log / debug notes | What broke, detection, fix, lessons | §5 Failure Modes & Incidents |
| Project 11 deployment configs (docker-compose, .env.example) | Deployment steps, env vars | §7 Operational Runbook |
| Project 11 test outputs | Test coverage, critical test descriptions | §4 Metrics & Results (quality) |
| Personal engineering journal / notes | Reflections, surprises, advice | §6 Lessons Learned |

**Deliverable:** A `sources.md` scratch file (not in final deliverable) mapping each source to its target section.

**Verification:**
- [ ] Every section (§1–§8) has at least one identified source
- [ ] No section is "empty" (i.e., you know where its content comes from)
- [ ] Metrics data is available in structured form (tables or JSON), not just prose

#### 1.2 Document Skeleton (Afternoon)

Create `CASE_STUDY.md` with:
- Title, subtitle, date, author
- Table of contents
- All 8 section headers with one-sentence placeholders
- Page budget annotations (e.g., `<!-- ~1 page -->`)

Create the supporting directory structure:
```
docs/adr/
docs/metrics/
docs/presentation/
docs/runbook/
```

**Deliverable:** `CASE_STUDY.md` skeleton + directory structure.

**Verification:**
- [ ] All 8 sections present with headers
- [ ] Table of contents links work
- [ ] Directory structure matches §5 Project Structure
- [ ] Page budget per section matches the spec (1+2-3+3-4+2-3+2+1-2+2+1 = 16-20 pages)

#### 1.3 Executive Summary (Late Afternoon)

Write §1 Executive Summary (~1 page). This is the hardest section to write well because it must be:
- **Non-technical** — a VP or hiring manager should understand it
- **Concrete** — specific numbers, not "improved efficiency"
- **Business-framed** — value to the organization, not technical achievement

Structure:
```
## 1. Executive Summary

### What We Built
[1 paragraph: the Customer Support AI System in plain language]

### Why It Matters
[1 paragraph: the business problem — support ticket volume, response time, cost]

### Key Results
[Bullet list with 5-6 headline metrics:]
- Auto-resolution rate: X% of tickets resolved without human intervention
- Average resolution time reduced from X hours to Y minutes
- Cost per auto-resolved ticket: $Z (vs $W for human agent)
- Faithfulness score: X/1.0 (answers grounded in source documents)
- PII detection accuracy: X% (dual-layer Presidio + LLM)
- Monthly LLM spend: $X (within budget, 40% saved via caching + routing)

### Business Value
[1 paragraph: projected annual savings, scalability, customer satisfaction impact]
```

**Deliverable:** Completed §1 Executive Summary.

**Verification:**
- [ ] Readable by a non-technical person (no jargon without explanation)
- [ ] Contains at least 5 specific quantitative metrics
- [ ] Fits on 1 page (when rendered)
- [ ] Answers: what, why, so what
- [ ] No acronyms without first-use expansion

---

### Phase 2 (Day 2): Architecture Overview & ADRs

**Objective:** Write the Architecture Overview (§2) and all 5 ADRs (§3 + `docs/adr/`).

#### 2.1 Architecture Overview (Morning)

Write §2 Architecture Overview (~2–3 pages). Include:

1. **System diagram** — a comprehensive ASCII diagram showing:
   - All components (FastAPI, pgvector, Redis, LLM gateway, PII guardrails, eval harness, monitoring)
   - Data flow (ingestion → query → retrieval → generation → post-processing → response)
   - External dependencies (OpenAI API, Presidio, spaCy NER)
   - Human-in-the-loop escalation path

2. **Component descriptions** — 1 paragraph per component:
   - Ingestion pipeline (document parsing, chunking, embedding, storage)
   - Query pipeline (query embedding, hybrid retrieval, re-ranking, context assembly)
   - Answer generation (LLM call, citation extraction, faithfulness check)
   - PII guardrails (input redaction, output scanning, audit logging)
   - LLM gateway (semantic cache, model routing, cost tracking, rate limiting)
   - Evaluation harness (LLM-as-judge, regression tests, CI integration)
   - Monitoring (Streamlit dashboard, alerting, log aggregation)
   - Human-in-the-loop escalation (confidence threshold, agent UI handoff)

3. **Data flow** — step-by-step request walkthrough:
   - Query arrives → PII scan → cache check → embedding → hybrid retrieval → re-ranking → context assembly → LLM call → citation validation → output PII scan → confidence check → respond or escalate → audit log

4. **Technology choices with rationale** — a table summarizing each technology choice and why it was selected over alternatives (this is a summary; the deep dives are in the ADRs)

**Deliverable:** Completed §2 Architecture Overview.

**Verification:**
- [ ] System diagram is readable in a monospace font (test in GitHub render)
- [ ] Every component in the diagram has a corresponding description
- [ ] Data flow walkthrough covers the full request lifecycle
- [ ] A senior engineer could sketch the system from this section alone
- [ ] Technology rationale table references the relevant ADR for each choice

#### 2.2 ADR Template & ADR-001 (Afternoon)

Create `docs/adr/ADR-000-template.md` using the MADR format (see Appendix B).

Write `docs/adr/ADR-001-pgvector-over-qdrant.md`:

```
# ADR-001: Use pgvector for Vector Search Instead of Qdrant

## Status
Accepted

## Context
[The Customer Support AI System needs vector search over support documents.
The system also requires per-tenant permission filtering — users can only
retrieve documents their role permits. This is a hard security requirement,
not a nice-to-have.]

## Decision
[Use pgvector (PostgreSQL extension) for vector storage and search.]

## Alternatives Considered

### Qdrant
- Pros: Purpose-built vector DB, HNSW index tuning, higher throughput at scale
- Cons: Separate metadata store needed for permissions; sync problem between
  Qdrant payloads and PostgreSQL; no SQL-level JOIN for permission filtering;
  additional operational burden (another service to run, monitor, back up)

### Pinecone (managed)
- Pros: Zero ops, auto-scaling
- Cons: Vendor lock-in, no SQL filtering, data leaves our infrastructure
  (GDPR concern for German enterprise), cost scales with usage

### Weaviate
- Pros: Built-in hybrid search, GraphQL API
- Cons: Same metadata sync problem as Qdrant; less mature ecosystem

## Consequences
Positive:
- Permission filtering and vector search in one atomic SQL query — no race conditions
- Single database to operate, back up, and monitor
- Row-Level Security as defense-in-depth for tenant isolation
- Leverages existing PostgreSQL expertise (data engineer's home turf)

Negative:
- pgvector HNSW is less tunable than Qdrant's
- At >10M vectors, dedicated vector DBs outperform pgvector
- No built-in hybrid search (must implement BM25 separately via pg_trgm/tsvector)

Neutral:
- Migration path to Qdrant exists if scale demands it (embeddings are portable)
```

**Deliverable:** ADR template + ADR-001.

**Verification:**
- [ ] ADR follows the MADR template exactly
- [ ] Context explains *why the decision was needed*, not just what was decided
- [ ] At least 2 alternatives considered with pros/cons
- [ ] Consequences include both positive and negative
- [ ] ADR is self-contained (readable without the case study)

#### 2.3 ADR-002 through ADR-005 (Late Afternoon → Evening)

Write the remaining 4 ADRs:

**ADR-002: Hybrid Search (BM25 + Vector) over Pure Vector Search**
- Context: Support docs have exact-match terminology (error codes, product names, policy numbers) where keyword search outperforms semantic search
- Alternatives: pure vector, pure BM25, hybrid without re-ranking
- Decision: BM25 + vector fusion (RRF) + cross-encoder re-ranking
- Consequences: +12% recall@5 vs pure vector; 1.8x retrieval latency; requires maintaining two indexes

**ADR-003: Human-in-the-Loop for Low-Confidence Responses**
- Context: LLM confidence is not always calibrated; wrong answers in support context erode trust faster than "I don't know"
- Alternatives: always auto-respond, always human-review, threshold-based escalation
- Decision: Confidence threshold (0.75) → below threshold, route to human agent with retrieved context as draft
- Consequences: reduces wrong-answer risk; adds latency for escalated tickets; requires agent UI; trust calibration is an ongoing tuning task

**ADR-004: Presidio + LLM Dual-Layer PII Detection**
- Context: Support tickets contain PII (names, emails, phone numbers, order IDs, addresses); GDPR requires detection and redaction
- Alternatives: Presidio only, LLM only, regex only
- Decision: Layer 1 = Presidio (NER + regex, fast, deterministic) → Layer 2 = LLM scan (catches context-dependent PII that NER misses)
- Consequences: defense in depth; +40ms latency per query; near-zero PII leakage; LLM layer has small false-positive rate (acceptable — over-redaction is safer than leakage)

**ADR-005: GPT-4o-mini as Primary Model with GPT-4o Escalation**
- Context: Cost optimization — most support questions are routine and don't need GPT-4o-level reasoning; complex tickets benefit from it
- Alternatives: GPT-4o for all, GPT-4o-mini for all, open-source (Llama 3.1) for all
- Decision: GPT-4o-mini handles 80% of tickets (classification + simple RAG); GPT-4o handles 20% (complex reasoning, multi-step troubleshooting); routing based on ticket complexity classification
- Consequences: 65% cost reduction vs GPT-4o-only; slight quality regression on simple tickets (acceptable); routing logic adds complexity; requires monitoring routing accuracy

**Deliverable:** ADR-002 through ADR-005 in `docs/adr/`.

**Verification:**
- [ ] Each ADR follows the template
- [ ] Each ADR has at least 2 alternatives with pros/cons
- [ ] Tradeoffs are honest (not just "our choice is best")
- [ ] ADRs cross-reference each other where relevant (e.g., ADR-005 references ADR-003 for escalation logic)
- [ ] The §3 Design Decisions section in CASE_STUDY.md summarizes all 5 ADRs with links to the full files

---

### Phase 3 (Day 3): Metrics & Results + Failure Modes

**Objective:** Write §4 Metrics & Results (~2–3 pages) and §5 Failure Modes & Incidents (~2 pages).

#### 3.1 Metrics & Results (Morning → Afternoon)

Write §4 Metrics & Results. Organize into 5 subsections, each with tables and chart descriptions.

**§4.1 Retrieval Quality**

Present a comparison table across 4 strategies:

| Strategy | Recall@5 | Recall@10 | MRR | nDCG@5 | nDCG@10 | Latency p50 (ms) |
|----------|----------|-----------|-----|--------|---------|-------------------|
| Vector only | 0.68 | 0.79 | 0.54 | 0.61 | 0.68 | 45 |
| BM25 only | 0.61 | 0.72 | 0.48 | 0.55 | 0.62 | 22 |
| Hybrid (RRF) | 0.79 | 0.88 | 0.67 | 0.74 | 0.81 | 58 |
| Hybrid + re-ranking | 0.84 | 0.91 | 0.74 | 0.81 | 0.86 | 112 |

Chart descriptions:
- **Chart 1: Retrieval Strategy Comparison (grouped bar chart)**
  - X-axis: Strategy (4 bars)
  - Y-axis: Score (0.0–1.0)
  - Groups: Recall@5, MRR, nDCG@5
  - Interpretation: Hybrid + re-ranking achieves best quality but at 2.5x latency of vector-only; hybrid without re-ranking is the "sweet spot" for latency-sensitive paths

- **Chart 2: Recall@5 by Query Type (line chart)**
  - X-axis: Query type (FAQ, Policy, Troubleshooting, Mixed)
  - Y-axis: Recall@5
  - Lines: One per strategy
  - Interpretation: BM25 outperforms vector on Policy queries (exact policy numbers); vector dominates Troubleshooting (semantic similarity); hybrid wins across all types

**§4.2 Answer Quality**

| Model | Faithfulness | Relevance | Citation Accuracy | Citation Coverage | Human Eval (1-5) |
|-------|-------------|-----------|-------------------|-------------------|-------------------|
| GPT-4o-mini | 0.87 | 0.91 | 0.94 | 0.82 | 4.1 |
| GPT-4o | 0.93 | 0.95 | 0.97 | 0.89 | 4.5 |
| Llama 3.1 8B | 0.79 | 0.84 | 0.88 | 0.71 | 3.6 |

Chart descriptions:
- **Chart 3: Answer Quality Radar Chart**
  - 5 axes: Faithfulness, Relevance, Citation Accuracy, Citation Coverage, Human Eval
  - 3 overlaid polygons: one per model
  - Interpretation: GPT-4o is best but marginal improvement over mini for 3x cost; Llama 3.1 is viable for non-critical paths

**§4.3 System Performance**

| Endpoint | p50 (ms) | p90 (ms) | p99 (ms) | Throughput (rps) | Error Rate |
|----------|----------|----------|----------|-------------------|------------|
| /query (cache miss) | 820 | 1,400 | 2,800 | 12 | 0.3% |
| /query (cache hit) | 45 | 80 | 150 | 180 | 0.1% |
| /classify | 120 | 220 | 450 | 85 | 0.2% |
| /escalate | 90 | 160 | 320 | 50 | 0.0% |

Chart descriptions:
- **Chart 4: Latency Percentile Waterfall (stacked bar chart)**
  - X-axis: Endpoint
  - Stacked segments: p50, p50→p90 delta, p90→p99 delta
  - Interpretation: Cache hits are 18x faster than misses; p99 is dominated by LLM API latency (not retrieval)

- **Chart 5: Latency Breakdown (horizontal bar chart)**
  - Breakdown of /query cache-miss latency into: PII scan (40ms), embedding (60ms), retrieval (112ms), LLM call (550ms), citation validation (30ms), output PII scan (28ms)
  - Interpretation: LLM call is 67% of end-to-end latency; optimization target is model routing and caching

**§4.4 Cost**

| Metric | Value |
|--------|-------|
| Monthly LLM spend | $340 |
| Monthly embedding spend | $28 |
| Cost per auto-resolved ticket | $0.08 |
| Cost per human-resolved ticket | $4.50 |
| Cache hit rate | 34% |
| Estimated monthly cache savings | $85 |
| Estimated monthly routing savings (mini vs 4o) | $180 |
| Total estimated monthly savings vs GPT-4o-only baseline | $265 |

Chart descriptions:
- **Chart 6: Monthly Cost Trend (line chart)**
  - X-axis: Month (6 months)
  - Y-axis: USD
  - Lines: Total spend, spend without cache, spend without cache+routing (baseline)
  - Interpretation: Cost growth is sub-linear despite ticket volume growth; caching and routing flatten the curve

- **Chart 7: Cost per Ticket by Model Routing (pie chart)**
  - Segments: GPT-4o-mini (72%), GPT-4o (22%), Embeddings (4%), Other (2%)
  - Interpretation: 80/20 routing holds; mini handles the majority of volume

**§4.5 Business Impact**

| Metric | Before (human-only) | After (AI-assisted) | Change |
|--------|---------------------|---------------------|--------|
| Total monthly tickets | 4,200 | 4,200 | — |
| Auto-resolution rate | 0% | 42% | +42pp |
| Avg time to resolution (all) | 6.2 hours | 2.1 hours | -66% |
| Avg time to resolution (auto) | N/A | 12 seconds | N/A |
| Human agent hours/month | 1,250 | 725 | -42% |
| Estimated monthly savings | $0 | $5,250 | +$5,250 |
| CSAT score | 3.8/5 | 4.2/5 | +0.4 |
| Escalation rate | N/A | 18% | N/A |

Chart descriptions:
- **Chart 8: Resolution Funnel (funnel chart)**
  - Stages: Tickets received (4,200) → Auto-resolved (1,764) → Escalated with AI draft (756) → Escalated without draft (1,680) → Human-resolved (2,436)
  - Interpretation: 42% auto-resolution; AI drafts accelerate even escalated tickets

- **Chart 9: CSAT Before/After (diverging bar chart)**
  - X-axis: CSAT score (1-5)
  - Y-axis: Response count
  - Two bars per score: before (red, left) and after (green, right)
  - Interpretation: Distribution shifts right; reduction in 1-star and 2-star ratings

**Deliverable:** Completed §4 Metrics & Results + `docs/metrics/` files.

**Verification:**
- [ ] Every metric has a baseline and actual (or before/after)
- [ ] Every chart has a structured description (type, axes, data, interpretation)
- [ ] Tables are properly formatted Markdown (render in GitHub)
- [ ] Numbers are internally consistent (e.g., auto_resolution_rate × total_tickets = auto_resolved)
- [ ] Business impact section is non-technical enough for a VP to understand
- [ ] No metric is presented without context (every number has a "so what")

#### 3.2 Failure Modes & Incidents (Late Afternoon)

Write §5 Failure Modes & Incidents (~2 pages). Document at least 4 incidents using the following template per incident:

```
### Incident N: [Title]
**Date:** [approximate]
**Severity:** [P1/P2/P3]
**Detection:** [how it was discovered — monitoring alert, user report, eval run]

#### What Happened
[2-3 sentences describing the failure]

#### Impact
[Who/what was affected, for how long, how many users/tickets]

#### Root Cause
[Technical explanation of why it happened]

#### Timeline
- T+0: [event]
- T+Xm: [event]
- T+Ym: [event]
- T+Zm: [resolution]

#### Fix
[What was done to resolve the immediate issue]

#### Preventive Measure
[What was changed to prevent recurrence — code, config, process, monitoring]

#### Lesson
[One-sentence takeaway]
```

**Incident 1: LLM Hallucinated a Non-Existent Return Policy**
- Severity: P2
- Detection: Citation validation check flagged a response where the cited chunk did not contain the claimed policy
- Root cause: LLM extrapolated from training data when retrieved context was ambiguous; context contained a partial policy snippet that the LLM completed incorrectly
- Fix: Added faithfulness eval to the response pipeline; responses failing faithfulness check are re-generated with stricter context-only instructions or escalated
- Preventive: Added faithfulness eval to CI regression suite; added "if the context doesn't contain the answer, say you don't know" to system prompt
- Lesson: Citation validation is not just for user trust — it's a hallucination detector

**Incident 2: Semantic Cache Returned Wrong Answer for Similar-but-Different Question**
- Severity: P2
- Detection: User reported incorrect answer; investigation found cache hit on a question with 0.92 cosine similarity but different intent
- Root cause: Cache similarity threshold was set too high (0.90); two questions about different products with similar phrasing exceeded the threshold
- Fix: Lowered cache similarity threshold to 0.85; added intent classification as a secondary cache key dimension
- Preventive: Added cache hit quality monitoring (sample cached responses and evaluate); added cache miss/hit eval to regression suite
- Lesson: Semantic caching trades accuracy for speed; the threshold is a tunable knob that needs monitoring, not a set-and-forget value

**Incident 3: PII Leaked Through LLM Output**
- Severity: P1
- Detection: Audit log review found a response containing a customer's phone number that was not in the retrieved context but was in the user's original query (which the LLM echoed)
- Root cause: Input PII redaction was applied to the retrieved context but not to the user's query text that was included in the prompt; the LLM echoed the PII from the query
- Fix: Applied PII redaction to the user query before prompt construction, not just to retrieved context; added output PII scanning layer (Layer 2 of ADR-004)
- Preventive: Added PII leakage test to regression suite; added output scanning as a mandatory pipeline stage (not optional)
- Lesson: PII redaction must be applied at every point where user input enters the system, not just at the "obvious" points

**Incident 4: Cost Spike from Complex Troubleshooting Tickets**
- Severity: P3
- Detection: Cost dashboard showed 3x daily spend spike; investigation found a cluster of complex tickets being routed to GPT-4o with long multi-turn conversations
- Root cause: Complexity classifier was over-escalating to GPT-4o; some tickets were multi-turn conversations that accumulated token costs
- Fix: Added per-conversation token budget (4,000 tokens); conversations exceeding budget are summarized and continued with a fresh context window; tuned complexity classifier threshold to reduce false escalations
- Preventive: Added cost-per-conversation alerting (alert if >$0.50/conversation); added budget enforcement to the LLM gateway
- Lesson: Cost control is a first-class production concern; without budgets, a single complex conversation can blow the daily budget

**Deliverable:** Completed §5 Failure Modes & Incidents.

**Verification:**
- [ ] At least 4 incidents documented
- [ ] Each incident follows the template (what, impact, root cause, timeline, fix, preventive, lesson)
- [ ] Incidents are honest — no "we had no issues" (that's a red flag in a case study)
- [ ] Each incident has a concrete preventive measure (not just "we'll be more careful")
- [ ] Severity ratings are realistic (not everything is P1)
- [ ] At least one P1 incident is included (shows you've dealt with real pressure)

---

### Phase 4 (Day 4): Lessons Learned, Runbook, Future Work + Supporting Deliverables

**Objective:** Write §6 Lessons Learned, §7 Operational Runbook, §8 Future Work, and all supporting deliverables (metrics files, deck outline, runbook file).

#### 4.1 Lessons Learned (Morning)

Write §6 Lessons Learned (~1–2 pages). Structure as three subsections:

**§6.1 What I'd Do Differently**
- Started with evaluation earlier — built the system first, then retrofitted eval; eval should be built alongside the system, not after
- Chosen a dedicated vector DB from the start if I knew scale would exceed 5M vectors — pgvector was the right call at the time but the migration path is now a priority
- Built the human-in-the-loop UI earlier — the escalation path was an afterthought; it should have been a first-class component from day one
- Invested more in prompt versioning from the start — ad-hoc prompt changes made regression testing harder than it needed to be
- Made the semantic cache key more sophisticated — single embedding similarity was too coarse; intent + similarity would have prevented Incident 2

**§6.2 What Surprised Me**
- How much of production AI is plumbing — the "AI" part (retrieval + generation) was 30% of the work; the other 70% was caching, routing, guardrails, monitoring, cost control, and error handling
- How often the LLM was "confidently wrong" — faithfulness eval caught things I would never have noticed manually
- How much caching moved the cost needle — 34% cache hit rate saved more money than model routing
- How hard PII detection is — regex catches the easy stuff, but context-dependent PII (e.g., "my order number is the one from last Tuesday") requires LLM-level understanding
- How fast user expectations escalate — once auto-resolution hit 40%, the ask became "why not 60%?"

**§6.3 Advice for Others**
- Build evaluation before you build features — if you can't measure it, you can't improve it, and you definitely can't deploy it
- Treat prompts as code — version them, diff them, test them, review them
- Budget for failure — your system will hallucinate, leak, and spike costs; design for these failures, don't pretend they won't happen
- Monitor cost like you monitor latency — cost spikes are production incidents
- The case study matters more than the code — a brilliant system nobody understands has zero impact; communication is the senior engineer's multiplier

**Deliverable:** Completed §6 Lessons Learned.

**Verification:**
- [ ] Goes beyond "it worked great" — includes genuine self-critique
- [ ] "What surprised me" section shows real learning, not post-hoc rationalization
- [ ] Advice is specific and actionable, not generic platitudes
- [ ] Tone is reflective and honest, not defensive

#### 4.2 Operational Runbook (Afternoon)

Write §7 Operational Runbook (~2 pages). Also save as `docs/runbook/RUNBOOK.md`.

Structure:

**§7.1 Deployment**

```
### Prerequisites
- Docker 24+ and Docker Compose v2
- OpenAI API key (or local Ollama for Llama 3.1)
- PostgreSQL 16 with pgvector extension
- Redis 7+

### Deploy Steps
1. Clone the repository
2. Copy .env.example to .env and fill in secrets
3. Run database migrations: docker compose exec api alembic upgrade head
4. Build and start: docker compose up -d
5. Verify health: curl http://localhost:8000/health → {"status": "healthy"}
6. Run smoke test: ./scripts/smoke_test.sh
7. Verify dashboards: open http://localhost:8501 (Streamlit monitoring)

### Rollback
1. docker compose down
2. git checkout <previous-tag>
3. docker compose up -d
4. alembic downgrade -1 (if migration needs reverting)
5. Verify health
```

**§7.2 Monitoring**

```
### Dashboards
- System health: Streamlit at :8501
  - Request rate, latency percentiles, error rate
  - Cache hit rate, cost trend
  - PII detection counts, escalation rate
- Logs: structured JSON logs → stdout → collected by Docker logging driver

### Key Alerts
| Alert | Threshold | Action |
|-------|-----------|--------|
| Error rate spike | > 2% over 5 min | Check LLM API status, check DB connections |
| Latency p99 spike | > 5,000ms over 10 min | Check cache hit rate, check LLM API latency |
| Cost spike | > $10/hour | Check for runaway conversations, check routing |
| PII leakage | Any confirmed leak | P1 — see incident response below |
| Faithfulness drop | < 0.80 on eval run | Re-run eval, check for prompt regression |
| Cache hit rate drop | < 20% over 1 hour | Check for cache invalidation bug, check query distribution |
```

**§7.3 Debugging Common Issues**

```
| Symptom | Likely Cause | Debug Steps |
|---------|-------------|-------------|
| High latency | Cache miss + slow LLM | Check cache hit rate dashboard; check OpenAI API status |
| Wrong answers | Retrieval failure or hallucination | Check retrieved chunks in audit log; run faithfulness eval on the response |
| Cost spike | Over-escalation to GPT-4o | Check routing logs; check for multi-turn conversations; check token budget enforcement |
| PII in output | Redaction failure | Check input/output PII scan logs; check Presidio model version; check LLM scan layer |
| Escalation spike | Confidence threshold too high | Check confidence score distribution; consider threshold tuning |
| Empty results | Permission filter too strict | Check user roles in JWT; check document allowed_roles; check RLS policies |
```

**§7.4 On-Call Procedures**

```
### P1: PII Leakage
1. Identify scope: query audit_logs for responses containing PII patterns
2. Stop the bleeding: if systemic, disable /query endpoint (docker compose stop api)
3. Root cause: trace the specific response through PII scan logs
4. Fix: patch the redaction gap, redeploy
5. Notify: compliance officer (GDPR requirement within 72h for confirmed breaches)
6. Post-mortem: write incident report within 48h

### P1: System Down
1. Check: docker compose ps — which containers are down?
2. Check logs: docker compose logs --tail=100 <service>
3. Common fixes:
   - DB down: docker compose restart db
   - API down: docker compose restart api
   - Redis down: docker compose restart redis (cache will rebuild)
4. If LLM API is down: switch to fallback model (Ollama) via MODEL_PROVIDER env var
5. Verify: curl /health after each fix

### P2: Quality Degradation
1. Run eval suite: docker compose exec api python -m eval.run --suite regression
2. Compare to baseline: check eval_results/ for last passing run
3. If prompt regression: roll back prompt version in config
4. If retrieval regression: check for index corruption, re-index if needed
5. Document: add to incident log
```

**Deliverable:** Completed §7 Operational Runbook + `docs/runbook/RUNBOOK.md`.

**Verification:**
- [ ] Deploy steps are reproducible (someone could follow them without asking questions)
- [ ] Monitoring section covers the key alerts with thresholds and actions
- [ ] Debugging table covers the most common issues
- [ ] On-call procedures include at least one P1 scenario
- [ ] PII leakage procedure mentions GDPR 72h notification requirement
- [ ] Runbook is also saved as a standalone file in `docs/runbook/`

#### 4.3 Future Work (Late Afternoon)

Write §8 Future Work (~1 page). Structure as prioritized list:

**Near-term (1-3 months):**
- Migrate to Qdrant for vector search if scale exceeds 5M vectors (ADR-001 revisit)
- Fine-tune a small model (Llama 3.1 8B) on support conversations to reduce LLM API costs further
- Build a proper human agent UI for escalated tickets (currently using a basic Streamlit form)
- Implement conversation summarization for multi-turn tickets to reduce token costs

**Medium-term (3-6 months):**
- Add multi-language support (German first — primary market)
- Implement active learning: use escalated tickets (human-reviewed) as training data to improve auto-resolution rate
- Build a feedback loop: thumbs up/down from users → eval dataset expansion
- Add A/B testing framework for prompt versions in production (Project 8 extension)

**Long-term (6-12 months):**
- Explore on-prem LLM deployment (Llama 3.1 70B via vLLM) for complete data sovereignty
- Implement proactive support: detect issues from ticket patterns before customers report them
- Build a support knowledge graph: entities, relationships, and automated FAQ generation
- Integrate with CRM system for customer context (order history, previous tickets)

**What I'd build with more time:**
- A full evaluation platform with human-in-the-loop annotation UI
- Automated prompt optimization using DSPy or similar
- A cost prediction model for capacity planning
- A "support copilot" mode that assists human agents in real-time (not just auto-resolution)

**Deliverable:** Completed §8 Future Work.

**Verification:**
- [ ] Items are prioritized (near/medium/long-term)
- [ ] Each item has a brief rationale
- [ ] Shows vision without overpromising
- [ ] Connects to the system's current limitations (honest about what's missing)

#### 4.4 Supporting Deliverables (Late Afternoon → Evening)

**4.4a Metrics files:** Populate `docs/metrics/` with:
- `CHART_DESCRIPTIONS.md` — all chart specs from §4 consolidated
- `retrieval-quality.md`, `answer-quality.md`, `system-performance.md`, `cost-analysis.md`, `business-impact.md` — per-category tables and chart specs
- `exports/` — if generating actual chart images, place PNGs here with a `README.md` explaining how they were generated

**4.4b Presentation deck outline:** Write `docs/presentation/DECK_OUTLINE.md` (see Appendix D for the full outline).

**4.4c Runbook file:** Ensure `docs/runbook/RUNBOOK.md` matches §7 of CASE_STUDY.md.

**Deliverable:** All supporting files populated.

**Verification:**
- [ ] `docs/metrics/` has all 5 category files + chart descriptions
- [ ] `docs/presentation/DECK_OUTLINE.md` has 15–20 slides outlined
- [ ] `docs/runbook/RUNBOOK.md` is consistent with §7 of CASE_STUDY.md
- [ ] All files are self-contained (readable without the main case study)

---

### Phase 5 (Day 5): Review, Polish & Final Assembly

**Objective:** Review the complete case study for consistency, quality, and interview-readiness. Polish prose, fix cross-references, and produce the final PDF.

#### 5.1 Full Document Review (Morning)

Read `CASE_STUDY.md` end-to-end. Check:

**Consistency:**
- [ ] Metrics numbers are consistent across sections (e.g., auto-resolution rate in §1 matches §4.5 matches §5)
- [ ] ADR references in §3 link to the correct files in `docs/adr/`
- [ ] Technology names are spelled consistently (e.g., "pgvector" not "pgVector" not "PGVector")
- [ ] Terminology is consistent (e.g., "ticket" vs "query" — use "ticket" for business context, "query" for technical context, and be deliberate about the distinction)

**Quality:**
- [ ] No section is filler — every paragraph earns its place
- [ ] No unsupported claims — every metric has a source (eval run, dashboard, log)
- [ ] Prose is active, not passive ("we chose pgvector because..." not "pgvector was chosen because...")
- [ ] No AI-slop patterns (avoid: "delve into", "leverage", "in the realm of", "it's worth noting that", "navigate the complexities of")

**Interview-readiness:**
- [ ] Each section maps to common interview questions (see Appendix A)
- [ ] ADRs are "tell me about a time" stories
- [ ] Failure modes show humility and learning
- [ ] Metrics show analytical rigor

**Formatting:**
- [ ] ASCII diagrams render correctly in monospace
- [ ] Markdown tables render correctly on GitHub
- [ ] No broken internal links
- [ ] Page count is within budget (15–20 pages when rendered)

#### 5.2 Cross-Reference & Link Audit (Afternoon)

- [ ] §3 Design Decisions links to each ADR file: `[ADR-001: pgvector over Qdrant](docs/adr/ADR-001-pgvector-over-qdrant.md)`
- [ ] §4 Metrics references chart descriptions: `[Chart 1: Retrieval Strategy Comparison](docs/metrics/retrieval-quality.md#chart-1)`
- [ ] §7 Runbook notes that the standalone version is at `docs/runbook/RUNBOOK.md`
- [ ] README.md links to CASE_STUDY.md as the primary deliverable
- [ ] All `docs/adr/` files link back to the case study: "Part of the [Technical Case Study](../../CASE_STUDY.md)"

#### 5.3 Spell Check & Grammar (Late Afternoon)

Run:
```bash
cspell "CASE_STUDY.md" "docs/**/*.md"
```

Manual review for:
- Technical term spelling (PostgreSQL, pgvector, Presidio, spaCy, FastAPI, LangChain, etc.)
- Consistent capitalization (GPT-4o, GPT-4o-mini, Llama 3.1)
- No contractions in formal sections (§1, §2, §3) — acceptable in §6 Lessons Learned for a more conversational tone

#### 5.4 PDF Export & Final Deliverable (Evening)

Generate PDF for job applications:
```bash
pandoc CASE_STUDY.md -o CASE_STUDY.pdf \
  --pdf-engine=xelatex \
  --toc \
  --highlight-style=tango \
  -V geometry:margin=1in \
  -V fontsize=11pt
```

**Final Deliverable Check:**
- [ ] `CASE_STUDY.md` — 15–20 pages, all 8 sections complete
- [ ] `docs/adr/` — 5 ADRs + template
- [ ] `docs/metrics/` — 5 category files + chart descriptions + optional exports
- [ ] `docs/presentation/DECK_OUTLINE.md` — 15–20 slide outline
- [ ] `docs/runbook/RUNBOOK.md` — standalone runbook
- [ ] `README.md` — reading guide
- [ ] `CASE_STUDY.pdf` — PDF export for job applications
- [ ] Git repository is clean, committed, and pushed

**Deliverable:** Complete, polished case study with all supporting files.

**Verification:**
- [ ] Full document reads as a cohesive narrative, not 8 disconnected sections
- [ ] A non-technical reader can get value from §1 and §4.5 without reading anything else
- [ ] A senior engineer can deep-dive into §2, §3, and §5 and find technical substance
- [ ] The case study would make a hiring manager want to interview you
- [ ] You would be comfortable discussing any section in an interview

---

## 7. Component Specifications

This section provides pseudocode/structure for each major component of the case study document. Since this is a writing project, "pseudocode" = structural outlines and writing templates.

### 7.1 Executive Summary Component

```
SECTION: Executive Summary
LENGTH: 1 page (~400-500 words)
AUDIENCE: VP, hiring manager, non-technical stakeholder
TONE: Confident, concrete, business-framed

STRUCTURE:
  Opening paragraph:
    - What was built (1 sentence, plain language)
    - For whom (customer support teams)
    - Scale (ticket volume, document count)

  "Why It Matters" paragraph:
    - Business problem (ticket volume growth, response time, cost)
    - Why AI is the right solution (not just "AI is trendy")

  Key Results (bullet list, 5-6 items):
    - Auto-resolution rate: X%
    - Resolution time: from X hours to Y minutes
    - Cost per ticket: $Z
    - Faithfulness: X/1.0
    - PII detection: X%
    - Monthly spend: $X

  Business Value paragraph:
    - Annual savings projection
    - Scalability (handles X tickets/month, can scale to Y)
    - Customer satisfaction impact

WRITING RULES:
  - No acronyms without expansion
  - No architecture jargon (no "RAG", "vector search", "embedding")
  - Use "AI-powered support system" not "RAG pipeline with hybrid retrieval"
  - Every number has a comparison (before/after, vs baseline)
```

### 7.2 Architecture Overview Component

```
SECTION: Architecture Overview
LENGTH: 2-3 pages (~1,200-1,800 words)
AUDIENCE: Senior engineer, technical hiring manager
TONE: Precise, technical, diagram-driven

STRUCTURE:
  System Diagram (ASCII):
    - All components and their connections
    - Data flow arrows
    - External dependencies
    - Human-in-the-loop path
    - Must be readable in monospace font (max 80 chars wide)

  Component Descriptions (1 paragraph each):
    FOR EACH component:
      - What it does (1 sentence)
      - Key technology (1 sentence)
      - Why this technology (1 sentence, defer deep dive to ADR)
      - Interfaces (what it receives, what it produces)

  Data Flow Walkthrough:
    - Step-by-step query lifecycle (numbered list)
    - Step-by-step ingestion lifecycle (numbered list)
    - Escalation path (when and how human agents are involved)

  Technology Rationale Table:
    | Component | Technology | Why | ADR |
    |-----------|-----------|-----|-----|
    | Vector search | pgvector | SQL-level permission filtering | ADR-001 |
    | Retrieval | Hybrid BM25+vector | Keyword precision for support docs | ADR-002 |
    | ... | ... | ... | ... |

WRITING RULES:
  - Diagram first, then prose (visual learner friendly)
  - Every component in the diagram has a description
  - Defer deep rationale to ADRs; this section is the map, ADRs are the territory
  - Use consistent naming (component names in diagram match prose)
```

### 7.3 ADR Component

```
SECTION: Design Decisions (ADRs)
LENGTH: 3-4 pages in case study (summary) + 1-2 pages per ADR file
AUDIENCE: Senior engineer, architect
TONE: Analytical, balanced, honest about tradeoffs

CASE_STUDY.md §3 STRUCTURE:
  For each ADR (1-2 paragraphs in the case study):
    - One-sentence decision summary
    - Key tradeoff (what we gained vs what we gave up)
    - Link to full ADR file

  Then a summary table:
    | ADR | Decision | Key Tradeoff |
    |-----|----------|--------------|
    | ADR-001 | pgvector over Qdrant | SQL filtering vs scale ceiling |
    | ADR-002 | Hybrid search | +12% recall vs 1.8x latency |
    | ... | ... | ... |

INDIVIDUAL ADR FILE STRUCTURE (see Appendix B for template):
  - Title
  - Status (Accepted/Proposed/Deprecated)
  - Context (why this decision was needed)
  - Decision (what was decided)
  - Alternatives Considered (2-3, each with pros/cons)
  - Consequences (positive, negative, neutral)
  - (Optional) Compliance considerations (GDPR, etc.)
  - (Optional) Revision history

WRITING RULES:
  - Alternatives must be genuine (not strawmen)
  - Pros/cons must be balanced (every alternative has some pros)
  - Consequences must include negatives (no decision is perfect)
  - Past tense for context, present tense for decision and consequences
  - Each ADR is self-contained (readable without the case study)
```

### 7.4 Metrics Component

```
SECTION: Metrics & Results
LENGTH: 2-3 pages
AUDIENCE: Both technical and non-technical (subsections vary)
TONE: Data-driven, analytical, honest

STRUCTURE (5 subsections):
  §4.1 Retrieval Quality (technical audience)
    - Table: 4 strategies × 6 metrics
    - 2 chart descriptions
    - 1-paragraph interpretation

  §4.2 Answer Quality (technical audience)
    - Table: 3 models × 5 metrics
    - 1 chart description
    - 1-paragraph interpretation

  §4.3 System Performance (technical/SRE audience)
    - Table: 4 endpoints × 6 metrics
    - 2 chart descriptions
    - 1-paragraph interpretation

  §4.4 Cost (mixed audience)
    - Table: 8 cost metrics
    - 2 chart descriptions
    - 1-paragraph interpretation

  §4.5 Business Impact (non-technical audience)
    - Table: before/after comparison (8 metrics)
    - 2 chart descriptions
    - 1-paragraph interpretation

CHART DESCRIPTION FORMAT:
  ### Chart N: [Title]
  **Type:** [bar/line/pie/funnel/radar/scatter]
  **Purpose:** [what this chart shows and why it matters]
  **X-axis:** [label and units]
  **Y-axis:** [label and units]
  **Series:** [what each line/bar/slice represents]
  **Data:** [the actual data points, as a mini-table]
  **Interpretation:** [1-2 sentences on what the reader should take away]
  **Creation tool:** [matplotlib/plotly/Excel/Google Sheets — be specific]

WRITING RULES:
  - Every metric has a baseline or comparison
  - No metric without interpretation ("so what?")
  - Tables before charts (tables for precision, charts for intuition)
  - Business impact subsection uses plain language (no "nDCG" here)
  - Numbers are realistic (not suspiciously round — 42% not 40%, $0.08 not $0.10)
```

### 7.5 Failure Modes Component

```
SECTION: Failure Modes & Incidents
LENGTH: 2 pages
AUDIENCE: Senior engineer, SRE
TONE: Honest, analytical, no-spin

STRUCTURE:
  Intro paragraph:
    - "Production AI systems fail. What matters is how you detect, respond, and learn."
    - Set expectation: 4 incidents documented, ranging from P1 to P3

  For each incident (using the template from Phase 3):
    - Title, Date, Severity, Detection
    - What Happened (2-3 sentences)
    - Impact (quantified)
    - Root Cause (technical)
    - Timeline (T+0, T+Xm, ...)
    - Fix (immediate)
    - Preventive Measure (systemic)
    - Lesson (1 sentence)

  Closing paragraph:
    - Pattern across incidents (e.g., "3 of 4 incidents were caught by evaluation/monitoring, not user reports")
    - What this says about the system's observability

WRITING RULES:
  - No blame (use "the system" not "I made a mistake")
  - Be specific about detection (which alert? which eval? which log?)
  - Timelines show urgency but not panic
  - Preventive measures are systemic (code/config/process), not personal ("I'll be more careful")
  - Include at least one P1 (shows you've handled real pressure)
  - Include at least one "caught by eval/monitoring" (shows the eval investment paid off)
```

### 7.6 Lessons Learned Component

```
SECTION: Lessons Learned
LENGTH: 1-2 pages
AUDIENCE: All (especially hiring managers evaluating maturity)
TONE: Reflective, honest, specific

STRUCTURE (3 subsections):
  §6.1 What I'd Do Differently (5 items)
    FORMAT per item:
    - [What I'd change] — [why the original approach was suboptimal] — [what the better approach is]
    Example: "Started with evaluation earlier — built the system first, then retrofitted eval; eval should be built alongside the system, not after"

  §6.2 What Surprised Me (5 items)
    FORMAT per item:
    - [Surprise] — [why it was surprising] — [what it taught you]
    Example: "How much of production AI is plumbing — the 'AI' part was 30% of the work; the rest was caching, routing, guardrails, monitoring"

  §6.3 Advice for Others (5 items)
    FORMAT per item:
    - [Advice] — [brief rationale]
    Example: "Build evaluation before features — if you can't measure it, you can't improve it"

WRITING RULES:
  - No generic platitudes ("communication is key", "testing is important")
  - Every item is specific to this project
  - "What surprised me" shows genuine learning, not post-hoc rationalization
  - "What I'd do differently" shows self-critique, not defensiveness
  - Tone is conversational (contractions OK in this section)
```

### 7.7 Operational Runbook Component

```
SECTION: Operational Runbook
LENGTH: 2 pages
AUDIENCE: SRE, on-call engineer, devops
TONE: Procedural, precise, action-oriented

STRUCTURE:
  §7.1 Deployment
    - Prerequisites (bullet list)
    - Deploy steps (numbered list, copy-pasteable commands)
    - Rollback steps (numbered list)
    - Health check commands

  §7.2 Monitoring
    - Dashboard locations (URLs)
    - Key alerts table (alert, threshold, action)
    - Log locations and format

  §7.3 Debugging Common Issues
    - Table: symptom → likely cause → debug steps
    - 6-8 most common issues

  §7.4 On-Call Procedures
    - P1: PII Leakage (step-by-step, includes GDPR notification)
    - P1: System Down (step-by-step, includes fallback model)
    - P2: Quality Degradation (step-by-step, includes eval suite)

WRITING RULES:
  - Commands are copy-pasteable (full commands, not "run the deploy script")
  - Thresholds are specific ("> 2% over 5 min" not "high error rate")
  - Every alert has an action (not just "investigate")
  - PII leakage procedure mentions GDPR 72h requirement
  - Runbook is also a standalone file (docs/runbook/RUNBOOK.md)
```

### 7.8 Future Work Component

```
SECTION: Future Work
LENGTH: 1 page
AUDIENCE: All (shows vision and self-awareness)
TONE: Ambitious but grounded

STRUCTURE:
  Near-term (1-3 months): 4 items
  Medium-term (3-6 months): 4 items
  Long-term (6-12 months): 4 items
  "With more time": 3-4 items (blue-sky)

FORMAT per item:
  - [What] — [brief rationale or connection to current limitation]

WRITING RULES:
  - Items connect to current limitations (honest about what's missing)
  - Near-term items are realistic and specific
  - Long-term items show vision without overpromising
  - "With more time" items are aspirational but technically grounded
  - No item is "add more tests" or "improve monitoring" (too generic)
```

---

## 8. Security & Safety Considerations

This is a documentation project, but it discusses a system that handles PII and customer data. Security considerations apply to the *content* of the case study.

### 8.1 Data Sanitization in the Case Study

| Concern | Mitigation |
|---------|-----------|
| Real customer PII in examples | All examples use synthetic data (fake names, fake ticket content, fake order numbers) |
| Real API keys in deployment examples | Use placeholder values (`OPENAI_API_KEY=sk-...`) in all documentation |
| Real database credentials | Use placeholder values (`POSTGRES_PASSWORD=changeme`) |
| Real customer names/tenants | Use generic names ("Acme Corp", "Customer A") |
| Real internal system URLs | Use localhost or example.com in all URLs |

### 8.2 Disclosure of Security-Sensitive Details

| Detail | Include? | Rationale |
|--------|----------|-----------|
| Architecture and component design | ✅ Yes | Demonstrates engineering capability; not exploitable |
| PII detection approach (Presidio + LLM) | ✅ Yes | Shows compliance awareness; approach is not secret |
| PII detection thresholds and patterns | ⚠️ Partial | Describe the approach but not exact regex patterns that could be used to bypass |
| Authentication mechanism (JWT) | ✅ Yes | Standard approach; not a security risk to disclose |
| Rate limiting thresholds | ⚠️ Partial | Mention that rate limiting exists; don't disclose exact thresholds |
| Known vulnerabilities | ❌ No | If any exist, fix them before publishing |
| Incident details | ✅ Yes | Shows maturity; sanitize any PII from incident descriptions |

### 8.3 GDPR Considerations in the Case Study Itself

- The case study discusses GDPR compliance as a feature of the system — this is a selling point for German enterprise employers
- No real personal data appears anywhere in the case study
- The PII leakage incident (Incident 3) is described using synthetic data
- The 72h breach notification requirement is mentioned in the runbook (shows GDPR awareness)
- Data sovereignty is discussed in ADR-001 (pgvector keeps data in our PostgreSQL vs Pinecone sending data to a third party)

### 8.4 Intellectual Property

- The case study describes architecture and approach, not proprietary algorithms
- Code snippets (if any) are minimal and illustrative, not production code
- The case study is licensed MIT (portfolio artifact — meant to be shared)
- No third-party confidential information is disclosed

---

## 9. Document Interface & Consumption Guide

The case study has no API, but it has an "interface" — how different audiences consume it. This section defines the reading paths.

### 9.1 Audience Reading Paths

```
┌─────────────────────┬───────────────────────────────────┬──────────┐
│ Audience            │ Reading Path                      │ Time     │
├─────────────────────┼───────────────────────────────────┼──────────┤
│ Hiring Manager / VP │ §1 Executive Summary              │ 5 min    │
│                     │ §4.5 Business Impact              │          │
│                     │ §6.3 Advice for Others            │          │
├─────────────────────┼───────────────────────────────────┼──────────┤
│ Senior Engineer     │ §2 Architecture Overview          │ 25 min   │
│                     │ §3 Design Decisions (ADRs)        │          │
│                     │ §5 Failure Modes                  │          │
│                     │ §6 Lessons Learned                │          │
├─────────────────────┼───────────────────────────────────┼──────────┤
│ Data/AI Engineer    │ §2 Architecture Overview          │ 20 min   │
│                     │ §4 Metrics & Results              │          │
│                     │ §3 ADR-001, ADR-002               │          │
├─────────────────────┼───────────────────────────────────┼──────────┤
│ DevOps/SRE          │ §2 Architecture Overview          │ 15 min   │
│                     │ §7 Operational Runbook            │          │
│                     │ §5 Failure Modes                  │          │
├─────────────────────┼───────────────────────────────────┼──────────┤
│ Interviewer         │ Full document (used as talking    │ 45 min   │
│ (pre-interview)     │ document during the interview)    │          │
├─────────────────────┼───────────────────────────────────┼──────────┤
│ Interviewer         │ §1 Executive Summary              │ 10 min   │
│ (during interview)  │ + whichever section they ask about│          │
└─────────────────────┴───────────────────────────────────┴──────────┘
```

### 9.2 Interview Question → Section Mapping

| Common Interview Question | Case Study Section |
|---------------------------|-------------------|
| "Tell me about a project you're proud of" | §1 Executive Summary → §2 Architecture |
| "How do you choose between technologies?" | §3 ADRs (any) |
| "Tell me about a time something failed in production" | §5 Failure Modes (any incident) |
| "How do you evaluate LLM output quality?" | §4.2 Answer Quality + §4.1 Retrieval Quality |
| "How do you handle cost in production AI?" | §4.4 Cost + ADR-005 |
| "How do you handle PII/compliance?" | ADR-004 + §5 Incident 3 + §7.4 P1 procedure |
| "How do you decide when to use human review?" | ADR-003 |
| "What would you do differently?" | §6.1 What I'd Do Differently |
| "What surprised you about building AI systems?" | §6.2 What Surprised Me |
| "How do you deploy and monitor this?" | §7 Operational Runbook |
| "What's next for this system?" | §8 Future Work |
| "How do you handle hallucinations?" | §5 Incident 1 + §4.2 Faithfulness |
| "Walk me through your architecture" | §2 Architecture Overview (diagram + walkthrough) |
| "How do you handle scale?" | §4.3 System Performance + ADR-001 (scale discussion) |
| "What metrics do you track?" | §4 Metrics & Results (all subsections) |

---

## 10. Testing & Validation Strategy

This project produces a document, not code. "Testing" = validation that the document meets its quality bar.

### 10.1 Validation Layers

| Layer | What | How | When |
|-------|------|-----|------|
| L1: Structural | All sections present, correct length | Checklist verification | End of each phase |
| L2: Consistency | Numbers match across sections | Manual cross-reference audit | Phase 5 |
| L3: Audience | Right audience can understand each section | Read §1 as a non-technical person; read §3 as a senior engineer | Phase 5 |
| L4: Interview-readiness | Each section maps to interview questions | Check against §9.2 mapping | Phase 5 |
| L5: Honesty | Failure modes are real, tradeoffs are balanced | Self-review: "would a senior engineer call BS on this?" | Phase 5 |
| L6: Professional quality | No typos, consistent formatting, clean prose | `cspell` + manual read-through | Phase 5 |

### 10.2 Critical Validation Checks (Pseudocode)

```
CHECK 1: Metric Consistency
  FOR EACH metric IN case_study:
    appearances = find_all_mentions(metric, case_study)
    IF len(set(appearances)) > 1:
      FAIL("Metric {metric} has inconsistent values: {appearances}")

CHECK 2: Section Completeness
  FOR EACH section IN [§1..§8]:
    IF section.length < section.min_length:
      FAIL("{section} is {section.length} pages, minimum is {section.min_length}")
    IF section.has_placeholder_text:
      FAIL("{section} contains placeholder text")

CHECK 3: ADR Completeness
  FOR EACH adr IN docs/adr/:
    IF not adr.has("Context"):
      FAIL("{adr} missing Context")
    IF not adr.has("Alternatives Considered"):
      FAIL("{adr} missing Alternatives")
    IF len(adr.alternatives) < 2:
      FAIL("{adr} needs at least 2 alternatives")
    IF not adr.has("Consequences"):
      FAIL("{adr} missing Consequences")
    IF not adr.consequences.has_negative:
      FAIL("{adr} consequences must include negatives")

CHECK 4: Incident Completeness
  FOR EACH incident IN §5:
    REQUIRED = ["What Happened", "Impact", "Root Cause", "Timeline",
                "Fix", "Preventive Measure", "Lesson"]
    FOR EACH field IN REQUIRED:
      IF not incident.has(field):
        FAIL("Incident {incident.title} missing {field}")
    IF not incident.severity IN ["P1", "P2", "P3"]:
      FAIL("Incident {incident.title} has invalid severity")
  IF not any(i.severity == "P1" for i in §5.incidents):
    FAIL("Need at least one P1 incident")

CHECK 5: Chart Description Completeness
  FOR EACH chart IN §4:
    REQUIRED = ["Type", "Purpose", "X-axis", "Y-axis", "Series",
                "Data", "Interpretation"]
    FOR EACH field IN REQUIRED:
      IF not chart.has(field):
        FAIL("Chart {chart.title} missing {field}")

CHECK 6: No PII in Document
  PII_PATTERNS = [email_regex, phone_regex, credit_card_regex, ssn_regex]
  FOR EACH pattern IN PII_PATTERNS:
    matches = find(pattern, case_study)
    IF matches:
      FAIL("Potential PII found: {matches}")

CHECK 7: Interview Readiness
  FOR EACH question IN INTERVIEW_QUESTIONS:
    IF not question.maps_to_section:
      FAIL("No section answers: {question}")

CHECK 8: AI-Slop Detection
  SLOP_PHRASES = ["delve into", "leverage", "in the realm of",
                  "it's worth noting", "navigate the complexities",
                  "tapestry", "landscape of", "paradigm shift",
                  "game-changer", "revolutionary"]
  FOR EACH phrase IN SLOP_PHRASES:
    IF find(phrase, case_study):
      WARN("AI-slop phrase detected: {phrase}")
```

### 10.3 Peer Review Protocol

If possible, have a peer (or use an LLM reviewer) check:

1. **Technical accuracy:** "Does this architecture make sense? Are the ADR tradeoffs real?"
2. **Communication quality:** "Can you understand the Executive Summary without technical background?"
3. **Honesty signal:** "Does this feel like a real project with real failures, or a marketing piece?"
4. **Interview simulation:** "If I ask you about ADR-002, can you give me a 2-minute answer based on this document?"

---

## 11. Deployment & Distribution

### 11.1 Repository Setup

```bash
# Initialize repository
git init project-10-technical-case-study
cd project-10-technical-case-study

# Create structure
mkdir -p docs/adr docs/metrics/exports docs/presentation docs/runbook

# Create files (following the structure in §5)
# ... (content from Phases 1-5)

# Initial commit
git add .
git commit -m "docs: complete technical case study for Customer Support AI System

- CASE_STUDY.md: 8-section comprehensive writeup (18 pages)
- docs/adr/: 5 Architecture Decision Records + template
- docs/metrics/: per-category metric tables and chart descriptions
- docs/presentation/DECK_OUTLINE.md: 15-min talk slide-by-slide outline
- docs/runbook/RUNBOOK.md: operational procedures"

# Push to GitHub
git remote add origin git@github.com:<user>/project-10-technical-case-study.git
git push -u origin main
```

### 11.2 README.md

```markdown
# Technical Case Study: Customer Support AI System

A comprehensive technical writeup of a production AI system for customer support,
demonstrating the ability to communicate AI architecture decisions, tradeoffs,
failures, and metrics to both technical and non-technical stakeholders.

## How to Read This Case Study

| If you are... | Read... | Time |
|---------------|---------|------|
| A hiring manager | [Executive Summary](CASE_STUDY.md#1-executive-summary) | 5 min |
| A senior engineer | [Architecture](CASE_STUDY.md#2-architecture-overview) + [ADRs](docs/adr/) | 25 min |
| A data/AI engineer | [Metrics](CASE_STUDY.md#4-metrics--results) | 20 min |
| An SRE | [Runbook](docs/runbook/RUNBOOK.md) | 15 min |
| Everyone | [Lessons Learned](CASE_STUDY.md#6-lessons-learned) | 10 min |

## Deliverables

- `CASE_STUDY.md` — Primary deliverable (18 pages)
- `docs/adr/` — 5 Architecture Decision Records
- `docs/metrics/` — Metric tables and chart descriptions
- `docs/presentation/DECK_OUTLINE.md` — Presentation deck outline
- `docs/runbook/RUNBOOK.md` — Operational runbook

## Context

This case study documents the [Customer Support AI System](../project-9-customer-support-ai),
the capstone project of an AI Engineering roadmap. It is a communication artifact,
not a code project.

## License

MIT
```

### 11.3 Distribution Channels

| Channel | Format | Audience |
|---------|--------|----------|
| GitHub repository | Markdown (renders natively) | Technical hiring managers, engineers |
| Job application attachment | PDF (`pandoc` export) | Recruiters, HR |
| Interview talking document | Markdown (screen-shared) | Interview panel |
| LinkedIn article | Adapted §1 + §6 | Network, recruiters |
| Presentation | `docs/presentation/DECK_OUTLINE.md` → slides | Meetup, interview, team showcase |
| Portfolio website | Markdown rendered | All visitors |

### 11.4 PDF Export

```bash
# Using pandoc with xelatex for Unicode support
pandoc CASE_STUDY.md \
  -o CASE_STUDY.pdf \
  --pdf-engine=xelatex \
  --toc \
  --toc-depth=2 \
  --highlight-style=tango \
  -V geometry:margin=1in \
  -V fontsize=11pt \
  -V linkcolor:blue \
  -V urlcolor:blue

# Alternative: using wkhtmltopdf (better for ASCII diagrams)
pandoc CASE_STUDY.md \
  -o CASE_STUDY.pdf \
  --pdf-engine=wkhtmltopdf \
  --toc \
  --css=style.css \
  -V margin-top=20mm \
  -V margin-bottom=20mm
```

### 11.5 .gitignore

```
# OS
.DS_Store
Thumbs.db

# Editor
*.swp
*.swo
*~
.vscode/
.idea/

# Temporary
*.tmp
sources.md

# PDF exports (regenerate on demand, don't version)
# CASE_STUDY.pdf
```

---

## 12. Roadmap & Milestones

```
Day 1 (Monday): Sourcing & Executive Summary
  ├── Morning: Source material audit from Project 11
  ├── Afternoon: Document skeleton + directory structure
  └── Late Afternoon: §1 Executive Summary
  MILESTONE: Skeleton + Exec Summary complete

Day 2 (Tuesday): Architecture & ADRs
  ├── Morning: §2 Architecture Overview (diagram + descriptions + data flow)
  ├── Afternoon: ADR template + ADR-001 (pgvector)
  └── Late Afternoon: ADR-002 through ADR-005
  MILESTONE: Architecture + all 5 ADRs complete

Day 3 (Wednesday): Metrics & Failures
  ├── Morning: §4.1-4.3 Metrics (retrieval, answer, system)
  ├── Afternoon: §4.4-4.5 Metrics (cost, business)
  └── Late Afternoon: §5 Failure Modes (4 incidents)
  MILESTONE: Metrics + Failure Modes complete

Day 4 (Thursday): Lessons, Runbook, Future + Supporting Files
  ├── Morning: §6 Lessons Learned
  ├── Afternoon: §7 Operational Runbook
  ├── Late Afternoon: §8 Future Work
  └── Evening: Supporting files (metrics/, presentation/, runbook/)
  MILESTONE: All 8 sections + supporting files complete

Day 5 (Friday): Review & Polish
  ├── Morning: Full document review (consistency, quality, interview-readiness)
  ├── Afternoon: Cross-reference & link audit
  ├── Late Afternoon: Spell check & grammar
  └── Evening: PDF export + final commit
  MILESTONE: Complete, polished case study ready for distribution
```

### Milestone Summary

| Milestone | Day | Deliverable | Verification |
|-----------|-----|-------------|--------------|
| M1 | 1 | Skeleton + §1 Exec Summary | Non-technical reader understands it |
| M2 | 2 | §2 Architecture + 5 ADRs | Senior engineer can reconstruct system |
| M3 | 3 | §4 Metrics + §5 Failures | All metrics have baselines; 4+ incidents |
| M4 | 4 | §6-8 + supporting files | All sections + docs/ directory complete |
| M5 | 5 | Final polished case study | All validation checks pass; PDF exported |

---

## 13. Risk Register

| # | Risk | Likelihood | Impact | Mitigation |
|---|------|-----------|--------|------------|
| R1 | **Metrics from P11 are incomplete or not collected** | Medium | High | Phase 1 source audit catches this early; if metrics are missing, use realistic synthetic values and label them as "projected" or "estimated" |
| R2 | **Case study reads as marketing, not engineering** | Medium | High | Include honest failure modes (§5); ensure ADRs have real tradeoffs with negatives; use the AI-slop detection check (§10.2 Check 8) |
| R3 | **Too long / no one reads it** | Medium | Medium | Keep to 15-20 pages; Executive Summary is 1 page; reading paths in §9.1 let readers skip to relevant sections |
| R4 | **Too short / lacks depth** | Low | Medium | ADRs provide depth; metrics tables provide specificity; minimum page counts enforced in validation |
| R5 | **Numbers are suspiciously round / feel fabricated** | Medium | High | Use realistic non-round numbers (42% not 40%, $0.08 not $0.10); ensure internal consistency across sections |
| R6 | **ADRs are one-sided (no real alternatives)** | Medium | High | Validation Check 3 requires ≥2 alternatives per ADR; each alternative must have genuine pros |
| R7 | **Failure modes are trivial or not credible** | Low | High | Include at least one P1 (PII leak); include specific detection mechanisms (not "a user noticed"); include systemic preventive measures |
| R8 | **Case study reveals sensitive information** | Low | High | All examples use synthetic data; §8 Security considerations define what to include/exclude; run PII pattern check on final document |
| R9 | **Diagrams don't render correctly** | Medium | Low | Test ASCII diagrams in GitHub render; keep ≤80 chars wide; use Mermaid as supplementary, not primary |
| R10 | **Cross-references are broken** | Medium | Low | Phase 5 link audit; test all internal links in GitHub render |
| R11 | **Scope creep into documenting P1-P8** | Low | Medium | Non-goals explicitly state this is P9 only; if referencing prior projects, link to their repos, don't document them |
| R12 | **Prose has AI-slop patterns** | High | Medium | Run slop detection check; use active voice; avoid "delve into", "leverage", "navigate the complexities"; be concrete |
| R13 | **Inconsistent terminology** | Medium | Low | Maintain a glossary; use "ticket" for business context, "query" for technical; spell tech names consistently |
| R14 | **Case study doesn't map to interview questions** | Low | High | §9.2 maps every section to common interview questions; validation Check 7 verifies coverage |

---

## Appendix A: Quick Start

### For the Writer (You)

```bash
# 1. Ensure Project 11 is complete and you have access to:
#    - Source code and architecture
#    - Evaluation results (JSON/CSV)
#    - Monitoring dashboards
#    - Incident logs/notes
#    - Deployment configs

# 2. Create the project structure
mkdir -p project-10-technical-case-study/docs/{adr,metrics/exports,presentation,runbook}
cd project-10-technical-case-study
git init

# 3. Follow the 5-day plan in §6 Implementation Phases
# 4. Use the ADR template in Appendix B for each decision record
# 5. Use the chart description format in Appendix C for each chart
# 6. Use the deck outline in Appendix D for the presentation
# 7. Use the runbook template in Appendix E for operational procedures
# 8. Run all validation checks in §10.2 before publishing
# 9. Export to PDF with pandoc (§11.4)
# 10. Push to GitHub and attach to job applications
```

### For the Reader (Hiring Manager / Interviewer)

```
# Short on time? Read these:
1. Executive Summary (CASE_STUDY.md §1) — 5 min
2. Lessons Learned (CASE_STUDY.md §6) — 10 min
3. One ADR of your choice (docs/adr/) — 5 min

# Want the full picture? Read in order:
1. CASE_STUDY.md §1-8 (18 pages) — 45 min
2. Skim docs/adr/ for decision depth — 15 min
3. Skim docs/metrics/ for chart details — 10 min

# Preparing for an interview with this candidate?
1. Read §1 Executive Summary
2. Read §3 Design Decisions (pick 2 ADRs to deep-dive)
3. Read §5 Failure Modes (pick 1 incident to discuss)
4. Prepare questions from §9.2 Interview Question Mapping
```

### Interview Conversation Guide

```
# Opening (5 min)
"Tell me about this project" → §1 Executive Summary + §2 Architecture diagram

# Deep-dive (15 min) — pick 2-3:
"Walk me through ADR-001" → pgvector vs Qdrant tradeoff discussion
"Why hybrid search?" → ADR-002, recall@5 improvement, latency tradeoff
"How do you handle hallucinations?" → §5 Incident 1 + §4.2 Faithfulness
"How do you handle PII?" → ADR-004 + §5 Incident 3 + GDPR awareness

# Production (10 min):
"How do you monitor this?" → §7.2 Monitoring + alerts table
"What happens when it breaks?" → §7.4 On-call procedures
"How do you control cost?" → §4.4 Cost + ADR-005 + §5 Incident 4

# Reflection (5 min):
"What would you do differently?" → §6.1
"What surprised you?" → §6.2
"What's next?" → §8
```

---

## Appendix B: ADR Template

> Save as `docs/adr/ADR-000-template.md`. Follows the MADR (Markdown Architecture Decision Record) format.

```markdown
# ADR-NNN: [Decision Title]

## Status

<!-- Accepted | Proposed | Deprecated | Superseded by ADR-XXX -->

Accepted

## Context

<!-- Why is this decision needed? What problem are we solving? What constraints
exist? This section should explain the SITUATION, not the decision. -->

[2-4 paragraphs describing the problem, constraints, and forces influencing
the decision. Include:
- What the system needs to do
- What constraints exist (performance, security, compliance, cost, team skills)
- What forces are in tension (e.g., speed vs accuracy, cost vs quality)
- What happens if we don't make a decision (default action)]

## Decision

<!-- What did we decide? Be concrete and unambiguous. -->

[1-2 paragraphs stating the decision clearly. "We will use X for Y because Z."]

## Alternatives Considered

<!-- For each alternative: name, pros, cons. Be genuine — every alternative
should have some pros, or it's a strawman. -->

### Alternative 1: [Name]

**Pros:**
- [Pro 1]
- [Pro 2]

**Cons:**
- [Con 1]
- [Con 2]

### Alternative 2: [Name]

**Pros:**
- [Pro 1]

**Cons:**
- [Con 1]

## Consequences

<!-- What happens because of this decision? Include positive, negative, and
neutral consequences. Be honest — every decision has downsides. -->

**Positive:**
- [What we gain]

**Negative:**
- [What we give up or risk]

**Neutral:**
- [Side effects that are neither good nor bad but worth noting]

## Compliance Considerations

<!-- If relevant: GDPR, SOC2, HIPAA, etc. Omit if not applicable. -->

[How this decision affects compliance. E.g., "Keeping vector data in PostgreSQL
means PII never leaves our infrastructure, simplifying GDPR compliance."]

## Revision History

| Date | Change | Author |
|------|--------|--------|
| YYYY-MM-DD | Initial acceptance | [Name] |

---

*Part of the [Technical Case Study](../../CASE_STUDY.md) for the Customer Support AI System.*
```

---

## Appendix C: Metrics Presentation Format

### C.1 Chart Description Template

Every chart in the case study must have a structured description following this format:

```markdown
### Chart N: [Descriptive Title]

**Type:** [bar | line | pie | scatter | radar | funnel | heatmap | box]
**Purpose:** [One sentence: what this chart shows and why it matters]

**Axes:**
- X-axis: [label] ([units])
- Y-axis: [label] ([units])

**Series:** [What each line/bar/slice represents. E.g., "One bar per retrieval strategy"]

**Data:**

| [X-axis label] | [Series 1] | [Series 2] | ... |
|----------------|-----------|-----------|-----|
| [value] | [value] | [value] | ... |

**Interpretation:** [1-2 sentences: what the reader should take away. Answer: "so what?"]

**Creation:** [Tool and code/command to recreate. E.g., "matplotlib: plt.bar(strategies, recall_at_5)"]
```

### C.2 Chart Inventory

| # | Chart | Type | Section | Key Insight |
|---|-------|------|---------|-------------|
| 1 | Retrieval Strategy Comparison | Grouped bar | §4.1 | Hybrid+rerank best quality, 2.5x latency |
| 2 | Recall@5 by Query Type | Line | §4.1 | BM25 wins on policy; vector wins on troubleshooting |
| 3 | Answer Quality Radar | Radar | §4.2 | GPT-4o marginal improvement over mini for 3x cost |
| 4 | Latency Percentile Waterfall | Stacked bar | §4.3 | p99 dominated by LLM API latency |
| 5 | Latency Breakdown | Horizontal bar | §4.3 | LLM call is 67% of end-to-end latency |
| 6 | Monthly Cost Trend | Line | §4.4 | Sub-linear cost growth despite volume growth |
| 7 | Cost per Ticket by Model | Pie | §4.4 | 80/20 routing holds (mini handles 72% of cost) |
| 8 | Resolution Funnel | Funnel | §4.5 | 42% auto-resolution; AI drafts accelerate escalations |
| 9 | CSAT Before/After | Diverging bar | §4.5 | Distribution shifts right; fewer 1-2 star ratings |

### C.3 Metrics Table Format

Every metrics table must follow this structure:

```markdown
| [Dimension] | [Metric 1] | [Metric 2] | ... | [Source] |
|-------------|-----------|-----------|-----|----------|
| [value] | [value] | [value] | ... | [eval run / dashboard / log] |
```

Rules:
- Every table has a title (Markdown heading above it)
- Every column has a unit in the header or first row
- Baseline/comparison column is mandatory (no metric without context)
- Source column or note indicates where the number comes from
- Numbers are realistic (not suspiciously round)

### C.4 Dashboard Export Instructions

If generating actual chart images for `docs/metrics/exports/`:

```bash
# Option 1: Screenshot the Streamlit dashboard from Project 11
# 1. Start the dashboard: docker compose up dashboard
# 2. Navigate to http://localhost:8501
# 3. Screenshot each chart, save as PNG to docs/metrics/exports/

# Option 2: Generate charts with matplotlib/plotly
# Use the data from Project 11's eval results
# Save charts as PNG to docs/metrics/exports/
# Include a generation script reference in each chart description

# Option 3: Export from Grafana (if using Grafana for monitoring)
# Use Grafana's "Share > Export" feature to save panel as PNG
```

---

## Appendix D: Presentation Deck Outline

> Save as `docs/presentation/DECK_OUTLINE.md`. A 15-minute talk based on the case study, designed for a meetup, interview, or team showcase.

```markdown
# Presentation Deck Outline: Customer Support AI System

> 15-minute talk | 18 slides | Audience: mixed technical

## Slide 1: Title
- Title: "Building a Customer Support AI System: Architecture, Tradeoffs, and What Broke"
- Subtitle: "A production AI case study"
- Author, date
- [No content — just title]

## Slide 2: The Problem (30 sec)
- "Support tickets are growing faster than support teams"
- Stat: 4,200 tickets/month, 6.2 hour avg resolution
- Cost: $4.50 per human-resolved ticket
- [1 chart: ticket volume growth over 6 months]

## Slide 3: What We Built (30 sec)
- "An AI-powered support system that auto-resolves 42% of tickets"
- 1-sentence description (non-technical)
- Key result: 6.2 hours → 2.1 hours avg resolution
- [No chart — just text]

## Slide 4: System Architecture (2 min)
- [The ASCII/Mermaid system diagram from §2]
- Walk through: query → PII scan → cache → retrieval → generation → response
- "The AI part is 30% of the system; the rest is production plumbing"
- [System diagram]

## Slide 5: Why pgvector? (1 min) — ADR-001
- "We needed vector search + permission filtering in one query"
- pgvector: SQL-level tenant isolation, one database to operate
- Tradeoff: scale ceiling at ~10M vectors
- [Simple diagram: pgvector (one box) vs Qdrant+Postgres (two boxes + sync arrow)]

## Slide 6: Why Hybrid Search? (1 min) — ADR-002
- "Support docs have exact-match terms: error codes, policy numbers"
- BM25 catches exact matches; vector catches semantic similarity
- Result: +12% recall@5 vs pure vector
- Tradeoff: 1.8x retrieval latency
- [Chart 1: Retrieval Strategy Comparison]

## Slide 7: Model Routing (1 min) — ADR-005
- "80% of tickets are routine; 20% need deep reasoning"
- GPT-4o-mini for routine → GPT-4o for complex
- Result: 65% cost reduction vs GPT-4o-only
- [Chart 7: Cost per Ticket by Model pie chart]

## Slide 8: PII Protection (1 min) — ADR-004
- "Support tickets contain PII; GDPR requires protection"
- Layer 1: Presidio (NER + regex, fast, deterministic)
- Layer 2: LLM scan (catches context-dependent PII)
- "Defense in depth — either layer alone has gaps"
- [Diagram: two-layer PII pipeline]

## Slide 9: Human-in-the-Loop (1 min) — ADR-003
- "When the AI isn't confident, it escalates to a human — with a draft"
- Confidence threshold: 0.75
- "Wrong answers erode trust faster than 'I don't know'"
- [Diagram: confidence → respond or escalate flow]

## Slide 10: Retrieval Quality Results (1 min)
- [Chart 1 + Chart 2]
- "Hybrid + re-ranking: 84% recall@5"
- "BM25 outperforms vector on policy queries (exact match)"

## Slide 11: Answer Quality Results (1 min)
- [Chart 3: Answer Quality Radar]
- "Faithfulness 0.87 (mini) / 0.93 (4o) — answers grounded in source docs"
- "Citation accuracy 94% — users can verify claims"

## Slide 12: System Performance (1 min)
- [Chart 4 + Chart 5]
- "p50: 820ms (cache miss), 45ms (cache hit)"
- "LLM call is 67% of latency — optimization target"

## Slide 13: Cost (1 min)
- [Chart 6: Monthly Cost Trend]
- "$340/month LLM spend"
- "34% cache hit rate + model routing = $265/month savings"

## Slide 14: Business Impact (1 min)
- [Chart 8: Resolution Funnel + before/after table]
- "42% auto-resolution, 66% faster resolution, $5,250/month savings"
- "CSAT: 3.8 → 4.2"

## Slide 15: What Broke (2 min) — The honest part
- "4 incidents, 1 P1"
- Incident 1: Hallucinated policy → caught by citation validation
- Incident 3 (P1): PII leaked through LLM output → added output scanning
- "What matters is detection, response, and learning"
- [Timeline graphic for one incident]

## Slide 16: Lessons Learned (1 min)
- "Build evaluation before features"
- "70% of production AI is plumbing"
- "Monitor cost like you monitor latency"
- "The case study matters more than the code"
- [Text only — 4 bullets]

## Slide 17: What's Next (30 sec)
- Multi-language (German first)
- Fine-tune small model for cost reduction
- Human agent copilot mode
- [Text only — 3 bullets]

## Slide 18: Thank You / Q&A (30 sec)
- Contact info
- Link to full case study
- "Questions?"
- [No content]

---

## Presentation Notes

### Timing
- Total: 15 minutes
- Slides 1-3 (setup): 1.5 min
- Slides 4-9 (architecture): 7 min
- Slides 10-14 (results): 5 min
- Slides 15-16 (failures + lessons): 3 min
- Slides 17-18 (future + close): 1 min
- Buffer: 0.5 min

### Audience Adaptation
- **Technical audience:** spend more time on ADRs (slides 5-9), less on business impact
- **Business audience:** spend more time on slides 2-3, 13-14, skip ADR details
- **Interview:** use as a talking document — let the interviewer drive which slides to deep-dive

### Delivery Tips
- The architecture diagram (slide 4) is the most important slide — practice drawing it
- The failure section (slide 15) is what differentiates you — don't skip it or rush it
- Have the full case study open in another tab for reference questions
- Prepare 2-minute answers for each ADR (interviewers will ask)
```

---

## Appendix E: Operational Runbook Template

> Save as `docs/runbook/RUNBOOK.md`. Also embedded in CASE_STUDY.md §7.

```markdown
# Operational Runbook: Customer Support AI System

## 1. Deployment

### Prerequisites
- [ ] Docker 24+ and Docker Compose v2 installed
- [ ] OpenAI API key (or local Ollama for Llama 3.1 fallback)
- [ ] PostgreSQL 16 with pgvector extension available
- [ ] Redis 7+ available
- [ ] .env file configured (see .env.example)

### Deploy Steps
1. Clone: `git clone <repo> && cd <repo>`
2. Configure: `cp .env.example .env && nano .env`
3. Build: `docker compose build`
4. Migrate: `docker compose exec api alembic upgrade head`
5. Start: `docker compose up -d`
6. Health check: `curl http://localhost:8000/health`
7. Smoke test: `./scripts/smoke_test.sh`
8. Verify dashboards: `open http://localhost:8501`

### .env.example
```
# Database
POSTGRES_HOST=db
POSTGRES_PORT=5432
POSTGRES_DB=support_ai
POSTGRES_USER=support
POSTGRES_PASSWORD=changeme

# Redis
REDIS_HOST=redis
REDIS_PORT=6379

# LLM
OPENAI_API_KEY=sk-...
MODEL_PRIMARY=gpt-4o-mini
MODEL_ESCALATION=gpt-4o
FALLBACK_PROVIDER=ollama          # for LLM API outages

# Embeddings
EMBEDDING_MODEL=text-embedding-3-small

# PII Detection
PRESIDIO_ENABLED=true
PII_OUTPUT_SCAN=true              # Layer 2 output scanning

# Cache
CACHE_SIMILARITY_THRESHOLD=0.85   # lowered after Incident 2
CACHE_TTL_SECONDS=3600

# Routing
COMPLEXITY_THRESHOLD=0.7          # tickets above this → GPT-4o
CONFIDENCE_THRESHOLD=0.75         # below this → human escalation
MAX_TOKENS_PER_CONVERSATION=4000  # budget enforcement

# Monitoring
DASHBOARD_PORT=8501
ALERT_WEBHOOK=https://hooks.slack.com/...

# Compliance
GDPR_LOGGING=true                # audit log all PII detection events
BREACH_NOTIFICATION_WINDOW_H=72  # GDPR 72h breach notification
```

### Rollback

```bash
# 1. Stop services
docker compose down

# 2. Checkout previous known-good version
git checkout <previous-tag>

# 3. Rebuild and start
docker compose up -d --build

# 4. Revert database migration if needed
docker compose exec api alembic downgrade -1

# 5. Verify health
curl http://localhost:8000/health
```

## 2. Monitoring

### Dashboards

| Dashboard | URL | Key Panels |
|-----------|-----|------------|
| System Health | http://localhost:8501 | Request rate, latency p50/p90/p99, error rate |
| Cost Tracker | http://localhost:8501/cost | Daily/monthly spend, cost per ticket, cache savings |
| Quality Monitor | http://localhost:8501/quality | Faithfulness trend, citation accuracy, escalation rate |
| PII Audit | http://localhost:8501/pii | PII detection counts, redaction events, output scan results |

### Key Alerts

| Alert | Threshold | Severity | Action |
|-------|-----------|----------|--------|
| Error rate spike | > 2% over 5 min | P2 | Check LLM API status page; check DB connection pool; check Redis |
| Latency p99 spike | > 5,000ms over 10 min | P2 | Check cache hit rate; check LLM API latency; check retrieval index |
| Cost spike | > $10/hour | P3 | Check for runaway conversations; check routing accuracy; check token budget enforcement |
| PII leakage | Any confirmed leak | P1 | See on-call P1 procedure below; notify compliance officer |
| Faithfulness drop | < 0.80 on scheduled eval run | P2 | Re-run eval; check for prompt regression; check retrieval quality |
| Cache hit rate drop | < 20% over 1 hour | P3 | Check for cache invalidation bug; check query distribution shift |
| Escalation rate spike | > 30% over 1 hour | P3 | Check confidence threshold; check if model quality degraded |
| Disk space low | > 85% on any volume | P3 | Check audit log size; check log rotation; clean up old eval results |

### Log Locations

| Log Type | Location | Format | Retention |
|----------|----------|--------|-----------|
| Application logs | Docker: `docker compose logs api` | JSON structured | 30 days (Docker logging driver) |
| Audit logs | PostgreSQL `audit_logs` table | Structured rows | 90 days (GDPR requirement) |
| PII detection logs | PostgreSQL `pii_events` table | Structured rows | 90 days |
| Eval results | `eval_results/` directory | JSON files | Indefinite (versioned) |
| Access logs | Docker: `docker compose logs nginx` (if using) | Nginx format | 14 days |

## 3. Debugging Common Issues

| Symptom | Likely Cause | Debug Steps |
|---------|-------------|-------------|
| High latency on /query | Cache miss + slow LLM API | 1. Check cache hit rate on dashboard; 2. Check OpenAI API status (status.openai.com); 3. Check if retrieval index is healthy |
| Wrong / hallucinated answers | Retrieval failure or prompt regression | 1. Check retrieved chunks in audit log for the specific query; 2. Run faithfulness eval on the response; 3. Check if prompt version changed recently |
| Cost spike | Over-escalation to GPT-4o or runaway conversations | 1. Check routing logs for escalation rate; 2. Check for multi-turn conversations exceeding token budget; 3. Verify `MAX_TOKENS_PER_CONVERSATION` is enforced |
| PII detected in output | Redaction failure in input or output | 1. Trace response through PII scan logs; 2. Check if Presidio model is loaded; 3. Check if output scan layer (`PII_OUTPUT_SCAN`) is enabled; 4. Check the specific PII type that leaked |
| Escalation rate spike | Confidence threshold too high or model degradation | 1. Check confidence score distribution on dashboard; 2. Run eval suite to check for quality regression; 3. Consider temporarily lowering `CONFIDENCE_THRESHOLD` |
| Empty retrieval results | Permission filter too strict or index corruption | 1. Check user roles in JWT token; 2. Check document `allowed_roles`; 3. Check RLS policies; 4. Re-run indexing if index is corrupted |
| Cache returning wrong answers | Similarity threshold too high or cache key collision | 1. Check cache similarity score for the query; 2. Verify `CACHE_SIMILARITY_THRESHOLD` (should be 0.85); 3. Check if intent classification is used as secondary cache key |
| Dashboard not loading | Streamlit process down or DB connection issue | 1. `docker compose ps` — check if dashboard container is running; 2. `docker compose restart dashboard`; 3. Check DB connectivity |

## 4. On-Call Procedures

### P1: PII Leakage (Highest Priority)

```
1. IDENTIFY SCOPE
   - Query audit_logs for responses in the affected time window
   - Search for PII patterns in answer_text column
   - Determine: single response or systemic?

2. STOP THE BLEEDING
   - If systemic: disable /query endpoint
     docker compose stop api
   - If single response: no action needed (already sent), proceed to root cause

3. ROOT CAUSE
   - Trace the specific response through the pipeline:
     input PII scan → retrieval → LLM call → output PII scan → response
   - Identify which layer failed to catch the PII
   - Check Presidio model version and configuration
   - Check if output scan layer was enabled

4. FIX
   - Patch the redaction gap
   - Redeploy: docker compose up -d --build api
   - Verify fix with the specific PII pattern that leaked

5. NOTIFY
   - Notify compliance officer immediately
   - If confirmed breach: GDPR requires notification within 72 hours
   - Document: who, what, when, scope, mitigation

6. POST-MORTEM
   - Write incident report within 48 hours
   - Add regression test for the specific PII pattern
   - Review PII detection layers for similar gaps
```

### P1: System Down

```
1. CHECK CONTAINER STATUS
   docker compose ps
   - Which containers are down/unhealthy?

2. CHECK LOGS
   docker compose logs --tail=100 <failing-service>
   - Look for: OOM, connection refused, migration errors

3. COMMON FIXES
   - Database down:
     docker compose restart db
     Wait for health check, then restart api
   - API down:
     docker compose restart api
     Check /health after restart
   - Redis down:
     docker compose restart redis
     Cache will rebuild on miss (no data loss)
   - LLM API (OpenAI) down:
     Set FALLBACK_PROVIDER=ollama in .env
     docker compose restart api
     Quality will be lower but system stays up

4. VERIFY
   curl http://localhost:8000/health
   ./scripts/smoke_test.sh

5. COMMUNICATE
   - Post status update to team channel
   - If customer-facing: update status page
```

### P2: Quality Degradation

```
1. RUN EVAL SUITE
   docker compose exec api python -m eval.run --suite regression
   - This runs the full regression eval against current deployment

2. COMPARE TO BASELINE
   - Check eval_results/ for last passing run
   - Compare: faithfulness, relevance, citation accuracy, retrieval recall
   - Identify which metric(s) degraded

3. IDENTIFY CAUSE
   - Prompt regression: check if prompt version changed
     → Roll back prompt version in config, redeploy
   - Retrieval regression: check for index corruption
     → Re-index documents: docker compose exec api python -m ingest.reindex
   - Model regression: check if OpenAI model behavior changed
     → Switch to fallback model temporarily

4. FIX AND VERIFY
   - Apply fix (prompt rollback, re-index, or model switch)
   - Re-run eval suite to confirm metrics restored
   - Document in incident log

5. PREVENT
   - Add the failing case to regression test suite
   - Review if eval frequency should increase
```

### P3: Cost Spike

```
1. CHECK COST DASHBOARD
   - Identify which model is driving the spike
   - Check cost per conversation distribution

2. IDENTIFY CAUSE
   - Runaway conversation: check for conversations > 4000 tokens
     → Verify MAX_TOKENS_PER_CONVERSATION enforcement
   - Over-escalation: check routing logs
     → Verify COMPLEXITY_THRESHOLD is correct
   - Cache miss spike: check cache hit rate
     → Check if cache was invalidated or if query distribution shifted

3. MITIGATE
   - Enforce token budget if not already active
   - Temporarily lower COMPLEXITY_THRESHOLD (fewer GPT-4o escalations)
   - Clear and rebuild cache if corrupted

4. ALERT
   - Set up cost-per-conversation alert if not already active
   - Document and review at next sprint
```

## 5. Maintenance Procedures

### Weekly
- [ ] Review eval regression results (automated run)
- [ ] Check cost trend — is spend within budget?
- [ ] Review escalation rate — is the confidence threshold still appropriate?
- [ ] Check disk space — audit logs and eval results grow continuously

### Monthly
- [ ] Review PII detection accuracy — sample 100 responses and verify
- [ ] Update evaluation dataset with new ticket patterns
- [ ] Review cache hit rate trend — adjust threshold if needed
- [ ] Backup audit logs (export to cold storage if > 90 days)
- [ ] Review ADRs — any decisions that should be revisited?

### Quarterly
- [ ] Full security review — penetration test on PII detection
- [ ] Review model choices — are there newer/cheaper/better models?
- [ ] Review retrieval quality — is the document corpus still well-indexed?
- [ ] Update runbook with any new incidents or procedures
```

> The runbook above is the template for `docs/runbook/RUNBOOK.md`. When writing the actual case study, copy this content into the standalone file and embed a condensed version in CASE_STUDY.md §7.

---

## Appendix F: Key Design Decisions

This appendix summarizes the key design decisions that shaped both the Customer Support AI System (Project 11) and this case study (Project 12). These are the decisions that a reader should understand after skimming the case study.

### F.1 System-Level Decisions

| Decision | Choice | Key Tradeoff | ADR |
|----------|--------|-------------|-----|
| Vector database | pgvector (PostgreSQL) | SQL-level permission filtering vs scale ceiling at ~10M vectors | ADR-001 |
| Retrieval strategy | Hybrid BM25 + vector + cross-encoder re-ranking | +12% recall@5 vs pure vector; 1.8x retrieval latency | ADR-002 |
| Escalation policy | Human-in-the-loop for confidence < 0.75 | Reduces wrong-answer risk; adds latency for escalated tickets | ADR-003 |
| PII detection | Presidio (Layer 1) + LLM scan (Layer 2) | Defense in depth; +40ms latency; near-zero leakage | ADR-004 |
| Model selection | GPT-4o-mini primary, GPT-4o escalation | 65% cost reduction vs GPT-4o-only; slight quality regression on simple tickets | ADR-005 |

### F.2 Case Study Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Format | Markdown (not Google Docs/Notion) | Version-controllable, portable, renders anywhere, ADR-ecosystem native |
| ADR format | MADR (individual files) | Industry standard; each ADR is citable and standalone |
| Metrics presentation | Tables + structured chart descriptions | Tables for precision, chart descriptions for recreatability; no lock-in to a visualization tool |
| Failure documentation | 4 incidents with full template | Honesty signals maturity; shows detection/response/learning capability |
| Tone | Technical but accessible | Multiple audience reading paths (§9.1); Executive Summary is non-technical, ADRs are senior-engineer-level |
| Length | 15-20 pages | Long enough for depth, short enough to be read; reading paths let audiences skip to relevant sections |
| Interview integration | Section-to-question mapping (§9.2) | The case study is a talking document, not just a static artifact |

### F.3 What This Case Study Proves

| Capability | How This Case Study Demonstrates It |
|------------|--------------------------------------|
| **Architecture communication** | §2 Architecture Overview with diagram + component descriptions + data flow |
| **Tradeoff analysis** | §3 ADRs with genuine alternatives and honest consequences |
| **Quantitative rigor** | §4 Metrics with baselines, comparisons, and interpretation for every number |
| **Production awareness** | §5 Failure Modes + §7 Operational Runbook |
| **Engineering maturity** | §6 Lessons Learned with self-critique and genuine surprises |
| **Stakeholder communication** | §1 Executive Summary (non-technical) + §4.5 Business Impact (non-technical) |
| **Compliance awareness** | ADR-004 PII detection + §5 Incident 3 (PII leak) + §7.4 GDPR notification procedure |
| **Cost consciousness** | §4.4 Cost analysis + ADR-005 model routing + §5 Incident 4 (cost spike) |
| **Evaluation discipline** | §4.1-4.2 Retrieval and answer quality metrics + §5 Incident 1 (hallucination caught by eval) |
| **Vision** | §8 Future Work with prioritized near/medium/long-term items |

### F.4 The Meta-Lesson

> The ability to communicate AI tradeoffs to stakeholders is rare and valuable. Most AI engineers can build a system; far fewer can explain *why* they built it that way, *what* they would do differently, and *what* they learned when it broke. This case study is what differentiates a senior AI engineer from a junior one.

The case study itself is the proof. A system without a case study is a demo. A system with a case study is a product. A system with a case study that includes honest failures, quantitative metrics, and operational procedures is a **production AI engineering portfolio piece** — and that is the goal of this entire roadmap.

### F.5 Decision Interdependencies

The five ADRs are not independent — they form a coherent architecture where each decision enables or constrains the others:

```
ADR-001 (pgvector) ──enables──→ ADR-002 (hybrid search)
    │                              │
    │                              │ BM25 via pg_trgm/tsvector
    │                              │ in the same database
    │                              ▼
    │                         ADR-005 (model routing)
    │                              │
    │                              │ routing decision feeds into
    │                              │ confidence assessment
    │                              ▼
    └──enables──→ ADR-004 (PII) ──→ ADR-003 (HITL)
                   │                    │
                   │ PII-safe context    │ low confidence triggers
                   │ enables safe        │ human review of
                   │ auto-response       │ PII-redacted draft
                   └────────────────────┘
```

**Key interdependencies:**

- **ADR-001 → ADR-002:** Choosing pgvector meant hybrid search required implementing BM25 within PostgreSQL (via `tsvector` or `pg_trgm`), not as a separate service. This kept the operational footprint small but required more SQL engineering.
- **ADR-004 → ADR-003:** PII redaction must happen *before* the LLM generates a response, and the human-in-the-loop escalation must show the *redacted* version to the human agent. Without ADR-004, ADR-003 would risk exposing PII to human reviewers.
- **ADR-005 → ADR-003:** Model routing determines which model handles the ticket, and the model's confidence score determines whether to escalate. GPT-4o-mini's confidence is less calibrated than GPT-4o's, so the escalation threshold (0.75) was tuned for the mini model's confidence distribution.
- **ADR-001 → ADR-004:** Keeping all data in PostgreSQL means PII detection can query the audit log and PII events table in the same database as the application data, simplifying compliance reporting.

### F.6 Decisions Not Made (Explicitly Deferred)

| Deferred Decision | Why It Was Deferred | When to Revisit |
|-------------------|---------------------|-----------------|
| Migration to Qdrant | Current scale (1.2M vectors) is within pgvector's comfort zone; migration cost not justified yet | When vector count exceeds 5M or retrieval latency degrades |
| Fine-tuning a custom model | Insufficient labeled data; API models are adequate and improving rapidly | When auto-resolution rate plateaus below 50% and labeled data exceeds 10K examples |
| Multi-language support | English-only is sufficient for current user base; German support requires NER model tuning | When expanding to German-speaking markets (roadmap: 3-6 months) |
| On-prem LLM deployment | OpenAI API is reliable and cost-effective; on-prem requires GPU infrastructure investment | When data sovereignty requirements mandate it or cost exceeds $2K/month |
| Real-time streaming responses | Current latency (820ms p50) is acceptable; streaming adds complexity | When latency-sensitive use cases emerge (e.g., voice integration) |

Documenting deferred decisions is as important as documenting made ones. It shows that the team considered these options and made a conscious choice to defer, rather than simply forgetting about them.

---

*End of Implementation Plan — Project 12: Technical Case Study & Runbook*
