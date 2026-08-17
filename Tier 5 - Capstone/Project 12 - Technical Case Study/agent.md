# Herdr Multi-Agent Orchestration Guide — Project 12: Technical Case Study

> **Project type:** Documentation (not code). A comprehensive technical writeup of the Customer Support AI System (Project 11).
> **Timeline:** 5 phases over 5 days.
> **Deliverable:** `CASE_STUDY.md` (15–20 pages) + `docs/adr/` (5 ADRs + template) + `docs/metrics/` (5 category files + chart descriptions) + `docs/presentation/DECK_OUTLINE.md` + `docs/runbook/RUNBOOK.md` + `README.md`.

---

## 1. Agent Roster

| Agent | Kind | Responsibility | Owns |
|-------|------|----------------|------|
| **writer** | codex | Main case study narrative — all 8 sections of `CASE_STUDY.md`, `README.md`, `docs/runbook/RUNBOOK.md` | `CASE_STUDY.md`, `README.md`, `docs/runbook/RUNBOOK.md`, `sources.md` (scratch) |
| **adr** | codex | Architecture Decision Records — MADR template + 5 ADRs, cross-referencing and interdependency documentation | `docs/adr/ADR-000-template.md`, `docs/adr/ADR-001-pgvector-over-qdrant.md`, `docs/adr/ADR-002-hybrid-search-over-pure-vector.md`, `docs/adr/ADR-003-human-in-the-loop-low-confidence.md`, `docs/adr/ADR-004-presidio-llm-dual-pii-detection.md`, `docs/adr/ADR-005-gpt-4o-mini-with-4o-escalation.md` |
| **deck** | codex | Metrics files with chart descriptions + presentation deck outline | `docs/metrics/CHART_DESCRIPTIONS.md`, `docs/metrics/retrieval-quality.md`, `docs/metrics/answer-quality.md`, `docs/metrics/system-performance.md`, `docs/metrics/cost-analysis.md`, `docs/metrics/business-impact.md`, `docs/metrics/exports/README.md`, `docs/presentation/DECK_OUTLINE.md` |
| **reviewer** | codex | Quality gate after each phase — consistency, completeness, AI-slop detection, interview-readiness | Read-only across all files |

---

## 2. Pane Layout

```
┌─────────────────────────────────────────────────────────────┐
│                    Herdr Pane Layout                         │
├──────────────────────┬──────────────────────────────────────┤
│                      │                                      │
│   Pane 1: writer     │   Pane 2: adr                        │
│   CASE_STUDY.md      │   docs/adr/*.md                      │
│   README.md          │   5 ADRs + template                  │
│   docs/runbook/      │                                      │
│                      │                                      │
├──────────────────────┼──────────────────────────────────────┤
│                      │                                      │
│   Pane 3: deck       │   Pane 4: reviewer                   │
│   docs/metrics/      │   Read-only review                   │
│   docs/presentation/ │   Quality gates per phase            │
│                      │                                      │
└──────────────────────┴──────────────────────────────────────┘
```

---

## 3. Setup Commands

```bash
# --- Create directory structure ---
mkdir -p "docs/adr" "docs/metrics/exports" "docs/presentation" "docs/runbook"

# --- Split panes (from project root) ---
herdr pane split --cwd "$PWD" --no-focus    # Pane 2 (adr)
herdr pane split --cwd "$PWD" --no-focus    # Pane 3 (deck)
herdr pane split --cwd "$PWD" --no-focus    # Pane 4 (reviewer)

# --- Start agents ---
herdr agent start writer --kind codex --pane 1
herdr agent start adr --kind codex --pane 2
herdr agent start deck --kind codex --pane 3
herdr agent start reviewer --kind codex --pane 4
```

---

## 4. Phase-by-Phase Orchestration

### Phase 1: Sourcing, Structure & Executive Summary (Day 1)

**Objective:** Gather raw material from Project 11, establish document skeleton, write Executive Summary, create ADR template, scaffold metrics directory.

#### Parallel Work

**writer:**
```
You are writing a technical case study documenting the Customer Support AI System (Project 11). This is a documentation project — no code.

TASK: Create the document skeleton and write §1 Executive Summary.

STEP 1 — Source Material Audit:
Read the IMPLEMENTATION_PLAN.md sections 1.1 (Source Material Audit table) and map each Project 11 source to its target section. Create a scratch file `sources.md` (not in final deliverable) with this mapping. Every section §1–§8 must have at least one identified source.

STEP 2 — Document Skeleton:
Create `CASE_STUDY.md` with:
- Title: "Technical Case Study: Customer Support AI System"
- Subtitle, date, author placeholder
- Table of contents linking all 8 sections
- All 8 section headers with one-sentence placeholders:
  §1 Executive Summary, §2 Architecture Overview, §3 Design Decisions,
  §4 Metrics & Results, §5 Failure Modes & Incidents, §6 Lessons Learned,
  §7 Operational Runbook, §8 Future Work
- Page budget annotations as HTML comments (e.g., `<!-- ~1 page -->`)
- Page budgets: §1=1, §2=2-3, §3=3-4, §4=2-3, §5=2, §6=1-2, §7=2, §8=1 (total 15-20)

STEP 3 — §1 Executive Summary (~1 page, ~400-500 words):
Follow IMPLEMENTATION_PLAN.md §7.1 (Executive Summary Component) exactly. Structure:
- "What We Built" paragraph: plain language, no jargon (no "RAG", "vector search", "embedding")
- "Why It Matters" paragraph: business problem — ticket volume, response time, cost
- "Key Results" bullet list with 5-6 headline metrics (auto-resolution rate, resolution time, cost per ticket, faithfulness, PII detection, monthly spend)
- "Business Value" paragraph: annual savings projection, scalability, CSAT impact

WRITING RULES (from §7.1):
- No acronyms without first-use expansion
- No architecture jargon — use "AI-powered support system" not "RAG pipeline"
- Every number has a comparison (before/after, vs baseline)
- Use realistic non-round numbers (42% not 40%, $0.08 not $0.10)
- Active voice, no AI-slop (avoid: "delve into", "leverage", "in the realm of", "it's worth noting")

Reference: IMPLEMENTATION_PLAN.md §1.3, §7.1, and the metrics in §4.5 Business Impact table for concrete numbers.
```

**adr:**
```
You are writing Architecture Decision Records (ADRs) for a technical case study documenting the Customer Support AI System (Project 11).

TASK: Create the MADR ADR template file.

Create `docs/adr/ADR-000-template.md` following the MADR format from IMPLEMENTATION_PLAN.md Appendix B exactly. The template must include these sections:
- Title: "# ADR-NNN: [Decision Title]"
- ## Status (Accepted | Proposed | Deprecated | Superseded)
- ## Context (2-4 paragraphs: problem, constraints, forces in tension, default action)
- ## Decision (1-2 paragraphs: concrete, unambiguous)
- ## Alternatives Considered (each with Pros/Cons — genuine, not strawmen)
- ## Consequences (Positive, Negative, Neutral — must include negatives)
- ## Compliance Considerations (GDPR etc., omit if not applicable)
- ## Revision History (table: Date, Change, Author)
- Footer: "*Part of the [Technical Case Study](../../CASE_STUDY.md) for the Customer Support AI System.*"

Include HTML comment hints in each section explaining what to write.

Reference: IMPLEMENTATION_PLAN.md Appendix B (lines 1795-1879).
```

