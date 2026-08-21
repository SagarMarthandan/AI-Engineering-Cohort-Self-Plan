# Extensive Interleaved Plan — Ed Donners × From-Scratch × Roadmap

> **What this is:** A concept-by-concept mapping of three learning resources, interleaved so that theory (Ed Donners video), depth (from-scratch lessons), and proof (roadmap projects) reinforce each other on the same topics at the same time.
>
> **Who this is for:** Sagar Marthandan — Data/AI Engineer who knows Python, SQL, dbt, Airflow, Docker. Skip fundamentals; focus on the 30% gap: LLM orchestration, retrieval engineering, AI evaluation, production AI infrastructure.

---

## The Three Resources

| Resource | Role | Format | Time |
|----------|------|--------|------|
| **Ed Donners** — *LLM Engineering: Master AI, Large Language Models and Agents* (Udemy, 33h, 8 weeks, 8 projects) | Concept & breadth | Video lectures at 1.5x + follow-along projects | ~5 days equivalent |
| **AI Engineering from Scratch** (17 phases, ~150 lessons) | Depth & first principles | Read + code [CORE] lessons; skip [DEEP]/[SKIP] | ~6 days equivalent |
| **AI-Engineering Roadmap** (12 projects, 5 tiers, this repo) | Proof & portfolio | Build projects from IMPLEMENTATION_PLAN.md specs | ~22 days equivalent |

### Interleaving Strategy

```
Morning:   Ed Donners video lectures (concept — see it done)
Afternoon: Roadmap project (proof — build it yourself)
Evening:   From-scratch lessons (depth — understand the math)
```

Some topics deviate from this pattern (e.g., when Ed Donners has no corresponding content, the full session goes to roadmap projects). Each mapping below states the allocation explicitly.

---

## Legend

- **W#** = Ed Donners Udemy course week number
- **P#-L##** = AI Engineering from Scratch, Phase #, Lesson ##
- **R-P#** = AI-Engineering Roadmap Project # (with specific phase from IMPLEMENTATION_PLAN.md)
- **[CORE]** = study alongside | **[DEEP]** = optional depth | **[SKIP]** = you know this

---

## Phase 1: Foundations (Ed Donners W1–W2)

**Concepts:** API calls, tokenization, transformers, tool calling, multimodal, first RAG setup.

### Topic 1.1 — LLM API Fundamentals

| Resource | Content | What You Do |
|----------|---------|-------------|
| **Ed Donners** | W1 Lectures 1–10: Course intro, Ollama setup, first local LLM call, OpenAI API setup, chat completions, system vs user prompts, your first LLM product | Install Ollama, pull a model, make API calls. Notes on: token costs, context windows, model selection criteria |
| **Ed Donners** | W1 Lectures 11–20: Website summarizer project, frontier models compared (GPT-4o, Claude, Gemini, Grok), transformers architecture overview (high-level), tokens and tokenization with tiktoken, context windows explained | Build the website summarizer alongside Ed |
| **Ed Donners** | W1 Lectures 21–37: Chaining GPT calls together, brochure generator (Project 1), environment setup, Cursor/UV/Git walkthrough [SKIP if you know this], first look at reasoning models | Finish the brochure generator — first shipped LLM product |

**Deliverable:** Brochure generator project complete. Ollama + OpenAI API working. Notes on token costs and model selection.

### Topic 1.2 — Transformer Architecture from Scratch

| Resource | Content | What You Do |
|----------|---------|-------------|
| **From-Scratch** | P7-L02: Self-Attention from Scratch [CORE] | Build attention by hand: Q, K, V matrices, softmax(QK^T/sqrt(d))V. Understand WHY attention works before Ed explains it conceptually. **Single most important from-scratch lesson.** |
| **From-Scratch** | P7-L03: Multi-Head Attention [CORE] | Extend single-head to multi-head: split into h heads, parallel attention, concatenate, linear projection. Understand why multi-head captures different representation subspaces |
| **From-Scratch** | P7-L05: The Full Transformer: Encoder + Decoder [CORE] | Positional encoding, encoder block, decoder block, masked attention. By end: draw the transformer architecture from memory |

**Deliverable:** Self-attention + multi-head attention code working. Full transformer architecture understood. Can explain Q/K/V without looking at code.

### Topic 1.3 — GPT, Tokenizers, and the Model Pipeline

| Resource | Content | What You Do |
|----------|---------|-------------|
| **Ed Donners** | W2 Lectures 1–12: Multi-model conversations (OpenAI + Claude together), LangChain vs LiteLLM, Gradio UI basics, streaming chatbot, system prompts and multi-shot prompting | Build the streaming chatbot in Gradio |
| **From-Scratch** | P7-L07: GPT - Causal Language Modeling [CORE] | Autoregressive next-token prediction, causal masking, decoder-only architecture. Code: minimal GPT block, generation loop, temperature sampling |
| **From-Scratch** | P10-L01: Tokenizers - BPE, WordPiece, SentencePiece [CORE] | Understand what tiktoken does. Code: BPE training loop, merge rules, encode/decode. Why token != word and how this affects cost |
| **Ed Donners** | W2 Lectures 13–24: Tool calling / function calling, airline AI assistant with tool calling + SQLite (Project 2), DALL-E 3, text-to-speech, multimodal AI, agentic AI intro | Build the airline assistant — first agent that calls tools |
| **From-Scratch** | P10-L02: Building a Tokenizer from Scratch [DEEP] | Train a BPE tokenizer on a real corpus, handle special tokens, chat templates. Compare output to tiktoken |
| **From-Scratch** | P10-L06: Instruction Tuning - SFT [CORE] (skim) + P10-L07: RLHF [DEEP] (skim) + P10-L08: DPO [DEEP] (skim) | Understand the 3-step pipeline: pretrain → SFT → RLHF/DPO at a conceptual level |

**Deliverable:** Streaming Gradio chatbot working. GPT causal LM architecture understood. BPE tokenizer code written. Airline assistant with tool calling complete. Understand base vs chat vs reasoning model pipeline.

### Topic 1.4 — First Roadmap Project: Advanced RAG Setup

