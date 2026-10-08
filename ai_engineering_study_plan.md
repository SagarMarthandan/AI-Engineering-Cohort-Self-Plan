# AI Engineering Study Plan
## A Structured 8-Week Roadmap to AI Implementation

> **Philosophy**: Theory + Coding every week. Build incrementally — each week's project compounds on the previous. By Week 8, you have a portfolio-ready capstone.

---

## 📋 Prerequisites Checklist (Complete Before Week 1)

### Knowledge
- [ ] Python proficiency (functions, classes, decorators, async/await)
- [ ] Basic linear algebra (vectors, dot products, matrices)
- [ ] REST API concepts (requests, JSON, headers, status codes)
- [ ] Git basics (clone, commit, branch, push)

### Tools to Install
- [ ] Python 3.11+ with `uv` or `poetry` for dependency management
- [ ] VS Code with Python + Jupyter extensions
- [ ] OpenAI API key (or Anthropic / open-source via Ollama)
- [ ] Docker Desktop (for vector DBs, Neo4j)
- [ ] Hugging Face account (for models & datasets)

### Accounts
- [ ] OpenAI Platform — https://platform.openai.com
- [ ] Anthropic Console — https://console.anthropic.com
- [ ] Hugging Face — https://huggingface.co
- [ ] Pinecone (free tier) — https://pinecone.io (optional, ChromaDB is local)
- [ ] Neo4j Aura (free tier) — https://neo4j.com/cloud/aura-free/

---

## Week 1: Terminology & Prerequisites
### Goal: Understand LLM fundamentals and make your first API calls

### 📖 Theory (Days 1-3)