**deck:**
```
You are creating metrics documentation files for a technical case study documenting the Customer Support AI System (Project 11).

TASK: Scaffold the docs/metrics/ directory and create the chart inventory.

STEP 1 — Create `docs/metrics/CHART_DESCRIPTIONS.md` with:
- Title and intro explaining this file consolidates all chart specs
- The Chart Inventory table from IMPLEMENTATION_PLAN.md Appendix C.2 (9 charts total):
  | # | Chart | Type | Section | Key Insight |
  Include all 9 rows from the plan.

STEP 2 — Create the Chart Description Template section at the top, following Appendix C.1:
  Every chart must have: Type, Purpose, Axes (X/Y), Series, Data (mini-table), Interpretation, Creation tool.

STEP 3 — Create 5 stub files with headers only (content comes in Phase 3):
- docs/metrics/retrieval-quality.md (§4.1: Recall@5, MRR, nDCG tables + Charts 1-2)
- docs/metrics/answer-quality.md (§4.2: Faithfulness, relevance, citation + Chart 3)
- docs/metrics/system-performance.md (§4.3: Latency percentiles, throughput + Charts 4-5)
- docs/metrics/cost-analysis.md (§4.4: Cost per ticket, spend breakdown + Charts 6-7)
- docs/metrics/business-impact.md (§4.5: Auto-resolution, time saved + Charts 8-9)

Each stub: title, one-line description, "## Tables" and "## Chart Descriptions" headers.

STEP 4 — Create `docs/metrics/exports/README.md` explaining how to generate chart PNGs (screenshot Streamlit dashboard, matplotlib, or Grafana export — see Appendix C.4).

Reference: IMPLEMENTATION_PLAN.md Appendix C (lines 1883-1960).
```

#### Sequential (after parallel)

**writer:**
```
Verify the skeleton: all 8 sections present with headers, TOC links work, directory structure matches §5 Project Structure. Confirm page budget annotations sum to 15-20 pages. Then verify §1 Executive Summary against the Phase 1 checklist (§1.3 Verification):
- Readable by non-technical person (no jargon without explanation)
- Contains at least 5 specific quantitative metrics
- Fits on 1 page when rendered
- Answers: what, why, so what
- No acronyms without first-use expansion
```

#### Review

**reviewer:**
```
Review Phase 1 deliverables against IMPLEMENTATION_PLAN.md verification checklists:

1. CASE_STUDY.md skeleton (§1.2 Verification):
   - All 8 sections present with headers
   - Table of contents links present
   - Page budgets sum to 15-20 pages

2. §1 Executive Summary (§1.3 Verification):
   - Readable by non-technical person
   - At least 5 specific quantitative metrics
   - Fits ~1 page
   - No acronyms without expansion
   - No AI-slop phrases (check against §10.2 Check 8 list)

3. docs/adr/ADR-000-template.md:
   - Follows MADR format from Appendix B
   - All required sections present
   - Footer links back to CASE_STUDY.md

4. docs/metrics/ scaffolding:
   - CHART_DESCRIPTIONS.md has all 9 charts in inventory table
   - 5 category stub files created with correct headers
   - exports/README.md present

Report any issues to the relevant agent via hub messaging. Do not edit files.
```

#### Commit

```bash
git add -A && git commit -m "Phase 1: document skeleton, executive summary, ADR template, metrics scaffolding"
```

---

### Phase 2: Architecture Overview & ADRs (Day 2)

**Objective:** Write §2 Architecture Overview and all 5 ADRs. Writer and adr work in parallel (no file conflicts). Deck continues metrics prep.

#### Parallel Work

**writer:**
```
TASK: Write §2 Architecture Overview (~2-3 pages, ~1,200-1,800 words) in CASE_STUDY.md.

Follow IMPLEMENTATION_PLAN.md §2.1 and §7.2 (Architecture Overview Component) exactly.

Include 4 parts:

1. SYSTEM DIAGRAM — ASCII diagram (max 80 chars wide) showing:
   - All components: FastAPI, pgvector, Redis, LLM gateway, PII guardrails, eval harness, monitoring
   - Data flow: ingestion → query → retrieval → generation → post-processing → response
   - External dependencies: OpenAI API, Presidio, spaCy NER
   - Human-in-the-loop escalation path
   Test that it renders in monospace font.

2. COMPONENT DESCRIPTIONS — 1 paragraph per component (8 components):
   - Ingestion pipeline (document parsing, chunking, embedding, storage)
   - Query pipeline (query embedding, hybrid retrieval, re-ranking, context assembly)
   - Answer generation (LLM call, citation extraction, faithfulness check)
   - PII guardrails (input redaction, output scanning, audit logging)
   - LLM gateway (semantic cache, model routing, cost tracking, rate limiting)
   - Evaluation harness (LLM-as-judge, regression tests, CI integration)
   - Monitoring (Streamlit dashboard, alerting, log aggregation)
   - Human-in-the-loop escalation (confidence threshold, agent UI handoff)
   Each paragraph: what it does, key technology, why this tech, interfaces.

3. DATA FLOW WALKTHROUGH — step-by-step numbered list:
   Query arrives → PII scan → cache check → embedding → hybrid retrieval →
   re-ranking → context assembly → LLM call → citation validation →
   output PII scan → confidence check → respond or escalate → audit log

4. TECHNOLOGY RATIONALE TABLE:
   | Component | Technology | Why | ADR |
   |-----------|-----------|-----|-----|
   | Vector search | pgvector | SQL-level permission filtering | ADR-001 |
   | Retrieval | Hybrid BM25+vector | Keyword precision for support docs | ADR-002 |
   | Escalation | Confidence threshold 0.75 | Wrong answers erode trust | ADR-003 |
   | PII detection | Presidio + LLM | Defense in depth | ADR-004 |
   | Model selection | GPT-4o-mini + 4o escalation | 65% cost reduction | ADR-005 |

WRITING RULES (from §7.2):
- Diagram first, then prose
- Every component in the diagram has a description
- Defer deep rationale to ADRs — this section is the map, ADRs are the territory
- Consistent naming (component names in diagram match prose)
- Active voice, no AI-slop

Reference: IMPLEMENTATION_PLAN.md §2.1, §7.2.
```