| Resource | Content | What You Do |
|----------|---------|-------------|
| **Roadmap** | R-P1 Setup: Read IMPLEMENTATION_PLAN.md for Project 1 (Advanced RAG) | Create git repo. Docker Compose (PostgreSQL 16 + pgvector). Schema: documents, chunks, queries, golden_questions. Verify pgvector with test vector insert |
| **Roadmap** | R-P1 Phase 2: Document ingestion pipeline | DocumentLoader (txt, md, pdf), ChunkingOrchestrator with 3 strategies (fixed_1000_200, semantic_spacy, sentence_window_3), EmbeddingGenerator (BGE-large-en-v1.5). Ingest 10 test documents |
| **Roadmap** | R-P1 Phase 2 continued | Test all 3 chunking strategies. Verify HNSW indexes. Run EXPLAIN ANALYZE on vector search. Confirm retrieval works |

**Deliverable:** R-P1 repo created. Docker Compose with PostgreSQL + pgvector running. 3 chunking strategies implemented. 10 documents ingested with embeddings. Vector search returns results.

---

## Phase 2: RAG Deep Dive (Ed Donners W3–W5)

**Concepts:** HuggingFace, model evaluation, full RAG stack, hybrid search, re-ranking, query transformation, tenant isolation, GraphRAG, agentic RAG, multimodal RAG. Roadmap Projects 1–3.

### Topic 2.1 — HuggingFace & Open-Source Models

| Resource | Content | What You Do |
|----------|---------|-------------|
| **Ed Donners** | W3 Lectures 1–12: HuggingFace platform tour, Colab + GPU setup [SKIP if you know Colab], HF Pipelines (sentiment, NER, Q&A, image, audio, diffusion) | Run each pipeline type. Understand what pipeline() does under the hood |
| **Ed Donners** | W3 Lectures 13–23: Tokenizers in action (chat templates, special tokens for LLaMA, Phi-4, DeepSeek, QWENCoder), transformers low-level API (model.generate, attention weights, quantization), inside LLaMA architecture (PyTorch inspection), running open-source models | Inspect a real model's architecture in PyTorch |
| **From-Scratch** | P7-L12: KV Cache, Flash Attention and Inference Optimization [CORE] | Understand what model.generate() does. KV cache: why it speeds up generation. Flash Attention: why it reduces memory. Connects W3 inside LLaMA to real inference performance |
| **From-Scratch** | P10-L11: Quantization: INT8, GPTQ, AWQ, GGUF [CORE] (skim) | Understand WHY Ollama can run models locally. 4-bit = 70B model on single GPU. Trade-offs: quality vs memory vs speed |

**Deliverable:** HF Pipelines tested. LLaMA architecture inspected in PyTorch. Understand KV cache, Flash Attention, and quantization trade-offs.

### Topic 2.2 — Hybrid Search & Query Transformation (R-P1 Completion)

| Resource | Content | What You Do |
|----------|---------|-------------|
| **Roadmap** | R-P1 Phase 3: Dual indexing | BM25 index (PostgreSQL tsvector) + vector index (pgvector HNSW). Hybrid retrieval: BM25 top-50 + Vector top-50 → Reciprocal Rank Fusion (RRF) → top-20. Code the RRF algorithm. Compare BM25-only vs vector-only vs hybrid on 5 queries |
| **Roadmap** | R-P1 Phase 4: Query transformation | HyDE (generate hypothetical document, embed, search) + Multi-Query expansion (3 reformulations, embed each, merge). Build QueryTransform class. Test: no-transform vs HyDE vs Multi-Query |
| **Roadmap** | R-P1 Phase 5: Cross-encoder re-ranking | BGE-reranker-large: top-20 → top-5. Build ReRanker class. Compare hybrid-only vs hybrid+rerank. Measure latency cost |
| **Roadmap** | R-P1 Phase 6: LLM generation + FastAPI | POST /query with strategy parameter (chunking + transform + rerank config). Full pipeline: query → transform → hybrid retrieve → rerank → LLM answer. Test with curl |
| **Roadmap** | R-P1 Phase 7: Evaluation dashboard | Streamlit app: recall@k, MRR, nDCG, precision@k across all strategy combinations. Golden Q&A dataset (10 questions). Verify >15% nDCG variance. Write README. Commit. Mark R-P1 complete |

**Deliverable:** R-P1 COMPLETE. Hybrid retrieval (BM25 + vector + RRF), query transformation (HyDE + Multi-Query), cross-encoder re-ranking, FastAPI /query endpoint, Streamlit evaluation dashboard. README written. Git committed.

### Topic 2.3 — Model Evaluation & Secure RAG (R-P2)

| Resource | Content | What You Do |
|----------|---------|-------------|
| **Ed Donners** | W4 Lectures 1–10: Model evaluation, comparing frontier models on tasks, code generation (Python to C++, Project 4), reasoning models and reasoning effort | Build the code generation project. What makes a model good at a task? How do you measure it? |
| **Ed Donners** | W4 Lectures 11–21: Business task evaluation, model selection for production, agentic AI in action (Deep Research, Claude Code, Agent Mode), LLM competition game | Model selection is a business decision: cost vs quality vs latency |
| **From-Scratch** | P10-L10: Evaluation - Benchmarks, Evals [CORE] | MMLU, HumanEval, GSM8K, MT-Bench. LLM-as-judge pattern. Connects Ed's W4 to the actual science |
| **From-Scratch** | P5-L27: LLM Evaluation: RAGAS, DeepEval, G-Eval [CORE] | Faithfulness, answer relevance, context precision/recall. Theory behind R-P1's eval dashboard and R-P7's eval harness |
| **Roadmap** | R-P2 Setup + Phase 1: Secure RAG | Docker Compose (PostgreSQL + pgvector). Schema: users, documents, chunks, audit_logs. Key difference from P1: customer_id (tenant isolation), allowed_roles (RBAC), audit_logs (append-only). JWT auth middleware. 2 test tenants with different roles |
| **Roadmap** | R-P2 Phase 2: Tenant isolation | Pre-filter vector search: WHERE customer_id = ? AND allowed_roles && user.roles. Row-Level Security (RLS) as defense-in-depth. Test: zero cross-tenant leakage |
| **Roadmap** | R-P2 Phase 3: Citation tracking + audit logging | Structured citations (doc_path, chunk_id, snippet). Audit_logs with append-only triggers (prevent UPDATE/DELETE). Log every query. Build audit log query API |