**Day 1 — LLM Core Concepts**
- [ ] Read: [OpenAI Tokenizer](https://platform.openai.com/tokenizer) — interactively tokenize text
- [ ] Study: Tokens vs words (~4 chars/token), BPE tokenization basics
- [ ] Study: Context window — what it is, why it limits input size
- [ ] Study: Parameters — what "7B" means, why scale matters
- [ ] Watch: 3Blue1Brown's [But what is a GPT?](https://youtu.be/wjZofJX0v4M)

**Day 2 — How LLMs Are Trained**
- [ ] Study: Pre-training (next-token prediction on internet-scale text)
- [ ] Study: Post-training (SFT → RLHF/DPO → instruction tuning)
- [ ] Study: Inference — autoregressive generation, KV cache, sampling
- [ ] Read: [Anthropic's prompt engineering guide](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview) (intro sections)

**Day 3 — Embeddings & Prompting**
- [ ] Study: Embeddings — mapping text to high-dimensional vectors
- [ ] Study: Semantic similarity — why cosine similarity works
- [ ] Study: Prompt types: system, user, assistant, few-shot
- [ ] Study: Structured outputs — JSON mode, function calling schema
- [ ] Read: [OpenAI Structured Outputs guide](https://platform.openai.com/docs/guides/structured-outputs)

### 💻 Coding (Days 4-7)

**Day 4 — First API Calls**
- [ ] Set up Python project with `uv init ai-eng-study`
- [ ] Install: `openai`, `anthropic`, `tiktoken`, `numpy`
- [ ] Exercise: Call OpenAI API with a system + user prompt
- [ ] Exercise: Call with `response_format={"type": "json_object"}` and parse result
- [ ] Exercise: Count tokens with `tiktoken` before and after a call

**Day 5 — Embeddings & Cosine Similarity**
- [ ] Exercise: Generate embeddings for 10 sentences using `text-embedding-3-small`
- [ ] Exercise: Implement cosine similarity from scratch (no sklearn):
  ```python
  def cosine_sim(a, b):
      return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))
  ```
- [ ] Exercise: Build a semantic search function — query → top-3 most similar sentences
- [ ] Exercise: Visualize embeddings with t-SNE or UMAP (optional)

**Day 6 — Prompting Patterns**
- [ ] Exercise: Zero-shot vs few-shot prompting on a classification task
- [ ] Exercise: System prompt engineering — set persona, constraints, output format
- [ ] Exercise: Chain-of-thought prompting on a reasoning task
- [ ] Exercise: Compare outputs across temperature 0.0, 0.7, 1.0

**Day 7 — Weekly Project + Review**
- [ ] **Mini-Project**: Build a "Semantic FAQ Matcher"
  - Input: a list of 20 Q&A pairs
  - User asks a question → embed it → find closest FAQ → return answer
  - Output: CLI tool with formatted results
- [ ] Write a 1-page summary of what you learned
- [ ] Push code to GitHub

### 📚 Recommended Reading
- *Build a Large Language Model (From Scratch)* — Sebastian Raschka (Ch. 1-2)
- [Karpathy's "Let's build GPT" video](https://youtu.be/kCc8FmEb1nY) (first 30 min)
- OpenAI Cookbook — [Embeddings section](https://cookbook.openai.com/articles/embeddings)

---

## Week 2: RAG — Components & Architecture
### Goal: Build an end-to-end RAG pipeline

### 📖 Theory (Days 1-3)

**Day 1 — Why RAG**
- [ ] Study: Knowledge cutoff problem — LLMs don't know recent events
- [ ] Study: Hallucination — confident fabrication, why it happens
- [ ] Study: RAG vs fine-tuning — when to use each
- [ ] Read: [Retrieval-Augmented Generation paper (Lewis et al., 2020)](https://arxiv.org/abs/2005.11401)

**Day 2 — The 5-Stage Pipeline**
- [ ] Study: Ingest — loading documents (PDF, HTML, Markdown, APIs)
- [ ] Study: Chunk — splitting strategies (fixed-size, recursive, semantic, document-based)
- [ ] Study: Embed — choosing embedding models (OpenAI, Cohere, open-source)
- [ ] Study: Index — vector DBs and indexing algorithms
- [ ] Study: Retrieve — similarity search, top-k

**Day 3 — Vector Databases Deep Dive**
- [ ] Study: HNSW (Hierarchical Navigable Small World) — graph-based ANN
- [ ] Study: IVF (Inverted File Index) — clustering-based ANN
- [ ] Study: PQ (Product Quantization) — compression for memory efficiency
- [ ] Compare: ChromaDB vs Pinecone vs Weaviate vs Qdrant
- [ ] Read: [Pinecone's "Vector Indexes" guide](https://www.pinecone.io/learn/series/faiss/vector-indexes/)

### 💻 Coding (Days 4-7)

**Day 4 — Document Ingestion & Chunking**
- [ ] Install: `langchain`, `langchain-community`, `chromadb`, `pypdf`
- [ ] Exercise: Load a PDF with `PyPDFLoader`
- [ ] Exercise: Chunk with `RecursiveCharacterTextSplitter` (chunk_size=1000, overlap=200)
- [ ] Exercise: Compare chunk outputs for different strategies
- [ ] Exercise: Inspect chunk metadata (source, page, char count)

**Day 5 — Embedding & Indexing**
- [ ] Exercise: Create ChromaDB collection
- [ ] Exercise: Embed chunks with OpenAI embeddings and store in ChromaDB
- [ ] Exercise: Query the collection and inspect raw results
- [ ] Exercise: Experiment with different embedding models — compare dimensionality

**Day 6 — Retrieval & Augmentation**
- [ ] Exercise: Build retrieval function — query → top-k chunks
- [ ] Exercise: Build augmentation — stuff retrieved chunks into prompt
- [ ] Exercise: Generate answer with LLM using retrieved context
- [ ] Exercise: Add source citations to the output

**Day 7 — Weekly Project**
- [ ] **Mini-Project**: "PDF Chatbot"
  - Load a technical PDF (e.g., a research paper or documentation)
  - Full RAG pipeline: ingest → chunk → embed → index → retrieve → answer
  - Interactive CLI: ask questions, get answers with citations
  - Log retrieval results for debugging
- [ ] Write summary: what chunking strategy worked best?
- [ ] Push to GitHub

### 📚 Recommended Reading
- [LangChain RAG tutorial](https://python.langchain.com/docs/tutorials/rag)
- [ChromaDB documentation](https://docs.trychroma.com/)
- *Designing Machine Learning Systems* — Chip Huyen (Ch. 7 on retrieval)

---

## Week 3: Advanced RAG
### Goal: Break through naive RAG accuracy ceilings

### 📖 Theory (Days 1-3)

**Day 1 — Why Naive RAG Fails**
- [ ] Study: Vocabulary mismatch — user query words ≠ document words
- [ ] Study: Lost in the middle — LLMs forget middle context
- [ ] Study: Chunk boundary problems — answers split across chunks
- [ ] Study: Retrieval vs generation quality — fixing the right stage

**Day 2 — Query Transformation**
- [ ] Study: Query expansion — adding synonyms and related terms
- [ ] Study: HyDE (Hypothetical Document Embeddings) — generate fake answer, embed that
- [ ] Study: Multi-query — LLM generates N reformulations, retrieve for each
- [ ] Study: Step-back prompting — abstract the query first
- [ ] Read: [HyDE paper](https://arxiv.org/abs/2212.10496)

**Day 3 — Reranking & Hybrid Search**
- [ ] Study: Cross-encoder reranking — why bi-encoders retrieve, cross-encoders rerank
- [ ] Study: Cohere Rerank API, BGE-reranker, Jina reranker
- [ ] Study: Hybrid search — dense (semantic) + sparse (BM25/keyword)
- [ ] Study: Metadata filtering — pre-filtering by source, date, category
- [ ] Study: RAG evals — faithfulness (answer grounded in context?), answer relevance
- [ ] Read: [RAGAS documentation](https://docs.ragas.io/)

### 💻 Coding (Days 4-7)

**Day 4 — Query Rewriting**
- [ ] Install: `ragas`, `cohere` (or use BGE reranker)
- [ ] Exercise: Implement HyDE — generate hypothetical answer, embed, retrieve
- [ ] Exercise: Implement multi-query — generate 3 reformulations, union results
- [ ] Exercise: Compare retrieval quality: naive vs HyDE vs multi-query

**Day 5 — Reranking & Hybrid Search**
- [ ] Exercise: Add cross-encoder reranking to Week 2 pipeline
- [ ] Exercise: Implement BM25 sparse retrieval alongside dense
- [ ] Exercise: Combine scores: `final = α * dense + (1-α) * sparse`
- [ ] Exercise: Tune α and measure retrieval accuracy

**Day 6 — Metadata Filtering & Self-RAG**
- [ ] Exercise: Add metadata to chunks (source, section, date)
- [ ] Exercise: Implement pre-filtering before retrieval
- [ ] Exercise: Implement Self-RAG — LLM decides if retrieval is needed
- [ ] Exercise: Implement answer grounding check — is answer supported by context?

**Day 7 — Weekly Project + Eval Suite**
- [ ] **Mini-Project**: "Advanced RAG with Benchmarking"
  - Take Week 2 PDF Chatbot
  - Add: HyDE + reranking + hybrid search + metadata filtering
  - Build eval suite with RAGAS: faithfulness, answer relevance, context precision
  - Create a comparison table: naive RAG vs advanced RAG metrics
  - Document which technique gave the biggest improvement
- [ ] Push to GitHub with eval results in README

### 📚 Recommended Reading
- [RAGAS: Automated Evaluation of RAG](https://arxiv.org/abs/2309.15217)
- [Cohere Reranker docs](https://docs.cohere.com/docs/reranking)
- [LangChain Advanced RAG guide](https://python.langchain.com/docs/how_to/#retrievers)

---

## Week 4: RAG Architectures & Specialised Types
### Goal: Understand when to use which RAG architecture

### 📖 Theory (Days 1-3)

**Day 1 — Production RAG Pitfalls**
- [ ] Study: Index freshness — keeping vectors up to date
- [ ] Study: Scale challenges — millions of documents
- [ ] Study: Multi-tenancy — per-user or per-org indices
- [ ] Study: Evaluation drift — production queries ≠ test queries
- [ ] Study: Cost optimization — embedding costs, retrieval latency

**Day 2 — GraphRAG & KAG**
- [ ] Study: GraphRAG — knowledge graphs as retrieval backends
- [ ] Study: Entity extraction → relationship extraction → graph construction
- [ ] Study: Community detection for summarization
- [ ] Study: KAG (Knowledge-Augmented Generation) — structured KB + LLM
- [ ] Study: When graphs beat vectors: multi-hop reasoning, relationship queries
- [ ] Read: [GraphRAG paper (Microsoft, 2024)](https://arxiv.org/abs/2404.16130)

**Day 3 — Agentic RAG & Multimodal**
- [ ] Study: Agentic RAG — LLM decides what to retrieve, when, how many times
- [ ] Study: Iterative retrieval — retrieve → assess → retrieve again if needed
- [ ] Study: Multimodal RAG — embedding images, tables, charts
- [ ] Study: LightRAG — simplified graph-based RAG
- [ ] Study: Decision framework — when to use vanilla vs GraphRAG vs Agentic RAG
- [ ] Read: [Agentic RAG with LangGraph](https://blog.langchain.dev/agentic-rag-with-langgraph/)

### 💻 Coding (Days 4-7)

**Day 4 — GraphRAG Setup**
- [ ] Install: `neo4j`, `neo4j-python-driver`, `langchain-graph`
- [ ] Set up Neo4j (Docker or Aura free tier)
- [ ] Exercise: Load a document, extract entities with LLM
- [ ] Exercise: Extract relationships, build graph in Neo4j
- [ ] Exercise: Query the graph with Cypher

**Day 5 — GraphRAG Retrieval**
- [ ] Exercise: Implement graph-based retrieval — entity → neighbors → context
- [ ] Exercise: Compare graph retrieval vs vector retrieval on multi-hop questions
- [ ] Exercise: Implement community summarization
- [ ] Exercise: Measure: which approach answers "Who works with X?" better?

**Day 6 — Agentic RAG**
- [ ] Exercise: Build an agentic RAG loop:
  1. LLM receives question
  2. LLM decides: retrieve more? or answer?
  3. If retrieve: formulate query, get chunks, add to context
  4. Repeat until confident or max iterations
- [ ] Exercise: Compare agentic RAG vs single-retrieval RAG on complex questions

**Day 7 — Weekly Project**
- [ ] **Mini-Project**: "GraphRAG vs Vanilla RAG Comparison"
  - Use a dataset with rich relationships (e.g., a company wiki, or a set of news articles about people/orgs)
  - Build both vanilla RAG (Week 2-3) and GraphRAG (Neo4j)
  - Create 20 test questions (10 simple, 10 multi-hop)
  - Benchmark: accuracy, latency, cost
  - Write a decision framework: when to use which
- [ ] Push to GitHub with comparison results

### 📚 Recommended Reading
- [Neo4j GraphRAG Python package](https://github.com/neo4j/neo4j-graphrag-python)
- [Microsoft GraphRAG](https://microsoft.github.io/graphrag/)
- [LightRAG GitHub](https://github.com/HKUDS/LightRAG)

---

## Week 5: Single-Agent Systems
### Goal: Build an agent with tools and structured outputs

### 📖 Theory (Days 1-3)

**Day 1 — What Is an Agent?**
- [ ] Study: Agent = LLM + tools + loop (autonomy)
- [ ] Study: LLM call vs pipeline vs agent — the autonomy spectrum
- [ ] Study: When you need an agent vs when a chain suffices
- [ ] Study: Agent components: planner, executor, memory, tools
- [ ] Read: [Lilian Weng's "LLM Powered Autonomous Agents"](https://lilianweng.github.io/posts/2023-06-23-agent/)

**Day 2 — Tool Calling & ReAct**
- [ ] Study: Function/tool calling — JSON schema → LLM selects tool → execute → feed back
- [ ] Study: ReAct pattern: Reason → Act → Observe → repeat
- [ ] Study: Tool design principles — clear names, descriptions, schemas
- [ ] Study: Error handling — what when a tool fails?
- [ ] Read: [ReAct paper (Yao et al., 2022)](https://arxiv.org/abs/2210.03629)

**Day 3 — Pydantic AI & Guardrails**
- [ ] Study: Pydantic AI — type-safe agents with structured outputs
- [ ] Study: Structured outputs as agent contracts
- [ ] Study: Guardrails — input validation, output validation, safety checks
- [ ] Study: Agent memory — conversation history management
- [ ] Read: [Pydantic AI documentation](https://ai.pydantic.dev/)

### 💻 Coding (Days 4-7)

**Day 4 — Tool Calling Basics**
- [ ] Install: `pydantic-ai`, `langchain`, `langgraph`
- [ ] Exercise: Define 3 tools with JSON schemas:
  - `get_weather(city: str) -> str`
  - `calculate(expression: str) -> float`
  - `search_wikipedia(query: str) -> str`
- [ ] Exercise: Call OpenAI with tools, parse tool calls, execute, feed results back
- [ ] Exercise: Build a simple tool-calling loop (manual ReAct)

**Day 5 — ReAct Agent**
- [ ] Exercise: Build a full ReAct loop:
  ```
  while not done:
      thought = llm.reason(question, observations)
      action = llm.select_tool(thought, available_tools)
      observation = execute(action)
      observations.append(observation)
  ```
- [ ] Exercise: Add max iterations limit
- [ ] Exercise: Add error handling for tool failures

**Day 6 — Pydantic AI Agent**
- [ ] Exercise: Build an agent with Pydantic AI:
  - Define typed tools with Pydantic models
  - Define structured output model
  - Add system prompt with guardrails
- [ ] Exercise: Test with edge cases — bad input, tool failure, ambiguous request
- [ ] Exercise: Add conversation memory

**Day 7 — Weekly Project**
- [ ] **Mini-Project**: "Research Assistant Agent"
  - Tools: web search, Wikipedia lookup, calculator, file reader
  - ReAct loop with max 10 iterations
  - Structured output: `{answer, sources, confidence, reasoning_steps}`
  - Guardrails: reject harmful requests, validate output schema
  - CLI interface: ask a question, watch the agent reason
- [ ] Push to GitHub with example traces

### 📚 Recommended Reading
- [Pydantic AI examples](https://ai.pydantic.dev/examples/)
- [LangGraph quickstart](https://langchain-ai.github.io/langgraph/tutorials/introduction/)
- [OpenAI Function Calling guide](https://platform.openai.com/docs/guides/function-calling)

---

## Week 6: Multi-Agent Systems
### Goal: Build a multi-agent pipeline with orchestration

### 📖 Theory (Days 1-3)

**Day 1 — Why Multi-Agent?**
- [ ] Study: Parallelism — multiple agents working simultaneously
- [ ] Study: Specialization — each agent expert in one domain
- [ ] Study: Separation of concerns — cleaner architecture
- [ ] Study: When NOT to use multi-agent — overhead, cost, complexity
- [ ] Study: Multi-agent vs single-agent-with-many-tools

**Day 2 — Design Patterns**
- [ ] Study: Orchestrator-Worker — one agent delegates to specialists
- [ ] Study: Routing — classifier agent sends to right specialist
- [ ] Study: Pipeline / Sequential — agent A → B → C
- [ ] Study: Debate — agents argue, judge picks best
- [ ] Study: Supervisor pattern — orchestrator manages state
- [ ] Read: [LangGraph multi-agent guide](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/multi-agent-collaboration/)

**Day 3 — Multi-Agent Challenges**
- [ ] Study: Orchestration — who talks to whom, when?
- [ ] Study: Information isolation — agents don't share all context
- [ ] Study: Planning — decomposing tasks across agents
- [ ] Study: Memory systems — short-term (conversation) vs long-term (persistent)
- [ ] Study: Cost & latency — N agents = N× cost, coordination overhead
- [ ] Study: Failure modes — cascading errors, infinite loops, deadlocks

### 💻 Coding (Days 4-7)

**Day 4 — LangGraph Basics**
- [ ] Install: `langgraph`
- [ ] Exercise: Build a simple graph: `node_a → node_b → node_c`
- [ ] Exercise: Add conditional edges (routing)
- [ ] Exercise: Define state with `TypedDict`
- [ ] Exercise: Visualize the graph

**Day 5 — Orchestrator-Worker Pattern**
- [ ] Exercise: Build orchestrator agent that:
  1. Receives a research task
  2. Decomposes into subtasks
  3. Assigns to worker agents
  4. Collects results
  5. Synthesizes final answer
- [ ] Exercise: Define 2 workers: "Researcher" (web search tool) and "Analyst" (data analysis tool)

**Day 6 — Routing + Critic Pattern**
- [ ] Exercise: Add a router agent — classifies query type → routes to specialist
- [ ] Exercise: Add a critic agent — reviews output, requests revision if needed
- [ ] Exercise: Build full pipeline: Router → Specialist → Critic → (revise or output)
- [ ] Exercise: Add short-term memory (conversation state)

**Day 7 — Weekly Project**
- [ ] **Mini-Project**: "Research + Writer + Critic Pipeline"
  - **Router Agent**: classifies input (technical? creative? analytical?)
  - **Research Agent**: gathers information using tools
  - **Writer Agent**: produces a draft based on research
  - **Critic Agent**: reviews draft, provides feedback
  - **Revision Loop**: writer revises based on critic feedback (max 3 rounds)
  - **Orchestrator**: manages the full flow with LangGraph
  - Output: a polished report with research sources
  - Log: total cost, latency, agent interactions
- [ ] Push to GitHub with architecture diagram

### 📚 Recommended Reading
- [LangGraph documentation](https://langchain-ai.github.io/langgraph/)
- [AutoGen multi-agent framework](https://microsoft.github.io/autogen/)
- [CrewAI examples](https://docs.crewai.com/examples/)

---

## Week 7: Context Engineering, Memory & Evaluation
### Goal: Run a full eval suite on your multi-agent system

### 📖 Theory (Days 1-3)

**Day 1 — Context Engineering**
- [ ] Study: Context engineering vs prompt engineering — the shift
- [ ] Study: What goes in context: instructions, tools, history, retrieved docs, examples
- [ ] Study: Context window management — prioritization, compression, sliding window
- [ ] Study: Context pollution — too much info degrades performance
- [ ] Study: System prompt as context architecture
- [ ] Read: [Anthropic's "Context Engineering" post](https://www.anthropic.com/news/contextual-retrieval)

**Day 2 — Memory Systems**
- [ ] Study: Short-term memory — conversation history, working memory
- [ ] Study: Long-term memory — persistent storage, episodic, semantic
- [ ] Study: Memory compression — summarization over time
- [ ] Study: Memory retrieval — bringing relevant past context back
- [ ] Study: Implementation: in-memory, Redis, vector DB for episodic memory

**Day 3 — Evaluation**
- [ ] Study: Why evals matter — you can't improve what you can't measure
- [ ] Study: Quantitative evals: accuracy, F1, exact match, tool-call accuracy
- [ ] Study: Qualitative evals: LLM-as-a-Judge, rubric design, pairwise comparison
- [ ] Study: Eval dataset creation — golden sets, adversarial examples
- [ ] Study: Observability — tracing, logging, monitoring in production
- [ ] Read: [LLM-as-a-Judge paper (Zheng et al., 2023)](https://arxiv.org/abs/2306.05685)

### 💻 Coding (Days 4-7)

**Day 4 — Context Engineering**
- [ ] Exercise: Audit your Week 6 pipeline's context usage
- [ ] Exercise: Implement context compression — summarize old conversation turns
- [ ] Exercise: Implement context prioritization — system prompt > tools > recent history > old history
- [ ] Exercise: Measure token usage before and after optimization

**Day 5 — Memory Systems**
- [ ] Exercise: Add short-term memory to your Week 6 agents (conversation buffer)
- [ ] Exercise: Implement conversation summary memory (compress every 5 turns)
- [ ] Exercise: Implement long-term memory — store key facts in a vector DB
- [ ] Exercise: Test: agent remembers facts from previous sessions

**Day 6 — Quantitative Evals**
- [ ] Install: `langsmith` or use custom eval framework
- [ ] Exercise: Create a golden dataset — 20 questions with expected answers
- [ ] Exercise: Implement exact match accuracy
- [ ] Exercise: Implement F1 score (token overlap)
- [ ] Exercise: Implement tool-call accuracy — did the agent call the right tools?
- [ ] Exercise: Run eval suite on Week 6 pipeline, record baseline metrics

**Day 7 — LLM-as-a-Judge + Weekly Project**
- [ ] Exercise: Design a rubric for evaluating agent outputs:
  - Correctness (1-5)
  - Completeness (1-5)
  - Source quality (1-5)
  - Reasoning clarity (1-5)
- [ ] Exercise: Implement LLM-as-a-Judge with GPT-4 or Claude
- [ ] Exercise: Run both quantitative + qualitative evals
- [ ] **Mini-Project**: "Eval Suite for Multi-Agent Pipeline"
  - Take Week 6 Research + Writer + Critic pipeline
  - Build comprehensive eval suite:
    - 30 test cases with golden answers
    - Quantitative: accuracy, F1, tool-call accuracy, latency, cost
    - Qualitative: LLM-as-a-Judge with rubric
  - Generate eval report with charts/tables
  - Identify top 3 failure modes and document fixes
- [ ] Push to GitHub with eval report

### 📚 Recommended Reading
- [LangSmith documentation](https://docs.smith.langchain.com/)
- [Braintrust eval framework](https://www.braintrust.dev/docs)
- [OpenAI Evals framework](https://github.com/openai/evals)

---

## Week 8: Capstone Project
### Goal: Build, evaluate, and present a production-grade AI system

### 📖 Theory (Days 1-2)

**Day 1 — From Prototype to Production**
- [ ] Study: Production considerations — reliability, cost, latency, monitoring
- [ ] Study: Engineering decision framework:
  - RAG type: vanilla vs GraphRAG vs Agentic — based on data complexity
  - Agent pattern: single vs multi-agent — based on task decomposition need
  - Stack: LangChain vs LangGraph vs Pydantic AI vs custom
- [ ] Study: MVP scoping — what to cut, what to keep
- [ ] Study: Architecture documentation — system diagrams, data flow

**Day 2 — Presenting AI Systems**
- [ ] Study: Interview presentation structure:
  1. Problem statement (30 sec)
  2. Architecture overview (1 min)
  3. Key engineering decisions (1 min)
  4. Demo (2 min)
  5. Eval results (1 min)
  6. Tradeoffs & future work (30 sec)
- [ ] Study: How to talk about evals in interviews — metrics, baselines, improvements
- [ ] Study: How to discuss tradeoffs honestly

### 💻 Coding (Days 3-6)

**Day 3 — Capstone Scoping & Architecture**
- [ ] Choose a real-world problem (see project ideas below)
- [ ] Define MVP: minimum features for a useful system
- [ ] Design architecture: draw system diagram
- [ ] Choose stack and justify decisions
- [ ] Create project skeleton

**Day 4-5 — Capstone Implementation**
- [ ] Implement core functionality
- [ ] Integrate components from previous weeks (RAG, agents, evals)
- [ ] Add error handling and edge case management
- [ ] Optimize: context engineering, cost, latency

**Day 6 — Capstone Evaluation**
- [ ] Build eval suite (reuse Week 7 framework)
- [ ] Run quantitative + qualitative evals
- [ ] Create comparison: baseline vs optimized
- [ ] Document failure modes and mitigations
- [ ] Write README with architecture, setup, results

**Day 7 — Demo & Presentation**
- [ ] Prepare 7-minute presentation
- [ ] Practice demo flow
- [ ] Prepare for Q&A: architecture decisions, tradeoffs, eval methodology
- [ ] Record a demo video (optional but recommended for portfolio)
- [ ] Polish GitHub repo: README, architecture diagram, eval results

### 🏗️ Capstone Project Ideas

1. **Intelligent Code Documentation Generator**
   - RAG over codebase + agent that writes docs
   - Eval: doc accuracy, completeness, developer satisfaction

2. **Multi-Source Research Assistant**
   - Agentic RAG across multiple data sources (PDFs, web, APIs)
   - Multi-agent: researcher + synthesizer + fact-checker
   - Eval: answer accuracy, source quality

3. **Customer Support Knowledge Bot**
   - GraphRAG over support tickets + product docs
   - Agent with tools: search, escalate, create ticket
   - Eval: resolution rate, faithfulness, latency

4. **Financial Report Analyzer**
   - Multimodal RAG over PDFs with tables/charts
   - Agent that answers questions about financial data
   - Eval: numerical accuracy, citation correctness

5. **Personalized Learning Path Generator**
   - RAG over course catalog + skills taxonomy
   - Multi-agent: assessor + planner + recommender
   - Eval: path relevance, coverage, personalization

### 📚 Recommended Reading
- [Chip Huyen's "AI Engineering" book](https://www.oreilly.com/library/view/ai-engineering/9781098188461/)
- [Eugene Yan's ML system design blog](https://eugeneyan.com/)
- [Hamel Husain's evals blog](https://hamel.dev/blog/evals/)

---

## 📊 Weekly Deliverables Summary

| Week | Project | Key Skills | Eval Component |
|------|---------|------------|----------------|
| 1 | Semantic FAQ Matcher | API calls, embeddings, cosine similarity | Manual testing |
| 2 | PDF Chatbot | RAG pipeline, ChromaDB, chunking | Manual testing |
| 3 | Advanced RAG + Benchmark | HyDE, reranking, hybrid search | RAGAS eval suite |
| 4 | GraphRAG vs Vanilla RAG | Neo4j, graph retrieval, agentic RAG | Comparison benchmark |
| 5 | Research Assistant Agent | ReAct, tool calling, Pydantic AI | Manual + edge cases |
| 6 | Research + Writer + Critic | LangGraph, orchestration, routing | Manual testing |
| 7 | Eval Suite for Multi-Agent | Context engineering, memory, LLM-as-Judge | Full eval suite |
| 8 | Capstone Project | End-to-end system | Comprehensive evals |

---

## 🛠️ Tech Stack Summary

### Core
| Tool | Purpose | Weeks |
|------|---------|-------|
| Python 3.11+ | Primary language | All |
| OpenAI API | LLM + embeddings | All |
| Anthropic API | LLM (Claude) | All |
| `tiktoken` | Token counting | 1+ |
| `numpy` | Vector math | 1+ |

### RAG
| Tool | Purpose | Weeks |
|------|---------|-------|
| LangChain | Orchestration framework | 2-8 |
| ChromaDB | Local vector database | 2-3 |
| Neo4j | Graph database for GraphRAG | 4 |
| RAGAS | RAG evaluation | 3, 7-8 |
| Cohere / BGE | Reranking | 3 |

### Agents
| Tool | Purpose | Weeks |
|------|---------|-------|
| LangGraph | Multi-agent orchestration | 5-8 |
| Pydantic AI | Type-safe agents | 5-8 |
| LangSmith | Tracing & observability | 7-8 |

### Infrastructure
| Tool | Purpose | Weeks |
|------|---------|-------|
| Docker | Containerized services | 2-8 |
| Git/GitHub | Version control + portfolio | All |
| `uv` / `poetry` | Dependency management | All |

---

## 📈 Progress Tracking

### Weekly Self-Assessment Questions

**After each week, answer:**
1. Can I explain the core concept to someone in 2 minutes?
2. Did I complete the coding project?
3. Did I push code to GitHub?
4. What was the hardest part?
5. What would I do differently?

### Skills Checklist

<details>
<summary>Click to expand full skills checklist</summary>

#### LLM Fundamentals
- [ ] Explain tokens, context window, parameters
- [ ] Describe pre-training vs post-training
- [ ] Explain attention mechanism (high-level)
- [ ] Make API calls with system/user prompts
- [ ] Generate and use embeddings
- [ ] Implement cosine similarity from scratch
- [ ] Use structured outputs (JSON mode)

#### RAG
- [ ] Explain the 5-stage RAG pipeline
- [ ] Choose appropriate chunking strategy
- [ ] Select embedding models
- [ ] Explain HNSW, IVF, PQ
- [ ] Build end-to-end RAG with LangChain + ChromaDB
- [ ] Implement HyDE
- [ ] Implement multi-query retrieval
- [ ] Apply cross-encoder reranking
- [ ] Implement hybrid search (dense + sparse)
- [ ] Add metadata filtering
- [ ] Run RAGAS evaluation
- [ ] Build GraphRAG with Neo4j
- [ ] Implement agentic RAG
- [ ] Choose RAG architecture using decision framework

#### Agents
- [ ] Explain agent vs pipeline vs API call
- [ ] Define tools with JSON schemas
- [ ] Implement ReAct pattern
- [ ] Build agent with Pydantic AI
- [ ] Add guardrails (input/output validation)
- [ ] Explain orchestrator-worker pattern
- [ ] Implement routing in multi-agent
- [ ] Build multi-agent pipeline with LangGraph
- [ ] Implement short-term and long-term memory
- [ ] Manage cost and latency in multi-agent

#### Evaluation
- [ ] Explain context engineering vs prompt engineering
- [ ] Optimize context window usage
- [ ] Implement conversation summary memory
- [ ] Create golden eval dataset
- [ ] Implement accuracy, F1, exact match
- [ ] Implement tool-call accuracy
- [ ] Design LLM-as-a-Judge rubric
- [ ] Run comprehensive eval suite
- [ ] Identify and document failure modes

#### Production
- [ ] Scope an MVP
- [ ] Design system architecture
- [ ] Justify engineering decisions
- [ ] Present AI system in interview format
- [ ] Discuss tradeoffs honestly
</details>

---

## 📚 Full Reading List

### Books
| Book | Author | Relevance |
|------|--------|-----------|
| AI Engineering | Chip Huyen | Full course companion |
| Build a Large Language Model (From Scratch) | Sebastian Raschka | Weeks 1-2 deep dive |
| Designing Machine Learning Systems | Chip Huyen | Production context |
| Hands-On Large Language Models | Jay Alammar | Visual intuition |

### Papers
| Paper | Year | Week |
|-------|------|------|
| Attention Is All You Need | 2017 | 1 (optional) |
| RAG (Lewis et al.) | 2020 | 2 |
| HyDE | 2022 | 3 |
| ReAct | 2022 | 5 |
| RAGAS | 2023 | 3, 7 |
| LLM-as-a-Judge | 2023 | 7 |
| GraphRAG (Microsoft) | 2024 | 4 |

### Free Resources
- [OpenAI Cookbook](https://cookbook.openai.com/)
- [LangChain documentation](https://python.langchain.com/docs/)
- [Anthropic prompt engineering guide](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview)
- [Karpathy's YouTube series](https://www.youtube.com/@AndrejKarpathy)
- [3Blue1Brown neural network series](https://www.3blue1brown.com/topics/neural-networks)
- [Pinecone Learn](https://www.pinecone.io/learn/)

---

## ⏰ Daily Time Commitment

| Activity | Hours/Day | Days/Week |
|----------|-----------|-----------|
| Theory (reading/videos) | 1.5 | 3 |
| Coding (exercises/projects) | 2.5 | 7 |
| Review & documentation | 0.5 | 7 |
| **Total** | **~4.5** | **7** |

> **Total course time**: ~250 hours over 8 weeks
> Adjust pace as needed — it's better to deeply understand Week 3 than to rush to Week 8.

---

## 🎯 Final Portfolio

By the end of this plan, your GitHub will contain:

1. **Semantic FAQ Matcher** — embeddings + cosine similarity
2. **PDF Chatbot** — end-to-end RAG pipeline
3. **Advanced RAG with Benchmarks** — HyDE, reranking, hybrid search + evals
4. **GraphRAG vs Vanilla RAG** — Neo4j + comparison study
5. **Research Assistant Agent** — ReAct + tool calling + Pydantic AI
6. **Multi-Agent Research Pipeline** — LangGraph + orchestration + routing
7. **Eval Suite** — quantitative + LLM-as-a-Judge
8. **Capstone Project** — production-grade AI system with full evals

Each repo should have:
- Clear README with architecture diagram
- Setup instructions
- Example usage
- Eval results
- Key engineering decisions documented