**adr:**
```
TASK: Write all 5 Architecture Decision Records using the MADR template from docs/adr/ADR-000-template.md.

Write each ADR as a standalone file. Each must be self-contained (readable without the case study). Follow the template exactly: Status, Context, Decision, Alternatives Considered (≥2 with genuine pros/cons), Consequences (positive + negative + neutral), Compliance (if relevant), Revision History, footer link.

ADR-001: docs/adr/ADR-001-pgvector-over-qdrant.md
- Context: System needs vector search + per-tenant permission filtering (hard security requirement)
- Decision: pgvector (PostgreSQL extension)
- Alternatives: Qdrant (purpose-built but separate metadata store + sync problem), Pinecone (vendor lock-in, data leaves infra — GDPR concern), Weaviate (same sync problem, less mature)
- Consequences +: atomic SQL query for permission filtering, single DB to operate, RLS as defense-in-depth
- Consequences -: HNSW less tunable, scale ceiling ~10M vectors, no built-in hybrid search
- Compliance: Keeping data in PostgreSQL means PII never leaves infrastructure (GDPR)
- Full content provided in IMPLEMENTATION_PLAN.md §2.2 (lines 437-485)

ADR-002: docs/adr/ADR-002-hybrid-search-over-pure-vector.md
- Context: Support docs have exact-match terminology (error codes, product names, policy numbers) where keyword search outperforms semantic
- Alternatives: pure vector, pure BM25, hybrid without re-ranking
- Decision: BM25 + vector fusion (RRF) + cross-encoder re-ranking
- Consequences: +12% recall@5 vs pure vector; 1.8x retrieval latency; requires maintaining two indexes
- Reference: §2.3 (lines 500-504)

ADR-003: docs/adr/ADR-003-human-in-the-loop-low-confidence.md
- Context: LLM confidence not always calibrated; wrong answers erode trust faster than "I don't know"
- Alternatives: always auto-respond, always human-review, threshold-based escalation
- Decision: Confidence threshold 0.75 → below threshold, route to human with retrieved context as draft
- Consequences: reduces wrong-answer risk; adds latency for escalated tickets; requires agent UI; trust calibration is ongoing
- Reference: §2.3 (lines 506-510)

ADR-004: docs/adr/ADR-004-presidio-llm-dual-pii-detection.md
- Context: Support tickets contain PII (names, emails, phone, order IDs, addresses); GDPR requires detection and redaction
- Alternatives: Presidio only, LLM only, regex only
- Decision: Layer 1 = Presidio (NER + regex, fast, deterministic) → Layer 2 = LLM scan (catches context-dependent PII)
- Consequences: defense in depth; +40ms latency; near-zero PII leakage; small false-positive rate (over-redaction safer than leakage)
- Compliance: GDPR — dual-layer approach ensures PII detection at every input/output boundary
- Reference: §2.3 (lines 512-516)

ADR-005: docs/adr/ADR-005-gpt-4o-mini-with-4o-escalation.md
- Context: Cost optimization — most support questions are routine
- Alternatives: GPT-4o for all, GPT-4o-mini for all, open-source (Llama 3.1) for all
- Decision: GPT-4o-mini handles 80% (classification + simple RAG); GPT-4o handles 20% (complex reasoning); routing based on complexity classification
- Consequences: 65% cost reduction vs GPT-4o-only; slight quality regression on simple tickets; routing logic adds complexity
- Reference: §2.3 (lines 518-522)

CROSS-REFERENCING (from Appendix F.5):
- ADR-005 references ADR-003 for escalation logic
- ADR-001 enables ADR-002 (BM25 via tsvector in same DB)
- ADR-004 enables ADR-003 (PII-redacted context for human review)
- ADR-001 enables ADR-004 (audit log + PII events in same DB)

WRITING RULES (from §7.3):
- Alternatives must be genuine (not strawmen) — every alternative has some pros
- Consequences must include negatives — no decision is perfect
- Past tense for context, present tense for decision and consequences
- Each ADR is self-contained
```

**deck:**
```
TASK: Populate docs/metrics/CHART_DESCRIPTIONS.md with full chart description templates for all 9 charts.

For each of the 9 charts from the inventory table, write a full structured description following the template from Appendix C.1:
- Type, Purpose, Axes (X/Y with units), Series, Data (mini-table), Interpretation, Creation tool

Use the chart data from IMPLEMENTATION_PLAN.md §3.1 (Phase 3 Metrics section):

Chart 1: Retrieval Strategy Comparison (grouped bar) — §4.1
  Data from the 4-strategy table: Vector only, BM25 only, Hybrid (RRF), Hybrid+re-ranking
  Groups: Recall@5, MRR, nDCG@5

Chart 2: Recall@5 by Query Type (line) — §4.1
  X: FAQ, Policy, Troubleshooting, Mixed; Lines: one per strategy

Chart 3: Answer Quality Radar — §4.2
  5 axes: Faithfulness, Relevance, Citation Accuracy, Citation Coverage, Human Eval
  3 polygons: GPT-4o-mini, GPT-4o, Llama 3.1 8B

Chart 4: Latency Percentile Waterfall (stacked bar) — §4.3
  X: /query cache miss, /query cache hit, /classify, /escalate
  Stacked: p50, p50→p90 delta, p90→p99 delta

Chart 5: Latency Breakdown (horizontal bar) — §4.3
  Segments: PII scan 40ms, embedding 60ms, retrieval 112ms, LLM call 550ms, citation validation 30ms, output PII scan 28ms

Chart 6: Monthly Cost Trend (line) — §4.4
  X: 6 months; Lines: Total spend, spend without cache, baseline (without cache+routing)

Chart 7: Cost per Ticket by Model (pie) — §4.4
  Segments: GPT-4o-mini 72%, GPT-4o 22%, Embeddings 4%, Other 2%

Chart 8: Resolution Funnel — §4.5
  Stages: 4,200 → 1,764 auto-resolved → 756 escalated with draft → 1,680 without draft → 2,436 human-resolved

Chart 9: CSAT Before/After (diverging bar) — §4.5
  X: CSAT 1-5; Two bars per score: before (red, left) and after (green, right)

Reference: IMPLEMENTATION_PLAN.md §3.1 (lines 539-647) for all data, Appendix C.1 for format.
```

#### Sequential (after parallel)

**writer:**
```
TASK: Write §3 Design Decisions section in CASE_STUDY.md (~3-4 pages summary).

Now that all 5 ADRs exist in docs/adr/, write §3 as a summary that:
1. For each ADR (1-2 paragraphs): one-sentence decision summary + key tradeoff (what we gained vs what we gave up) + link to full ADR file
2. Summary table:
   | ADR | Decision | Key Tradeoff |
   |-----|----------|--------------|
   | ADR-001 | pgvector over Qdrant | SQL filtering vs scale ceiling |
   | ADR-002 | Hybrid search | +12% recall vs 1.8x latency |
   | ADR-003 | Human-in-the-loop | Reduces wrong-answer risk vs added latency |
   | ADR-004 | Dual-layer PII | Defense in depth vs +40ms latency |
   | ADR-005 | Model routing | 65% cost reduction vs routing complexity |
3. Include the ADR interdependency diagram from Appendix F.5 showing how ADRs enable/constrain each other

Link format: [ADR-001: pgvector over Qdrant](docs/adr/ADR-001-pgvector-over-qdrant.md)

Reference: IMPLEMENTATION_PLAN.md §7.3 (ADR Component), Appendix F.5 (Decision Interdependencies).
```

#### Review

**reviewer:**
```
Review Phase 2 deliverables:

1. §2 Architecture Overview (§2.1 Verification):
   - System diagram readable in monospace (max 80 chars wide)
   - Every component in diagram has a description
   - Data flow walkthrough covers full request lifecycle
   - Technology rationale table references relevant ADR for each choice

2. ADRs (§2.2 + §2.3 Verification):
   - Each ADR follows MADR template exactly
   - Each has ≥2 alternatives with genuine pros/cons (not strawmen)
   - Consequences include both positive AND negative
   - Tradeoffs are honest (not just "our choice is best")
   - ADRs cross-reference each other (ADR-005→ADR-003, ADR-001→ADR-002, etc.)
   - Each ADR is self-contained (readable without case study)
   - Footer links back to CASE_STUDY.md

3. §3 Design Decisions:
   - Summarizes all 5 ADRs with links to full files
   - Summary table present with key tradeoffs
   - Interdependency diagram included

4. CHART_DESCRIPTIONS.md:
   - All 9 charts have full structured descriptions
   - Each has: Type, Purpose, Axes, Series, Data, Interpretation, Creation

5. Run validation Check 3 (ADR Completeness) from §10.2 on each ADR file.
6. Run AI-slop detection (Check 8) on all new content.

Report issues to relevant agents. Do not edit files.
```