**Deliverable:** R-P2 repo created. JWT auth working. Tenant isolation + RLS working. Zero cross-tenant leakage verified. Citations + audit logging with append-only triggers. Understand model evaluation benchmarks and RAGAS/DeepEval frameworks.

### Topic 2.4 — RAG from Scratch & Advanced Chunking

| Resource | Content | What You Do |
|----------|---------|-------------|
| **Ed Donners** | W5 Lectures 1–10: RAG big idea, vector data stores, vectors and LangChain, vector databases (Chroma, FAISS), embeddings and embedding models, t-SNE visualization | Build Ed's basic RAG pipeline. Visualize embeddings with t-SNE |
| **Ed Donners** | W5 Lectures 11–20: Document chunking, text splitters, RAG pipeline with LangChain + Chroma, RAG with conversation history, Gradio UI for RAG | Build Ed's full RAG pipeline. See how chunking affects retrieval quality |
| **From-Scratch** | P11-L06: RAG: Retrieval-Augmented Generation [CORE] | Build RAG from first principles (no LangChain): embed query, vector search, build context, call LLM. Compare to Ed's LangChain RAG. Understand what LangChain adds and hides |
| **From-Scratch** | P5-L23: Chunking Strategies for RAG [CORE] + P11-L07: Advanced RAG: Chunking, Reranking [CORE] | Fixed-size, semantic, sentence-window chunking. When to use each. Bi-encoder vs cross-encoder re-ranking. Theory behind R-P1's implementations |
| **Roadmap** | R-P2 Phase 4: Finish + polish | FastAPI endpoints: POST /upload (permission check), POST /query (tenant filter + citations + audit). README with architecture diagram. Docker Compose up. Tests: tenant isolation, RBAC, audit immutability. Mark R-P2 complete |

**Deliverable:** R-P2 COMPLETE. From-scratch RAG (no framework) built and compared to LangChain version. Understand chunking strategies and re-ranking theory.

### Topic 2.5 — Advanced RAG: GraphRAG, Agentic RAG, Multimodal RAG (R-P3)

| Resource | Content | What You Do |
|----------|---------|-------------|
| **Ed Donners** | W5 Lectures 21–32: RAG evaluation (MRR, NDCG, LLM-as-judge, golden data), advanced RAG (query rewriting, re-ranking, query expansion, GraphRAG), semantic chunking, production RAG | Most RAG-intensive content. Detailed notes on evaluation metrics |
| **Ed Donners** | W5: Knowledge worker project (Project 5) + buffer | Build Ed's knowledge worker — a production-quality RAG system over a real document set |
| **From-Scratch** | P5-L14: Information Retrieval and Search [CORE] | Boolean retrieval, TF-IDF, BM25, vector space model, cosine similarity. Code: implement BM25 from scratch. Compare to PostgreSQL's tsvector |
| **From-Scratch** | P5-L22: Embedding Models Deep Dive [CORE] | Contrastive learning, triplet loss, hard negative mining. Why BGE > OpenAI embeddings for some tasks. Theory behind embedding choices in R-P1 and R-P2 |
| **Roadmap** | R-P3 Setup + Phase 1: GraphRAG | Docker Compose (PostgreSQL + pgvector + Neo4j). LLM entity extraction → entities + relationships → Neo4j graph. Test on 5 documents |
| **Roadmap** | R-P3 Phase 2: Agentic RAG | Iterative retrieval loop: LLM decides what to retrieve, assesses if context is sufficient, retrieves again (max 5 iterations). Self-RAG reflection tokens: [RETRIEVE]/[NO RETRIEVE], [RELEVANT]/[IRRELEVANT]. Decision framework: vanilla vs GraphRAG vs Agentic vs Multimodal. Streamlit comparison dashboard |
| **Roadmap** | R-P3 Phase 3: Multimodal RAG | CLIP (openai/clip-vit-base-patch32) for image embeddings (512-dim) alongside BGE text (1024-dim). Cross-modal retrieval: text→image, image→text. Extract images from PDFs with PyMuPDF |
| **Roadmap** | R-P3 Phase 4: Finish | FastAPI comparison API: POST /query with architecture parameter (vanilla, graph, agentic, multimodal). KAG/LightRAG comparison table (survey). README with architecture decision framework. Mark R-P3 complete |

**Deliverable:** R-P3 COMPLETE. GraphRAG with Neo4j, Agentic RAG with Self-RAG reflection, Multimodal RAG with CLIP. Comparison API deployed. BM25 implemented from scratch. IR theory and embedding model training understood. All 3 RAG projects (P1, P2, P3) complete.

---

## Phase 3: Agents (Ed Donners W6–W8)

**Concepts:** Fine-tuning GPT-4o-mini, QLoRA on LLaMA, multi-agent capstone, function calling agents, document processing pipelines, LangGraph orchestration. Roadmap Projects 4–6.

### Topic 3.1 — Fine-Tuning & Neural Network Foundations

