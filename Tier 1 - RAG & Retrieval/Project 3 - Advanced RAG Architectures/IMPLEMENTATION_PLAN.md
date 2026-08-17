# Advanced RAG Architectures — GraphRAG, Agentic RAG, Multimodal RAG & Decision Framework

## Implementation Plan

> **Tier 1 — Project 3** | Estimated duration: 10 working days (~2 weeks)
> Target audience: Data engineer / analytics engineer transitioning to AI engineering
> Prerequisites: Project 1 (Advanced RAG with hybrid search + re-ranking) and Project 2 (Secure RAG with tenant isolation)

---

## Table of Contents

1. [Project Goals & Non-Goals](#1-project-goals--non-goals)
2. [Architecture Overview](#2-architecture-overview)
3. [Tech Stack with Rationale](#3-tech-stack-with-rationale)
4. [Database & Schema Design](#4-database--schema-design)
5. [Project Structure](#5-project-structure)
6. [Implementation Phases](#6-implementation-phases)
7. [Component Specifications](#7-component-specifications)
8. [RAG Architecture Decision Framework](#8-rag-architecture-decision-framework)
9. [Comparison & Evaluation](#9-comparison--evaluation)
10. [API Specification](#10-api-specification)
11. [Security & Safety Considerations](#11-security--safety-considerations)
12. [Testing Strategy](#12-testing-strategy)
13. [Deployment](#13-deployment)
14. [Roadmap & Milestones](#14-roadmap--milestones)
15. [Risk Register](#15-risk-register)
16. [Appendix](#16-appendix)

---

## 1. Project Goals & Non-Goals

### Goals

| # | Goal | Success Criterion |
|---|------|-------------------|
| G1 | Build a **GraphRAG pipeline** with Neo4j that extracts entities and relationships from text and retrieves via graph traversal | Entity/relationship extraction, graph construction in Neo4j, community detection, and graph-based retrieval all operational with Cypher queries |
| G2 | **Benchmark GraphRAG vs vanilla vector RAG** on the same corpus | Side-by-side comparison showing recall@k, MRR, and answer quality for multi-hop reasoning queries where graph retrieval outperforms vector retrieval |
| G3 | Build an **Agentic RAG** pipeline with an iterative retrieval loop where the LLM decides what to retrieve, when, and how many times | LLM-driven retrieve → assess → retrieve-again loop with Self-RAG-style reflection tokens, max 5 iterations, termination on sufficient context |
| G4 | Build a **Multimodal RAG** pipeline that embeds images alongside text using CLIP | CLIP image embeddings + text embeddings stored in pgvector, cross-modal retrieval (text query → image results, image query → text results), table/chart handling |
| G5 | Produce a **KAG/LightRAG overview** with a structured comparison table explaining when structured KB + LLM beats vector RAG | Comprehensive comparison table covering architecture, indexing, retrieval, strengths, weaknesses, and use-case fit for KAG and LightRAG |
| G6 | Deliver a **RAG Architecture Decision Framework** — a structured table/flowchart for choosing vanilla RAG vs GraphRAG vs Agentic RAG vs KAG vs Multimodal RAG | Decision matrix with criteria (data type, query complexity, relationship density, modality, latency budget, corpus size) and clear recommendations |
| G7 | Expose all architectures via a **FastAPI comparison API** | `POST /query` accepts `architecture` parameter (vanilla, graph, agentic, multimodal) and returns ranked results + answer + retrieval trace |
| G8 | Build a **Streamlit comparison dashboard** showing all architectures side-by-side | Dashboard with architecture selector, query input, retrieval trace visualization, benchmark charts, and decision framework explorer |

### Non-Goals

| # | Non-Goal | Rationale |
|---|----------|-----------|
| NG1 | Full KAG or LightRAG implementation | These are surveyed and compared, not built from scratch — the focus is on understanding when to use them |
| NG2 | Training or fine-tuning CLIP or entity extraction models | We use pre-trained CLIP (openai/clip-vit-base-patch32) and LLM-based entity extraction |
| NG3 | Production-scale graph serving (billions of nodes) | Focus is on architecture understanding and comparison, not Neo4j cluster tuning |
| NG4 | Real-time graph updates / streaming ingestion | Batch graph construction from documents is sufficient for comparison |
| NG5 | Video or audio modality RAG | Image + text multimodal is sufficient to demonstrate the cross-modal retrieval pattern |
| NG6 | Multi-tenant isolation for graph and multimodal stores | Single-user comparison tool; tenant isolation was covered in Project 2 |

---

## 2. Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                        ADVANCED RAG ARCHITECTURES COMPARISON SYSTEM                     │
├──────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌─────────────────────────────── SHARED INGESTION ──────────────────────────┐        │
│  │                                                                           │        │
│  │  ┌──────────┐   ┌──────────────┐   ┌──────────────────────────────┐       │        │
│  │  │ Document │──▶│  Text        │──▶│  Chunking (semantic_spacy)   │       │        │
│  │  │  Loader  │   │  Extraction  │   │  + Image Extraction (PDF)    │       │        │
│  │  │          │   │  (.txt,.md,  │   └──────────┬───────────────────┘       │        │
│  │  │ .txt     │   │   .pdf)      │              │                           │        │
│  │  │ .md      │   └──────────────┘              │                           │        │
│  │  │ .pdf     │                                 │                           │        │
│  │  │ (images) │                                 ▼                           │        │
│  │  └──────────┘                    ┌──────────────────────────────┐          │        │
│  │                                   │  Text Embeddings (BGE-large) │          │        │
│  │                                   │  + CLIP Image Embeddings     │          │        │
│  │                                   └──────────┬───────────────────┘          │        │
│  │                                              │                              │        │
│  └──────────────────────────────────────────────┼──────────────────────────────┘        │
│                                                 │                                      │
│  ┌─────────────── ARCHITECTURE A: VANILLA VECTOR RAG ──────────────────────────┐       │
│  │                                                                              │       │
│  │  Query ──▶ Embed ──▶ pgvector cosine search ──▶ top-k chunks ──▶ LLM answer  │       │
│  │                                                                              │       │
│  └──────────────────────────────────────────────────────────────────────────────┘       │
│                                                                                      │
│  ┌─────────────────────── ARCHITECTURE B: GraphRAG (Neo4j) ─────────────────────┐       │
│  │                                                                              │       │
│  │  INGEST:                                                                     │       │
│  │  Chunks ──▶ LLM Entity Extraction ──▶ Entities + Relationships              │       │
│  │  ──▶ Neo4j Graph Construction ──▶ Community Detection (Leiden)              │       │
│  │  ──▶ Community Summarization (LLM)                                          │       │
│  │                                                                              │       │
│  │  QUERY:                                                                      │       │
│  │  Query ──▶ Entity Linking ──▶ Graph Traversal (Cypher)                      │       │
│  │  ──▶ Community Summaries ──▶ Subgraph Context ──▶ LLM answer                │       │
│  │                                                                              │       │
│  └──────────────────────────────────────────────────────────────────────────────┘       │
│                                                                                      │
│  ┌─────────────────────── ARCHITECTURE C: AGENTIC RAG ───────────────────────────┐       │
│  │                                                                              │       │
│  │  Query ──▶ LLM Planner                                                       │       │
│  │         ┌───────────────────────────────────┐                                │       │
│  │         │  Iteration Loop (max 5):          │                                │       │
│  │         │  1. Decide: retrieve more?        │                                │       │
│  │         │  2. Generate sub-query            │                                │       │
│  │         │  3. Retrieve (vector or graph)    │                                │       │
│  │         │  4. Assess: is context sufficient?│                                │       │
│  │         │  5. If yes → generate answer      │                                │       │
│  │         │  6. If no → loop back to step 1   │                                │       │
│  │         │  Self-RAG reflection tokens:      │                                │       │
│  │         │  [RETRIEVE] / [NO RETRIEVE]       │                                │       │
│  │         │  [RELEVANT] / [IRRELEVANT]        │                                │       │
│  │         │  [GENERATE] / [NO GENERATE]       │                                │       │
│  │         └───────────────────────────────────┘                                │       │
│  │         ──▶ Final Answer + Retrieval Trace                                  │       │
│  │                                                                              │       │
│  └──────────────────────────────────────────────────────────────────────────────┘       │
│                                                                                      │
│  ┌─────────────────────── ARCHITECTURE D: MULTIMODAL RAG ────────────────────────┐       │
│  │                                                                              │       │
│  │  INGEST:                                                                     │       │
│  │  Document ──▶ Text Chunks (BGE embeddings, 1024-dim)                        │       │
│  │           ──▶ Image Chunks (CLIP embeddings, 512-dim)                       │       │
│  │           ──▶ Table/Chart Extraction (text representation)                  │       │
│  │           ──▶ All stored in pgvector (separate vector dimensions)           │       │
│  │                                                                              │       │
│  │  QUERY:                                                                      │       │
│  │  Text query ──▶ CLIP text encoder ──▶ cross-modal search                    │       │
│  │  Image query ──▶ CLIP image encoder ──▶ cross-modal search                  │       │
│  │  ──▶ Fuse text + image results ──▶ LLM answer (with image refs)             │       │
│  │                                                                              │       │
│  └──────────────────────────────────────────────────────────────────────────────┘       │
│                                                                                      │
│  ┌─────────────────────── DECISION FRAMEWORK LAYER ─────────────────────────────┐       │
│  │                                                                              │       │
│  │  ┌────────────────┐   ┌──────────────────────┐   ┌───────────────────────┐  │       │
│  │  │  Query         │──▶│  Decision Engine      │──▶│  Architecture         │  │       │
│  │  │  Analyzer      │   │  (criteria matrix)    │   │  Recommendation       │  │       │
│  │  │  (data type,   │   │  • data type          │   │  + routing to         │  │       │
│  │  │   complexity,  │   │  • query complexity   │   │    A / B / C / D      │  │       │
│  │  │   density,     │   │  • relationship dens. │   │                       │  │       │
│  │  │   modality)    │   │  • modality           │   │                       │  │       │
│  │  └────────────────┘   └──────────────────────┘   └───────────────────────┘  │       │
│  │                                                                              │       │
│  └──────────────────────────────────────────────────────────────────────────────┘       │
│                                                                                      │
│  ┌─────────────────────── COMPARISON DASHBOARD (Streamlit) ──────────────────────┐       │
│  │                                                                              │       │
│  │  • Architecture selector (A / B / C / D)                                    │       │
│  │  • Query input + side-by-side results                                       │       │
│  │  • Retrieval trace visualization (especially Agentic loop)                  │       │
│  │  • Benchmark charts (GraphRAG vs Vector RAG, multimodal recall)             │       │
│  │  • Decision Framework explorer (interactive criteria sliders)               │       │
│  │  • KAG / LightRAG comparison table                                          │       │
│  │                                                                              │       │
│  └──────────────────────────────────────────────────────────────────────────────┘       │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘

External Services:
  ┌────────────────┐     ┌──────────────────┐     ┌─────────────────────┐     ┌──────────────┐
  │  PostgreSQL    │     │  Neo4j           │     │  LLM API            │     │  HuggingFace │
  │  + pgvector    │     │  (graph store)   │     │  (OpenAI / Ollama)  │     │  (CLIP, BGE) │
  │  (text + image │     │                  │     │                     │     │              │
  │   vectors)     │     │                  │     │                     │     │              │
  └────────────────┘     └──────────────────┘     └─────────────────────┘     └──────────────┘
```

### Data Flow Summary

```
INGEST (shared):  Documents → Extract text + images → Chunk → Embed (BGE text + CLIP image) → Store (pgvector)
                  → LLM entity extraction → Build Neo4j graph → Community detection → Community summaries

QUERY (vanilla):  Query → Embed → pgvector cosine → top-k → LLM answer
QUERY (graph):    Query → Entity linking → Cypher traversal → Community summaries + subgraph → LLM answer
QUERY (agentic):  Query → LLM planner → [retrieve → assess] loop (max 5) → LLM answer + trace
QUERY (multimodal): Query → CLIP encode → cross-modal pgvector search → fuse text+image → LLM answer

BENCHMARK:        Golden Q&A → Run all 4 architectures → Compute metrics → Streamlit dashboard
DECISION:         Query characteristics → Criteria matrix → Architecture recommendation
```

### Architecture Selection Flow

```
                         ┌──────────────────┐
                         │  User Query +     │
                         │  Corpus Metadata  │
                         └────────┬─────────┘
                                  │
                         ┌────────▼─────────┐
                         │  Q1: Is corpus   │
                         │  multimodal?      │
                         └────────┬─────────┘
                            YES   │   NO
                         ┌────────┴────────────────┐
                         ▼                         ▼
                  ┌──────────────┐        ┌──────────────────┐
                  │ MULTIMODAL   │        │ Q2: High entity  │
                  │ RAG          │        │ relationship     │
                  │ (Arch D)     │        │ density?         │
                  └──────────────┘        └────────┬─────────┘
                                              YES  │   NO
                                          ┌────────┴────────────────┐
                                          ▼                         ▼
                                   ┌──────────────┐        ┌──────────────────┐
                                   │ Q3: Multi-hop│        │ Q4: Complex      │
                                   │ reasoning?   │        │ multi-step query?│
                                   └──────┬───────┘        └────────┬─────────┘
                                     YES  │  NO              YES    │   NO
                                   ┌──────┴────┐         ┌─────────┴─────────┐
                                   ▼           ▼         ▼                   ▼
                            ┌──────────┐ ┌─────────┐ ┌──────────┐    ┌──────────────┐
                            │ GRAPHRAG │ │ Q5:     │ │ AGENTIC  │    │ VANILLA RAG  │
                            │ (Arch B) │ │ Struct. │ │ RAG      │    │ (Arch A)     │
                            └──────────┘ │ KB +    │ │ (Arch C) │    └──────────────┘
                                         │ rules?  │ └──────────┘
                                         └───┬────┘
                                        YES  │  NO
                                        ┌────┴────┐
                                        ▼         ▼
                                  ┌─────────┐ ┌──────────────┐
                                  │ KAG /   │ │ GRAPHRAG or  │
                                  │ LIGHTRAG│ │ VANILLA RAG  │
                                  └─────────┘ └──────────────┘
```

---

## 3. Tech Stack with Rationale

| Layer | Technology | Version | Rationale |
|-------|-----------|---------|-----------|
| **Language** | Python | 3.11+ | Type hints, async support; standard for ML/AI |
| **Web Framework** | FastAPI | 0.110+ | Async, auto-docs (OpenAPI), type-safe with Pydantic v2 |
| **Relational DB** | PostgreSQL | 16 | Metadata store, vector store (pgvector), query logging |
| **Vector Extension** | pgvector | 0.7+ | Text embeddings (1024-dim BGE) + image embeddings (512-dim CLIP) in one store |
| **Graph Database** | Neo4j | 5.11+ (Community) | Purpose-built graph DB; Cypher query language; GDS library for community detection |
| **Graph Driver** | neo4j (Python driver) | 5.11+ | Official async-capable driver; Cypher query execution |
| **Graph Analytics** | Neo4j GDS (Graph Data Science) | 2.4+ | Leiden algorithm for community detection; graph embeddings (node2vec) |
| **Text Embedding** | BAAI/bge-large-en-v1.5 | — | Same as P1; 1024-dim; top-tier MTEB scores |
| **Image Embedding** | openai/clip-vit-base-patch32 | — | Open-source CLIP; 512-dim joint text-image embedding space; cross-modal retrieval |
| **CLIP Library** | transformers (HuggingFace) | 4.40+ | CLIPModel + CLIPProcessor for image+text encoding |
| **Image Processing** | Pillow (PIL) | 10.0+ | Image loading, resizing, format conversion for CLIP |
| **PDF Image Extraction** | PyMuPDF (fitz) | 1.24+ | Extract embedded images from PDFs; render PDF pages as images |
| **Entity Extraction** | LLM-based (gpt-4o-mini / llama3.1:8b) | — | LLM extracts entities + relationships from text; more flexible than NER models |
| **LLM Orchestration** | LangChain | 0.2+ | Agent loop abstractions, tool calling, LLM client wrappers |
| **LLM** | OpenAI `gpt-4o-mini` or local Ollama (`llama3.1:8b`) | — | `gpt-4o-mini` for quality/latency; Ollama for zero-cost local development |
| **Comparison Dashboard** | Streamlit | 1.35+ | Interactive architecture comparison, benchmark charts, decision framework explorer |
| **Containerization** | Docker + Docker Compose | — | Neo4j + Postgres + API + Dashboard in one `docker compose up` |
| **Testing** | pytest | 8.0+ | Standard Python testing; `pytest-asyncio` for async endpoints |
| **Data Validation** | Pydantic | 2.6+ | Schema validation for API, graph entities, multimodal results |
| **HTTP Client** | httpx | 0.27+ | Async HTTP for LLM API calls |
| **Graph Visualization** | streamlit-agraph | 0.0.45+ | Interactive graph visualization in Streamlit for Neo4j subgraph display |
| **Charting** | plotly | 5.20+ | Interactive benchmark charts in Streamlit dashboard |

### Why Neo4j over a Postgres-based graph?

For a GraphRAG project, **Neo4j** is the recommended choice:
- **Purpose-built**: Native graph storage, traversal, and pathfinding — Cypher is designed for graph patterns, not SQL recursion.
- **GDS library**: Leiden community detection, node2vec embeddings, PageRank — all built-in, no external pipeline needed.
- **Graph visualization**: Neo4j Browser + streamlit-agraph provide immediate visual feedback for graph construction.
- **Trade-off**: At >100M nodes, a distributed graph DB (Neo4j Cluster, TigerGraph, Amazon Neptune) becomes necessary. For this project's corpus (1K–10K entities), Neo4j Community Edition is sufficient.

> **Alternative**: If you prefer to keep everything in Postgres, use Apache AGE (Postgres extension for Cypher) or recursive CTEs for graph traversal. The `GraphStore` interface (Section 7.3) abstracts this — only the implementation class changes.

### Why CLIP over a dedicated image embedding model?

For multimodal RAG, **CLIP** is the recommended choice:
- **Joint embedding space**: Text and images live in the same 512-dim vector space — cross-modal retrieval (text→image, image→text) works out of the box.
- **Open-source**: `openai/clip-vit-base-patch32` runs locally, no API calls needed.
- **Sufficient quality**: For document images (charts, diagrams, tables), CLIP's zero-shot performance is adequate.
- **Trade-off**: For domain-specific images (medical, satellite), a fine-tuned model or a vision-language model like LLaVA may be needed. This is noted as a future enhancement.

---

## 4. Database & Schema Design

This project uses **two databases**: PostgreSQL (metadata, vector store, query logging) and Neo4j (knowledge graph). The schemas are designed to be complementary — Postgres stores document/chunk metadata and embeddings, while Neo4j stores the extracted entity-relationship graph.

### 4.1 PostgreSQL Schema

```sql
-- Enable extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "vector";       -- pgvector
CREATE EXTENSION IF NOT EXISTS "pg_trgm";      -- trigram fuzzy search

-- ============================================================
-- Documents: source documents (shared across all architectures)
-- ============================================================
CREATE TABLE documents (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    source_path     TEXT NOT NULL,
    title           TEXT NOT NULL,
    doc_type        TEXT NOT NULL CHECK (doc_type IN ('txt', 'md', 'pdf', 'html', 'image')),
    content_hash    TEXT NOT NULL UNIQUE,
    char_count      INTEGER NOT NULL,
    has_images      BOOLEAN NOT NULL DEFAULT FALSE,
    image_count     INTEGER NOT NULL DEFAULT 0,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_documents_content_hash ON documents(content_hash);
CREATE INDEX idx_documents_metadata_gin ON documents USING GIN(metadata);

-- ============================================================
-- Chunks: text chunks for vector RAG (vanilla + agentic + multimodal text)
-- ============================================================
CREATE TABLE chunks (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    document_id     UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    chunk_index     INTEGER NOT NULL,
    chunking_strategy TEXT NOT NULL DEFAULT 'semantic_spacy',
    content         TEXT NOT NULL,
    token_count     INTEGER NOT NULL,
    tsv             TSVECTOR GENERATED ALWAYS AS (
        to_tsvector('english', content)
    ) STORED,
    embedding       vector(1024),           -- BGE-large-en-v1.5 text embedding
    window_context  TEXT,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(document_id, chunking_strategy, chunk_index)
);

CREATE INDEX idx_chunks_tsv ON chunks USING GIN(tsv);
CREATE INDEX idx_chunks_strategy ON chunks(chunking_strategy);
CREATE INDEX idx_chunks_embedding
    ON chunks USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64)
    WHERE embedding IS NOT NULL;

-- ============================================================
-- Images: extracted images for multimodal RAG
-- ============================================================
CREATE TABLE images (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    document_id     UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    image_index     INTEGER NOT NULL,           -- position within document
    image_path      TEXT NOT NULL,              -- path to stored image file
    image_format    TEXT NOT NULL CHECK (image_format IN ('png', 'jpeg', 'webp', 'tiff')),
    width           INTEGER NOT NULL,
    height          INTEGER NOT NULL,
    -- CLIP embedding (512-dim for clip-vit-base-patch32)
    clip_embedding  vector(512),
    -- text extracted from image (OCR or caption)
    caption         TEXT,                       -- LLM-generated or OCR text
    caption_embedding vector(1024),             -- BGE embedding of caption (for text-based image search)
    -- image type classification
    image_type      TEXT NOT NULL DEFAULT 'image' CHECK (
        image_type IN ('image', 'chart', 'table', 'diagram', 'figure')
    ),
    -- surrounding text context (text near the image in the document)
    context_text    TEXT,
    context_chunk_id UUID REFERENCES chunks(id), -- nearest text chunk
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(document_id, image_index)
);

CREATE INDEX idx_images_doc ON images(document_id);
CREATE INDEX idx_images_clip_embedding
    ON images USING hnsw (clip_embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64)
    WHERE clip_embedding IS NOT NULL;

CREATE INDEX idx_images_caption_embedding
    ON images USING hnsw (caption_embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64)
    WHERE caption_embedding IS NOT NULL;

-- ============================================================
-- Graph Entities: metadata for entities extracted and stored in Neo4j
-- (Neo4j stores the actual graph; this table tracks extraction state)
-- ============================================================
CREATE TABLE graph_entities (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    document_id     UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    chunk_id        UUID REFERENCES chunks(id),
    entity_name     TEXT NOT NULL,
    entity_type     TEXT NOT NULL,              -- PERSON, ORG, TECH, CONCEPT, LOCATION, etc.
    entity_description TEXT,
    -- Neo4j node ID (for cross-referencing)
    neo4j_node_id   TEXT,                       -- internal Neo4j ID or custom elementId
    -- extraction metadata
    extraction_confidence FLOAT,
    mention_count   INTEGER NOT NULL DEFAULT 1,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(document_id, entity_name, entity_type)
);

CREATE INDEX idx_graph_entities_doc ON graph_entities(document_id);
CREATE INDEX idx_graph_entities_name ON graph_entities(entity_name);
CREATE INDEX idx_graph_entities_type ON graph_entities(entity_type);

-- ============================================================
-- Graph Relationships: metadata for relationships in Neo4j
-- ============================================================
CREATE TABLE graph_relationships (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    source_entity_id UUID NOT NULL REFERENCES graph_entities(id) ON DELETE CASCADE,
    target_entity_id UUID NOT NULL REFERENCES graph_entities(id) ON DELETE CASCADE,
    relationship_type TEXT NOT NULL,            -- WORKS_FOR, DEVELOPED_BY, RELATED_TO, etc.
    -- chunk where this relationship was extracted from
    source_chunk_id  UUID REFERENCES chunks(id),
    neo4j_rel_id     TEXT,
    extraction_confidence FLOAT,
    metadata         JSONB NOT NULL DEFAULT '{}',
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(source_entity_id, target_entity_id, relationship_type)
);

CREATE INDEX idx_graph_rels_source ON graph_relationships(source_entity_id);
CREATE INDEX idx_graph_rels_target ON graph_relationships(target_entity_id);
CREATE INDEX idx_graph_rels_type ON graph_relationships(relationship_type);

-- ============================================================
-- Communities: community detection results from Neo4j GDS
-- ============================================================
CREATE TABLE graph_communities (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    community_id    INTEGER NOT NULL,           -- GDS community ID
    community_level INTEGER NOT NULL DEFAULT 0, -- hierarchy level (Leiden)
    -- LLM-generated summary of the community
    summary         TEXT NOT NULL,
    summary_embedding vector(1024),             -- BGE embedding of summary
    -- entities and relationships in this community
    entity_count    INTEGER NOT NULL,
    rel_count       INTEGER NOT NULL,
    -- key entities (top by degree centrality)
    key_entities    JSONB NOT NULL DEFAULT '[]',
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(community_id, community_level)
);

CREATE INDEX idx_communities_level ON graph_communities(community_level);
CREATE INDEX idx_communities_summary_embedding
    ON graph_communities USING hnsw (summary_embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64)
    WHERE summary_embedding IS NOT NULL;

-- ============================================================
-- Queries: logged user queries across all architectures
-- ============================================================
CREATE TABLE queries (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    query_text      TEXT NOT NULL,
    architecture    TEXT NOT NULL CHECK (
        architecture IN ('vanilla', 'graph', 'agentic', 'multimodal')
    ),
    strategy_config JSONB NOT NULL,             -- architecture-specific config
    retrieved_ids   UUID[] NOT NULL,            -- chunk IDs (text)
    retrieved_image_ids UUID[] NOT NULL DEFAULT '{}', -- image IDs (multimodal)
    retrieved_entity_ids UUID[] NOT NULL DEFAULT '{}', -- entity IDs (graph)
    final_answer    TEXT,
    retrieval_trace JSONB,                      -- step-by-step trace (especially agentic)
    latency_ms      INTEGER,
    iterations      INTEGER NOT NULL DEFAULT 1, -- agentic loop iterations
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_queries_arch ON queries(architecture);
CREATE INDEX idx_queries_created ON queries(created_at);

-- ============================================================
-- Golden Questions: evaluation ground truth
-- ============================================================
CREATE TABLE golden_questions (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    question        TEXT NOT NULL,
    expected_answer TEXT NOT NULL,
    -- which architectures should handle this query well
    expected_architectures TEXT[] NOT NULL,     -- e.g., {graph, agentic}
    -- query characteristics for decision framework validation
    query_type      TEXT NOT NULL CHECK (
        query_type IN ('factual', 'multi_hop', 'temporal', 'comparative', 'multimodal', 'agentic')
    ),
    -- relevant chunks for evaluation
    relevant_chunk_ids UUID[] NOT NULL DEFAULT '{}',
    relevant_entity_ids UUID[] NOT NULL DEFAULT '{}',
    relevant_image_ids  UUID[] NOT NULL DEFAULT '{}',
    -- relevance grades for nDCG: {chunk_id: grade (0-3)}
    relevance_grades  JSONB NOT NULL DEFAULT '{}',
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_golden_type ON golden_questions(query_type);

-- ============================================================
-- Eval Results: benchmark results across architectures
-- ============================================================
CREATE TABLE eval_results (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    question_id     UUID NOT NULL REFERENCES golden_questions(id) ON DELETE CASCADE,
    architecture    TEXT NOT NULL CHECK (
        architecture IN ('vanilla', 'graph', 'agentic', 'multimodal')
    ),
    retrieved_ids   UUID[] NOT NULL,
    final_answer    TEXT,
    recall_at_k     FLOAT,
    precision_at_k  FLOAT,
    mrr             FLOAT,
    ndcg_at_k       FLOAT,
    answer_score    FLOAT,                      -- LLM-judge score (0-1)
    latency_ms      INTEGER,
    iterations      INTEGER NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(question_id, architecture)
);

CREATE INDEX idx_eval_arch ON eval_results(architecture);
CREATE INDEX idx_eval_question ON eval_results(question_id);
```

### 4.2 Neo4j Graph Schema (Cypher)

The Neo4j graph stores entities as nodes and extracted relationships as edges. The schema is defined by constraints and indexes, not by rigid table definitions.

```cypher
// ============================================================
// CONSTRAINTS — enforce uniqueness and improve query performance
// ============================================================

// Entity nodes: unique by name + type
CREATE CONSTRAINT entity_unique IF NOT EXISTS
FOR (e:Entity) REQUIRE (e.name, e.entity_type) IS UNIQUE;

// Document nodes: unique by document ID (UUID from Postgres)
CREATE CONSTRAINT document_unique IF NOT EXISTS
FOR (d:Document) REQUIRE d.doc_id IS UNIQUE;

// Chunk nodes: unique by chunk ID (UUID from Postgres)
CREATE CONSTRAINT chunk_unique IF NOT EXISTS
FOR (c:Chunk) REQUIRE c.chunk_id IS UNIQUE;

// Community nodes: unique by community ID + level
CREATE CONSTRAINT community_unique IF NOT EXISTS
FOR (com:Community) REQUIRE (com.community_id, com.level) IS UNIQUE;

// ============================================================
// INDEXES — speed up common query patterns
// ============================================================
CREATE INDEX entity_name_index IF NOT EXISTS FOR (e:Entity) ON (e.name);
CREATE INDEX entity_type_index IF NOT EXISTS FOR (e:Entity) ON (e.entity_type);
CREATE INDEX entity_doc_index IF NOT EXISTS FOR (e:Entity) ON (e.doc_id);
CREATE INDEX community_id_index IF NOT EXISTS FOR (com:Community) ON (com.community_id);

// Full-text index for entity search (fuzzy matching for entity linking)
CREATE FULLTEXT INDEX entity_fulltext IF NOT EXISTS
FOR (e:Entity) ON EACH [e.name, e.description];

// ============================================================
// NODE LABELS & PROPERTIES
// ============================================================
// (:Entity {
//     name: string,           -- "OpenAI", "GPT-4", "Sam Altman"
//     entity_type: string,    -- "ORG", "TECH", "PERSON", "CONCEPT", "LOCATION"
//     description: string,    -- LLM-generated description
//     doc_id: string,         -- Postgres document UUID (first mention)
//     chunk_id: string,       -- Postgres chunk UUID (first mention)
//     mention_count: int,     -- how many chunks mention this entity
//     extraction_confidence: float,
//     created_at: datetime
// })
//
// (:Document {
//     doc_id: string,         -- Postgres UUID
//     title: string,
//     doc_type: string,
//     source_path: string,
//     created_at: datetime
// })
//
// (:Chunk {
//     chunk_id: string,       -- Postgres UUID
//     doc_id: string,
//     content: string,        -- full chunk text
//     chunk_index: int,
//     created_at: datetime
// })
//
// (:Community {
//     community_id: int,      -- GDS Leiden community ID
//     level: int,             -- hierarchy level
//     summary: string,        -- LLM-generated community summary
//     entity_count: int,
//     rel_count: int,
//     key_entities: [string], -- top entity names
//     created_at: datetime
// })

// ============================================================
// RELATIONSHIP TYPES & PROPERTIES
// ============================================================
// (:Entity)-[:MENTIONS {chunk_id: string, confidence: float}]->(:Chunk)
//     -- Entity is mentioned in a chunk
//
// (:Entity)-[:RELATED_TO {type: string, description: string, chunk_id: string, confidence: float}]->(:Entity)
//     -- Generic relationship between entities
//     -- type: "DEVELOPED_BY", "WORKS_FOR", "COMPETES_WITH", "PART_OF", "USES", etc.
//
// (:Entity)-[:BELONGS_TO {level: int}]->(:Community)
//     -- Entity belongs to a community (from Leiden detection)
//
// (:Chunk)-[:PART_OF]->(:Document)
//     -- Chunk belongs to a document
//
// (:Entity)-[:EXTRACTED_FROM {chunk_id: string}]->(:Document)
//     -- Entity was first extracted from this document
//
// (:Community)-[:PARENT_COMMUNITY]->(:Community)
//     -- Hierarchical community structure (Leiden levels)

// ============================================================
// EXAMPLE GRAPH CONSTRUCTION QUERIES
// ============================================================

// Create a document node
CREATE (d:Document {
    doc_id: '550e8400-e29b-41d4-a716-446655440000',
    title: 'Attention Is All You Need',
    doc_type: 'pdf',
    source_path: 'data/raw/papers/attention_is_all_you_need.pdf',
    created_at: datetime()
})

// Create chunk nodes
CREATE (c:Chunk {
    chunk_id: '660e8400-e29b-41d4-a716-446655440001',
    doc_id: '550e8400-e29b-41d4-a716-446655440000',
    content: 'We propose a new simple network architecture, the Transformer...',
    chunk_index: 0,
    created_at: datetime()
})

// Create entity nodes (from LLM extraction)
CREATE (e1:Entity {
    name: 'Transformer',
    entity_type: 'TECH',
    description: 'A neural network architecture based solely on attention mechanisms',
    doc_id: '550e8400-e29b-41d4-a716-446655440000',
    chunk_id: '660e8400-e29b-41d4-a716-446655440001',
    mention_count: 15,
    extraction_confidence: 0.95,
    created_at: datetime()
})

CREATE (e2:Entity {
    name: 'Google',
    entity_type: 'ORG',
    description: 'Technology company, employer of the paper authors',
    doc_id: '550e8400-e29b-41d4-a716-446655440000',
    chunk_id: '660e8400-e29b-41d4-a716-446655440001',
    mention_count: 3,
    extraction_confidence: 0.90,
    created_at: datetime()
})

// Create relationships
MATCH (e1:Entity {name: 'Transformer', entity_type: 'TECH'}),
      (e2:Entity {name: 'Google', entity_type: 'ORG'})
CREATE (e2)-[:RELATED_TO {
    type: 'DEVELOPED_BY',
    description: 'Google researchers developed the Transformer architecture',
    chunk_id: '660e8400-e29b-41d4-a716-446655440001',
    confidence: 0.92
}]->(e1)

// Link entities to chunks (MENTIONS)
MATCH (e:Entity {name: 'Transformer', entity_type: 'TECH'}),
      (c:Chunk {chunk_id: '660e8400-e29b-41d4-a716-446655440001'})
CREATE (e)-[:MENTIONS {confidence: 0.95}]->(c)

// Link chunk to document
MATCH (c:Chunk {chunk_id: '660e8400-e29b-41d4-a716-446655440001'}),
      (d:Document {doc_id: '550e8400-e29b-41d4-a716-446655440000'})
CREATE (c)-[:PART_OF]->(d)
```

### 4.3 Neo4j GDS Graph Projection (for community detection)

```cypher
// ============================================================
// GRAPH PROJECTION for GDS (Graph Data Science)
// ============================================================

// Project a graph for community detection
// Nodes: Entity, Relationships: RELATED_TO (undirected for community detection)
CALL gds.graph.project(
    'entityGraph',
    ['Entity'],
    {
        RELATED_TO: {
            orientation: 'UNDIRECTED',
            properties: ['confidence']
        }
    }
);

// ============================================================
// LEIDEN COMMUNITY DETECTION
// ============================================================

// Run Leiden algorithm and write community IDs back to nodes
CALL gds.leiden.write(
    'entityGraph',
    {
        writeProperty: 'communityId',
        maxIterations: 10,
        relationshipWeightProperty: 'confidence',
        includeIntermediateCommunities: true
    }
);

// ============================================================
// COMMUNITY SUMMARIZATION
// ============================================================

// Get community members and create Community nodes
// (This is done in Python after Leiden — see Section 7.3)
// For each community:
// 1. Query all entities in the community
// 2. Query all relationships within the community
// 3. Send to LLM for summarization
// 4. Create Community node with summary

// Example: get community 0 members
MATCH (e:Entity)
WHERE e.communityId = 0
RETURN e.name, e.entity_type, e.description
ORDER BY e.mention_count DESC;

// Example: get relationships within community 0
MATCH (e1:Entity)-[r:RELATED_TO]->(e2:Entity)
WHERE e1.communityId = 0 AND e2.communityId = 0
RETURN e1.name, r.type, e2.name, r.description;

// Create community node
CREATE (com:Community {
    community_id: 0,
    level: 0,
    summary: 'This community covers the Transformer architecture and its development at Google...',
    entity_count: 12,
    rel_count: 28,
    key_entities: ['Transformer', 'Google', 'Attention Mechanism'],
    created_at: datetime()
})

// Link entities to community
MATCH (e:Entity), (com:Community {community_id: 0})
WHERE e.communityId = 0
CREATE (e)-[:BELONGS_TO {level: 0}]->(com);

// ============================================================
// GRAPH RETRIEVAL QUERIES (used at query time)
// ============================================================

// 1. Entity linking: find entities mentioned in the query
// (Done via fulltext search in Python, then Cypher for traversal)

// 2. Graph traversal: find related entities within N hops
MATCH (e:Entity {name: 'Transformer', entity_type: 'TECH'})
CALL {
    WITH e
    MATCH path = (e)-[:RELATED_TO*1..3]-(related:Entity)
    RETURN related, relationships(path) as rels, length(path) as hopCount
    LIMIT 20
}
RETURN related.name, related.entity_type, related.description,
       [r in rels | r.type] as relationshipTypes,
       hopCount
ORDER BY hopCount ASC;

// 3. Community-based retrieval: get community summaries for linked entities
MATCH (e:Entity)-[:BELONGS_TO]->(com:Community)
WHERE e.name IN ['Transformer', 'Google']
RETURN DISTINCT com.community_id, com.summary, com.key_entities
ORDER BY com.level ASC;

// 4. Subgraph extraction: get all entities, relationships, and chunks
// within 2 hops of the linked entities
MATCH (e:Entity {name: 'Transformer', entity_type: 'TECH'})
CALL {
    WITH e
    MATCH (e)-[:RELATED_TO*1..2]-(related:Entity)-[:MENTIONS]->(c:Chunk)
    RETURN DISTINCT c.chunk_id, c.content, related.name as entity
    LIMIT 10
}
RETURN c.chunk_id, c.content, collect(entity) as related_entities;

// 5. Clean up projection (after community detection is done)
CALL gds.graph.drop('entityGraph');
```

---

## 5. Project Structure

```
advanced-rag-architectures/
├── docker-compose.yml
├── Dockerfile
├── pyproject.toml
├── .env.example
├── README.md
├── IMPLEMENTATION_PLAN.md
├── agent.md
│
├── sql/
│   ├── schema.sql                    # PostgreSQL schema (Section 4.1)
│   └── neo4j/
│       ├── constraints.cypher        # Neo4j constraints + indexes
│       ├── community_detection.cypher # GDS Leiden + community creation
│       └── example_queries.cypher    # Example graph retrieval queries
│
├── src/
│   ├── __init__.py
│   ├── config.py                     # Pydantic Settings (all env vars)
│   │
│   ├── ingestion/
│   │   ├── __init__.py
│   │   ├── loader.py                 # Document loader (.txt, .md, .pdf, images)
│   │   ├── chunker.py                # Semantic chunker (spaCy, from P1)
│   │   ├── text_embedder.py          # BGE text embeddings (from P1)
│   │   ├── image_extractor.py        # Extract images from PDFs (PyMuPDF)
│   │   ├── clip_embedder.py          # CLIP image + text embeddings
│   │   └── ingest_pipeline.py        # Orchestrates full ingestion
│   │
│   ├── graph/
│   │   ├── __init__.py
│   │   ├── neo4j_client.py           # Neo4j driver wrapper (async-capable)
│   │   ├── entity_extractor.py       # LLM-based entity + relationship extraction
│   │   ├── graph_builder.py          # Build Neo4j graph from extracted entities
│   │   ├── community_detector.py     # Leiden community detection + summarization
│   │   └── graph_retriever.py        # Graph-based retrieval (entity linking + traversal)
│   │
│   ├── retrieval/
│   │   ├── __init__.py
│   │   ├── vector_retriever.py       # Vanilla vector RAG (pgvector cosine)
│   │   ├── hybrid_retriever.py       # Hybrid BM25 + vector (from P1, for comparison)
│   │   ├── multimodal_retriever.py   # CLIP cross-modal retrieval (text + image)
│   │   └── retrieval_result.py       # Shared dataclass for retrieval results
│   │
│   ├── agentic/
│   │   ├── __init__.py
│   │   ├── planner.py                # LLM planner: decides what to retrieve
│   │   ├── retrieval_loop.py         # Iterative retrieve → assess → retrieve loop
│   │   ├── self_rag.py               # Self-RAG reflection tokens
│   │   └── trace.py                  # Retrieval trace dataclass + logging
│   │
│   ├── generation/
│   │   ├── __init__.py
│   │   ├── llm_client.py             # LLM client (OpenAI / Ollama, from P1)
│   │   └── prompt_templates.py       # Prompt templates for all architectures
│   │
│   ├── framework/
│   │   ├── __init__.py
│   │   ├── decision_engine.py        # RAG architecture decision framework
│   │   ├── query_analyzer.py         # Analyze query characteristics
│   │   └── comparison_tables.py      # KAG/LightRAG comparison data
│   │
│   ├── evaluation/
│   │   ├── __init__.py
│   │   ├── metrics.py                # Retrieval metrics (from P1)
│   │   ├── benchmark_runner.py       # Run all 4 architectures on golden Q&A
│   │   └── golden_dataset.py         # Golden question management
│   │
│   ├── api/
│   │   ├── __init__.py
│   │   ├── main.py                   # FastAPI app
│   │   ├── routes.py                 # API routes
│   │   └── schemas.py                # Pydantic request/response schemas
│   │
│   └── dashboard/
│       ├── __init__.py
│       ├── app.py                    # Streamlit main app
│       ├── pages/
│       │   ├── 1_Comparison.py       # Side-by-side architecture comparison
│       │   ├── 2_GraphRAG_Explorer.py # Neo4j graph visualization
│       │   ├── 3_Agentic_Trace.py    # Agentic retrieval loop trace
│       │   ├── 4_Multimodal_Gallery.py # Image + text retrieval gallery
│       │   ├── 5_Decision_Framework.py # Interactive decision framework
│       │   └── 6_KAG_LightRAG.py     # KAG/LightRAG comparison table
│       └── components/
│           ├── graph_viz.py          # streamlit-agraph wrapper
│           └── benchmark_charts.py   # Plotly benchmark charts
│
├── scripts/
│   ├── init_db.py                    # Initialize PostgreSQL schema
│   ├── init_neo4j.py                 # Initialize Neo4j constraints + indexes
│   ├── ingest.py                     # CLI: ingest documents (text + images)
│   ├── build_graph.py                # CLI: build Neo4j graph from ingested chunks
│   ├── run_communities.py            # CLI: run community detection + summarization
│   ├── run_benchmark.py              # CLI: run benchmarks across all architectures
│   └── create_golden.py              # CLI: create/manage golden questions
│
├── data/
│   ├── raw/
│   │   ├── papers/                   # AI/ML research papers (PDF with images)
│   │   ├── tech_articles/            # Technology articles (text + images)
│   │   └── company_profiles/         # Company descriptions (entity-rich text)
│   ├── images/                       # Extracted images from PDFs
│   ├── golden/
│   │   └── golden_questions.json     # Golden Q&A dataset
│   └── benchmarks/
│       └── benchmark_results.json    # Benchmark output
│
└── tests/
    ├── __init__.py
    ├── conftest.py                   # Shared fixtures (DB pools, Neo4j driver, mock LLM)
    ├── test_chunker.py               # Semantic chunker tests
    ├── test_text_embedder.py         # BGE embedder tests
    ├── test_clip_embedder.py         # CLIP embedder tests
    ├── test_image_extractor.py       # PDF image extraction tests
    ├── test_entity_extractor.py      # LLM entity extraction tests
    ├── test_graph_builder.py         # Neo4j graph construction tests
    ├── test_community_detector.py    # Community detection tests
    ├── test_graph_retriever.py       # Graph retrieval tests
    ├── test_vector_retriever.py      # Vanilla vector RAG tests
    ├── test_multimodal_retriever.py  # Multimodal retrieval tests
    ├── test_agentic_loop.py          # Agentic retrieval loop tests
    ├── test_self_rag.py              # Self-RAG reflection token tests
    ├── test_decision_engine.py       # Decision framework tests
    ├── test_metrics.py               # Retrieval metrics tests
    ├── test_benchmark_runner.py      # Benchmark runner tests
    └── test_api.py                   # API integration tests
```

---

## 6. Implementation Phases

### Phase 1: Environment Setup — Neo4j + Postgres + Docker Compose (Day 1)

**Goal**: Docker Compose running with Neo4j, Postgres, API, and Dashboard services. Both database schemas deployed.

**Deliverables**:
- `docker-compose.yml` with 4 services: `postgres` (pgvector), `neo4j` (Community + GDS), `api` (FastAPI), `dashboard` (Streamlit)
- `Dockerfile` with all dependencies (neo4j driver, transformers, CLIP, PyMuPDF, etc.)
- `pyproject.toml` with all dependencies and optional-dependencies groups
- `.env.example` with all environment variables (DATABASE_URL, NEO4J_URI, NEO4J_USER, NEO4J_PASSWORD, LLM config, CLIP model, etc.)
- `sql/schema.sql` — full PostgreSQL DDL from Section 4.1
- `sql/neo4j/constraints.cypher` — Neo4j constraints and indexes from Section 4.2
- `src/config.py` — Pydantic Settings loading all env vars
- `scripts/init_db.py` — initialize PostgreSQL schema
- `scripts/init_neo4j.py` — initialize Neo4j constraints + indexes
- All package `__init__.py` files

**Verification**:
- `docker compose up -d postgres neo4j` starts both services
- `docker compose exec postgres psql -U rag -d ragdb -c "\dt"` shows all tables
- `docker compose exec neo4j cypher-shell -u neo4j -p testpassword "SHOW CONSTRAINTS;"` shows entity/document/chunk/community constraints
- `python scripts/init_db.py` → "Schema created successfully"
- `python scripts/init_neo4j.py` → "Neo4j constraints and indexes created successfully"

---

### Phase 2: GraphRAG — Entity/Relationship Extraction & Graph Construction (Day 2–3)

**Goal**: Documents ingested, entities and relationships extracted via LLM, Neo4j graph populated.

**Deliverables**:
- `src/ingestion/loader.py` — document loader for .txt, .md, .pdf (with image extraction flag)
- `src/ingestion/chunker.py` — semantic chunker (spaCy, reused from P1)
- `src/ingestion/text_embedder.py` — BGE text embedder (reused from P1)
- `src/graph/neo4j_client.py` — Neo4j async driver wrapper with connection pooling
- `src/graph/entity_extractor.py` — LLM-based entity + relationship extraction
  - Prompt template: given a chunk, extract entities (name, type, description) and relationships (source, target, type, description)
  - Entity types: PERSON, ORG, TECH, CONCEPT, LOCATION, EVENT, PRODUCT
  - Relationship types: DEVELOPED_BY, WORKS_FOR, COMPETES_WITH, PART_OF, USES, RELATED_TO, FOUNDED_BY, LOCATED_IN
  - Batch extraction: process chunks in batches, merge entities across chunks
- `src/graph/graph_builder.py` — build Neo4j graph from extracted entities
  - Create Document, Chunk, Entity nodes
  - Create MENTIONS, RELATED_TO, PART_OF, EXTRACTED_FROM relationships
  - Merge duplicate entities (same name + type) across chunks, increment mention_count
- `scripts/ingest.py` — CLI: load → chunk → embed → store in Postgres
- `scripts/build_graph.py` — CLI: read chunks from Postgres → LLM extraction → build Neo4j graph
- `data/raw/` — sample corpus: 5–10 AI/ML papers and tech articles with entity-rich content
- `tests/test_entity_extractor.py` — test LLM extraction with mock LLM
- `tests/test_graph_builder.py` — test graph construction with mock Neo4j driver

**Verification**:
- `python scripts/ingest.py --source data/raw/` → documents and chunks in Postgres
- `python scripts/build_graph.py` → entities and relationships in Neo4j
- `docker compose exec neo4j cypher-shell -u neo4j -p testpassword "MATCH (e:Entity) RETURN count(e) as entity_count;"` → >50 entities
- `docker compose exec neo4j cypher-shell -u neo4j -p testpassword "MATCH ()-[r:RELATED_TO]->() RETURN count(r) as rel_count;"` → >30 relationships
- `pytest tests/test_entity_extractor.py tests/test_graph_builder.py -v` → all pass

---

### Phase 3: GraphRAG Retrieval & Benchmarking vs Vanilla Vector RAG (Day 4)

**Goal**: Graph-based retrieval operational, benchmarked against vanilla vector RAG on the same corpus.

**Deliverables**:
- `src/graph/graph_retriever.py` — graph-based retrieval
  - Entity linking: fulltext search in Neo4j to find entities mentioned in the query
  - Graph traversal: Cypher query to find related entities within N hops (default 2)
  - Community lookup: get community summaries for linked entities
  - Subgraph context: extract relevant chunks via MENTIONS relationships
  - Return: entity names, community summaries, related chunks, subgraph context
- `src/retrieval/vector_retriever.py` — vanilla vector RAG (pgvector cosine, from P1)
- `src/retrieval/retrieval_result.py` — shared RetrievalResult dataclass
- `src/generation/llm_client.py` — LLM client (OpenAI / Ollama, from P1)
- `src/generation/prompt_templates.py` — prompt templates for vanilla + graph RAG
- `src/evaluation/metrics.py` — retrieval metrics (recall@k, MRR, nDCG, from P1)
- `src/evaluation/golden_dataset.py` — golden question management
- `data/golden/golden_questions.json` — 15–20 golden questions with expected answers
  - 5 factual (vanilla should win)
  - 5 multi-hop reasoning (graph should win)
  - 5 comparative (graph should win)
- `scripts/run_benchmark.py` — CLI: run vanilla + graph on golden questions, compute metrics
- `tests/test_graph_retriever.py` — test graph retrieval with mock Neo4j
- `tests/test_vector_retriever.py` — test vector retrieval with mock pool

**Verification**:
- `python scripts/run_benchmark.py --architectures vanilla graph` → benchmark results
- Graph RAG outperforms vanilla on multi-hop questions (higher recall@5 or MRR)
- Vanilla RAG outperforms graph on simple factual questions (lower latency)
- `pytest tests/test_graph_retriever.py tests/test_vector_retriever.py -v` → all pass

---

### Phase 4: Agentic RAG — Iterative Retrieval Loop (Day 5–6)

**Goal**: LLM-driven iterative retrieval loop with Self-RAG reflection tokens, capable of deciding what to retrieve and when to stop.

**Deliverables**:
- `src/agentic/planner.py` — LLM planner that decides retrieval strategy
  - Input: user query + current context (empty on first iteration)
  - Output: action plan with reflection tokens
  - Self-RAG tokens: `[RETRIEVE]`, `[NO RETRIEVE]`, `[RELEVANT]`, `[IRRELEVANT]`, `[GENERATE]`, `[NO GENERATE]`
  - Sub-query generation: reformulate query for better retrieval
  - Architecture selection: choose vector or graph retrieval based on query type
- `src/agentic/retrieval_loop.py` — iterative retrieval loop
  - Max iterations: 5 (configurable)
  - Loop: plan → retrieve → assess → (generate or loop)
  - Context accumulation: append retrieved chunks across iterations
  - Deduplication: skip chunks already retrieved
  - Termination conditions: (1) LLM says [GENERATE], (2) max iterations, (3) no new relevant chunks
- `src/agentic/self_rag.py` — Self-RAG reflection token parser
  - Parse LLM output for reflection tokens
  - Token state machine: RETRIEVE → retrieve → RELEVANT/IRRELEVANT → GENERATE/NO GENERATE
  - Fallback: if no tokens, use heuristic (always retrieve on first iteration, assess on subsequent)
- `src/agentic/trace.py` — retrieval trace dataclass
  - Per-iteration: sub-query, architecture used, retrieved chunks, relevance assessment, token sequence
  - Final: total iterations, total chunks retrieved, total latency, final answer
- `src/generation/prompt_templates.py` — add agentic prompt templates
  - Planner prompt: "Given the query and current context, decide whether to retrieve more information..."
  - Assessor prompt: "Given the retrieved chunks and the query, assess if the context is sufficient..."
  - Generator prompt: "Given the query and all retrieved context, generate a final answer..."
- `tests/test_agentic_loop.py` — test iterative loop with mock LLM
  - Test: loop terminates when LLM says [GENERATE]
  - Test: loop continues when LLM says [NO GENERATE]
  - Test: max iterations enforced
  - Test: context accumulates across iterations
  - Test: deduplication works
- `tests/test_self_rag.py` — test reflection token parsing
  - Test: parse [RETRIEVE] token
  - Test: parse [RELEVANT] / [IRRELEVANT] tokens
  - Test: fallback when no tokens present

**Verification**:
- Agentic RAG answers multi-hop questions correctly by retrieving iteratively
- Retrieval trace shows 2–4 iterations for complex queries, 1 for simple queries
- `pytest tests/test_agentic_loop.py tests/test_self_rag.py -v` → all pass

---

### Phase 5: Multimodal RAG — CLIP Embeddings & Cross-Modal Retrieval (Day 7)

**Goal**: Images extracted from documents, CLIP embeddings generated, cross-modal retrieval (text→image, image→text) operational.

**Deliverables**:
- `src/ingestion/image_extractor.py` — extract images from PDFs using PyMuPDF
  - Extract embedded images (JPEG, PNG)
  - Render full pages as images (for page-level multimodal)
  - Classify image type: image, chart, table, diagram, figure (using simple heuristics or LLM)
  - Extract surrounding text context (text near the image)
  - Generate image caption using LLM (vision model or text-based description)
- `src/ingestion/clip_embedder.py` — CLIP image + text encoder
  - Model: `openai/clip-vit-base-patch32` (512-dim)
  - `encode_image(image_path)` → 512-dim numpy array
  - `encode_images(image_paths, batch_size=32)` → batch encoding
  - `encode_text(text)` → 512-dim numpy array (for cross-modal text queries)
  - `encode_texts(texts)` → batch text encoding
  - Normalize embeddings (unit vectors for cosine similarity)
- `src/ingestion/ingest_pipeline.py` — full ingestion orchestrator
  - Load documents → extract text + images → chunk text → embed text (BGE) + images (CLIP) → store in Postgres
  - Generate captions for images using LLM → embed captions (BGE) for text-based image search
- `src/retrieval/multimodal_retriever.py` — cross-modal retrieval
  - `search_by_text(query_text, k=10)` → CLIP-encode query → search images by clip_embedding + search chunks by BGE embedding → fuse results
  - `search_by_image(query_image_path, k=10)` → CLIP-encode image → search images by clip_embedding + search chunks by caption_embedding → fuse results
  - Fusion: weighted combination of text and image results (configurable weights)
  - Return: text chunks + image references + scores
- `src/generation/prompt_templates.py` — add multimodal prompt template
  - "Given the query, text context, and image descriptions, generate an answer that references relevant images..."
- `data/raw/papers/` — ensure PDFs contain extractable images (charts, diagrams, figures)
- `tests/test_clip_embedder.py` — test CLIP encoding
  - Test: encode_image returns 512-dim array
  - Test: encode_text returns 512-dim array
  - Test: embeddings are normalized (unit vectors)
  - Test: image and text of same concept have high cosine similarity (sanity check)
- `tests/test_image_extractor.py` — test PDF image extraction
  - Test: extracts images from PDF
  - Test: classifies image types
  - Test: extracts surrounding text context
- `tests/test_multimodal_retriever.py` — test cross-modal retrieval
  - Test: text query returns relevant images
  - Test: image query returns relevant text
  - Test: fusion combines results correctly

**Verification**:
- `python scripts/ingest.py --source data/raw/ --extract-images` → images in Postgres
- `docker compose exec postgres psql -U rag -d ragdb -c "SELECT COUNT(*) FROM images;"` → >10 images
- Text query "architecture diagram" returns relevant images
- `pytest tests/test_clip_embedder.py tests/test_image_extractor.py tests/test_multimodal_retriever.py -v` → all pass

---

### Phase 6: KAG/LightRAG Overview + Comparison Table + Decision Framework (Day 8)

**Goal**: Comprehensive KAG/LightRAG comparison table and interactive RAG architecture decision framework.

**Deliverables**:
- `src/framework/comparison_tables.py` — KAG/LightRAG comparison data
  - Structured comparison table covering:
    - Architecture: indexing, retrieval, generation approach
    - Knowledge representation: vector embeddings vs structured KB vs graph
    - Retrieval mechanism: similarity search vs logical reasoning vs graph traversal
    - Strengths: what each system does best
    - Weaknesses: known limitations
    - Use cases: when to choose each
    - Setup complexity: low/medium/high
    - Maintenance: update frequency, re-indexing needs
    - Latency: typical query latency
    - Accuracy: on multi-hop, factual, temporal queries
  - Systems compared: Vanilla RAG, GraphRAG, KAG, LightRAG, Agentic RAG, Multimodal RAG
- `src/framework/query_analyzer.py` — analyze query characteristics
  - Input: user query + corpus metadata
  - Output: query profile with scores for:
    - `is_multimodal`: does the query ask about images/charts/visual content?
    - `entity_density`: how many entities are mentioned? (NER or LLM-based)
    - `relationship_complexity`: does the query require multi-hop reasoning?
    - `query_complexity`: simple factual vs complex analytical
    - `temporal_aspect`: does the query involve time-based reasoning?
    - `expected_latency_budget`: is this real-time or batch?
- `src/framework/decision_engine.py` — RAG architecture decision engine
  - Input: query profile (from query_analyzer) + corpus metadata
  - Output: architecture recommendation with confidence scores
  - Decision matrix (see Section 8 for full table):
    - If `is_multimodal` → Multimodal RAG
    - If `entity_density` high and `relationship_complexity` high → GraphRAG
    - If `query_complexity` high and needs iterative reasoning → Agentic RAG
    - If structured KB available and logical reasoning needed → KAG
    - If simple factual and low latency → Vanilla RAG
    - If lightweight graph + vector hybrid needed → LightRAG
  - Confidence scores: based on how clearly the criteria point to one architecture
- `tests/test_decision_engine.py` — test decision framework
  - Test: multimodal query → recommends Multimodal RAG
  - Test: multi-hop query with high entity density → recommends GraphRAG
  - Test: simple factual query → recommends Vanilla RAG
  - Test: complex analytical query → recommends Agentic RAG
  - Test: structured KB query → recommends KAG

**Verification**:
- Decision engine correctly routes all golden questions to their expected architectures
- `pytest tests/test_decision_engine.py -v` → all pass

---

### Phase 7: FastAPI Endpoints + Streamlit Comparison Dashboard (Day 9)

**Goal**: All 4 architectures exposed via FastAPI, interactive Streamlit dashboard for comparison.

**Deliverables**:
- `src/api/main.py` — FastAPI app with CORS, lifespan, router includes
- `src/api/routes.py` — API routes:
  - `POST /query` — query with architecture selector (vanilla, graph, agentic, multimodal)
  - `POST /query/compare` — run query across all architectures, return side-by-side results
  - `GET /graph/entities` — list entities in Neo4j graph
  - `GET /graph/subgraph` — get subgraph around an entity (for visualization)
  - `GET /images/{image_id}` — serve extracted image
  - `POST /decision` — get architecture recommendation for a query
  - `GET /benchmark` — get benchmark results
  - `GET /health` — health check (Postgres + Neo4j connectivity)
- `src/api/schemas.py` — Pydantic request/response schemas
  - QueryRequest: query, architecture, config (top_k, max_iterations, etc.)
  - QueryResponse: answer, retrieved_chunks, retrieved_images, retrieval_trace, latency_ms
  - CompareResponse: results for all 4 architectures
  - DecisionResponse: recommended_architecture, confidence, reasoning
- `src/dashboard/app.py` — Streamlit main app with navigation
- `src/dashboard/pages/1_Comparison.py` — side-by-side architecture comparison
  - Query input, architecture selector, results display
  - Retrieval trace visualization (especially agentic loop iterations)
  - Latency comparison
- `src/dashboard/pages/2_GraphRAG_Explorer.py` — Neo4j graph visualization
  - streamlit-agraph interactive graph display
  - Entity search, subgraph exploration
  - Community summaries display
- `src/dashboard/pages/3_Agentic_Trace.py` — agentic retrieval loop trace
  - Step-by-step iteration display
  - Self-RAG reflection tokens visualization
  - Sub-query evolution across iterations
- `src/dashboard/pages/4_Multimodal_Gallery.py` — image + text retrieval gallery
  - Text query → image results grid
  - Image upload → similar images + related text
  - Image metadata display (type, caption, context)
- `src/dashboard/pages/5_Decision_Framework.py` — interactive decision framework
  - Query input → query profile display
  - Interactive criteria sliders (entity density, complexity, etc.)
  - Architecture recommendation with confidence bars
  - Decision flowchart visualization
- `src/dashboard/pages/6_KAG_LightRAG.py` — KAG/LightRAG comparison table
  - Full comparison table from comparison_tables.py
  - Architecture diagrams (ASCII or rendered)
  - Use case examples
- `src/dashboard/components/graph_viz.py` — streamlit-agraph wrapper
- `src/dashboard/components/benchmark_charts.py` — Plotly benchmark charts
- `tests/test_api.py` — API integration tests

**Verification**:
- `docker compose up -d` starts all 4 services
- `curl -X POST http://localhost:8000/query -d '{"query": "Who developed the Transformer?", "architecture": "graph"}'` → answer with graph retrieval
- `curl -X POST http://localhost:8000/query/compare -d '{"query": "Compare GPT-4 and Claude"}'` → results from all 4 architectures
- Streamlit dashboard at `http://localhost:8501` → all 6 pages functional
- `pytest tests/test_api.py -v` → all pass

---

### Phase 8: Testing, Documentation, README (Day 10)

**Goal**: Full test suite passing, README complete, benchmark results documented.

**Deliverables**:
- `tests/conftest.py` — shared fixtures:
  - `db_pool`: asyncpg connection pool to test database
  - `neo4j_driver`: Neo4j driver to test database (or mock)
  - `mock_llm`: mock LLM client returning canned responses
  - `sample_chunks`: sample chunk data for tests
  - `sample_entities`: sample entity data for tests
  - `sample_images`: sample image paths for tests
- Complete test suite execution:
  - `pytest tests/ -v --tb=short` → all tests pass
  - Coverage report: `pytest tests/ --cov=src --cov-report=term-missing`
- `README.md` — project README with:
  - Project overview and goals
  - Architecture summary (4 architectures + decision framework)
  - Quickstart: `docker compose up -d` + ingest + query
  - API documentation (link to OpenAPI at `/docs`)
  - Dashboard guide
  - Benchmark results summary
  - Decision framework guide
  - KAG/LightRAG summary
  - Development setup
- `scripts/run_benchmark.py` — final benchmark run across all architectures
- `data/benchmarks/benchmark_results.json` — final benchmark results

**Verification**:
- `pytest tests/ -v` → all tests pass (30+ tests)
- `docker compose up -d` → all services healthy
- `curl http://localhost:8000/health` → `{"status": "healthy", "postgres": "connected", "neo4j": "connected"}`
- README is complete and accurate
- Benchmark results show clear differentiation between architectures

---

## 7. Component Specifications

### 7.1 Document Loader (`src/ingestion/loader.py`)

```python
from dataclasses import dataclass, field
from pathlib import Path
import hashlib
import fitz  # PyMuPDF

@dataclass
class Document:
    source_path: str
    title: str
    doc_type: str          # 'txt', 'md', 'pdf', 'html', 'image'
    content: str
    content_hash: str      # SHA-256
    char_count: int
    has_images: bool = False
    image_count: int = 0
    metadata: dict = field(default_factory=dict)

class DocumentLoader:
    """Load documents from files. Supports .txt, .md, .pdf."""

    def load(self, path: Path) -> list[Document]:
        """Load all supported documents from a directory tree."""
        documents = []
        for file_path in path.rglob("*"):
            if file_path.suffix.lower() in (".txt", ".md", ".pdf"):
                doc = self._load_file(file_path)
                if doc:
                    documents.append(doc)
        # Deduplicate by content_hash
        seen = set()
        unique = []
        for doc in documents:
            if doc.content_hash not in seen:
                seen.add(doc.content_hash)
                unique.append(doc)
        return unique

    def _load_file(self, path: Path) -> Document | None:
        suffix = path.suffix.lower()
        if suffix == ".txt":
            content = path.read_text(encoding="utf-8")
            return self._make_doc(path, content, "txt")
        elif suffix == ".md":
            content = path.read_text(encoding="utf-8")
            # Strip markdown formatting (simple regex or markdown library)
            content = self._strip_markdown(content)
            return self._make_doc(path, content, "md")
        elif suffix == ".pdf":
            content, has_images, image_count = self._extract_pdf_text(path)
            doc = self._make_doc(path, content, "pdf")
            doc.has_images = has_images
            doc.image_count = image_count
            return doc
        return None

    def _extract_pdf_text(self, path: Path) -> tuple[str, bool, int]:
        """Extract text from PDF using PyMuPDF. Returns (text, has_images, image_count)."""
        doc = fitz.open(path)
        text_parts = []
        image_count = 0
        for page in doc:
            text_parts.append(page.get_text())
            image_list = page.get_images(full=True)
            image_count += len(image_list)
        doc.close()
        text = "\n\n".join(text_parts)
        return text, image_count > 0, image_count

    def _make_doc(self, path: Path, content: str, doc_type: str) -> Document:
        content_hash = hashlib.sha256(content.encode()).hexdigest()
        title = path.stem.replace("_", " ").title()
        return Document(
            source_path=str(path),
            title=title,
            doc_type=doc_type,
            content=content,
            content_hash=content_hash,
            char_count=len(content),
            metadata={"source": str(path)},
        )

    def _strip_markdown(self, content: str) -> str:
        """Simple markdown stripping: remove headers, links, images, formatting."""
        import re
        # Remove code blocks
        content = re.sub(r'```[\s\S]*?```', '', content)
        # Remove images
        content = re.sub(r'!\[.*?\]\(.*?\)', '', content)
        # Remove links, keep text
        content = re.sub(r'\[([^\]]+)\]\([^\)]+\)', r'\1', content)
        # Remove headers
        content = re.sub(r'^#+\s+', '', content, flags=re.MULTILINE)
        # Remove bold/italic
        content = re.sub(r'\*{1,3}([^\*]+)\*{1,3}', r'\1', content)
        return content.strip()
```

### 7.2 Semantic Chunker (`src/ingestion/chunker.py`)

Reused from Project 1. The `SemanticChunker` uses spaCy `en_core_web_sm` for sentence segmentation, groups sentences until ~1000 chars, never breaks a sentence. Strategy label: `semantic_spacy`.

```python
from dataclasses import dataclass, field
import spacy

@dataclass
class Chunk:
    content: str
    chunk_index: int
    chunking_strategy: str
    token_count: int
    window_context: str | None = None
    metadata: dict = field(default_factory=dict)

class SemanticChunker:
    def __init__(self, target_size: int = 1000, nlp_model: str = "en_core_web_sm"):
        self.target_size = target_size
        self.nlp = spacy.load(nlp_model)

    def chunk(self, text: str) -> list[Chunk]:
        doc = self.nlp(text)
        sentences = [s.text.strip() for s in doc.sents if s.text.strip()]
        chunks = []
        current = []
        current_size = 0
        for sent in sentences:
            if current_size + len(sent) > self.target_size and current:
                content = " ".join(current)
                chunks.append(Chunk(
                    content=content,
                    chunk_index=len(chunks),
                    chunking_strategy="semantic_spacy",
                    token_count=len(content.split()),
                ))
                current = [sent]
                current_size = len(sent)
            else:
                current.append(sent)
                current_size += len(sent)
        if current:
            content = " ".join(current)
            chunks.append(Chunk(
                content=content,
                chunk_index=len(chunks),
                chunking_strategy="semantic_spacy",
                token_count=len(content.split()),
            ))
        return chunks
```

### 7.3 Neo4j Client & Graph Builder (`src/graph/neo4j_client.py`, `src/graph/graph_builder.py`)

```python
# src/graph/neo4j_client.py
from neo4j import AsyncGraphDatabase
from dataclasses import dataclass

@dataclass
class Neo4jConfig:
    uri: str           # bolt://localhost:7687
    user: str          # neo4j
    password: str      # testpassword
    database: str      # neo4j

class Neo4jClient:
    """Async Neo4j driver wrapper with connection pooling."""

    def __init__(self, config: Neo4jConfig):
        self.config = config
        self._driver = None

    async def connect(self):
        self._driver = AsyncGraphDatabase.driver(
            self.config.uri,
            auth=(self.config.user, self.config.password),
        )

    async def close(self):
        if self._driver:
            await self._driver.close()

    async def execute(self, query: str, params: dict = None) -> list[dict]:
        """Execute a Cypher query and return results as list of dicts."""
        async with self._driver.session(database=self.config.database) as session:
            result = await session.run(query, params or {})
            return [record.data() async for record in result]

    async def execute_write(self, query: str, params: dict = None) -> dict:
        """Execute a write query and return summary."""
        async with self._driver.session(database=self.config.database) as session:
            result = await session.run(query, params or {})
            summary = await result.consume()
            return {
                "nodes_created": summary.counters.nodes_created,
                "relationships_created": summary.counters.relationships_created,
                "properties_set": summary.counters.properties_set,
            }

    async def health_check(self) -> bool:
        """Check if Neo4j is reachable."""
        try:
            result = await self.execute("RETURN 1 as test")
            return result[0]["test"] == 1
        except Exception:
            return False
```

```python
# src/graph/graph_builder.py
from dataclasses import dataclass
from src.graph.neo4j_client import Neo4jClient

@dataclass
class ExtractedEntity:
    name: str
    entity_type: str
    description: str
    chunk_id: str
    confidence: float

@dataclass
class ExtractedRelationship:
    source_name: str
    source_type: str
    target_name: str
    target_type: str
    relationship_type: str
    description: str
    chunk_id: str
    confidence: float

class GraphBuilder:
    """Build Neo4j graph from extracted entities and relationships."""

    def __init__(self, client: Neo4jClient):
        self.client = client

    async def create_document_node(self, doc_id: str, title: str, doc_type: str, source_path: str):
        query = """
        MERGE (d:Document {doc_id: $doc_id})
        SET d.title = $title, d.doc_type = $doc_type, d.source_path = $source_path,
            d.created_at = datetime()
        """
        await self.client.execute_write(query, {
            "doc_id": doc_id, "title": title, "doc_type": doc_type, "source_path": source_path
        })

    async def create_chunk_node(self, chunk_id: str, doc_id: str, content: str, chunk_index: int):
        query = """
        MERGE (c:Chunk {chunk_id: $chunk_id})
        SET c.doc_id = $doc_id, c.content = $content, c.chunk_index = $chunk_index,
            c.created_at = datetime()
        WITH c
        MATCH (d:Document {doc_id: $doc_id})
        MERGE (c)-[:PART_OF]->(d)
        """
        await self.client.execute_write(query, {
            "chunk_id": chunk_id, "doc_id": doc_id, "content": content, "chunk_index": chunk_index
        })

    async def create_entity(self, entity: ExtractedEntity, doc_id: str):
        query = """
        MERGE (e:Entity {name: $name, entity_type: $entity_type})
        SET e.description = COALESCE(e.description, $description),
            e.doc_id = COALESCE(e.doc_id, $doc_id),
            e.chunk_id = COALESCE(e.chunk_id, $chunk_id),
            e.mention_count = COALESCE(e.mention_count, 0) + 1,
            e.extraction_confidence = $confidence,
            e.created_at = COALESCE(e.created_at, datetime())
        WITH e
        MATCH (c:Chunk {chunk_id: $chunk_id})
        MERGE (e)-[:MENTIONS {confidence: $confidence}]->(c)
        """
        await self.client.execute_write(query, {
            "name": entity.name, "entity_type": entity.entity_type,
            "description": entity.description, "doc_id": doc_id,
            "chunk_id": entity.chunk_id, "confidence": entity.confidence
        })

    async def create_relationship(self, rel: ExtractedRelationship):
        query = """
        MATCH (src:Entity {name: $src_name, entity_type: $src_type})
        MATCH (tgt:Entity {name: $tgt_name, entity_type: $tgt_type})
        MERGE (src)-[r:RELATED_TO {
            type: $rel_type,
            chunk_id: $chunk_id
        }]->(tgt)
        SET r.description = $description, r.confidence = $confidence
        """
        await self.client.execute_write(query, {
            "src_name": rel.source_name, "src_type": rel.source_type,
            "tgt_name": rel.target_name, "tgt_type": rel.target_type,
            "rel_type": rel.relationship_type, "chunk_id": rel.chunk_id,
            "description": rel.description, "confidence": rel.confidence
        })

    async def build_from_extractions(
        self,
        doc_id: str,
        title: str,
        doc_type: str,
        source_path: str,
        chunks: list[dict],           # [{chunk_id, content, chunk_index}]
        entities: list[ExtractedEntity],
        relationships: list[ExtractedRelationship],
    ):
        """Full graph construction from extraction results."""
        await self.create_document_node(doc_id, title, doc_type, source_path)
        for chunk in chunks:
            await self.create_chunk_node(chunk["chunk_id"], doc_id, chunk["content"], chunk["chunk_index"])
        for entity in entities:
            await self.create_entity(entity, doc_id)
        for rel in relationships:
            await self.create_relationship(rel)

    async def get_graph_stats(self) -> dict:
        """Return entity count, relationship count, chunk count."""
        stats = {}
        result = await self.client.execute("MATCH (e:Entity) RETURN count(e) as count")
        stats["entities"] = result[0]["count"] if result else 0
        result = await self.client.execute("MATCH ()-[r:RELATED_TO]->() RETURN count(r) as count")
        stats["relationships"] = result[0]["count"] if result else 0
        result = await self.client.execute("MATCH (c:Chunk) RETURN count(c) as count")
        stats["chunks"] = result[0]["count"] if result else 0
        result = await self.client.execute("MATCH (d:Document) RETURN count(d) as count")
        stats["documents"] = result[0]["count"] if result else 0
        return stats
```

### 7.4 Entity Extractor (`src/graph/entity_extractor.py`)

```python
from dataclasses import dataclass
from src.graph.graph_builder import ExtractedEntity, ExtractedRelationship
from src.generation.llm_client import LLMClient
import json

ENTITY_EXTRACTION_PROMPT = """You are an expert at extracting entities and relationships from text.

Given the following text chunk, extract:
1. ENTITIES: named entities with their type and a brief description
2. RELATIONSHIPS: relationships between entities

Entity types: PERSON, ORG, TECH, CONCEPT, LOCATION, EVENT, PRODUCT
Relationship types: DEVELOPED_BY, WORKS_FOR, COMPETES_WITH, PART_OF, USES, RELATED_TO, FOUNDED_BY, LOCATED_IN, CREATED, MEMBER_OF

Text chunk:
{chunk_text}

Respond in JSON format:
{{
  "entities": [
    {{"name": "Entity Name", "type": "ORG", "description": "Brief description"}}
  ],
  "relationships": [
    {{"source": "Entity A", "source_type": "ORG", "target": "Entity B", "target_type": "TECH", "type": "DEVELOPED_BY", "description": "A developed B"}}
  ]
}}

Extract only entities and relationships that are clearly stated or strongly implied in the text.
Be precise with entity names (use canonical names, not pronouns).
"""

class EntityExtractor:
    """LLM-based entity and relationship extraction."""

    def __init__(self, llm_client: LLMClient):
        self.llm = llm_client

    async def extract(self, chunk_text: str, chunk_id: str) -> tuple[list[ExtractedEntity], list[ExtractedRelationship]]:
        """Extract entities and relationships from a single chunk."""
        prompt = ENTITY_EXTRACTION_PROMPT.format(chunk_text=chunk_text)
        response = await self.llm.generate(prompt, temperature=0.0)

        try:
            data = json.loads(response)
        except json.JSONDecodeError:
            # Try to extract JSON from the response
            import re
            match = re.search(r'\{[\s\S]*\}', response)
            if match:
                data = json.loads(match.group())
            else:
                return [], []

        entities = []
        for ent in data.get("entities", []):
            entities.append(ExtractedEntity(
                name=ent["name"].strip(),
                entity_type=ent["type"].strip().upper(),
                description=ent.get("description", "").strip(),
                chunk_id=chunk_id,
                confidence=0.85,  # Could be LLM-scored in the future
            ))

        relationships = []
        for rel in data.get("relationships", []):
            relationships.append(ExtractedRelationship(
                source_name=rel["source"].strip(),
                source_type=rel.get("source_type", "CONCEPT").strip().upper(),
                target_name=rel["target"].strip(),
                target_type=rel.get("target_type", "CONCEPT").strip().upper(),
                relationship_type=rel["type"].strip().upper(),
                description=rel.get("description", "").strip(),
                chunk_id=chunk_id,
                confidence=0.80,
            ))

        return entities, relationships

    async def extract_batch(self, chunks: list[dict]) -> tuple[list[ExtractedEntity], list[ExtractedRelationship]]:
        """Extract entities from multiple chunks and merge duplicates."""
        all_entities = []
        all_relationships = []
        for chunk in chunks:
            entities, rels = await self.extract(chunk["content"], chunk["chunk_id"])
            all_entities.extend(entities)
            all_relationships.extend(rels)

        # Merge duplicate entities (same name + type), keep first description
        seen = {}
        merged_entities = []
        for ent in all_entities:
            key = (ent.name.lower(), ent.entity_type)
            if key not in seen:
                seen[key] = True
                merged_entities.append(ent)

        return merged_entities, all_relationships
```

### 7.5 Community Detector (`src/graph/community_detector.py`)

```python
from src.graph.neo4j_client import Neo4jClient
from src.generation.llm_client import LLMClient

class CommunityDetector:
    """Run Leiden community detection and generate community summaries."""

    def __init__(self, client: Neo4jClient, llm: LLMClient):
        self.client = client
        self.llm = llm

    async def run_leiden(self) -> dict:
        """Run Leiden algorithm via Neo4j GDS and write community IDs to nodes."""
        # Project graph (if not already projected)
        project_query = """
        CALL gds.graph.project(
            'entityGraph',
            ['Entity'],
            {
                RELATED_TO: {
                    orientation: 'UNDIRECTED',
                    properties: ['confidence']
                }
            }
        )
        """
        try:
            await self.client.execute(project_query)
        except Exception:
            pass  # Graph may already be projected

        # Run Leiden
        leiden_query = """
        CALL gds.leiden.write(
            'entityGraph',
            {
                writeProperty: 'communityId',
                maxIterations: 10,
                relationshipWeightProperty: 'confidence',
                includeIntermediateCommunities: true
            }
        )
        YIELD communityCount, modularity
        """
        result = await self.client.execute(leiden_query)
        stats = result[0] if result else {}
        return {"communities": stats.get("communityCount", 0), "modularity": stats.get("modularity", 0)}

    async def get_communities(self) -> list[dict]:
        """Get all communities with their entity counts."""
        query = """
        MATCH (e:Entity)
        WHERE e.communityId IS NOT NULL
        WITH e.communityId as cid, collect(e) as entities
        RETURN cid, size(entities) as entity_count,
               [e in entities | e.name] as entity_names
        ORDER BY entity_count DESC
        """
        return await self.client.execute(query)

    async def summarize_community(self, community_id: int, entity_names: list[str]) -> str:
        """Generate an LLM summary of a community."""
        # Get relationships within the community
        rel_query = """
        MATCH (e1:Entity)-[r:RELATED_TO]->(e2:Entity)
        WHERE e1.communityId = $cid AND e2.communityId = $cid
        RETURN e1.name as source, r.type as type, e2.name as target, r.description as description
        LIMIT 20
        """
        rels = await self.client.execute(rel_query, {"cid": community_id})

        # Build context for LLM
        entity_list = ", ".join(entity_names[:20])
        rel_list = "\n".join([f"- {r['source']} {r['type']} {r['target']}: {r['description']}" for r in rels[:15]])

        prompt = f"""Summarize the following group of related entities and their relationships in 2-3 sentences:

Entities: {entity_list}

Relationships:
{rel_list}

Summary:"""
        summary = await self.llm.generate(prompt, temperature=0.3, max_tokens=200)
        return summary.strip()

    async def create_community_nodes(self):
        """Create Community nodes in Neo4j and link entities to them."""
        communities = await self.get_communities()
        for comm in communities:
            cid = comm["cid"]
            entity_names = comm["entity_names"]
            entity_count = comm["entity_count"]

            # Generate summary
            summary = await self.summarize_community(cid, entity_names)

            # Get relationship count within community
            rel_count_query = """
            MATCH (e1:Entity)-[r:RELATED_TO]->(e2:Entity)
            WHERE e1.communityId = $cid AND e2.communityId = $cid
            RETURN count(r) as count
            """
            rel_result = await self.client.execute(rel_count_query, {"cid": cid})
            rel_count = rel_result[0]["count"] if rel_result else 0

            # Key entities (top 5 by mention count)
            key_entities = entity_names[:5]

            # Create Community node
            create_query = """
            CREATE (com:Community {
                community_id: $cid,
                level: 0,
                summary: $summary,
                entity_count: $entity_count,
                rel_count: $rel_count,
                key_entities: $key_entities,
                created_at: datetime()
            })
            WITH com
            MATCH (e:Entity)
            WHERE e.communityId = $cid
            CREATE (e)-[:BELONGS_TO {level: 0}]->(com)
            """
            await self.client.execute_write(create_query, {
                "cid": cid, "summary": summary, "entity_count": entity_count,
                "rel_count": rel_count, "key_entities": key_entities
            })

    async def cleanup(self):
        """Drop the GDS graph projection."""
        try:
            await self.client.execute("CALL gds.graph.drop('entityGraph')")
        except Exception:
            pass
```

### 7.6 Graph Retriever (`src/graph/graph_retriever.py`)

```python
from dataclasses import dataclass, field
from src.graph.neo4j_client import Neo4jClient
from src.generation.llm_client import LLMClient

@dataclass
class GraphRetrievalResult:
    query: str
    linked_entities: list[dict] = field(default_factory=list)     # [{name, type, description}]
    related_entities: list[dict] = field(default_factory=list)    # [{name, type, hop_count, relationship_path}]
    community_summaries: list[str] = field(default_factory=list)
    retrieved_chunks: list[dict] = field(default_factory=list)    # [{chunk_id, content, score}]
    subgraph_context: str = ""
    latency_ms: int = 0

class GraphRetriever:
    """Graph-based retrieval using entity linking and graph traversal."""

    def __init__(self, client: Neo4jClient, llm: LLMClient, max_hops: int = 2, top_k: int = 10):
        self.client = client
        self.llm = llm
        self.max_hops = max_hops
        self.top_k = top_k

    async def retrieve(self, query: str) -> GraphRetrievalResult:
        """Full graph retrieval pipeline: entity linking → traversal → community → chunks."""
        import time
        start = time.time()

        result = GraphRetrievalResult(query=query)

        # Step 1: Entity linking — find entities mentioned in the query
        linked = await self._link_entities(query)
        result.linked_entities = linked

        if not linked:
            result.latency_ms = int((time.time() - start) * 1000)
            return result

        # Step 2: Graph traversal — find related entities within N hops
        related = await self._traverse_graph([e["name"] for e in linked])
        result.related_entities = related

        # Step 3: Community lookup — get community summaries
        communities = await self._get_communities([e["name"] for e in linked])
        result.community_summaries = communities

        # Step 4: Chunk retrieval — get chunks mentioned by linked + related entities
        chunks = await self._get_chunks([e["name"] for e in linked + related])
        result.retrieved_chunks = chunks[:self.top_k]

        # Step 5: Build subgraph context string for LLM
        result.subgraph_context = self._build_context(result)

        result.latency_ms = int((time.time() - start) * 1000)
        return result

    async def _link_entities(self, query: str) -> list[dict]:
        """Find entities in the graph that are mentioned in the query using fulltext search."""
        query_str = """
        CALL db.index.fulltext.queryNodes('entity_fulltext', $query)
        YIELD node, score
        RETURN node.name as name, node.entity_type as entity_type,
               node.description as description, score
        ORDER BY score DESC
        LIMIT 5
        """
        try:
            results = await self.client.execute(query_str, {"query": query})
            return results
        except Exception:
            return []

    async def _traverse_graph(self, entity_names: list[str], max_hops: int = None) -> list[dict]:
        """Find related entities within N hops via Cypher traversal."""
        hops = max_hops or self.max_hops
        query_str = f"""
        UNWIND $names as name
        MATCH (e:Entity {{name: name}})
        CALL {{
            WITH e
            MATCH path = (e)-[:RELATED_TO*1..{hops}]-(related:Entity)
            RETURN related, relationships(path) as rels, length(path) as hopCount
            LIMIT 20
        }}
        WITH related, min(hopCount) as hopCount,
             collect(DISTINCT [r in rels | r.type])[0] as relTypes
        RETURN related.name as name, related.entity_type as entity_type,
               related.description as description, hopCount, relTypes
        ORDER BY hopCount ASC
        LIMIT 15
        """
        results = await self.client.execute(query_str, {"names": entity_names})
        return results

    async def _get_communities(self, entity_names: list[str]) -> list[str]:
        """Get community summaries for entities."""
        query_str = """
        UNWIND $names as name
        MATCH (e:Entity {{name: name}})-[:BELONGS_TO]->(com:Community)
        RETURN DISTINCT com.community_id as cid, com.summary as summary, com.key_entities as key_entities
        ORDER BY com.level ASC
        """
        results = await self.client.execute(query_str, {"names": entity_names})
        return [r["summary"] for r in results]

    async def _get_chunks(self, entity_names: list[str]) -> list[dict]:
        """Get chunks mentioned by the given entities."""
        query_str = """
        UNWIND $names as name
        MATCH (e:Entity {{name: name}})-[:MENTIONS]->(c:Chunk)
        RETURN DISTINCT c.chunk_id as chunk_id, c.content as content, c.doc_id as doc_id
        LIMIT 20
        """
        results = await self.client.execute(query_str, {"names": entity_names})
        return results

    def _build_context(self, result: GraphRetrievalResult) -> str:
        """Build a context string from graph retrieval results for the LLM."""
        parts = []
        if result.linked_entities:
            parts.append("## Linked Entities")
            for e in result.linked_entities:
                parts.append(f"- {e['name']} ({e['entity_type']}): {e.get('description', '')}")

        if result.related_entities:
            parts.append("\n## Related Entities")
            for e in result.related_entities:
                parts.append(f"- {e['name']} ({e['entity_type']}, {e['hopCount']} hops away): {e.get('description', '')}")

        if result.community_summaries:
            parts.append("\n## Community Summaries")
            for i, s in enumerate(result.community_summaries):
                parts.append(f"### Community {i+1}\n{s}")

        if result.retrieved_chunks:
            parts.append("\n## Relevant Chunks")
            for i, c in enumerate(result.retrieved_chunks):
                parts.append(f"### Chunk {i+1}\n{c['content'][:500]}")

        return "\n".join(parts)
```

### 7.7 Agentic Retrieval Loop (`src/agentic/retrieval_loop.py`, `src/agentic/self_rag.py`)

```python
# src/agentic/self_rag.py
from dataclasses import dataclass
from enum import Enum
import re

class ReflectionToken(Enum):
    RETRIEVE = "RETRIEVE"
    NO_RETRIEVE = "NO RETRIEVE"
    RELEVANT = "RELEVANT"
    IRRELEVANT = "IRRELEVANT"
    GENERATE = "GENERATE"
    NO_GENERATE = "NO GENERATE"

@dataclass
class TokenParseResult:
    retrieve: bool = False
    relevant: bool = False
    generate: bool = False
    raw_tokens: list[str] = None
    sub_query: str = ""
    reasoning: str = ""

class SelfRAGParser:
    """Parse Self-RAG reflection tokens from LLM output."""

    TOKEN_PATTERN = r'\[(RETRIEVE|NO RETRIEVE|RELEVANT|IRRELEVANT|GENERATE|NO GENERATE)\]'
    SUB_QUERY_PATTERN = r'SUB_QUERY:\s*(.*?)(?:\n|$)'
    REASONING_PATTERN = r'REASONING:\s*(.*?)(?:\n\[|\Z)'

    def parse(self, llm_output: str) -> TokenParseResult:
        """Parse reflection tokens and sub-query from LLM output."""
        tokens = re.findall(self.TOKEN_PATTERN, llm_output, re.IGNORECASE)
        sub_query_match = re.search(self.SUB_QUERY_PATTERN, llm_output, re.IGNORECASE | re.DOTALL)
        reasoning_match = re.search(self.REASONING_PATTERN, llm_output, re.IGNORECASE | re.DOTALL)

        result = TokenParseResult(
            retrieve=any(t.upper() == "RETRIEVE" for t in tokens),
            relevant=any(t.upper() == "RELEVANT" for t in tokens),
            generate=any(t.upper() == "GENERATE" for t in tokens),
            raw_tokens=[t.upper() for t in tokens],
            sub_query=sub_query_match.group(1).strip() if sub_query_match else "",
            reasoning=reasoning_match.group(1).strip() if reasoning_match else llm_output,
        )

        # Fallback: if no tokens, use heuristics
        if not tokens:
            # First iteration: always retrieve
            # Subsequent: assess based on output content
            result.retrieve = True  # Default to retrieve if no tokens

        return result
```

```python
# src/agentic/retrieval_loop.py
from dataclasses import dataclass, field
from src.agentic.self_rag import SelfRAGParser, TokenParseResult
from src.generation.llm_client import LLMClient
from src.retrieval.vector_retriever import VectorRetriever
from src.graph.graph_retriever import GraphRetriever
import time

@dataclass
class IterationTrace:
    iteration: int
    sub_query: str
    architecture_used: str          # "vector" or "graph"
    tokens: list[str]               # reflection tokens
    retrieved_chunk_ids: list[str]
    retrieved_count: int
    is_relevant: bool
    reasoning: str
    latency_ms: int

@dataclass
class AgenticTrace:
    query: str
    iterations: list[IterationTrace] = field(default_factory=list)
    total_chunks_retrieved: int = 0
    total_latency_ms: int = 0
    final_answer: str = ""
    terminated_reason: str = ""     # "generate", "max_iterations", "no_new_chunks"

PLANNER_PROMPT = """You are an intelligent retrieval planner for a RAG system.

Given:
- User query: {query}
- Current context (chunks retrieved so far): {context}
- Iteration: {iteration} of {max_iterations}

Decide whether to retrieve more information or generate an answer.

If you need to retrieve more:
- Output [RETRIEVE]
- Provide a SUB_QUERY: a reformulated query optimized for retrieval
- Choose retrieval architecture: "vector" for factual lookups, "graph" for entity-relationship queries

If the context is sufficient to answer:
- Output [GENERATE]
- Explain why the context is sufficient

If retrieved chunks are not relevant:
- Output [IRRELEVANT]
- Reformulate the query differently

Format:
[RETRIEVE] or [GENERATE] or [NO RETRIEVE]
SUB_QUERY: <reformulated query or "N/A">
ARCHITECTURE: <vector or graph>
REASONING: <your reasoning>
"""

class AgenticRetrievalLoop:
    """Iterative retrieval loop where the LLM decides what to retrieve."""

    def __init__(
        self,
        llm: LLMClient,
        vector_retriever: VectorRetriever,
        graph_retriever: GraphRetriever = None,
        max_iterations: int = 5,
        top_k: int = 5,
    ):
        self.llm = llm
        self.vector_retriever = vector_retriever
        self.graph_retriever = graph_retriever
        self.max_iterations = max_iterations
        self.top_k = top_k
        self.parser = SelfRAGParser()

    async def run(self, query: str) -> tuple[str, AgenticTrace]:
        """Run the agentic retrieval loop. Returns (answer, trace)."""
        trace = AgenticTrace(query=query)
        start = time.time()
        accumulated_chunks = {}  # chunk_id -> content (dedup)
        seen_chunk_ids = set()

        for iteration in range(1, self.max_iterations + 1):
            iter_start = time.time()

            # Step 1: Plan — ask LLM what to do
            context = self._format_context(accumulated_chunks)
            prompt = PLANNER_PROMPT.format(
                query=query,
                context=context if context else "(no context yet)",
                iteration=iteration,
                max_iterations=self.max_iterations,
            )
            plan_response = await self.llm.generate(prompt, temperature=0.0)
            parsed = self.parser.parse(plan_response)

            # Step 2: Check termination — if LLM says GENERATE, generate answer
            if parsed.generate and accumulated_chunks:
                answer = await self._generate_answer(query, accumulated_chunks)
                trace.iterations.append(IterationTrace(
                    iteration=iteration,
                    sub_query="N/A",
                    architecture_used="none",
                    tokens=parsed.raw_tokens,
                    retrieved_chunk_ids=[],
                    retrieved_count=0,
                    is_relevant=True,
                    reasoning=parsed.reasoning,
                    latency_ms=int((time.time() - iter_start) * 1000),
                ))
                trace.final_answer = answer
                trace.terminated_reason = "generate"
                break

            # Step 3: Retrieve — use the sub-query to retrieve more chunks
            sub_query = parsed.sub_query if parsed.sub_query else query
            architecture = "vector"  # default
            if "graph" in plan_response.lower() and self.graph_retriever:
                architecture = "graph"

            new_chunks = []
            if architecture == "graph" and self.graph_retriever:
                graph_result = await self.graph_retriever.retrieve(sub_query)
                new_chunks = graph_result.retrieved_chunks
            else:
                new_chunks = await self.vector_retriever.search(sub_query, k=self.top_k)

            # Step 4: Deduplicate — skip chunks already retrieved
            new_chunk_ids = set()
            relevant_chunks = []
            for chunk in new_chunks:
                cid = chunk.get("chunk_id", str(hash(chunk.get("content", ""))))
                if cid not in seen_chunk_ids:
                    seen_chunk_ids.add(cid)
                    new_chunk_ids.add(cid)
                    accumulated_chunks[cid] = chunk.get("content", "")
                    relevant_chunks.append(cid)

            # Step 5: Assess — are the new chunks relevant?
            is_relevant = len(relevant_chunks) > 0

            trace.iterations.append(IterationTrace(
                iteration=iteration,
                sub_query=sub_query,
                architecture_used=architecture,
                tokens=parsed.raw_tokens,
                retrieved_chunk_ids=list(new_chunk_ids),
                retrieved_count=len(relevant_chunks),
                is_relevant=is_relevant,
                reasoning=parsed.reasoning,
                latency_ms=int((time.time() - iter_start) * 1000),
            ))

            # Step 6: Check termination — no new relevant chunks
            if not is_relevant and iteration > 1:
                answer = await self._generate_answer(query, accumulated_chunks)
                trace.final_answer = answer
                trace.terminated_reason = "no_new_chunks"
                break

        else:
            # Max iterations reached — force generation
            answer = await self._generate_answer(query, accumulated_chunks)
            trace.final_answer = answer
            trace.terminated_reason = "max_iterations"

        trace.total_chunks_retrieved = len(accumulated_chunks)
        trace.total_latency_ms = int((time.time() - start) * 1000)
        return answer, trace

    def _format_context(self, chunks: dict) -> str:
        """Format accumulated chunks into a context string."""
        if not chunks:
            return ""
        parts = []
        for i, (cid, content) in enumerate(chunks.items(), 1):
            parts.append(f"### Context {i}\n{content[:500]}")
        return "\n\n".join(parts)

    async def _generate_answer(self, query: str, chunks: dict) -> str:
        """Generate final answer from accumulated context."""
        context = self._format_context(chunks)
        prompt = f"""Given the following context and query, generate a comprehensive answer.

Query: {query}

Context:
{context}

Answer:"""
        return await self.llm.generate(prompt, temperature=0.3)
```

### 7.8 CLIP Embedder (`src/ingestion/clip_embedder.py`)

```python
import numpy as np
from PIL import Image
from transformers import CLIPModel, CLIPProcessor
import torch

class CLIPEmbedder:
    """CLIP image and text encoder for multimodal RAG."""

    MODEL_NAME = "openai/clip-vit-base-patch32"
    DIMENSION = 512

    def __init__(self, model_name: str = None, device: str = "cpu"):
        model_name = model_name or self.MODEL_NAME
        self.device = device
        self.model = CLIPModel.from_pretrained(model_name).to(device)
        self.processor = CLIPProcessor.from_pretrained(model_name)
        self.model.eval()

    def encode_image(self, image_path: str) -> np.ndarray:
        """Encode a single image into a 512-dim normalized vector."""
        image = Image.open(image_path).convert("RGB")
        inputs = self.processor(images=image, return_tensors="pt").to(self.device)
        with torch.no_grad():
            features = self.model.get_image_features(**inputs)
        embedding = features.cpu().numpy()[0]
        # Normalize to unit vector
        norm = np.linalg.norm(embedding)
        if norm > 0:
            embedding = embedding / norm
        return embedding

    def encode_images(self, image_paths: list[str], batch_size: int = 32) -> np.ndarray:
        """Encode multiple images. Returns (N, 512) array."""
        embeddings = []
        for i in range(0, len(image_paths), batch_size):
            batch = image_paths[i:i + batch_size]
            images = [Image.open(p).convert("RGB") for p in batch]
            inputs = self.processor(images=images, return_tensors="pt").to(self.device)
            with torch.no_grad():
                features = self.model.get_image_features(**inputs)
            batch_emb = features.cpu().numpy()
            # Normalize
            norms = np.linalg.norm(batch_emb, axis=1, keepdims=True)
            norms[norms == 0] = 1
            embeddings.append(batch_emb / norms)
        return np.vstack(embeddings)

    def encode_text(self, text: str) -> np.ndarray:
        """Encode a text string into a 512-dim normalized vector (for cross-modal queries)."""
        inputs = self.processor(text=[text], return_tensors="pt", padding=True).to(self.device)
        with torch.no_grad():
            features = self.model.get_text_features(**inputs)
        embedding = features.cpu().numpy()[0]
        norm = np.linalg.norm(embedding)
        if norm > 0:
            embedding = embedding / norm
        return embedding

    def encode_texts(self, texts: list[str], batch_size: int = 32) -> np.ndarray:
        """Encode multiple text strings. Returns (N, 512) array."""
        embeddings = []
        for i in range(0, len(texts), batch_size):
            batch = texts[i:i + batch_size]
            inputs = self.processor(text=batch, return_tensors="pt", padding=True).to(self.device)
            with torch.no_grad():
                features = self.model.get_text_features(**inputs)
            batch_emb = features.cpu().numpy()
            norms = np.linalg.norm(batch_emb, axis=1, keepdims=True)
            norms[norms == 0] = 1
            embeddings.append(batch_emb / norms)
        return np.vstack(embeddings)
```

### 7.9 Multimodal Retriever (`src/retrieval/multimodal_retriever.py`)

```python
from dataclasses import dataclass, field
import asyncpg
import numpy as np
from src.ingestion.clip_embedder import CLIPEmbedder
from src.ingestion.text_embedder import TextEmbedder

@dataclass
class MultimodalResult:
    chunk_id: str | None = None
    image_id: str | None = None
    content: str = ""
    image_path: str | None = None
    caption: str | None = None
    score: float = 0.0
    modality: str = "text"  # "text", "image", "both"
    metadata: dict = field(default_factory=dict)

class MultimodalRetriever:
    """Cross-modal retrieval using CLIP embeddings + BGE text embeddings."""

    def __init__(
        self,
        pool: asyncpg.Pool,
        clip_embedder: CLIPEmbedder,
        text_embedder: TextEmbedder,
        text_weight: float = 0.6,
        image_weight: float = 0.4,
    ):
        self.pool = pool
        self.clip = clip_embedder
        self.text_embedder = text_embedder
        self.text_weight = text_weight
        self.image_weight = image_weight

    async def search_by_text(self, query: str, k: int = 10) -> list[MultimodalResult]:
        """Search for both text chunks and images using a text query.

        Uses BGE embedding for text chunk search and CLIP text encoding for image search.
        Fuses results with configurable weights.
        """
        # Encode query with both embedders
        text_embedding = self.text_embedder.embed_query(query)
        clip_text_embedding = self.clip.encode_text(query)

        # Search text chunks (BGE, 1024-dim)
        text_results = await self._search_chunks(text_embedding, k=k)

        # Search images (CLIP, 512-dim)
        image_results = await self._search_images_by_clip(clip_text_embedding, k=k)

        # Also search images by caption (BGE, 1024-dim) — for text-based image search
        caption_results = await self._search_images_by_caption(text_embedding, k=k)

        # Merge image results (CLIP + caption)
        image_merged = self._merge_image_results(image_results, caption_results)

        # Fuse text + image results
        fused = self._fuse_results(text_results, image_merged, k=k)
        return fused

    async def search_by_image(self, image_path: str, k: int = 10) -> list[MultimodalResult]:
        """Search for similar images and related text using an image query."""
        clip_image_embedding = self.clip.encode_image(image_path)

        # Search images by CLIP embedding
        image_results = await self._search_images_by_clip(clip_image_embedding, k=k)

        # Search text chunks by caption embedding (find chunks that describe similar images)
        # Use CLIP text encoding of a generated caption as bridge
        # For simplicity, search images and return their context chunks
        chunk_ids = [r.get("context_chunk_id") for r in image_results if r.get("context_chunk_id")]
        text_results = []
        if chunk_ids:
            text_results = await self._get_chunks_by_ids(chunk_ids, k=k)

        fused = self._fuse_results(text_results, image_results, k=k)
        return fused

    async def _search_chunks(self, embedding: np.ndarray, k: int = 10) -> list[dict]:
        """Search text chunks by BGE embedding using pgvector cosine distance."""
        vec_str = self._format_vector(embedding)
        async with self.pool.acquire() as conn:
            rows = await conn.fetch("""
                SELECT id, content, 1 - (embedding <=> $1) as score
                FROM chunks
                WHERE embedding IS NOT NULL
                ORDER BY embedding <=> $1
                LIMIT $2
            """, vec_str, k)
            return [{"chunk_id": str(r["id"]), "content": r["content"], "score": float(r["score"])} for r in rows]

    async def _search_images_by_clip(self, embedding: np.ndarray, k: int = 10) -> list[dict]:
        """Search images by CLIP embedding using pgvector cosine distance."""
        vec_str = self._format_vector(embedding)
        async with self.pool.acquire() as conn:
            rows = await conn.fetch("""
                SELECT id, image_path, caption, image_type, context_text, context_chunk_id,
                       1 - (clip_embedding <=> $1) as score
                FROM images
                WHERE clip_embedding IS NOT NULL
                ORDER BY clip_embedding <=> $1
                LIMIT $2
            """, vec_str, k)
            return [{"image_id": str(r["id"]), "image_path": r["image_path"],
                     "caption": r["caption"], "image_type": r["image_type"],
                     "context_text": r["context_text"],
                     "context_chunk_id": str(r["context_chunk_id"]) if r["context_chunk_id"] else None,
                     "score": float(r["score"])} for r in rows]

    async def _search_images_by_caption(self, embedding: np.ndarray, k: int = 10) -> list[dict]:
        """Search images by caption BGE embedding (text-based image search)."""
        vec_str = self._format_vector(embedding)
        async with self.pool.acquire() as conn:
            rows = await conn.fetch("""
                SELECT id, image_path, caption, image_type, context_text, context_chunk_id,
                       1 - (caption_embedding <=> $1) as score
                FROM images
                WHERE caption_embedding IS NOT NULL
                ORDER BY caption_embedding <=> $1
                LIMIT $2
            """, vec_str, k)
            return [{"image_id": str(r["id"]), "image_path": r["image_path"],
                     "caption": r["caption"], "image_type": r["image_type"],
                     "context_text": r["context_text"],
                     "context_chunk_id": str(r["context_chunk_id"]) if r["context_chunk_id"] else None,
                     "score": float(r["score"])} for r in rows]

    async def _get_chunks_by_ids(self, chunk_ids: list[str], k: int = 10) -> list[dict]:
        """Get text chunks by their IDs."""
        async with self.pool.acquire() as conn:
            rows = await conn.fetch("""
                SELECT id, content
                FROM chunks
                WHERE id = ANY($1::uuid[])
                LIMIT $2
            """, chunk_ids, k)
            return [{"chunk_id": str(r["id"]), "content": r["content"], "score": 0.5} for r in rows]

    def _merge_image_results(self, clip_results: list[dict], caption_results: list[dict]) -> list[dict]:
        """Merge CLIP and caption image search results, deduplicating by image_id."""
        merged = {}
        for r in clip_results:
            merged[r["image_id"]] = r
        for r in caption_results:
            if r["image_id"] in merged:
                # Take max score
                merged[r["image_id"]]["score"] = max(merged[r["image_id"]]["score"], r["score"])
            else:
                merged[r["image_id"]] = r
        return sorted(merged.values(), key=lambda x: x["score"], reverse=True)

    def _fuse_results(self, text_results: list[dict], image_results: list[dict], k: int = 10) -> list[MultimodalResult]:
        """Fuse text and image results with configurable weights."""
        fused = []
        for r in text_results[:k]:
            fused.append(MultimodalResult(
                chunk_id=r.get("chunk_id"),
                content=r.get("content", ""),
                score=r["score"] * self.text_weight,
                modality="text",
            ))
        for r in image_results[:k]:
            fused.append(MultimodalResult(
                image_id=r.get("image_id"),
                image_path=r.get("image_path"),
                caption=r.get("caption"),
                content=r.get("context_text", "") or r.get("caption", ""),
                score=r["score"] * self.image_weight,
                modality="image",
                metadata={"image_type": r.get("image_type")},
            ))
        fused.sort(key=lambda x: x.score, reverse=True)
        return fused[:k]

    @staticmethod
    def _format_vector(vec: np.ndarray) -> str:
        """Convert numpy array to pgvector string format."""
        return "[" + ",".join(f"{x:.8f}" for x in vec) + "]"
```

### 7.10 Decision Engine (`src/framework/decision_engine.py`)

```python
from dataclasses import dataclass
from src.framework.query_analyzer import QueryProfile

@dataclass
class ArchitectureRecommendation:
    architecture: str        # "vanilla", "graph", "agentic", "multimodal", "kag", "lightrag"
    confidence: float        # 0.0 - 1.0
    reasoning: str
    alternatives: list[dict]  # [{architecture, confidence, reasoning}]

class DecisionEngine:
    """RAG architecture decision engine based on query and corpus characteristics."""

    def recommend(self, profile: QueryProfile, corpus_metadata: dict = None) -> ArchitectureRecommendation:
        """Recommend the best RAG architecture based on query profile."""
        scores = {
            "vanilla": 0.0,
            "graph": 0.0,
            "agentic": 0.0,
            "multimodal": 0.0,
            "kag": 0.0,
            "lightrag": 0.0,
        }
        reasoning = []

        # Criterion 1: Modality
        if profile.is_multimodal:
            scores["multimodal"] += 0.4
            reasoning.append("Query involves visual content → Multimodal RAG preferred")

        # Criterion 2: Entity density + relationship complexity
        if profile.entity_density > 0.5:
            scores["graph"] += 0.2
            scores["lightrag"] += 0.1
            reasoning.append(f"High entity density ({profile.entity_density:.1f}) → Graph-based approaches favored")

        if profile.relationship_complexity > 0.6:
            scores["graph"] += 0.25
            reasoning.append(f"High relationship complexity ({profile.relationship_complexity:.1f}) → GraphRAG strongly favored")

        # Criterion 3: Query complexity
        if profile.query_complexity > 0.7:
            scores["agentic"] += 0.3
            reasoning.append(f"High query complexity ({profile.query_complexity:.1f}) → Agentic RAG for iterative retrieval")
        elif profile.query_complexity < 0.3:
            scores["vanilla"] += 0.3
            reasoning.append(f"Low query complexity ({profile.query_complexity:.1f}) → Vanilla RAG sufficient")

        # Criterion 4: Structured KB availability
        if profile.has_structured_kb:
            scores["kag"] += 0.3
            reasoning.append("Structured knowledge base available → KAG can leverage formal reasoning")

        # Criterion 5: Latency budget
        if profile.latency_budget_ms < 500:
            scores["vanilla"] += 0.15
            scores["lightrag"] += 0.1
            scores["agentic"] -= 0.1
            reasoning.append("Low latency budget → prefer fast architectures (vanilla, LightRAG)")
        elif profile.latency_budget_ms > 3000:
            scores["agentic"] += 0.1
            scores["graph"] += 0.05
            reasoning.append("High latency budget → can afford complex architectures (agentic, graph)")

        # Criterion 6: Corpus size
        corpus_size = (corpus_metadata or {}).get("chunk_count", 1000)
        if corpus_size > 100000:
            scores["lightrag"] += 0.1
            scores["vanilla"] += 0.05
            reasoning.append("Large corpus → LightRAG's dual-level retrieval scales well")

        # Select best architecture
        best = max(scores, key=scores.get)
        best_score = scores[best]

        # Build alternatives
        alternatives = []
        for arch, score in sorted(scores.items(), key=lambda x: x[1], reverse=True)[1:3]:
            if score > 0.1:
                alternatives.append({"architecture": arch, "confidence": score, "reasoning": "Secondary option"})

        # Normalize confidence
        total = sum(scores.values())
        confidence = best_score / total if total > 0 else 0.0

        return ArchitectureRecommendation(
            architecture=best,
            confidence=confidence,
            reasoning="; ".join(reasoning),
            alternatives=alternatives,
        )
```

### 7.11 Query Analyzer (`src/framework/query_analyzer.py`)

```python
from dataclasses import dataclass
import re

@dataclass
class QueryProfile:
    query: str
    is_multimodal: bool = False          # asks about images/charts/visual content
    entity_density: float = 0.0          # 0.0 - 1.0, how entity-rich the query is
    relationship_complexity: float = 0.0  # 0.0 - 1.0, how many hops needed
    query_complexity: float = 0.0         # 0.0 - 1.0, simple factual vs complex analytical
    temporal_aspect: bool = False         # involves time-based reasoning
    has_structured_kb: bool = False       # structured KB available for this domain
    latency_budget_ms: int = 2000         # expected latency budget
    estimated_entities: list[str] = None  # detected entity names

class QueryAnalyzer:
    """Analyze query characteristics for architecture decision."""

    MULTIMODAL_KEYWORDS = {"image", "chart", "diagram", "figure", "table", "plot", "visual", "picture", "graph visualization"}
    TEMPORAL_KEYWORDS = {"when", "before", "after", "timeline", "history", "evolved", "changed", "timeline"}
    COMPLEXITY_INDICATORS = {"compare", "contrast", "analyze", "why", "how does", "relationship between", "impact of", "consequence"}
    MULTI_HOP_INDICATORS = {"who.*developed.*that", "which.*used by.*to", "relationship between.*and", "connected to"}

    def analyze(self, query: str, has_structured_kb: bool = False, latency_budget_ms: int = 2000) -> QueryProfile:
        """Analyze a query and return its profile."""
        query_lower = query.lower()
        words = query_lower.split()

        # Multimodal detection
        is_multimodal = any(kw in query_lower for kw in self.MULTIMODAL_KEYWORDS)

        # Temporal detection
        temporal = any(kw in query_lower for kw in self.TEMPORAL_KEYWORDS)

        # Entity density estimation (simple heuristic: capitalized words / total words)
        capitalized = len([w for w in query.split() if w[0].isupper() and w.isalpha()])
        entity_density = min(capitalized / max(len(words), 1), 1.0)

        # Relationship complexity (multi-hop indicators)
        multi_hop_matches = sum(1 for pattern in self.MULTI_HOP_INDICATORS if re.search(pattern, query_lower))
        relationship_complexity = min(multi_hop_matches * 0.3 + (entity_density * 0.3), 1.0)

        # Query complexity
        complexity_matches = sum(1 for indicator in self.COMPLEXITY_INDICATORS if indicator in query_lower)
        query_complexity = min(complexity_matches * 0.25 + (len(words) / 30), 1.0)

        # Estimated entities (capitalized words, 3+ chars)
        estimated_entities = [w.strip(".,;:!?") for w in query.split() if len(w) >= 3 and w[0].isupper()]

        return QueryProfile(
            query=query,
            is_multimodal=is_multimodal,
            entity_density=entity_density,
            relationship_complexity=relationship_complexity,
            query_complexity=query_complexity,
            temporal_aspect=temporal,
            has_structured_kb=has_structured_kb,
            latency_budget_ms=latency_budget_ms,
            estimated_entities=estimated_entities,
        )
```

### 7.12 Retrieval Metrics (`src/evaluation/metrics.py`)

Reused from Project 1. Implements `RetrievalMetrics` class with `recall_at_k`, `precision_at_k`, `mrr`, and `ndcg_at_k` as static methods.

```python
import math
from typing import Sequence

class RetrievalMetrics:
    """Retrieval quality metrics — reused from Project 1."""

    @staticmethod
    def recall_at_k(retrieved_ids: Sequence[str], relevant_ids: Sequence[str], k: int) -> float:
        if not relevant_ids:
            return 0.0
        retrieved_set = set(retrieved_ids[:k])
        relevant_set = set(relevant_ids)
        return len(retrieved_set & relevant_set) / len(relevant_set)

    @staticmethod
    def precision_at_k(retrieved_ids: Sequence[str], relevant_ids: Sequence[str], k: int) -> float:
        if k == 0:
            return 0.0
        retrieved_set = set(retrieved_ids[:k])
        relevant_set = set(relevant_ids)
        return len(retrieved_set & relevant_set) / k

    @staticmethod
    def mrr(retrieved_ids: Sequence[str], relevant_ids: Sequence[str]) -> float:
        relevant_set = set(relevant_ids)
        for i, rid in enumerate(retrieved_ids, 1):
            if rid in relevant_set:
                return 1.0 / i
        return 0.0

    @staticmethod
    def ndcg_at_k(retrieved_ids: Sequence[str], relevance_grades: dict, k: int) -> float:
        dcg = 0.0
        for i, rid in enumerate(retrieved_ids[:k], 1):
            grade = relevance_grades.get(rid, 0)
            dcg += (2 ** grade - 1) / math.log2(i + 1)
        ideal_grades = sorted(relevance_grades.values(), reverse=True)[:k]
        idcg = sum((2 ** g - 1) / math.log2(i + 1) for i, g in enumerate(ideal_grades, 1))
        return dcg / idcg if idcg > 0 else 0.0
```

### 7.13 Benchmark Runner (`src/evaluation/benchmark_runner.py`)

```python
import asyncio
import json
import time
from dataclasses import dataclass, asdict
from src.evaluation.metrics import RetrievalMetrics

@dataclass
class BenchmarkResult:
    question_id: str
    question: str
    architecture: str
    retrieved_ids: list[str]
    final_answer: str
    recall_at_5: float
    mrr: float
    ndcg_at_5: float
    answer_score: float
    latency_ms: int
    iterations: int

class BenchmarkRunner:
    """Run all architectures on golden questions and compute metrics."""

    def __init__(self, architectures: dict, golden_questions: list[dict], metrics_calculator=None):
        self.architectures = architectures  # {"vanilla": vanilla_rag, "graph": graph_rag, ...}
        self.golden_questions = golden_questions
        self.metrics = metrics_calculator or RetrievalMetrics

    async def run_all(self) -> list[BenchmarkResult]:
        """Run all architectures on all golden questions."""
        results = []
        for q in self.golden_questions:
            for arch_name, arch_impl in self.architectures.items():
                result = await self._run_single(q, arch_name, arch_impl)
                results.append(result)
        return results

    async def _run_single(self, question: dict, arch_name: str, arch_impl) -> BenchmarkResult:
        """Run a single architecture on a single question."""
        start = time.time()
        answer, retrieved_ids, iterations = await arch_impl.query(question["question"])
        latency_ms = int((time.time() - start) * 1000)

        relevant_ids = question.get("relevant_chunk_ids", [])
        relevance_grades = question.get("relevance_grades", {})

        recall = self.metrics.recall_at_k(retrieved_ids, relevant_ids, 5)
        mrr_score = self.metrics.mrr(retrieved_ids, relevant_ids)
        ndcg = self.metrics.ndcg_at_k(retrieved_ids, relevance_grades, 5)

        # Answer score: simple LLM judge or string similarity
        answer_score = self._score_answer(answer, question.get("expected_answer", ""))

        return BenchmarkResult(
            question_id=question["id"],
            question=question["question"],
            architecture=arch_name,
            retrieved_ids=retrieved_ids,
            final_answer=answer,
            recall_at_5=recall,
            mrr=mrr_score,
            ndcg_at_5=ndcg,
            answer_score=answer_score,
            latency_ms=latency_ms,
            iterations=iterations,
        )

    def _score_answer(self, answer: str, expected: str) -> float:
        """Simple answer scoring: token overlap (could be replaced with LLM judge)."""
        answer_tokens = set(answer.lower().split())
        expected_tokens = set(expected.lower().split())
        if not expected_tokens:
            return 0.0
        return len(answer_tokens & expected_tokens) / len(expected_tokens)

    def save_results(self, results: list[BenchmarkResult], path: str):
        """Save benchmark results to JSON."""
        with open(path, "w") as f:
            json.dump([asdict(r) for r in results], f, indent=2)
```

---

## 8. RAG Architecture Decision Framework

This is the **key deliverable** of the project — a structured framework for choosing the right RAG architecture based on query and corpus characteristics.

### 8.1 Decision Matrix

| Criterion | Vanilla RAG | GraphRAG | Agentic RAG | Multimodal RAG | KAG | LightRAG |
|-----------|-------------|----------|-------------|----------------|-----|----------|
| **Data Type** | Text only | Text (entity-rich) | Text (any) | Text + Images | Structured KB + Text | Text (any) |
| **Query Complexity** | Low (factual) | Medium-High (multi-hop) | High (multi-step) | Medium | High (logical reasoning) | Medium |
| **Relationship Density** | Low | High | Any | Low-Medium | High | Medium |
| **Modality** | Text | Text | Text | Text + Image | Text + Structured | Text |
| **Latency Budget** | <200ms | 200-1000ms | 1000-5000ms | 200-500ms | 500-2000ms | 200-500ms |
| **Corpus Size** | Any | <1M entities | Any | <100K images | Any | <10M chunks |
| **Setup Complexity** | Low | High | Medium | Medium | High | Medium |
| **Maintenance** | Low (re-index) | High (re-extract + re-detect) | Low | Medium (re-embed images) | High (KB updates) | Medium |
| **Multi-hop Reasoning** | Poor | Excellent | Good | Poor | Excellent | Good |
| **Factual Lookup** | Excellent | Good | Good | Good | Good | Excellent |
| **Temporal Reasoning** | Poor | Fair | Good | Poor | Good | Fair |
| **Cross-modal Retrieval** | N/A | N/A | N/A | Excellent | N/A | N/A |
| **Iterative Refinement** | No | No | Yes | No | No | No |
| **Structured Reasoning** | No | Partial | No | No | Yes | Partial |
| **Scalability** | High | Medium | Medium | Medium | High | High |
| **Cost per Query** | Low | Medium | High (multiple LLM calls) | Medium | Medium | Low-Medium |

### 8.2 Decision Criteria Scoring

Each criterion is scored 0.0–1.0 based on query and corpus analysis:

| Score | Meaning | Example |
|-------|---------|---------|
| 0.0–0.2 | Not relevant | No entities in query → entity_density = 0.1 |
| 0.2–0.4 | Low relevance | Simple factual query → query_complexity = 0.2 |
| 0.4–0.6 | Medium relevance | Some entities, some relationships → entity_density = 0.5 |
| 0.6–0.8 | High relevance | Multi-hop query with many entities → relationship_complexity = 0.7 |
| 0.8–1.0 | Critical | Query explicitly about images → is_multimodal = 1.0 |

### 8.3 Decision Rules (If-Then)

```
RULE 1: If is_multimodal == True
        → Recommend MULTIMODAL RAG
        (confidence: high — no other architecture handles images)

RULE 2: If entity_density > 0.5 AND relationship_complexity > 0.6
        → Recommend GRAPHRAG
        (confidence: high — graph traversal excels at multi-hop entity queries)

RULE 3: If query_complexity > 0.7 AND relationship_complexity < 0.5
        → Recommend AGENTIC RAG
        (confidence: medium — iterative retrieval handles complex queries without graph)

RULE 4: If query_complexity > 0.7 AND relationship_complexity > 0.6
        → Recommend AGENTIC RAG with GRAPH retrieval
        (confidence: high — combines iterative reasoning with graph traversal)

RULE 5: If has_structured_kb == True AND query requires logical reasoning
        → Recommend KAG
        (confidence: high — structured KB + LLM reasoning beats vector similarity)

RULE 6: If query_complexity < 0.3 AND latency_budget < 500ms
        → Recommend VANILLA RAG
        (confidence: high — simple factual queries don't need complex architectures)

RULE 7: If entity_density > 0.3 AND corpus_size > 100K chunks
        → Recommend LIGHTRAG
        (confidence: medium — dual-level retrieval scales better than full GraphRAG)

RULE 8: If temporal_aspect == True AND has_structured_kb == False
        → Recommend AGENTIC RAG
        (confidence: medium — iterative retrieval can handle temporal reasoning)

FALLBACK: → Recommend VANILLA RAG
          (confidence: low — default to simplest architecture)
```

### 8.4 Architecture Comparison Summary

| Architecture | Best For | Key Strength | Key Weakness | When to Avoid |
|-------------|----------|--------------|--------------|---------------|
| **Vanilla RAG** | Simple factual Q&A | Fast, simple, well-understood | Fails on multi-hop, no reasoning | Multi-hop queries, entity relationships |
| **GraphRAG** | Multi-hop entity reasoning | Graph traversal finds connected info | High setup cost, re-extraction needed | Simple factual queries, no entities |
| **Agentic RAG** | Complex multi-step queries | LLM decides retrieval strategy | High latency, multiple LLM calls | Simple queries, low latency budget |
| **Multimodal RAG** | Image + text retrieval | Cross-modal search | Limited to CLIP's understanding | Text-only queries, no images in corpus |
| **KAG** | Structured domain reasoning | Formal logic + KB + LLM | Requires structured KB, high maintenance | Unstructured text, no KB available |
| **LightRAG** | Lightweight graph + vector | Fast, scalable, dual-level | Less powerful than full GraphRAG | Complex multi-hop requiring full graph |

### 8.5 KAG / LightRAG Overview

#### KAG (Knowledge-Augmented Generation)

KAG is an architecture that combines a **structured knowledge base** (formal logic, rules, ontologies) with LLM generation. Unlike vector RAG which retrieves by similarity, KAG retrieves by **logical reasoning** over the knowledge base.

**Key concepts**:
- **Knowledge Base**: Formal representation of domain knowledge (OWL ontologies, rule bases, knowledge graphs with typed relations)
- **Logical Reasoning**: Deductive, inductive, or abductive reasoning over the KB to find answers
- **LLM Augmentation**: LLM generates natural language from the logically-derived answer, or fills gaps when the KB is incomplete
- **Verification**: The LLM's output can be verified against the KB for consistency

**When KAG beats vector RAG**:
- Queries requiring **precise logical inference** (e.g., "If A is a subclass of B, and B has property C, does A have property C?")
- Queries requiring **rule application** (e.g., "According to regulation X, is company Y compliant?")
- Queries where **exact matching** matters more than similarity (e.g., "What is the exact definition of X?")
- Domains with **well-defined ontologies** (medical, legal, financial)

**When KAG is worse than vector RAG**:
- Unstructured text without formal representation
- Open-ended exploratory queries
- Domains where the KB is incomplete or outdated
- Rapidly changing knowledge (KB maintenance is expensive)

#### LightRAG

LightRAG is a **lightweight graph-enhanced RAG** that combines vector search with graph traversal without the full overhead of GraphRAG. It uses a **dual-level retrieval** paradigm:

**Key concepts**:
- **Dual-level retrieval**: Low-level (specific entity retrieval) + High-level (broader topic retrieval via graph communities)
- **Incremental graph construction**: Graph is built incrementally as new documents are added, no full rebuild needed
- **Lightweight**: No community detection or summarization required — uses entity co-occurrence and relationship types directly
- **Hybrid search**: Combines vector similarity (pgvector) with graph traversal (entity relationships)

**When LightRAG beats GraphRAG**:
- When **setup cost matters** — LightRAG is much faster to set up (no community detection)
- When **incremental updates** are needed — LightRAG supports incremental graph construction
- When **corpus is large** — LightRAG's dual-level retrieval scales better than full graph traversal
- When **latency is critical** — LightRAG is faster than GraphRAG (no community summary generation)

**When LightRAG is worse than GraphRAG**:
- When **community-level reasoning** is needed — LightRAG lacks community summaries
- When **deep multi-hop** queries are common — LightRAG's traversal is shallower
- When **graph structure is complex** — LightRAG's lightweight approach may miss nuanced relationships

### 8.6 KAG / LightRAG / GraphRAG / Vanilla RAG Comparison Table

| Feature | Vanilla RAG | GraphRAG | LightRAG | KAG |
|---------|-------------|----------|----------|-----|
| **Knowledge Representation** | Vector embeddings | Graph (entities + relationships) | Graph (lightweight) + vectors | Structured KB (ontology + rules) |
| **Indexing** | Embed + store in vector DB | Extract entities → build graph → community detection | Extract entities → build graph (no communities) | Build ontology + rule base + vector index |
| **Retrieval** | Cosine similarity | Entity linking → graph traversal → community summaries | Dual-level: vector + graph traversal | Logical reasoning + vector fallback |
| **Generation** | LLM with retrieved chunks | LLM with subgraph + community summaries | LLM with graph context + vector chunks | LLM with logically-derived answer + KB context |
| **Multi-hop Reasoning** | Poor (1-hop only) | Excellent (N-hop traversal) | Good (2-3 hop) | Excellent (logical inference) |
| **Setup Complexity** | Low | High | Medium | High |
| **Incremental Updates** | Easy (re-embed new docs) | Hard (re-extract + re-detect) | Easy (incremental graph) | Hard (update ontology + rules) |
| **Latency** | <100ms | 200-1000ms | 100-300ms | 500-2000ms |
| **Scalability** | High (vector DB scales) | Medium (graph DB limits) | High (hybrid approach) | Medium (KB reasoning is expensive) |
| **Best Use Case** | FAQ, document search | Knowledge graph Q&A, entity relationships | Large-scale document Q&A with entity awareness | Compliance, legal, medical reasoning |
| **Example Query** | "What is RAG?" | "Who are all the people who worked on GPT-4?" | "What technologies does OpenAI use?" | "Is this drug interaction safe per FDA guidelines?" |
| **Strength** | Simplicity, speed | Deep relationship reasoning | Balance of speed and graph awareness | Precise logical inference |
| **Weakness** | No reasoning, no relationships | High setup + maintenance cost | Shallower than full GraphRAG | Requires structured KB, expensive to maintain |

---

## 9. Comparison & Evaluation

### 9.1 Benchmark Methodology

The benchmark runs all 4 implemented architectures (vanilla, graph, agentic, multimodal) on the same golden question set and computes:
- **Retrieval metrics**: recall@5, MRR, nDCG@5
- **Answer quality**: LLM-judge score (0–1) or token overlap with expected answer
- **Latency**: end-to-end query latency in milliseconds
- **Efficiency**: number of LLM calls (especially for agentic), iterations

### 9.2 Golden Question Categories

| Category | Count | Expected Best Architecture | Example |
|----------|-------|---------------------------|---------|
| Factual | 5 | Vanilla RAG | "What is the Transformer architecture?" |
| Multi-hop | 5 | GraphRAG | "Which organizations contributed to both BERT and GPT?" |
| Comparative | 3 | GraphRAG / Agentic | "Compare the attention mechanisms in Transformer and Linformer" |
| Multimodal | 3 | Multimodal RAG | "What does the architecture diagram in the Transformer paper look like?" |
| Agentic | 4 | Agentic RAG | "What are the key differences between RAG approaches and how do they compare?" |

### 9.3 Expected Benchmark Results

| Architecture | Factual (recall@5) | Multi-hop (recall@5) | Comparative (recall@5) | Multimodal (recall@5) | Avg Latency |
|-------------|--------------------|-----------------------|------------------------|------------------------|-------------|
| Vanilla RAG | 0.85+ | 0.30-0.50 | 0.40-0.60 | N/A | <100ms |
| GraphRAG | 0.70-0.85 | 0.75+ | 0.70+ | N/A | 200-500ms |
| Agentic RAG | 0.75-0.90 | 0.65-0.80 | 0.70-0.85 | N/A | 1000-3000ms |
| Multimodal RAG | 0.70-0.85 | N/A | N/A | 0.65+ | 200-400ms |

### 9.4 Evaluation Dashboard

The Streamlit dashboard displays:
- **Bar charts**: recall@5, MRR, nDCG by architecture and question type
- **Latency comparison**: box plots of latency by architecture
- **Per-question drill-down**: which chunks/entities/images were retrieved
- **Agentic trace**: iteration-by-iteration breakdown of the agentic loop
- **Decision framework validation**: did the recommended architecture actually perform best?

---

## 10. API Specification

### 10.1 Endpoints

| Method | Path | Description | Request Body | Response |
|--------|------|-------------|--------------|----------|
| POST | `/query` | Query with specific architecture | `QueryRequest` | `QueryResponse` |
| POST | `/query/compare` | Run query across all architectures | `CompareRequest` | `CompareResponse` |
| POST | `/decision` | Get architecture recommendation | `DecisionRequest` | `DecisionResponse` |
| GET | `/graph/entities` | List entities in Neo4j | `?limit=50` | `EntityListResponse` |
| GET | `/graph/subgraph` | Get subgraph around entity | `?entity_name=X&depth=2` | `SubgraphResponse` |
| GET | `/images/{image_id}` | Serve extracted image | — | image file |
| GET | `/benchmark` | Get benchmark results | — | `BenchmarkResponse` |
| GET | `/health` | Health check | — | `HealthResponse` |

### 10.2 Request/Response Schemas

```python
from pydantic import BaseModel, Field
from typing import Optional

class QueryRequest(BaseModel):
    query: str
    architecture: str = Field(default="vanilla", pattern="^(vanilla|graph|agentic|multimodal)$")
    top_k: int = Field(default=5, ge=1, le=50)
    max_iterations: int = Field(default=5, ge=1, le=10)  # agentic only
    config: dict = Field(default_factory=dict)

class RetrievedChunk(BaseModel):
    chunk_id: str
    content: str
    score: float
    source: str = "text"

class RetrievedImage(BaseModel):
    image_id: str
    image_path: str
    caption: Optional[str] = None
    image_type: str = "image"
    score: float

class RetrievalTraceStep(BaseModel):
    iteration: int
    sub_query: str
    architecture_used: str
    tokens: list[str] = []
    retrieved_count: int
    reasoning: str

class QueryResponse(BaseModel):
    query: str
    architecture: str
    answer: str
    retrieved_chunks: list[RetrievedChunk] = []
    retrieved_images: list[RetrievedImage] = []
    retrieved_entities: list[dict] = []
    retrieval_trace: list[RetrievalTraceStep] = []
    latency_ms: int
    iterations: int = 1

class CompareRequest(BaseModel):
    query: str
    architectures: list[str] = Field(default=["vanilla", "graph", "agentic", "multimodal"])
    top_k: int = 5

class CompareResponse(BaseModel):
    query: str
    results: dict[str, QueryResponse]  # architecture -> response

class DecisionRequest(BaseModel):
    query: str
    has_structured_kb: bool = False
    latency_budget_ms: int = 2000

class DecisionResponse(BaseModel):
    recommended_architecture: str
    confidence: float
    reasoning: str
    alternatives: list[dict] = []
    query_profile: dict

class HealthResponse(BaseModel):
    status: str
    postgres: str
    neo4j: str
```

### 10.3 Example API Calls

```bash
# Query with GraphRAG
curl -X POST http://localhost:8000/query \
  -H "Content-Type: application/json" \
  -d '{"query": "Who developed the Transformer and what company do they work for?", "architecture": "graph"}'

# Compare all architectures
curl -X POST http://localhost:8000/query/compare \
  -H "Content-Type: application/json" \
  -d '{"query": "Compare GPT-4 and Claude 3"}'

# Get architecture recommendation
curl -X POST http://localhost:8000/decision \
  -H "Content-Type: application/json" \
  -d '{"query": "What does the architecture diagram in the Transformer paper show?"}'

# Get subgraph for visualization
curl "http://localhost:8000/graph/subgraph?entity_name=Transformer&depth=2"
```

---

## 11. Security & Safety Considerations

| Concern | Mitigation |
|---------|-----------|
| **LLM prompt injection via entity extraction** | Validate extracted entity names/types; reject entries with control characters or excessive length |
| **Neo4j Cypher injection** | Use parameterized queries exclusively; never interpolate user input into Cypher strings |
| **Image path traversal** | Validate image paths are within the `data/images/` directory; reject `../` sequences |
| **LLM API key exposure** | Keys stored in `.env` (gitignored); never logged or returned in API responses |
| **Large corpus ingestion DoS** | Rate-limit ingestion; cap batch sizes; timeout on LLM extraction calls |
| **Agentic loop runaway** | Max 5 iterations enforced; total LLM call budget per query; timeout on each iteration |
| **Neo4j credential exposure** | Neo4j password in `.env` (gitignored); Docker internal network only |
| **Image content safety** | CLIP embeddings don't expose image content; images served only via authenticated API |

---

## 12. Testing Strategy

### 12.1 Test Pyramid

| Level | Count | What | Tools |
|-------|-------|------|-------|
| Unit | 25+ | Individual components (chunker, embedder, entity extractor, CLIP, Self-RAG parser, decision engine, metrics) | pytest, mocks |
| Integration | 5+ | Component interactions (graph builder → Neo4j, multimodal retriever → pgvector, API → all architectures) | pytest, Docker Compose |
| End-to-end | 3+ | Full pipeline (ingest → build graph → query → benchmark) | pytest, Docker Compose |

### 12.2 Test Files and Cases

#### `tests/test_chunker.py`
- `test_semantic_chunking_respects_sentence_boundaries`
- `test_chunking_strategy_label`
- `test_empty_text`
- `test_short_text`

#### `tests/test_text_embedder.py`
- `test_embed_passages_returns_correct_dimension` (1024)
- `test_embed_query_returns_1d_array`
- `test_embed_query_prepends_instruction_prefix` (mock model)

#### `tests/test_clip_embedder.py`
- `test_encode_image_returns_512_dim` (mock CLIP model)
- `test_encode_text_returns_512_dim` (mock CLIP model)
- `test_embeddings_are_normalized` (unit vectors)
- `test_image_and_text_same_concept_have_high_similarity` (sanity check)

#### `tests/test_image_extractor.py`
- `test_extracts_images_from_pdf`
- `test_classifies_image_types`
- `test_extracts_surrounding_text_context`
- `test_handles_pdf_without_images`

#### `tests/test_entity_extractor.py`
- `test_extracts_entities_from_text` (mock LLM returns JSON)
- `test_extracts_relationships_from_text` (mock LLM returns JSON)
- `test_handles_malformed_llm_response` (JSON parse error fallback)
- `test_merges_duplicate_entities_across_chunks`

#### `tests/test_graph_builder.py`
- `test_creates_document_node` (mock Neo4j)
- `test_creates_entity_node_with_mentions` (mock Neo4j)
- `test_creates_relationship_between_entities` (mock Neo4j)
- `test_merges_duplicate_entities` (same name + type → mention_count increments)

#### `tests/test_community_detector.py`
- `test_run_leiden_writes_community_ids` (mock GDS)
- `test_summarize_community_uses_llm` (mock LLM)
- `test_creates_community_nodes` (mock Neo4j)

#### `tests/test_graph_retriever.py`
- `test_entity_linking_finds_entities` (mock Neo4j fulltext)
- `test_graph_traversal_finds_related_entities` (mock Neo4j)
- `test_community_lookup_returns_summaries` (mock Neo4j)
- `test_build_context_string` (unit test on string building)

#### `tests/test_vector_retriever.py`
- `test_vector_search_returns_top_k` (mock asyncpg pool)
- `test_vector_search_returns_correct_scores` (mock asyncpg pool)

#### `tests/test_multimodal_retriever.py`
- `test_text_query_returns_text_and_image_results` (mock pool + CLIP)
- `test_image_query_returns_similar_images` (mock pool + CLIP)
- `test_fusion_combines_results_with_weights` (unit test on fusion logic)
- `test_merge_image_results_deduplicates` (unit test on merge logic)

#### `tests/test_agentic_loop.py`
- `test_loop_terminates_on_generate_token` (mock LLM returns [GENERATE])
- `test_loop_continues_on_no_generate` (mock LLM returns [RETRIEVE])
- `test_max_iterations_enforced` (mock LLM always returns [RETRIEVE])
- `test_context_accumulates_across_iterations`
- `test_deduplication_skips_already_retrieved_chunks`
- `test_terminates_on_no_new_relevant_chunks`

#### `tests/test_self_rag.py`
- `test_parse_retrieve_token`
- `test_parse_generate_token`
- `test_parse_relevant_irrelevant_tokens`
- `test_fallback_when_no_tokens` (defaults to retrieve)
- `test_parse_sub_query`

#### `tests/test_decision_engine.py`
- `test_multimodal_query_recommends_multimodal`
- `test_multi_hop_query_recommends_graph`
- `test_simple_factual_recommends_vanilla`
- `test_complex_analytical_recommends_agentic`
- `test_structured_kb_recommends_kag`
- `test_low_latency_prefers_fast_architectures`

#### `tests/test_metrics.py`
- `test_recall_at_k_perfect`
- `test_recall_at_k_partial`
- `test_precision_at_k`
- `test_mrr_first_result`
- `test_ndcg_perfect_ranking`
- `test_ndcg_partial_ranking`

#### `tests/test_api.py`
- `test_health_check` (all services connected)
- `test_query_vanilla` (POST /query with architecture=vanilla)
- `test_query_graph` (POST /query with architecture=graph)
- `test_query_agentic` (POST /query with architecture=agentic)
- `test_query_multimodal` (POST /query with architecture=multimodal)
- `test_compare_all_architectures` (POST /query/compare)
- `test_decision_endpoint` (POST /decision)

### 12.3 Test Fixtures (`tests/conftest.py`)

```python
import pytest
import asyncio
from unittest.mock import AsyncMock, MagicMock

@pytest.fixture
def mock_llm():
    """Mock LLM client returning canned responses."""
    llm = AsyncMock()
    llm.generate = AsyncMock(return_value='{"entities": [{"name": "Test", "type": "CONCEPT", "description": "Test entity"}], "relationships": []}')
    return llm

@pytest.fixture
def mock_neo4j_client():
    """Mock Neo4j client."""
    client = AsyncMock()
    client.execute = AsyncMock(return_value=[{"name": "Test", "entity_type": "CONCEPT"}])
    client.execute_write = AsyncMock(return_value={"nodes_created": 1, "relationships_created": 0})
    client.health_check = AsyncMock(return_value=True)
    return client

@pytest.fixture
def mock_db_pool():
    """Mock asyncpg pool."""
    pool = MagicMock()
    conn = AsyncMock()
    conn.fetch = AsyncMock(return_value=[{"id": "test-id", "content": "test content", "score": 0.9}])
    pool.acquire.return_value.__aenter__ = AsyncMock(return_value=conn)
    pool.acquire.return_value.__aexit__ = AsyncMock(return_value=None)
    return pool

@pytest.fixture
def sample_chunks():
    """Sample chunk data for tests."""
    return [
        {"chunk_id": "chunk-1", "content": "The Transformer architecture was proposed by Google researchers.", "chunk_index": 0},
        {"chunk_id": "chunk-2", "content": "OpenAI developed GPT-4, a large language model.", "chunk_index": 1},
    ]

@pytest.fixture
def sample_entities():
    """Sample extracted entities for tests."""
    return [
        {"name": "Transformer", "entity_type": "TECH", "description": "Neural network architecture"},
        {"name": "Google", "entity_type": "ORG", "description": "Technology company"},
        {"name": "GPT-4", "entity_type": "TECH", "description": "Large language model by OpenAI"},
    ]
```

### 12.4 Coverage Targets

| Module | Target Coverage |
|--------|----------------|
| `src/agentic/` | 90% (critical logic: loop, termination, dedup) |
| `src/graph/` | 80% (graph construction + retrieval) |
| `src/framework/` | 90% (decision engine logic) |
| `src/ingestion/` | 75% (embedders, extractors) |
| `src/retrieval/` | 80% (all retrievers) |
| `src/evaluation/` | 85% (metrics, benchmark) |
| `src/api/` | 70% (integration tests) |
| **Overall** | **80%+** |

---

## 13. Deployment

### 13.1 Docker Compose

```yaml
version: "3.9"

services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_USER: rag
      POSTGRES_PASSWORD: ragpassword
      POSTGRES_DB: ragdb
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./sql/schema.sql:/docker-entrypoint-initdb.d/schema.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U rag -d ragdb"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - rag-network

  neo4j:
    image: neo4j:5.11-community
    environment:
      NEO4J_AUTH: neo4j/testpassword
      NEO4J_PLUGINS: '["gds"]'
      NEO4J_dbms_security_procedures_unrestricted: gds.*
      NEO4J_dbms_memory_heap_max__size: 1G
    ports:
      - "7474:7474"   # Neo4j Browser
      - "7687:7687"   # Bolt protocol
    volumes:
      - neo4j_data:/data
      - neo4j_logs:/logs
      - ./sql/neo4j/constraints.cypher:/var/lib/neo4j/import/constraints.cypher:ro
    healthcheck:
      test: ["CMD-SHELL", "wget -q -O - http://localhost:7474 || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 10
    networks:
      - rag-network

  api:
    build: .
    environment:
      DATABASE_URL: postgresql://rag:ragpassword@postgres:5432/ragdb
      NEO4J_URI: bolt://neo4j:7687
      NEO4J_USER: neo4j
      NEO4J_PASSWORD: testpassword
      LLM_PROVIDER: openai
      OPENAI_API_KEY: ${OPENAI_API_KEY:-}
      OPENAI_MODEL: gpt-4o-mini
      OLLAMA_BASE_URL: http://host.docker.internal:11434
      OLLAMA_MODEL: llama3.1:8b
      EMBEDDING_MODEL: BAAI/bge-large-en-v1.5
      CLIP_MODEL: openai/clip-vit-base-patch32
      DEVICE: cpu
      API_HOST: 0.0.0.0
      API_PORT: 8000
      CORS_ORIGINS: '["http://localhost:8501"]'
    ports:
      - "8000:8000"
    depends_on:
      postgres:
        condition: service_healthy
      neo4j:
        condition: service_healthy
    volumes:
      - ./data:/app/data
      - model_cache:/root/.cache/huggingface
    networks:
      - rag-network

  dashboard:
    build: .
    command: streamlit run src/dashboard/app.py --server.port=8501 --server.address=0.0.0.0
    environment:
      DATABASE_URL: postgresql://rag:ragpassword@postgres:5432/ragdb
      NEO4J_URI: bolt://neo4j:7687
      NEO4J_USER: neo4j
      NEO4J_PASSWORD: testpassword
      API_URL: http://api:8000
      LLM_PROVIDER: openai
      OPENAI_API_KEY: ${OPENAI_API_KEY:-}
      OPENAI_MODEL: gpt-4o-mini
      EMBEDDING_MODEL: BAAI/bge-large-en-v1.5
      CLIP_MODEL: openai/clip-vit-base-patch32
      DEVICE: cpu
    ports:
      - "8501:8501"
    depends_on:
      api:
        condition: service_started
    volumes:
      - ./data:/app/data
      - model_cache:/root/.cache/huggingface
    networks:
      - rag-network

volumes:
  postgres_data:
  neo4j_data:
  neo4j_logs:
  model_cache:

networks:
  rag-network:
    driver: bridge
```

### 13.2 Dockerfile

```dockerfile
FROM python:3.11-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    libgl1-mesa-glx \
    libglib2.0-0 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY pyproject.toml .
RUN pip install --no-cache-dir -e ".[api,dashboard,dev]"

# Download models at build time (optional — can be done at runtime)
RUN python -c "from transformers import CLIPModel, CLIPProcessor; CLIPModel.from_pretrained('openai/clip-vit-base-patch32'); CLIPProcessor.from_pretrained('openai/clip-vit-base-patch32')" || true
RUN python -c "from sentence_transformers import SentenceTransformer; SentenceTransformer('BAAI/bge-large-en-v1.5')" || true
RUN python -m spacy download en_core_web_sm || true

COPY . .

EXPOSE 8000

CMD ["uvicorn", "src.api.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 13.3 .env.example

```env
# PostgreSQL
DATABASE_URL=postgresql://rag:ragpassword@localhost:5432/ragdb
TEST_DATABASE_URL=postgresql://rag:ragpassword@localhost:5432/ragdb_test

# Neo4j
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=testpassword
NEO4J_DATABASE=neo4j

# LLM
LLM_PROVIDER=openai
OPENAI_API_KEY=your-api-key-here
OPENAI_MODEL=gpt-4o-mini
OPENAI_BASE_URL=https://api.openai.com/v1
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_MODEL=llama3.1:8b

# Embedding Models
EMBEDDING_MODEL=BAAI/bge-large-en-v1.5
CLIP_MODEL=openai/clip-vit-base-patch32
DEVICE=cpu

# Retrieval Defaults
DEFAULT_TOP_K=5
DEFAULT_MAX_HOPS=2
DEFAULT_MAX_ITERATIONS=5
TEXT_WEIGHT=0.6
IMAGE_WEIGHT=0.4

# API
API_HOST=0.0.0.0
API_PORT=8000
CORS_ORIGINS=["http://localhost:8501"]

# Dashboard
API_URL=http://localhost:8000
```

### 13.4 pyproject.toml

```toml
[build-system]
requires = ["setuptools>=68.0", "wheel"]
build-backend = "setuptools.backends._legacy:_Backend"

[project]
name = "advanced-rag-architectures"
version = "0.1.0"
description = "Advanced RAG Architectures: GraphRAG, Agentic RAG, Multimodal RAG & Decision Framework"
requires-python = ">=3.11"
dependencies = [
    "fastapi>=0.110",
    "uvicorn[standard]>=0.27",
    "pydantic>=2.6",
    "pydantic-settings>=2.1",
    "asyncpg>=0.29",
    "pgvector>=0.3",
    "sentence-transformers>=2.5",
    "transformers>=4.40",
    "torch>=2.2",
    "Pillow>=10.0",
    "PyMuPDF>=1.24",
    "spacy>=3.7",
    "neo4j>=5.11",
    "langchain>=0.2",
    "httpx>=0.27",
    "numpy>=1.26",
    "python-dotenv>=1.0",
]

[project.optional-dependencies]
api = [
    "uvicorn[standard]>=0.27",
    "python-multipart>=0.0.9",
]
dashboard = [
    "streamlit>=1.35",
    "streamlit-agraph>=0.0.45",
    "plotly>=5.20",
]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
    "pytest-cov>=4.1",
    "ruff>=0.3",
    "mypy>=1.8",
]

[tool.setuptools.packages.find]
where = ["."]
include = ["src*"]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]

[tool.ruff]
line-length = 120
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "W"]

[tool.mypy]
python_version = "3.11"
warn_return_any = true
warn_unused_configs = true
ignore_missing_imports = true
```

### 13.5 Deployment Steps

```bash
# 1. Clone and enter project
cd advanced-rag-architectures/

# 2. Copy environment file
cp .env.example .env
# Edit .env with your OPENAI_API_KEY (or use Ollama)

# 3. Start all services
docker compose up -d

# 4. Verify health
curl http://localhost:8000/health
# → {"status": "healthy", "postgres": "connected", "neo4j": "connected"}

# 5. Initialize Neo4j constraints
python scripts/init_neo4j.py

# 6. Ingest documents
python scripts/ingest.py --source data/raw/ --extract-images

# 7. Build knowledge graph
python scripts/build_graph.py

# 8. Run community detection
python scripts/run_communities.py

# 9. Run benchmarks
python scripts/run_benchmark.py

# 10. Open dashboard
open http://localhost:8501

# 11. Open API docs
open http://localhost:8000/docs

# 12. Open Neo4j Browser
open http://localhost:7474
```

---

## 14. Roadmap & Milestones

| Milestone | Day | Deliverable | Success Criterion |
|-----------|-----|-------------|------------------|
| M1 | Day 1 | Environment + schemas | Docker Compose up, both DBs initialized |
| M2 | Day 3 | GraphRAG graph built | Neo4j has entities, relationships, chunks |
| M3 | Day 4 | GraphRAG retrieval + benchmark | Graph RAG outperforms vanilla on multi-hop |
| M4 | Day 6 | Agentic RAG loop | Iterative retrieval with Self-RAG tokens |
| M5 | Day 7 | Multimodal RAG | CLIP cross-modal retrieval operational |
| M6 | Day 8 | Decision framework | Correctly routes golden questions |
| M7 | Day 9 | API + Dashboard | All endpoints + 6 dashboard pages |
| M8 | Day 10 | Testing + docs | Full test suite passing, README complete |

### Future Enhancements (Post-Project)

| Enhancement | Priority | Effort | Description |
|-------------|----------|--------|-------------|
| KAG full implementation | Medium | 2 weeks | Build a structured KB + logical reasoning engine |
| LightRAG implementation | Medium | 1 week | Implement dual-level retrieval as a lightweight alternative |
| Video modality | Low | 2 weeks | Add video frame extraction + CLIP embeddings |
| Fine-tuned CLIP | Low | 1 week | Fine-tune CLIP on domain-specific images |
| GraphRAG with LLM-judge eval | Medium | 3 days | Use LLM as judge for answer quality scoring |
| Streaming responses | Low | 2 days | SSE streaming for agentic retrieval trace |
| Multi-tenant graph isolation | Medium | 1 week | Extend P2 tenant isolation to Neo4j |
| Production Neo4j cluster | Low | 1 week | Neo4j Cluster setup for >1M entities |

---

## 15. Risk Register

| # | Risk | Likelihood | Impact | Mitigation |
|---|------|-----------|--------|-----------|
| R1 | Neo4j GDS plugin not available in Docker Community Edition | Medium | High | Use `NEO4J_PLUGINS: '["gds"]'` in Docker Compose; fallback to manual community detection (connected components via Cypher) |
| R2 | LLM entity extraction quality too low for useful graph | Medium | High | Use gpt-4o-mini (not Ollama) for extraction; validate entities post-extraction; manual review for small corpus |
| R3 | CLIP model too large for Docker image | Low | Medium | Download at build time; use `clip-vit-base-patch32` (smallest variant); cache in volume |
| R4 | Agentic loop never terminates | Low | High | Max 5 iterations hard limit; no-new-chunks termination; per-iteration timeout |
| R5 | Neo4j and Postgres data inconsistency | Medium | Medium | Single ingestion pipeline writes to both; Postgres tracks Neo4j node IDs for cross-reference |
| R6 | PDF image extraction fails on certain PDFs | Medium | Low | Fallback: render full page as image; log failures; skip problematic PDFs |
| R7 | Graph retrieval slower than expected | Medium | Medium | Limit traversal depth (2 hops); limit results (15 entities); cache common queries |
| R8 | Decision framework misroutes queries | Medium | Medium | Validate against golden questions; tune scoring weights; allow manual override |
| R9 | Docker Compose resource usage too high | Medium | Medium | Neo4j heap limit 1G; use CPU mode for models; separate dashboard container |
| R10 | LLM API costs for agentic loop (multiple calls per query) | Medium | Medium | Use gpt-4o-mini (cheapest); cap iterations at 5; cache common sub-queries |

---

## 16. Appendix

### 16.1 Neo4j Cypher Cheat Sheet

```cypher
// === CREATE ===
CREATE (n:Label {property: "value"})
CREATE (a:Entity {name: "GPT-4"})-[:RELATED_TO {type: "DEVELOPED_BY"}]->(b:Entity {name: "OpenAI"})

// === MERGE (create if not exists) ===
MERGE (e:Entity {name: "Transformer", entity_type: "TECH"})
SET e.description = "Neural network architecture based on attention"

// === MATCH (read) ===
MATCH (e:Entity) RETURN e
MATCH (e:Entity {name: "Transformer"}) RETURN e
MATCH (e:Entity) WHERE e.entity_type = "TECH" RETURN e

// === RELATIONSHIPS ===
MATCH (a:Entity {name: "Google"})-[:RELATED_TO]->(b:Entity)
RETURN a.name, b.name

MATCH (a)-[r:RELATED_TO]->(b)
WHERE r.type = "DEVELOPED_BY"
RETURN a.name, b.name

// === PATH TRAVERSAL (variable length) ===
MATCH path = (e:Entity {name: "Transformer"})-[:RELATED_TO*1..3]-(related)
RETURN path

// === AGGREGATION ===
MATCH (e:Entity)
RETURN e.entity_type, count(e) as count
ORDER BY count DESC

// === FULLTEXT SEARCH ===
CALL db.index.fulltext.queryNodes('entity_fulltext', 'transformer attention')
YIELD node, score
RETURN node.name, score
ORDER BY score DESC

// === GDS: GRAPH PROJECTION ===
CALL gds.graph.project('myGraph', ['Entity'], {RELATED_TO: {orientation: 'UNDIRECTED'}})

// === GDS: LEIDEN COMMUNITY DETECTION ===
CALL gds.leiden.write('myGraph', {writeProperty: 'communityId'})
YIELD communityCount, modularity

// === GDS: PAGERANK ===
CALL gds.pageRank.write('myGraph', {writeProperty: 'pagerank'})

// === DELETE ===
MATCH (n) DETACH DELETE n  // Delete all nodes and relationships (use with caution!)
MATCH (e:Entity {name: "Test"}) DETACH DELETE e

// === DROP GRAPH PROJECTION ===
CALL gds.graph.drop('myGraph')

// === INDEXES ===
CREATE INDEX entity_name IF NOT EXISTS FOR (e:Entity) ON (e.name)
CREATE FULLTEXT INDEX entity_fulltext IF NOT EXISTS FOR (e:Entity) ON EACH [e.name, e.description]

// === CONSTRAINTS ===
CREATE CONSTRAINT entity_unique IF NOT EXISTS FOR (e:Entity) REQUIRE (e.name, e.entity_type) IS UNIQUE
```

### 16.2 CLIP Setup Guide

```python
# Install dependencies
# pip install transformers torch Pillow

from transformers import CLIPModel, CLIPProcessor
from PIL import Image
import torch

# Load model and processor
model = CLIPModel.from_pretrained("openai/clip-vit-base-patch32")
processor = CLIPProcessor.from_pretrained("openai/clip-vit-base-patch32")
model.eval()

# Encode an image
image = Image.open("path/to/image.png").convert("RGB")
inputs = processor(images=image, return_tensors="pt")
with torch.no_grad():
    image_features = model.get_image_features(**inputs)
# image_features shape: (1, 512)

# Encode text
inputs = processor(text=["a diagram of a neural network"], return_tensors="pt", padding=True)
with torch.no_grad():
    text_features = model.get_text_features(**inputs)
# text_features shape: (1, 512)

# Compute similarity (both are in the same embedding space)
image_features = image_features / image_features.norm(dim=-1, keepdim=True)
text_features = text_features / text_features.norm(dim=-1, keepdim=True)
similarity = (image_features @ text_features.T).item()
# similarity is a float between -1 and 1
```

### 16.3 Self-RAG Reflection Tokens Reference

| Token | Meaning | When Used |
|-------|---------|-----------|
| `[RETRIEVE]` | Retrieve more information | Context insufficient, need more chunks |
| `[NO RETRIEVE]` | Do not retrieve | Context is sufficient or query is self-contained |
| `[RELEVANT]` | Retrieved chunk is relevant | After assessing a retrieved chunk |
| `[IRRELEVANT]` | Retrieved chunk is not relevant | After assessing a retrieved chunk |
| `[GENERATE]` | Generate the final answer | Context is sufficient to answer |
| `[NO GENERATE]` | Do not generate yet | Need more iterations before answering |

### 16.4 Entity Types and Relationship Types

**Entity Types**:
| Type | Description | Example |
|------|-------------|---------|
| PERSON | A specific person | "Sam Altman", "Ashish Vaswani" |
| ORG | An organization/company | "OpenAI", "Google", "Microsoft" |
| TECH | A technology or method | "Transformer", "GPT-4", "BERT" |
| CONCEPT | An abstract concept | "Attention Mechanism", "Transfer Learning" |
| LOCATION | A geographic location | "San Francisco", "United States" |
| EVENT | A specific event | "NeurIPS 2017", "GPT-4 Launch" |
| PRODUCT | A specific product | "ChatGPT", "Claude 3" |

**Relationship Types**:
| Type | Description | Example |
|------|-------------|---------|
| DEVELOPED_BY | Entity A was developed by entity B | Transformer DEVELOPED_BY Google |
| WORKS_FOR | Person A works for organization B | Sam Altman WORKS_FOR OpenAI |
| COMPETES_WITH | Entity A competes with entity B | OpenAI COMPETES_WITH Anthropic |
| PART_OF | Entity A is part of entity B | GPT-4 PART_OF OpenAI |
| USES | Entity A uses entity B | GPT-4 USES Transformer |
| RELATED_TO | Generic relationship (fallback) | Any entities that are related |
| FOUNDED_BY | Organization A was founded by person B | OpenAI FOUNDED_BY Sam Altman |
| LOCATED_IN | Entity A is located in location B | OpenAI LOCATED_IN San Francisco |
| CREATED | Entity A created entity B | Google CREATED Transformer |
| MEMBER_OF | Person A is a member of organization B | Researcher MEMBER_OF Google |

### 16.5 Comparison Table: All RAG Architectures

| Feature | Vanilla RAG | GraphRAG | Agentic RAG | Multimodal RAG | KAG | LightRAG |
|---------|-------------|----------|-------------|----------------|-----|----------|
| **Retrieval Method** | Vector similarity | Graph traversal + community | LLM-directed iterative | Cross-modal CLIP | Logical reasoning + vector | Dual-level vector + graph |
| **Indexing** | Embed chunks | Extract entities → graph → communities | Same as vanilla/graph | Embed text + images | Build KB + embed | Extract entities → graph |
| **LLM Calls per Query** | 1 (generation) | 1 (generation) | 2-6 (plan + retrieve + generate) | 1 (generation) | 1-2 (reasoning + generation) | 1 (generation) |
| **Setup Time** | Minutes | Hours (extraction + graph build) | Minutes (uses existing indexes) | Minutes (embed images) | Days (KB construction) | Minutes-Hours |
| **Maintenance** | Low | High (re-extract on updates) | Low | Medium (re-embed images) | High (KB updates) | Medium |
| **Best Query Type** | "What is X?" | "Who is connected to X?" | "Compare X and Y across dimensions" | "What does the diagram show?" | "Is X compliant with rule Y?" | "What technologies does X use?" |
| **Worst Query Type** | Multi-hop reasoning | Simple factual | Simple factual | Text-only reasoning | Unstructured exploratory | Deep multi-hop |
| **Corpus Requirement** | Any text | Entity-rich text | Any text | Text + images | Structured KB + text | Any text |
| **Latency** | <100ms | 200-1000ms | 1000-5000ms | 200-500ms | 500-2000ms | 100-300ms |
| **Implementation Complexity** | Low | High | Medium | Medium | High | Medium |

### 16.6 Glossary

| Term | Definition |
|------|-----------|
| **GraphRAG** | RAG approach that uses a knowledge graph (entities + relationships) for retrieval instead of (or in addition to) vector similarity |
| **Agentic RAG** | RAG approach where an LLM agent decides what to retrieve, when, and how many times, using an iterative retrieval loop |
| **Self-RAG** | A training approach for LLMs that teaches them to generate reflection tokens ([RETRIEVE], [RELEVANT], [GENERATE]) to self-assess retrieval needs |
| **Multimodal RAG** | RAG approach that handles multiple modalities (text, images, audio, video) by embedding them in a shared or comparable vector space |
| **CLIP** | Contrastive Language-Image Pre-training; a model that encodes images and text into a shared 512-dim embedding space |
| **KAG** | Knowledge-Augmented Generation; combines structured knowledge bases (ontologies, rules) with LLM generation for precise logical reasoning |
| **LightRAG** | Lightweight graph-enhanced RAG that combines vector search with graph traversal without full GraphRAG overhead |
| **Community Detection** | Algorithm (e.g., Leiden) that groups graph nodes into communities based on connection density |
| **Entity Linking** | The process of matching entities mentioned in a query to entities in the knowledge graph |
| **Leiden Algorithm** | A community detection algorithm that finds groups of densely connected nodes in a graph |
| **Cross-modal Retrieval** | Retrieving results of one modality (e.g., images) using a query of another modality (e.g., text) |
| **Reflection Tokens** | Special tokens ([RETRIEVE], [GENERATE], etc.) that an LLM generates to indicate its retrieval/generation decisions |
| **Subgraph Extraction** | Extracting a portion of a knowledge graph around specific entities for use as LLM context |
| **pgvector** | PostgreSQL extension for vector similarity search, supporting HNSW and IVFFlat indexes |
| **Neo4j GDS** | Neo4j Graph Data Science library; provides graph algorithms like Leiden, PageRank, node2vec |
| **Cypher** | Neo4j's declarative query language for graph databases, similar to SQL but designed for graph patterns |
| **HNSW** | Hierarchical Navigable Small World; an approximate nearest neighbor index structure used by pgvector |
| **BM25** | Best Matching 25; a ranking function used in full-text search to estimate the relevance of documents |
| **nDCG** | Normalized Discounted Cumulative Gain; a metric that measures ranking quality with graded relevance |
| **MRR** | Mean Reciprocal Rank; the average of reciprocal ranks of the first relevant result |