#### Commit

```bash
git add -A && git commit -m "Phase 2: architecture overview, 5 ADRs, design decisions summary, chart descriptions"
```

---

### Phase 3: Metrics & Results + Failure Modes (Day 3)

**Objective:** Write §4 Metrics & Results (~2-3 pages) and §5 Failure Modes & Incidents (~2 pages). Writer handles CASE_STUDY.md sections; deck populates per-category metrics files in parallel.

#### Parallel Work

**writer:**
```
TASK: Write §4 Metrics & Results and §5 Failure Modes & Incidents in CASE_STUDY.md.

=== §4 METRICS & RESULTS (~2-3 pages) ===
Follow IMPLEMENTATION_PLAN.md §3.1 and §7.4 (Metrics Component). Organize into 5 subsections, each with tables and chart description references.

§4.1 Retrieval Quality (technical audience):
  Table — 4 strategies × 6 metrics (from §3.1 lines 547-553):
  | Strategy | Recall@5 | Recall@10 | MRR | nDCG@5 | nDCG@10 | Latency p50 (ms) |
  | Vector only | 0.68 | 0.79 | 0.54 | 0.61 | 0.68 | 45 |
  | BM25 only | 0.61 | 0.72 | 0.48 | 0.55 | 0.62 | 22 |
  | Hybrid (RRF) | 0.79 | 0.88 | 0.67 | 0.74 | 0.81 | 58 |
  | Hybrid + re-ranking | 0.84 | 0.91 | 0.74 | 0.81 | 0.86 | 112 |
  Reference Charts 1-2 (link to docs/metrics/retrieval-quality.md).
  1-paragraph interpretation.

§4.2 Answer Quality (technical audience):
  Table — 3 models × 5 metrics (from §3.1 lines 569-573):
  | Model | Faithfulness | Relevance | Citation Accuracy | Citation Coverage | Human Eval (1-5) |
  | GPT-4o-mini | 0.87 | 0.91 | 0.94 | 0.82 | 4.1 |
  | GPT-4o | 0.93 | 0.95 | 0.97 | 0.89 | 4.5 |
  | Llama 3.1 8B | 0.79 | 0.84 | 0.88 | 0.71 | 3.6 |
  Reference Chart 3. 1-paragraph interpretation.

§4.3 System Performance (technical/SRE):
  Table — 4 endpoints × 6 metrics (from §3.1 lines 583-588):
  | Endpoint | p50 (ms) | p90 (ms) | p99 (ms) | Throughput (rps) | Error Rate |
  | /query (cache miss) | 820 | 1,400 | 2,800 | 12 | 0.3% |
  | /query (cache hit) | 45 | 80 | 150 | 180 | 0.1% |
  | /classify | 120 | 220 | 450 | 85 | 0.2% |
  | /escalate | 90 | 160 | 320 | 50 | 0.0% |
  Reference Charts 4-5. 1-paragraph interpretation.

§4.4 Cost (mixed audience):
  Table — 8 cost metrics (from §3.1 lines 602-611):
  Monthly LLM spend $340, embedding $28, cost per auto-resolved ticket $0.08,
  cost per human-resolved $4.50, cache hit rate 34%, cache savings $85,
  routing savings $180, total savings vs baseline $265.
  Reference Charts 6-7. 1-paragraph interpretation.

§4.5 Business Impact (non-technical audience):
  Before/after table — 8 metrics (from §3.1 lines 626-636):
  Total tickets 4,200, auto-resolution 0%→42%, avg resolution 6.2h→2.1h,
  human hours 1,250→725, savings $0→$5,250, CSAT 3.8→4.2, escalation 18%.
  Reference Charts 8-9. 1-paragraph interpretation (plain language, no "nDCG").

CHART DESCRIPTIONS: In CASE_STUDY.md, include a brief chart description inline for each chart AND link to the full spec in docs/metrics/. Use the format from Appendix C.1.

WRITING RULES (from §7.4):
- Every metric has a baseline or comparison
- No metric without interpretation ("so what?")
- Tables before charts (tables for precision, charts for intuition)
- §4.5 uses plain language (no "nDCG" here)
- Numbers are realistic (42% not 40%, $0.08 not $0.10)
- Verify internal consistency: auto_resolution_rate × total_tickets = auto_resolved (0.42 × 4200 = 1764 ✓)

=== §5 FAILURE MODES & INCIDENTS (~2 pages) ===
Follow IMPLEMENTATION_PLAN.md §3.2 and §7.5 (Failure Modes Component).

Intro paragraph: "Production AI systems fail. What matters is how you detect, respond, and learn." Set expectation: 4 incidents, P1 to P3.

Document 4 incidents using this template per incident:
  ### Incident N: [Title]
  **Date:** [approximate] | **Severity:** [P1/P2/P3] | **Detection:** [how discovered]
  #### What Happened (2-3 sentences)
  #### Impact (quantified)
  #### Root Cause (technical)
  #### Timeline (T+0, T+Xm, T+Ym, T+Zm)
  #### Fix (immediate)
  #### Preventive Measure (systemic — code/config/process, not personal)
  #### Lesson (1 sentence)

Incident 1: LLM Hallucinated Non-Existent Return Policy (P2)
  Detection: Citation validation flagged response where cited chunk didn't contain claimed policy
  Root cause: LLM extrapolated from training data when context was ambiguous
  Fix: Added faithfulness eval to response pipeline; failing responses re-generated or escalated
  Preventive: Faithfulness eval in CI regression; "if context doesn't contain answer, say you don't know" in system prompt
  Lesson: Citation validation is not just for user trust — it's a hallucination detector
  Full details: §3.2 lines 693-699

Incident 2: Semantic Cache Returned Wrong Answer (P2)
  Detection: User reported incorrect answer; cache hit on 0.92 cosine similarity but different intent
  Root cause: Cache threshold too high (0.90); similar phrasing, different products
  Fix: Lowered threshold to 0.85; added intent classification as secondary cache key
  Preventive: Cache hit quality monitoring; cache eval in regression suite
  Lesson: Semantic caching trades accuracy for speed; threshold needs monitoring
  Full details: §3.2 lines 701-707

Incident 3: PII Leaked Through LLM Output (P1)
  Detection: Audit log review found response with customer phone number not in retrieved context but in user query (LLM echoed it)
  Root cause: Input PII redaction applied to retrieved context but NOT to user query text in prompt
  Fix: Applied PII redaction to user query before prompt construction; added output PII scanning (Layer 2 of ADR-004)
  Preventive: PII leakage test in regression; output scanning as mandatory pipeline stage
  Lesson: PII redaction must be applied at every point where user input enters the system
  Full details: §3.2 lines 709-715

Incident 4: Cost Spike from Complex Troubleshooting Tickets (P3)
  Detection: Cost dashboard showed 3x daily spend spike; cluster of complex tickets routed to GPT-4o with long multi-turn conversations
  Root cause: Complexity classifier over-escalating; multi-turn conversations accumulating token costs
  Fix: Added per-conversation token budget (4,000 tokens); conversations exceeding budget summarized and continued fresh; tuned classifier threshold
  Preventive: Cost-per-conversation alerting (>$0.50); budget enforcement in LLM gateway
  Lesson: Cost control is a first-class production concern
  Full details: §3.2 lines 717-723

Closing paragraph: Pattern across incidents (3 of 4 caught by eval/monitoring, not user reports — shows observability investment paid off).

WRITING RULES (from §7.5):
- No blame (use "the system" not "I made a mistake")
- Be specific about detection (which alert? which eval? which log?)
- Timelines show urgency but not panic
- Preventive measures are systemic (code/config/process), not personal
- Include at least one P1 (Incident 3)
- Include at least one "caught by eval/monitoring" (Incidents 1, 2, 4)

Reference: IMPLEMENTATION_PLAN.md §3.1, §3.2, §7.4, §7.5.
```