| Resource | Content | What You Do |
|----------|---------|-------------|
| **Ed Donners** | W6 Lectures 1–10: Datasets, Amazon data curation, HF datasets, weighted sampling, 5-step AI process [SKIP traditional ML baselines — you know RF/XGBoost] | Focus on: dataset distribution analysis, deduplication, HF datasets API |
| **Ed Donners** | W6 Lectures 11–20: Fine-tuning GPT-4o-mini with OpenAI SFT API, when fine-tuning fails, neural networks in PyTorch, testing frontier models on price prediction (Project 6) | Build Ed's fine-tuning job. See training data format, API call, evaluation. Fine-tuning is a data problem, not a code problem |
| **Ed Donners** | W6 Lectures 21–27: Deep neural network redemption (Project 6 continued), MLOps, Groq batch processing, baseline models vs LLM comparison | When does fine-tuning beat prompt engineering? When does traditional ML beat LLMs? For structured/tabular data, XGBoost often beats LLMs |
| **From-Scratch** | P3-L03: Backpropagation from Scratch [CORE] (if you haven't written it by hand) | Forward pass, backward pass, gradient computation, weight update. Chain rule, why gradients flow backwards, why ReLU helps. If done before → skip to P3-L05: Loss Functions [CORE] — cross-entropy, the loss in every LLM fine-tuning job |

**Deliverable:** Backprop or loss functions understood from scratch. GPT-4o-mini fine-tuning understood. When to use fine-tuning vs prompt engineering vs traditional ML.

### Topic 3.2 — Data Analysis Agent (R-P4)

| Resource | Content | What You Do |
|----------|---------|-------------|
| **Roadmap** | R-P4 Setup + Phase 1: Data Analysis Agent | Docker Compose (PostgreSQL with Northwind sample data). Read-only DB user: data_agent_reader (SELECT only, statement_timeout=30s, work_mem=64MB). Function calling tools: get_schema, execute_sql, generate_chart as OpenAI function schemas. Test: LLM calls get_schema |
| **Roadmap** | R-P4 Phase 2: SQL validation layer | sqlglot parse (syntax check), safety check (no DDL/DML, read-only), LIMIT enforcement, EXPLAIN cost estimation. ReAct agent loop: Reason → Act → Observe → Reason. Test: agent writes SQL for top 10 customers by revenue |
| **Roadmap** | R-P4 Phase 3: Error recovery + memory | If SQL fails → feed error to LLM → regenerate → retry (max 3). Conversation memory (in-memory dict + SQLite/Postgres persistence). Multi-turn: "show me top customers → now filter by Germany". Write README. Mark R-P4 complete |

**Deliverable:** R-P4 COMPLETE. SQL validation layer working. ReAct agent loop with error recovery. Multi-turn conversation memory. README written.

### Topic 3.3 — QLoRA & LoRA Math

| Resource | Content | What You Do |
|----------|---------|-------------|
| **Ed Donners** | W7 Lectures 1–10: QLoRA intro, LoRA training LLaMA 3.2, LoRA hyperparameters, 4-bit/8-bit quantization explained | LoRA = low-rank adaptation (freeze base weights, train small adapter). QLoRA = 4-bit base + LoRA adapter. See the training loop |
| **Ed Donners** | W7 Lectures 11–20: Dataset prep for fine-tuning, base vs chat models, training hyperparameters (lr, optimizers, batch size), Weights & Biases, TRL SFT Trainer | See Ed's full training run. Learning rate is the most important hyperparameter. W&B tracks loss curves, gradient norms, lr schedule |
| **Ed Donners** | W7 Lectures 21–27: Monitoring training loss, overfitting, checkpoint selection, cross-entropy loss, full dataset training on A100, testing fine-tuned model vs GPT-4o (Project 7) | Training loss vs validation loss, when to stop, how to select best checkpoint |
| **From-Scratch** | P11-L08: Fine-Tuning with LoRA and QLoRA [CORE] | Math behind what Ed showed: low-rank decomposition (W = W0 + BA where B and A are small), gradient flow through frozen weights, why LoRA is parameter-efficient. Code: minimal LoRA layer in PyTorch |

**Deliverable:** LoRA math understood. Minimal LoRA layer coded. QLoRA training process understood (training loss vs validation loss, checkpoint selection).

### Topic 3.4 — Document Processing Pipeline (R-P5)

| Resource | Content | What You Do |
|----------|---------|-------------|
| **Roadmap** | R-P5 Setup + Phase 1: Document Processing Pipeline | Docker Compose (PostgreSQL + Airflow). Ingestion layer: parsers for PDF (pypdf), DOCX (python-docx), TXT, EML (mailparser), images (Tesseract OCR). LLM classifier: invoice/contract/email/receipt/other with confidence. Test: classify 10 mixed documents |
| **Roadmap** | R-P5 Phase 2: Structured extraction | Pydantic v2 schemas (InvoiceSchema, ContractSchema, EmailSchema, ReceiptSchema). OpenAI structured output: client.beta.chat.completions.parse() with Pydantic response_format. Test: extract fields from 5 invoices. Pydantic validation catches bad data |
| **Roadmap** | R-P5 Phase 3: Validation + retry + routing | Pydantic validation + business rule validators (invoice total = sum of line items, date plausibility, tax rate ranges). Retry with error feedback (max 2). Routing: validated → PostgreSQL, files → document store, audit log. Airflow DAG: scan → ingest → classify → extract → validate → route. Test: run DAG on 20 documents |
| **Roadmap** | R-P5 Phase 4: Human review queue + finish | Streamlit review UI: original doc + extracted data + validation errors + confidence. Reviewer edits, approves, or rejects. README. Docker Compose up (Airflow + Postgres + API + Streamlit). Mark R-P5 complete |

**Deliverable:** R-P5 COMPLETE. Ingestion + classification + structured extraction + validation + retry + routing + Airflow DAG + human review queue. README written.

### Topic 3.5 — Multi-Agent Systems (R-P6)

| Resource | Content | What You Do |
|----------|---------|-------------|
| **Ed Donners** | W8 Lectures 1–11: Agentic AI intro, Modal serverless deployment, designing agent architectures, Modal platform setup | See how Ed deploys an agent to Modal. Understand serverless AI: cold starts, GPU provisioning, cost-per-invocation. [Your roadmap uses Docker, not Modal — but understand the pattern] |
| **Ed Donners** | W8 Lectures 12–22: Multi-agent system (Project 8), autonomous deal-spotting system | Build Ed's final project — multi-agent system that autonomously finds and reports deals. All 8 weeks of skills come together |
| **From-Scratch** | P14-L01: The Agent Loop [CORE] | Fundamental agent pattern: perceive → reason → act → observe → repeat. Code: minimal agent loop |
| **From-Scratch** | P14-L02: ReWOO and Plan-and-Execute [CORE] | Plan all steps first, then execute |
| **From-Scratch** | P14-L12: Anthropic's Workflow Patterns [CORE] | Canonical patterns: prompt chaining, routing, parallelization, orchestrator-worker, evaluator-optimizer |
| **From-Scratch** | P14-L25: Multi-Agent Debate and Collaboration [CORE] + P14-L28: Orchestration Patterns - Supervisor, Swarm, Hierarchical [CORE] | Patterns behind R-P6. Map each pattern to what you built. Theory that makes R-P6 code make sense |
| **Roadmap** | R-P6 Setup: Multi-Agent Systems | Docker Compose (PostgreSQL + Redis). Install LangGraph. StateGraph TypedDict: query, query_type, research_notes, draft, critic_feedback, revision_count, final_output, cost_log, messages, memory_context, status. Graph topology: router → research → writer → critic → synthesizer |
| **Roadmap** | R-P6 Phase 2: Router + Research agents | Router Agent: classifies query as technical/creative/analytical. 3 Research Agents (one per type) with tools: web_search_stub, knowledge_base_lookup. Conditional edges in LangGraph. Test routing |
| **Roadmap** | R-P6 Phase 3: Writer + Critic agents | Writer: produces draft from research notes. Critic: structured feedback {strengths, weaknesses, suggestions, approved: bool}. Revision loop: if not approved AND revision_count < 3 → back to writer. Test loop terminates |
| **Roadmap** | R-P6 Phase 4: Orchestrator-Worker pattern | Orchestrator: decomposes complex task into subtasks, delegates to workers, synthesizes. Map-Reduce parallel agents: fan out ≥3 subtasks concurrently, reduce. Supervisor: monitors worker status, handles failures, routes to fallback |
| **Roadmap** | R-P6 Phase 5: Agent memory + cost tracking + finish | Short-term: Redis buffer (last 10 messages), summarization after 10 turns. Long-term: pgvector episodic memory. Cost tracker: per-agent token usage, total pipeline cost, latency per node. Streamlit dashboard: agent graph visualization + cost/latency. README. Mark R-P6 complete |

**Deliverable:** R-P6 COMPLETE. Router + Research + Writer + Critic agents. Revision loop. Orchestrator-Worker + Map-Reduce + Supervisor patterns. Agent memory (Redis + pgvector). Cost tracking. Streamlit dashboard. Ed Donners course COMPLETE (all 8 weeks, all 8 projects). Agent loop, plan-execute, workflow patterns, multi-agent orchestration understood from first principles.

**Milestone:** 6 portfolio projects + 8 Ed Donners projects + transformers/tokenizers/RAG/agents built from scratch.

---

## Phase 4: Evaluation + Infrastructure (No Ed Donners — Full Focus on Roadmap)

**Concepts:** LLM evaluation harness, prompt A/B testing, LLM serving gateway, PII redaction guardrails. This is the tier that separates AI engineers from API callers.

### Topic 4.1 — LLM Evaluation Harness (R-P7)

| Resource | Content | What You Do |
|----------|---------|-------------|
| **Roadmap** | R-P7 Setup + Phase 1: LLM Evaluation Harness | Docker Compose (PostgreSQL). Schema: prompt_versions, eval_datasets, eval_runs, eval_case_results, human_feedback. Eval dataset format: YAML with test cases (question, expected_answer, relevant_contexts, metadata). 20 test cases for customer support FAQ. System Under Test (SUT) adapter: pluggable interface for any RAG system |
| **Roadmap** | R-P7 Phase 2: Deterministic metrics | Latency (p50, p95), cost (token count × price), context precision (fraction of retrieved context that is relevant), context recall (fraction of relevant context that is retrieved), citation accuracy. Unit test each with known-good and known-bad fixtures. Eval Runner: call SUT → compute metrics → assemble CaseResult |
| **Roadmap** | R-P7 Phase 3: LLM-as-judge metrics | Faithfulness (answer grounded in context?) and answer relevance (addresses question?). GPT-4 as judge, GPT-4o-mini as SUT (judge must be stronger). Structured judge output: JSON with score (0-1) + reasoning. Calibrate: 50 cases, Cohen's kappa vs human gold labels. Target: kappa ≥ 0.6 |
| **Roadmap** | R-P7 Phase 4: Context engineering + finish | Context utilization, compression ratio, position bias (lost-in-middle), context overflow. Streamlit eval dashboard: metric trends, per-version comparisons, failing cases. CI: pytest with @pytest.mark.eval, GitHub Actions on every PR. README. Mark R-P7 complete |

**Deliverable:** R-P7 COMPLETE. 5 deterministic metrics + LLM-as-judge (faithfulness, relevance) + context engineering metrics. Streamlit eval dashboard. CI integration with GitHub Actions.

### Topic 4.2 — Prompt Versioning & A/B Testing (R-P8)

| Resource | Content | What You Do |
|----------|---------|-------------|
| **Roadmap** | R-P8 Setup + Phase 1: Prompt Versioning | Docker Compose (PostgreSQL). Schema: prompts, prompt_versions (git-like, status: draft/staging/production/archived), eval_datasets, eval_items, ab_tests, test_results, metric_history, promotions. Prompt store: CRUD + versioning. Jinja2 SandboxedEnvironment. StrictUndefined. Test: create prompt, 3 versions, verify history |
| **Roadmap** | R-P8 Phase 2: A/B test runner | Load 2+ versions, render with eval dataset, call LLM, measure latency_ms + token_count + output_length, LLM-as-judge → score (0-1). Per-variant: mean_accuracy, mean_latency, mean_cost. Test: A/B test on 2 versions with 20 eval items |
| **Roadmap** | R-P8 Phase 3: Statistical significance | Win-rate, Wilson confidence interval, two-proportion z-test p-value, required sample size. Declare winner ONLY if p < 0.05 AND sample size adequate. scipy + statsmodels. Promotion/rollback API: POST /promote (atomic transaction). Python SDK: PromptClient.get_production_prompt() |
| **Roadmap** | R-P8 Phase 4: Streamlit dashboard + finish | Win-rate chart, metric comparison, sample outputs side-by-side, version history, metric trends. Promotion/rollback UI. README. Docker Compose up. Mark R-P8 complete |

**Deliverable:** R-P8 COMPLETE. Prompt versioning with Jinja2. A/B test runner. Statistical significance testing (z-test, Wilson CI). Promotion/rollback API + SDK. Streamlit dashboard.

### Topic 4.3 — LLM Serving Gateway (R-P9)

| Resource | Content | What You Do |
|----------|---------|-------------|
| **Roadmap** | R-P9 Setup + Phase 1: LLM Serving Gateway | Docker Compose (PostgreSQL + Redis). Schema: cost_records, tenants, budget_usage, model_pricing. FastAPI gateway: auth + tenant resolve → rate limiter (Redis sliding window) → budget check → semantic cache. Semantic cache: embed prompt, search Redis for cosine sim ≥ 0.95, return cached if hit. Test: same query twice → second is cache hit |
| **Roadmap** | R-P9 Phase 2: Model routing | Task type detection (simple vs complex). Simple → GPT-4o-mini, complex → GPT-4o. Confidence check: if mini_response.confidence < 7, escalate to GPT-4o. Provider failover: OpenAI → Anthropic → Ollama. Test: simulate OpenAI 5xx → automatic failover |
| **Roadmap** | R-P9 Phase 3: Cost tracking + structured logging | Log every request: tenant, model, tokens (in/out), latency, cache hit/miss, cost. Daily cost report. Verify matches provider invoices within ±2%. Admin dashboard (Streamlit): cache hit rate, cost per tenant, model distribution, budget usage |
| **Roadmap** | R-P9 Phase 4: Polish + finish | Rate limiting: sliding-window (per-minute, per-hour, per-day). HTTP 429 with Retry-After. Test: concurrent burst, no burst-through. README with cost math (87% reduction example). Mark R-P9 complete |

**Deliverable:** R-P9 COMPLETE. Semantic cache (≥50% hit rate on repeated queries). Model routing with confidence escalation. Provider failover chain. Cost tracking + structured logging. Admin dashboard. Rate limiting tested. 87% cost reduction demonstrated.

### Topic 4.4 — PII Redaction Guardrails (R-P10)

| Resource | Content | What You Do |
|----------|---------|-------------|
| **Roadmap** | R-P10 Setup + Phase 1: PII Redaction | Docker Compose (PostgreSQL). Schema: tenants, tenant_config, allowlist, redaction_events, audit_logs. FastAPI proxy middleware: intercepts every LLM API call. Layer 1: Regex patterns — email, phone, SSN, IBAN, credit card, Steuer-ID, PLZ, German address. Test: "My name is Anna Mueller, email: a.mueller@gmx.de, Steuer-ID: 12345678901" → all 3 PII types detected |
| **Roadmap** | R-P10 Phase 2: Layer 2 - NER | Microsoft Presidio + spaCy. en_core_web_lg + de_core_news_lg for German. Detect: person names, organizations, locations, dates, addresses. Merge + deduplicate Layer 1 + Layer 2. Apply tenant allowlist |
| **Roadmap** | R-P10 Phase 3: Layer 3 - LLM contextual + redaction engine | LLM catches implicit PII ("my boss is the CEO" → PERSON). Only runs if Layer 1+2 found fewer entities than expected. Redaction strategies: mask ([REDACTED_PERSON]), hash ([HASH_a3f9...]), synthetic (Faker de_DE). Output scanner: same 3-layer pipeline on LLM response. Log every redaction event |
| **Roadmap** | R-P10 Phase 4: Compliance + finish | Audit log: append-only (DB trigger blocks UPDATE/DELETE). Streamlit compliance dashboard: redaction statistics, PII type distribution, compliance report export (CSV/PDF). GDPR compliance mapping table. End-to-end test: PII prompt → redacted → LLM → no leaked PII → audit log entry. Mark R-P10 complete |

**Deliverable:** R-P10 COMPLETE. 3-layer PII detection (regex + NER + LLM). Redaction engine (mask/hash/synthetic). Compliance audit log (append-only). Streamlit dashboard. GDPR mapping. German PII patterns (Steuer-ID, PLZ) detected.

**Milestone:** 10 of 12 roadmap projects complete. Only the capstone (P11) and case study (P12) remain.

---

## Phase 5: Capstone (Projects 11–12)

**Concepts:** Full AI product architecture combining RAG + agents + multi-agent + eval + guardrails + monitoring. Architecture documentation, ADRs, presentation.

### Topic 5.1 — Customer Support AI System (R-P11)

| Resource | Content | What You Do |
|----------|---------|-------------|
| **Roadmap** | R-P11 Setup + Phase 1: Ticket ingestion | Docker Compose (PostgreSQL+pgvector + Redis + Neo4j + Airflow). Schema: tickets, ticket_classifications, responses, response_evaluations, cost_records, audit_logs, pii_redactions, human_reviews. POST /tickets accepts {customer_id, subject, body, channel}. Ticket router |
| **Roadmap** | R-P11 Phase 2: PII redaction + classification | Reuse R-P10's 3-layer detection. Every ticket: scan → redact → store. Classification: urgency (low/medium/high/critical), category (billing/technical/account/general), sentiment (positive/neutral/frustrated/angry). Test: 10 sample tickets |
| **Roadmap** | R-P11 Phase 3: RAG knowledge base | Reuse R-P1's hybrid retrieval (BM25 + pgvector + RRF) + chunking + re-ranking. Ingest KB articles (FAQ, product docs, policies). Retrieve top-5 relevant articles per ticket |
| **Roadmap** | R-P11 Phase 4: Response generation | Draft response from ticket + KB context + classification. Template: greeting + acknowledge + solution from KB + next steps + signature. Confidence score: LLM self-assessment + retrieval confidence. ≥0.8: auto-respond. <0.8: human review |
| **Roadmap** | R-P11 Phase 5: Evaluation layer | Reuse R-P7's eval harness. Evaluate every auto-response: faithfulness, answer relevance, citation accuracy. If faithfulness < 0.7: flag for human review even if confidence was high. Human review queue (Streamlit) |
| **Roadmap** | R-P11 Phase 6: Cost tracking + audit | Reuse R-P9's cost tracking. Log every LLM call: ticket_id, model, tokens, cost. Cost dashboard. Reuse R-P2's audit logging: every action logged (ingest, redact, classify, retrieve, generate, evaluate, auto-respond, human-review, send). Append-only |
| **Roadmap** | R-P11 Phase 7: End-to-end integration | Full pipeline: ticket ingest → PII redact → classify → RAG retrieve → draft → confidence → eval → auto-respond or human-review → cost track → audit log. Streamlit dashboard: ticket queue, processing status, stats, cost breakdown, eval trends. Test: 20 diverse tickets end-to-end |
| **Roadmap** | R-P11 Phase 8: Polish | README with full architecture diagram. Document 14 component-to-project mappings. Docker Compose up starts entire system. Integration tests. Mark R-P11 complete |

**Deliverable:** R-P11 COMPLETE. Full end-to-end pipeline working. 20 tickets processed. Streamlit dashboard. README with architecture + component mappings. Integration tests passing. **This is the capstone — it proves you can architect and build a production AI system.**

### Topic 5.2 — Technical Case Study (R-P12)

| Resource | Content | What You Do |
|----------|---------|-------------|
| **Roadmap** | R-P12 Phase 1: CASE_STUDY.md (first 8 sections) | Executive Summary, System Architecture, Component Design (one section per component: ingestion, PII, classification, RAG, generation, evaluation, human review, cost tracking, audit), Data Flow Diagrams, Technology Decisions, Performance & Metrics |
| **Roadmap** | R-P12 Phase 2: 5 ADRs + metrics tables | ADR-001: pgvector over Qdrant. ADR-002: LangGraph over CrewAI. ADR-003: GPT-4o-mini as default. ADR-004: Presidio + spaCy for PII. ADR-005: Streamlit for dashboards. Each: Context, Decision, Consequences, Alternatives. Metrics: latency by component, cost per ticket, eval scores by category, cache hit rate, auto-respond rate |
| **Roadmap** | R-P12 Phase 3: Remaining CASE_STUDY.md sections | Failure Modes & Mitigations (hallucination, PII leakage, cost overrun, eval false positives, retrieval failure, agent loop recursion, model outage, cache poisoning). Operational Runbook (deployment, monitoring, incident response, rollback). Lessons Learned. Target: 15–20 pages |
| **Roadmap** | R-P12 Phase 4: Presentation deck | 15-slide structure: Title, Problem, Solution Architecture, Component Deep-Dive (3–4 slides), Live Demo Screenshots, Metrics & Results, Tech Stack, Challenges & Solutions, Future Improvements, Q&A. Speaker notes. Mark R-P12 complete |

**Deliverable:** R-P12 COMPLETE. CASE_STUDY.md (15–20 pages). 5 ADRs. Metrics tables. Presentation deck outline with speaker notes. **All 12 roadmap projects done.**

---

## Resource Mapping Summary

### Ed Donners Week → Roadmap Tier → From-Scratch Phase

| Ed Donners Week | Topic | Roadmap Projects | From-Scratch Lessons |
|----------------|-------|-----------------|---------------------|
| W1 | API fundamentals, first LLM product | — (just learn) | — |
| W2 | Tool calling, multimodal, Gradio | — (just learn) | P7-L02, L03, L05, L07; P10-L01, L02, L06, L07, L08 |
| W3 | HuggingFace, open-source models, LLaMA | R-P1 (Phases 3–4) | P7-L12; P10-L11 |
| W4 | Model evaluation, code generation | R-P2 (Setup + Phases 1–2) | P10-L10; P5-L27 |
| W5 | RAG, embeddings, chunking, knowledge worker | R-P2 (Phases 3–4), R-P3 (all) | P11-L06, L07; P5-L14, L22, L23 |
| W6 | Datasets, fine-tuning GPT-4o-mini, neural nets | R-P4 (all), R-P5 (Setup + Phase 1) | P3-L03, L05 |
| W7 | QLoRA, LoRA, TRL SFT Trainer, W&B | R-P5 (Phases 2–4), R-P6 (Setup) | P11-L08 |
| W8 | Multi-agent, Modal, agent architectures | R-P6 (Phases 2–5) | P14-L01, L02, L12, L25, L28 |
| After Ed Donners | Evaluation + Infrastructure + Capstone | R-P7 through R-P12 | — (theory complete) |

### Roadmap Project → Ed Donners Week → From-Scratch Lessons

| Roadmap Project | Ed Donners | From-Scratch |
|----------------|------------|-------------|
| R-P1: Advanced RAG | W3, W5 | P7-L12; P10-L11; P11-L06, L07; P5-L14, L22, L23 |
| R-P2: Secure RAG | W4, W5 | P10-L10; P5-L27; P11-L06 |
| R-P3: Advanced RAG Architectures | W5 | P5-L14, L22 |
| R-P4: Data Analysis Agent | W6 | P3-L03, L05 |
| R-P5: Document Processing Pipeline | W6, W7 | P11-L08 |
| R-P6: Multi-Agent Systems | W7, W8 | P11-L08; P14-L01, L02, L12, L25, L28 |
| R-P7: LLM Evaluation Harness | — | P10-L10; P5-L27 |
| R-P8: Prompt A/B Testing | — | — |
| R-P9: LLM Serving Gateway | — | — |
| R-P10: PII Redaction Guardrails | — | — |
| R-P11: Customer Support AI System | — (reuses P1–P10) | — |
| R-P12: Technical Case Study | — (documents P11) | — |

### From-Scratch [CORE] Lessons → When You Study Them

| Lesson | Title | Paired With |
|--------|-------|-------------|
| P7-L02 | Self-Attention from Scratch | Before W3 (concept first, then Ed's high-level) |
| P7-L03 | Multi-Head Attention | Before W3 |
| P7-L05 | Full Transformer: Encoder + Decoder | Before W3 |
| P7-L07 | GPT - Causal Language Modeling | W2 (after first API calls) |
| P7-L12 | KV Cache, Flash Attention, Inference Optimization | W3 (inside LLaMA) |
| P10-L01 | Tokenizers - BPE, WordPiece, SentencePiece | W2 (after tiktoken intro) |
| P10-L02 | Building a Tokenizer from Scratch [DEEP] | W3 (tokenizers in action) |
| P10-L06 | Instruction Tuning - SFT | W2 (before fine-tuning) |
| P10-L07 | RLHF - Reward Model + PPO [DEEP] | W2 (skim) |
| P10-L08 | DPO [DEEP] | W2 (skim) |
| P10-L10 | Evaluation - Benchmarks, Evals | W4 (model evaluation) |
| P10-L11 | Quantization: INT8, GPTQ, AWQ, GGUF | W3 (running open-source models) |
| P3-L03 | Backpropagation from Scratch | W6 (before neural nets in PyTorch) |
| P3-L05 | Loss Functions: MSE, Cross-Entropy, Contrastive | W6 (before fine-tuning) |
| P5-L14 | Information Retrieval and Search | W5 (RAG evaluation + advanced RAG) |
| P5-L22 | Embedding Models Deep Dive | W5 (embeddings) |
| P5-L23 | Chunking Strategies for RAG | W5 (document chunking) |
| P5-L27 | LLM Evaluation: RAGAS, DeepEval, G-Eval | W4 (model evaluation) |
| P11-L06 | RAG: Retrieval-Augmented Generation | W5 (RAG pipeline) |
| P11-L07 | Advanced RAG: Chunking, Reranking | W5 (advanced RAG) |
| P11-L08 | Fine-Tuning with LoRA and QLoRA | W7 (QLoRA training) |
| P14-L01 | The Agent Loop | W8 (agentic AI) |
| P14-L02 | ReWOO and Plan-and-Execute | W8 |
| P14-L12 | Anthropic's Workflow Patterns | W8 (agent architectures) |
| P14-L25 | Multi-Agent Debate and Collaboration | W8 (multi-agent capstone) |
| P14-L28 | Orchestration Patterns - Supervisor, Swarm, Hierarchical | W8 |

---

## What to Ruthlessly Skip

### Ed Donners — Skip These

- W1: Environment setup, Cursor/UV/Git walkthrough (you know your tools)
- W3: Google Colab + GPU setup (you know Colab)
- W6: Random Forest, XGBoost, Bag of Words, traditional ML baselines (you know these from data engineering)
- W6: MLOps general concepts (you know MLOps; focus on the LLM-specific parts)
- Any lecture where Ed spends > 5 minutes on basic Python or git

### From-Scratch Curriculum — Skip These Phases Entirely

- Phase 0: Python Foundations (you know Python)
- Phase 1: Data Structures (you know data structures)
- Phase 2: Math for ML (you know linear algebra, calculus, probability)
- Phase 3: NumPy, Pandas, Matplotlib basics (you use these daily)

### From-Scratch Curriculum — Skip These Lesson Types

- All [SKIP] lessons (marked for experienced developers)
- Most [DEEP] lessons in Phase 1 (unless shaky on a specific topic)
- P4 (CNNs): convolution, pooling, ResNet — skip unless working with vision models
- P6 (RNNs): vanilla RNN, LSTM, GRU — skip (transformers replaced RNNs)
- P8 (Diffusion): all lessons — skip unless working with image generation
- P9 (Reinforcement Learning): all lessons — skip (RLHF covered conceptually in P10)
- P12 (Distributed Training): all lessons — skip (you won't train models at scale)
- P13 (Inference at Scale): TensorRT, vLLM, model serving — skim only
- P15 (Security): prompt injection — skim (R-P10 covers PII, which is more relevant)
- P16 (Future Directions): all lessons — skip (speculative, not actionable)

### Roadmap Projects — Do Not Skip Any

All 12 roadmap projects are essential. They are the proof of your skills. Even if a project seems simple (e.g., P8 prompt versioning), it teaches patterns you'll need for the capstone (P11).

---

## AI Code Generation Workflow

The roadmap projects use AI to generate boilerplate. This is the disciplined workflow that ensures you actually understand what you ship.

1. **Read the Plan.** Before generating any code, read the full IMPLEMENTATION_PLAN.md. Understand: what are we building, architecture, database tables, API surface. If you can't explain the project after reading the plan, read it again.

2. **Generate in Small Chunks.** Ask AI to generate one component at a time (e.g., "write the DocumentLoader class"). Never ask for the whole project at once. If a chunk is > 100 lines, ask for smaller pieces.

3. **Read Every Line.** Before pasting generated code, read every line. Ask: What does this line do? Why is it here? If you can't answer, ask the AI to explain. If the explanation doesn't make sense, ask again or look it up. Never ship code you don't understand.

4. **Run + Test Immediately.** After each component, run it. Write a quick test. Does it do what the plan says? If not, debug. Don't move to the next component until the current one works.

5. **Document Your Understanding.** After finishing a component, write a 2-3 sentence comment at the top explaining what it does and why. If you can't write the comment, you don't understand it yet.

6. **Connect Components.** Once individual components work, connect them. Integration bugs are in how YOU connected things — AI doesn't have that context. Debug them yourself.

7. **Write the README.** The README is your proof of understanding. Write it yourself, not with AI. Include: architecture diagram, component descriptions, how to run, metrics. If you can't write the README, you don't understand the project.

> **The golden rule:** AI generates code, you generate understanding. If you ship 12 projects but can't explain any of them, you wasted your time. If you ship 12 projects and can explain every architectural decision, you're an AI engineer.

---

## Tech Stack Summary

| Category | Technology | Where Used |
|----------|-----------|------------|
| LLM Provider | OpenAI (GPT-4o, GPT-4o-mini) | All projects |
| Alternative Provider | Anthropic Claude, Ollama (local) | R-P9 failover |
| Vector DB | PostgreSQL + pgvector | R-P1, P2, P3, P6, P11 |
| Graph DB | Neo4j | R-P3 (GraphRAG) |
| Cache | Redis | R-P6, P9 |
| Framework | LangGraph | R-P6 (multi-agent) |
| API | FastAPI | All projects |
| UI | Streamlit | All projects (dashboards) |
| Orchestration | Apache Airflow | R-P5 (document pipeline) |
| Embeddings | BGE-large-en-v1.5, CLIP | R-P1, P3 |
| Re-ranker | BGE-reranker-large | R-P1 |
| PII Detection | Presidio + spaCy | R-P10, P11 |
| SQL Validation | sqlglot | R-P4 |
| Prompt Templating | Jinja2 | R-P8 |
| Statistics | scipy + statsmodels | R-P8 |
| Fine-tuning | TRL SFT Trainer, QLoRA | Ed Donners W7 (not roadmap) |
| Experiment Tracking | Weights & Biases | Ed Donners W7 (not roadmap) |
| Container | Docker + Docker Compose | All projects |
| Testing | pytest + pytest-asyncio | All projects |
| Database | PostgreSQL 16 | All projects |
| CI/CD | GitHub Actions | All projects |

---

## German Market Focus

For AI engineering roles in Germany, emphasize these projects and patterns:

| Concern | Project | What to Highlight |
|---------|---------|-------------------|
| GDPR compliance | R-P10 (PII Redaction) | 3-layer detection, German PII patterns (Steuer-ID, PLZ), compliance logging, GDPR mapping |
| Audit trails | R-P2 (Secure RAG), R-P11 (Capstone) | Append-only audit logs, immutable triggers, query API for compliance |
| Enterprise patterns | R-P9 (LLM Gateway) | Cost control, rate limiting, provider failover, tenant budgets |
| Production reliability | R-P7 (Eval Harness), R-P8 (A/B Testing) | Systematic evaluation, regression testing, statistical significance |
| Data residency | R-P9 (failover to Ollama) | Local model fallback keeps data on-premise when needed |

---

*This plan interleaves three resources: Ed Donners for concept, from-scratch for depth, roadmap projects for proof. All three together make you an AI engineer.*