**deck:**
```
TASK: Populate the 5 per-category metrics files in docs/metrics/ with full tables and chart descriptions.

Each file must be self-contained (readable without CASE_STUDY.md). Include the actual data tables and full chart descriptions.

1. docs/metrics/retrieval-quality.md:
   - §4.1 Retrieval Quality tables (4 strategies × 6 metrics)
   - Chart 1: Retrieval Strategy Comparison (full spec with data mini-table)
   - Chart 2: Recall@5 by Query Type (full spec)
   - Interpretation paragraph

2. docs/metrics/answer-quality.md:
   - §4.2 Answer Quality table (3 models × 5 metrics)
   - Chart 3: Answer Quality Radar (full spec with data)
   - Interpretation paragraph

3. docs/metrics/system-performance.md:
   - §4.3 System Performance table (4 endpoints × 6 metrics)
   - Chart 4: Latency Percentile Waterfall (full spec)
   - Chart 5: Latency Breakdown (full spec with segment data: PII 40ms, embedding 60ms, retrieval 112ms, LLM 550ms, citation 30ms, output PII 28ms)
   - Interpretation paragraph

4. docs/metrics/cost-analysis.md:
   - §4.4 Cost table (8 metrics)
   - Chart 6: Monthly Cost Trend (full spec)
   - Chart 7: Cost per Ticket by Model (pie, full spec with segments)
   - Interpretation paragraph

5. docs/metrics/business-impact.md:
   - §4.5 Business Impact before/after table (8 metrics)
   - Chart 8: Resolution Funnel (full spec with stage data)
   - Chart 9: CSAT Before/After (full spec)
   - Interpretation paragraph (plain language, no jargon)

Use the Chart Description Template from Appendix C.1 for every chart:
Type, Purpose, Axes (X/Y with units), Series, Data (mini-table), Interpretation, Creation tool.

All data is in IMPLEMENTATION_PLAN.md §3.1 (lines 539-647). Copy exact numbers — do not round or change values.

Reference: IMPLEMENTATION_PLAN.md §3.1, §4.4a, Appendix C.
```

#### Sequential (after parallel)

No sequential work needed — writer and deck work on different files.

#### Review

**reviewer:**
```
Review Phase 3 deliverables:

1. §4 Metrics & Results (§3.1 Verification):
   - Every metric has a baseline and actual (or before/after)
   - Every chart has a structured description (type, axes, data, interpretation)
   - Tables are properly formatted Markdown
   - Numbers are internally consistent (verify: 0.42 × 4200 = 1764; 4200 - 1764 = 2436; 2436 × $4.50 ≈ human cost; etc.)
   - §4.5 Business Impact is non-technical enough for a VP
   - No metric without context ("so what?")
   - Run validation Check 5 (Chart Description Completeness) from §10.2

2. §5 Failure Modes (§3.2 Verification):
   - At least 4 incidents documented
   - Each follows template: What Happened, Impact, Root Cause, Timeline, Fix, Preventive, Lesson
   - Incidents are honest (no "we had no issues")
   - Each has concrete preventive measure (not "we'll be more careful")
   - Severity ratings realistic (not everything P1)
   - At least one P1 (Incident 3)
   - Run validation Check 4 (Incident Completeness) from §10.2

3. docs/metrics/ files:
   - All 5 category files populated with tables + chart descriptions
   - Each file self-contained
   - Numbers match CASE_STUDY.md §4 exactly (cross-check)

4. AI-slop detection on all new content.

Report issues to relevant agents. Do not edit files.
```

#### Commit

```bash
git add -A && git commit -m "Phase 3: metrics & results, failure modes, per-category metrics files"
```

---

### Phase 4: Lessons, Runbook, Future Work + Supporting Deliverables (Day 4)

**Objective:** Write §6 Lessons Learned, §7 Operational Runbook, §8 Future Work, and all supporting deliverables (runbook file, deck outline). Writer and deck work in parallel.

#### Parallel Work

**writer:**
```
TASK: Write §6 Lessons Learned, §7 Operational Runbook, §8 Future Work in CASE_STUDY.md, and create docs/runbook/RUNBOOK.md + README.md.

=== §6 LESSONS LEARNED (~1-2 pages) ===
Follow IMPLEMENTATION_PLAN.md §4.1 and §7.6. Three subsections:

§6.1 What I'd Do Differently (5 items):
  Format: [What I'd change] — [why original was suboptimal] — [better approach]
  1. Started with evaluation earlier — built system first, retrofitted eval; eval should be built alongside
  2. Chosen dedicated vector DB if scale >5M vectors — pgvector was right at the time but migration is now priority
  3. Built human-in-the-loop UI earlier — escalation was an afterthought; should be first-class from day one
  4. Invested in prompt versioning from start — ad-hoc prompt changes made regression testing harder
  5. Made semantic cache key more sophisticated — single embedding similarity too coarse; intent + similarity would have prevented Incident 2

§6.2 What Surprised Me (5 items):
  Format: [Surprise] — [why surprising] — [what it taught you]
  1. 70% of production AI is plumbing — caching, routing, guardrails, monitoring, cost control, error handling
  2. How often LLM was "confidently wrong" — faithfulness eval caught things I'd never notice manually
  3. How much caching moved the cost needle — 34% cache hit rate saved more than model routing
  4. How hard PII detection is — regex catches easy stuff, context-dependent PII needs LLM-level understanding
  5. How fast user expectations escalate — once auto-resolution hit 40%, ask became "why not 60%?"

§6.3 Advice for Others (5 items):
  Format: [Advice] — [brief rationale]
  1. Build evaluation before features — if you can't measure it, you can't improve or deploy it
  2. Treat prompts as code — version, diff, test, review them
  3. Budget for failure — system will hallucinate, leak, spike costs; design for these
  4. Monitor cost like you monitor latency — cost spikes are production incidents
  5. The case study matters more than the code — communication is the senior engineer's multiplier

Tone: reflective, honest, conversational (contractions OK in this section).
No generic platitudes. Every item specific to this project.

=== §7 OPERATIONAL RUNBOOK (~2 pages) ===
Follow IMPLEMENTATION_PLAN.md §4.2, §7.7, and Appendix E. Four subsections:

§7.1 Deployment:
  - Prerequisites (Docker 24+, OpenAI API key, PostgreSQL 16 + pgvector, Redis 7+)
  - Deploy steps (numbered, copy-pasteable: clone, cp .env.example, docker compose build, alembic upgrade, up -d, curl /health, smoke test, verify dashboards)
  - Rollback steps (docker compose down, git checkout, up -d --build, alembic downgrade -1, verify)
  - Include .env.example with all env vars from Appendix E (POSTGRES_*, REDIS_*, OPENAI_API_KEY, MODEL_PRIMARY, MODEL_ESCALATION, FALLBACK_PROVIDER, EMBEDDING_MODEL, PRESIDIO_ENABLED, PII_OUTPUT_SCAN, CACHE_SIMILARITY_THRESHOLD=0.85, CACHE_TTL, COMPLEXITY_THRESHOLD=0.7, CONFIDENCE_THRESHOLD=0.75, MAX_TOKENS_PER_CONVERSATION=4000, DASHBOARD_PORT, ALERT_WEBHOOK, GDPR_LOGGING, BREACH_NOTIFICATION_WINDOW_H=72)

§7.2 Monitoring:
  - Dashboard table (System Health, Cost Tracker, Quality Monitor, PII Audit — URLs and key panels)
  - Key alerts table (8 alerts with threshold, severity, action — from Appendix E lines 2209-2218)
  - Log locations table (5 log types with location, format, retention — from Appendix E lines 2222-2228)

§7.3 Debugging Common Issues:
  - Table: 8 symptoms → likely cause → debug steps (from Appendix E lines 2232-2241)

§7.4 On-Call Procedures:
  - P1: PII Leakage (6 steps: identify scope, stop bleeding, root cause, fix, notify — GDPR 72h, post-mortem)
  - P1: System Down (5 steps: check containers, check logs, common fixes, verify, communicate)
  - P2: Quality Degradation (5 steps: run eval suite, compare baseline, identify cause, fix, prevent)
  - P3: Cost Spike (4 steps: check dashboard, identify cause, mitigate, alert)

Also include §5 Maintenance Procedures (weekly/monthly/quarterly checklists from Appendix E lines 2373-2390).

=== §8 FUTURE WORK (~1 page) ===
Follow IMPLEMENTATION_PLAN.md §4.3 and §7.8. Prioritized list:

Near-term (1-3 months): 4 items
  - Migrate to Qdrant if scale >5M vectors (ADR-001 revisit)
  - Fine-tune Llama 3.1 8B on support conversations for cost reduction
  - Build proper human agent UI for escalated tickets
  - Implement conversation summarization for multi-turn tickets

Medium-term (3-6 months): 4 items
  - Multi-language support (German first — primary market)
  - Active learning: use escalated tickets as training data
  - Feedback loop: thumbs up/down → eval dataset expansion
  - A/B testing framework for prompt versions (Project 8 extension)

Long-term (6-12 months): 4 items
  - On-prem LLM deployment (Llama 3.1 70B via vLLM) for data sovereignty
  - Proactive support: detect issues from ticket patterns before customers report
  - Support knowledge graph: entities, relationships, automated FAQ generation
  - CRM integration for customer context (order history, previous tickets)

With more time: 4 items (blue-sky)
  - Full evaluation platform with human-in-the-loop annotation UI
  - Automated prompt optimization using DSPy
  - Cost prediction model for capacity planning
  - Support copilot mode assisting human agents in real-time

Format per item: [What] — [brief rationale or connection to current limitation]
No generic items like "add more tests" or "improve monitoring".

=== STANDALONE RUNBOOK ===
Create docs/runbook/RUNBOOK.md with the full runbook content from §7 (including maintenance procedures). This is the standalone version — CASE_STUDY.md §7 has a condensed version. Add header: "# Operational Runbook: Customer Support AI System" and note that it's also embedded in CASE_STUDY.md §7.

=== README.md ===
Create README.md following IMPLEMENTATION_PLAN.md §11.2 exactly:
- Title, description
- "How to Read This Case Study" table (audience → read → time)
- Deliverables list
- Context (links to Project 11)
- License: MIT

Reference: IMPLEMENTATION_PLAN.md §4.1-4.3, §7.6-7.8, Appendix E, §11.2.
```

**deck:**
```
TASK: Write docs/presentation/DECK_OUTLINE.md — slide-by-slide outline for a 15-minute talk.

Follow IMPLEMENTATION_PLAN.md Appendix D (lines 1964-2103) EXACTLY. This is the full deck outline with 18 slides:

Slide 1: Title — "Building a Customer Support AI System: Architecture, Tradeoffs, and What Broke"
Slide 2: The Problem (30 sec) — ticket volume, resolution time, cost
Slide 3: What We Built (30 sec) — 42% auto-resolution, 6.2h→2.1h
Slide 4: System Architecture (2 min) — diagram from §2, query lifecycle walkthrough
Slide 5: Why pgvector? (1 min) — ADR-001, SQL filtering vs scale ceiling
Slide 6: Why Hybrid Search? (1 min) — ADR-002, +12% recall, 1.8x latency
Slide 7: Model Routing (1 min) — ADR-005, 65% cost reduction
Slide 8: PII Protection (1 min) — ADR-004, two-layer pipeline
Slide 9: Human-in-the-Loop (1 min) — ADR-003, confidence threshold 0.75
Slide 10: Retrieval Quality Results (1 min) — Charts 1-2, 84% recall@5
Slide 11: Answer Quality Results (1 min) — Chart 3, faithfulness 0.87/0.93
Slide 12: System Performance (1 min) — Charts 4-5, p50 820ms/45ms
Slide 13: Cost (1 min) — Chart 6, $340/month, $265 savings
Slide 14: Business Impact (1 min) — Chart 8, 42% auto-resolution, $5,250/month
Slide 15: What Broke (2 min) — 4 incidents, 1 P1, hallucination + PII leak
Slide 16: Lessons Learned (1 min) — 4 key bullets
Slide 17: What's Next (30 sec) — German, fine-tuning, copilot
Slide 18: Thank You / Q&A (30 sec)

Include the Presentation Notes section:
- Timing breakdown (total 15 min: setup 1.5, architecture 7, results 5, failures+lessons 3, close 1)
- Audience adaptation (technical vs business vs interview)
- Delivery tips (architecture diagram is most important slide; failure section differentiates you)

Each slide must specify:
- Slide number and title
- Time allocation
- Key content points (bullets)
- Visual element (chart reference, diagram, or "text only")

Reference: IMPLEMENTATION_PLAN.md Appendix D (lines 1964-2103). Copy the structure exactly.
```

#### Sequential (after parallel)

No sequential work — writer and deck own separate files.

#### Review

**reviewer:**
```
Review Phase 4 deliverables:

1. §6 Lessons Learned (§4.1 Verification):
   - Goes beyond "it worked great" — includes genuine self-critique
   - "What surprised me" shows real learning, not post-hoc rationalization
   - Advice is specific and actionable, not generic platitudes
   - Tone is reflective and honest, not defensive
   - 5 items per subsection

2. §7 Operational Runbook (§4.2 Verification):
   - Deploy steps reproducible (someone could follow without asking questions)
   - Monitoring section covers key alerts with thresholds and actions
   - Debugging table covers most common issues (8 entries)
   - On-call procedures include at least one P1 scenario
   - PII leakage procedure mentions GDPR 72h notification requirement
   - .env.example includes all env vars from Appendix E

3. §8 Future Work (§4.3 Verification):
   - Items prioritized (near/medium/long-term)
   - Each item has brief rationale
   - Shows vision without overpromising
   - Connects to current limitations
   - No generic "add more tests" items

4. docs/runbook/RUNBOOK.md:
   - Consistent with §7 of CASE_STUDY.md
   - Self-contained (readable without case study)

5. docs/presentation/DECK_OUTLINE.md:
   - 15-20 slides outlined (should be 18)
   - Each slide has time allocation, content, visual element
   - Timing adds up to ~15 minutes
   - Presentation notes included (timing, audience adaptation, delivery tips)

6. README.md:
   - Reading guide table present
   - Deliverables listed
   - Links to CASE_STUDY.md as primary deliverable

7. AI-slop detection on all new content.

Report issues to relevant agents. Do not edit files.
```

#### Commit

```bash
git add -A && git commit -m "Phase 4: lessons learned, operational runbook, future work, deck outline, README"
```

---

### Phase 5: Review, Polish & Final Assembly (Day 5)

**Objective:** Full document review for consistency, quality, interview-readiness. Polish prose, fix cross-references, spell check, PDF export.

#### Parallel Work

**writer:**
```
TASK: Full document review and polish of CASE_STUDY.md.

Read CASE_STUDY.md end-to-end and fix:

CONSISTENCY (from §5.1):
- Metrics numbers consistent across sections (auto-resolution rate in §1 matches §4.5 matches §5)
- ADR references in §3 link to correct files in docs/adr/
- Technology names spelled consistently ("pgvector" not "pgVector" not "PGVector")
- Terminology consistent ("ticket" for business context, "query" for technical context)
- GPT-4o, GPT-4o-mini, Llama 3.1 — consistent capitalization throughout

QUALITY (from §5.1):
- No section is filler — every paragraph earns its place
- No unsupported claims — every metric has a source
- Prose is active, not passive ("we chose pgvector because..." not "pgvector was chosen because...")
- No AI-slop patterns (check against §10.2 Check 8 list: "delve into", "leverage", "in the realm of", "it's worth noting that", "navigate the complexities of", "tapestry", "landscape of", "paradigm shift", "game-changer", "revolutionary")

FORMATTING:
- ASCII diagrams render correctly in monospace (max 80 chars wide)
- Markdown tables render correctly
- No broken internal links
- Page count within budget (15-20 pages when rendered)

Run: cspell "CASE_STUDY.md" "docs/**/*.md" (if cspell available)
Manual review for: PostgreSQL, pgvector, Presidio, spaCy, FastAPI, LangChain spelling.
No contractions in formal sections (§1, §2, §3) — acceptable in §6.
```

**adr:**
```
TASK: Final ADR cross-reference audit and consistency check.

Review all 6 files in docs/adr/:
1. Verify each ADR footer links back to CASE_STUDY.md: "*Part of the [Technical Case Study](../../CASE_STUDY.md) for the Customer Support AI System.*"
2. Verify cross-references between ADRs are correct:
   - ADR-005 references ADR-003 (escalation logic)
   - ADR-001 enables ADR-002 (BM25 via tsvector)
   - ADR-004 enables ADR-003 (PII-redacted context for human review)
   - ADR-001 enables ADR-004 (audit log in same DB)
3. Verify ADR numbering in CASE_STUDY.md §3 matches file names
4. Check that ADR titles in §3 summary table match ADR file titles
5. Verify Status is "Accepted" in all 5 ADRs
6. Run validation Check 3 (ADR Completeness) from §10.2 on each file
7. Spell check: pgvector, Qdrant, Pinecone, Weaviate, Presidio, GPT-4o-mini, Llama, HNSW, RRF, nDCG, GDPR

Fix any issues found directly in the ADR files.
```

**deck:**
```
TASK: Final metrics file consistency check and cross-reference audit.

Review all files in docs/metrics/:
1. Verify all numbers in docs/metrics/ files match CASE_STUDY.md §4 exactly:
   - retrieval-quality.md: 4 strategies, recall/MRR/nDCG values match §4.1 table
   - answer-quality.md: 3 models, faithfulness/relevance/citation values match §4.2 table
   - system-performance.md: 4 endpoints, latency/throughput/error values match §4.3 table
   - cost-analysis.md: 8 cost metrics match §4.4 table
   - business-impact.md: before/after values match §4.5 table
2. Verify chart numbering is consistent (Charts 1-9) across CHART_DESCRIPTIONS.md and per-category files
3. Verify each chart description has all required fields: Type, Purpose, Axes, Series, Data, Interpretation, Creation
4. Run validation Check 5 (Chart Description Completeness) from §10.2
5. Verify docs/presentation/DECK_OUTLINE.md chart references match chart numbers in metrics files
6. Check exports/README.md is present and explains generation options

Fix any issues found directly in the metrics files.
```

#### Sequential (after parallel — reviewer runs full audit)

**reviewer:**
```
TASK: Final comprehensive review of the complete case study.

Run ALL validation checks from IMPLEMENTATION_PLAN.md §10.2:

CHECK 1: Metric Consistency — verify every metric appears with the same value across all sections and files. Cross-check §1 Exec Summary, §4 Metrics, §5 Failures, docs/metrics/ files.

CHECK 2: Section Completeness — verify each section §1-§8 meets minimum length, no placeholder text remains.

CHECK 3: ADR Completeness — each ADR has Context, ≥2 Alternatives, Consequences with negatives.

CHECK 4: Incident Completeness — each incident has all 7 required fields, valid severity, at least one P1.

CHECK 5: Chart Description Completeness — each chart has Type, Purpose, X-axis, Y-axis, Series, Data, Interpretation.

CHECK 6: No PII in Document — scan for email/phone/credit card/SSN patterns. All examples should use synthetic data.

CHECK 7: Interview Readiness — verify each section maps to interview questions from §9.2.

CHECK 8: AI-Slop Detection — scan for: "delve into", "leverage", "in the realm of", "it's worth noting", "navigate the complexities", "tapestry", "landscape of", "paradigm shift", "game-changer", "revolutionary".

CROSS-REFERENCE AUDIT (from §5.2):
- §3 Design Decisions links to each ADR file
- §4 Metrics references chart descriptions in docs/metrics/
- §7 Runbook notes standalone version at docs/runbook/RUNBOOK.md
- README.md links to CASE_STUDY.md as primary deliverable
- All docs/adr/ files link back to case study

FINAL DELIVERABLE CHECK (from §5.4):
- CASE_STUDY.md — 15-20 pages, all 8 sections complete
- docs/adr/ — 5 ADRs + template
- docs/metrics/ — 5 category files + chart descriptions + exports/README
- docs/presentation/DECK_OUTLINE.md — 18 slide outline
- docs/runbook/RUNBOOK.md — standalone runbook
- README.md — reading guide

Report ALL issues to the relevant agent. Categorize as:
- BLOCKER: must fix before commit (inconsistent numbers, broken links, missing sections)
- WARNING: should fix (AI-slop phrases, passive voice, minor formatting)
- SUGGESTION: nice to have (prose polish, word choice)

Do not edit files. Send issues to writer, adr, or deck via hub messaging.
```

**writer** (after reviewer report):
```
TASK: Apply fixes from reviewer's final audit.

Address all BLOCKER issues first, then WARNINGs. Fix directly in CASE_STUDY.md, README.md, and docs/runbook/RUNBOOK.md.

After fixes, generate PDF export:
pandoc CASE_STUDY.md -o CASE_STUDY.pdf \
  --pdf-engine=xelatex \
  --toc \
  --toc-depth=2 \
  --highlight-style=tango \
  -V geometry:margin=1in \
  -V fontsize=11pt \
  -V linkcolor:blue \
  -V urlcolor:blue

If xelatex not available, use: pandoc CASE_STUDY.md -o CASE_STUDY.pdf --toc (with whatever engine is available).

Verify final deliverable checklist from §5.4:
- [ ] CASE_STUDY.md — 15-20 pages, all 8 sections
- [ ] docs/adr/ — 5 ADRs + template
- [ ] docs/metrics/ — 5 category files + CHART_DESCRIPTIONS.md + exports/README.md
- [ ] docs/presentation/DECK_OUTLINE.md — 18 slides
- [ ] docs/runbook/RUNBOOK.md — standalone
- [ ] README.md — reading guide
- [ ] CASE_STUDY.pdf — PDF export
```

#### Review

**reviewer:**
```
Final sign-off review. Verify all BLOCKER issues from the audit have been resolved. Confirm:
- Full document reads as a cohesive narrative, not 8 disconnected sections
- A non-technical reader can get value from §1 and §4.5 without reading anything else
- A senior engineer can deep-dive into §2, §3, and §5 and find technical substance
- The case study would make a hiring manager want to interview the candidate
- PDF export exists (if pandoc available)

Give GO/NO-GO for final commit.
```

#### Commit

```bash
git add -A && git commit -m "Phase 5: final review, polish, cross-reference audit, PDF export

- All 8 sections of CASE_STUDY.md complete (15-20 pages)
- 5 ADRs + template in docs/adr/
- 5 metrics category files + chart descriptions in docs/metrics/
- 18-slide presentation outline in docs/presentation/
- Standalone runbook in docs/runbook/
- README.md reading guide
- CASE_STUDY.pdf export for job applications

All validation checks passed (§10.2 Checks 1-8)."
```

---

## 5. Coordination Rules

### File Ownership Boundaries

| Agent | Can Write | Can Read |
|-------|-----------|----------|
| **writer** | `CASE_STUDY.md`, `README.md`, `docs/runbook/RUNBOOK.md`, `sources.md` | All files |
| **adr** | `docs/adr/*.md` | All files |
| **deck** | `docs/metrics/*.md`, `docs/metrics/exports/README.md`, `docs/presentation/DECK_OUTLINE.md` | All files |
| **reviewer** | None (read-only) | All files |

### Conflict Avoidance

- **No agent edits another agent's files.** If writer needs an ADR cross-reference fixed, message `adr` via hub — do not edit `docs/adr/` directly.
- **CASE_STUDY.md is writer's exclusive domain.** Other agents create standalone files; writer embeds/links to them.
- **Number consistency is a shared concern.** The canonical metric values live in IMPLEMENTATION_PLAN.md §3.1. All agents copy from there — never from another agent's file (prevents error propagation).
- **Cross-references flow one way:** ADR files link back to CASE_STUDY.md (footer). CASE_STUDY.md links out to ADR files (§3), metrics files (§4), runbook (§7). Metrics files and deck outline are self-contained but may reference chart numbers consistently.

### Parallel vs Sequential

- **Parallel:** When agents work on non-overlapping files with no content dependency. Phases 1-4 have parallel work blocks.
- **Sequential:** When one agent's output is needed as input for another. Example: Phase 2 — writer's §3 Design Decisions must run after adr's ADRs exist (needs links to ADR files). Phase 5 — reviewer runs after all agents complete their parallel fixes.
- **Rule of thumb:** If agent B needs to link to or reference agent A's file content, B runs after A. If they just need to agree on numbers (from the plan), they run in parallel.

### Handling Blocked Agents

- An agent that cannot proceed (missing dependency, unclear requirement) should:
  1. Message the blocking agent via `hub send` with a specific question
  2. If blocked on the writer (who owns CASE_STUDY.md), wait for response — do not edit CASE_STUDY.md
  3. If blocked on a number or fact, consult IMPLEMENTATION_PLAN.md directly — all canonical data is there
  4. If blocked on a design decision not in the plan, message `reviewer` for guidance
- The reviewer can unblock by making a ruling on ambiguous requirements
- No agent should guess or fabricate content to unblock itself — ask first

---

## 6. Quick Reference

### Common Herdr Commands

```bash
# --- Pane management ---
herdr pane split --cwd "$PWD" --no-focus     # Split a new pane
herdr pane list                               # List all panes
herdr pane focus <id>                         # Focus a pane

# --- Agent management ---
herdr agent start <name> --kind codex --pane <id>  # Start an agent
herdr agent list                                    # List running agents
herdr agent stop <name>                             # Stop an agent
herdr agent send <name> "<message>"                 # Send a prompt to an agent

# --- Monitoring ---
herdr agent logs <name>                       # View agent output
herdr agent status <name>                     # Check agent status

# --- Project workflow ---
git add -A && git commit -m "Phase N: <description>"  # Commit after each phase
git log --oneline                                     # Review commit history
```

### Phase Summary

| Phase | Day | Parallel Agents | Sequential | Key Deliverable |
|-------|-----|-----------------|------------|-----------------|
| 1 | 1 | writer, adr, deck | writer (verify) | Skeleton + §1 Exec Summary + ADR template + metrics scaffold |
| 2 | 2 | writer (§2), adr (5 ADRs), deck (chart specs) | writer (§3) | Architecture + 5 ADRs + chart descriptions |
| 3 | 3 | writer (§4+§5), deck (metrics files) | — | Metrics & Results + Failure Modes + per-category files |
| 4 | 4 | writer (§6+§7+§8+runbook+README), deck (deck outline) | — | Lessons + Runbook + Future + Deck + README |
| 5 | 5 | writer (polish), adr (audit), deck (consistency) | reviewer (full audit), writer (fixes+PDF) | Final polished case study + PDF |

### Key Numbers (Canonical — All Agents Copy From Here)

| Metric | Value | Source |
|--------|-------|--------|
| Auto-resolution rate | 42% | §4.5 |
| Avg resolution time (before) | 6.2 hours | §4.5 |
| Avg resolution time (after) | 2.1 hours | §4.5 |
| Cost per auto-resolved ticket | $0.08 | §4.4 |
| Cost per human-resolved ticket | $4.50 | §4.4 |
| Monthly LLM spend | $340 | §4.4 |
| Monthly savings vs baseline | $265 | §4.4 |
| Monthly business savings | $5,250 | §4.5 |
| Cache hit rate | 34% | §4.4 |
| Faithfulness (GPT-4o-mini) | 0.87 | §4.2 |
| Faithfulness (GPT-4o) | 0.93 | §4.2 |
| Recall@5 (hybrid+rerank) | 0.84 | §4.1 |
| CSAT (before → after) | 3.8 → 4.2 | §4.5 |
| Total monthly tickets | 4,200 | §4.5 |
| Escalation rate | 18% | §4.5 |
| Confidence threshold | 0.75 | ADR-003 |
| Cache similarity threshold | 0.85 | Incident 2 |
| Max tokens per conversation | 4,000 | Incident 4 |
| Cost reduction (routing vs 4o-only) | 65% | ADR-005 |
| Recall improvement (hybrid vs vector) | +12% | ADR-002 |
