# PII Redaction Middleware for LLM Inputs/Outputs — Detailed Implementation Plan

> **Elevator pitch:** A drop-in FastAPI proxy middleware that intercepts every LLM API call, scans user prompts and model responses for personally identifiable information using a three-layer detection pipeline (regex → NER → LLM-contextual), redacts or masks PII before it reaches the model or the user, and writes every redaction event to an immutable GDPR/SOC 2 compliance audit log. Built specifically for the German market — with native patterns for Steuer-ID, German IBANs, phone formats, postal codes, and addresses — this is an AI safety component, not generic middleware: the LLM never sees raw PII, and the user never receives leaked PII.

---

## Table of Contents

1. [Project Goals & Non-Goals](#1-project-goals--non-goals)
2. [Architecture Overview](#2-architecture-overview)
3. [Tech Stack](#3-tech-stack)
4. [Database Schema](#4-database-schema)
5. [Project Structure](#5-project-structure)
6. [Implementation Phases](#6-implementation-phases)
7. [Component Specifications](#7-component-specifications)
8. [Security & Safety Model](#8-security--safety-model)
9. [API Specification](#9-api-specification)
10. [Testing Strategy](#10-testing-strategy)
11. [Deployment](#11-deployment)
12. [Roadmap & Milestones](#12-roadmap--milestones)
13. [Risk Register](#13-risk-register)
14. [Appendix A: Quick Start](#appendix-a-quick-start)
15. [Appendix B: Key Design Decisions](#appendix-b-key-design-decisions)
16. [Appendix C: Regex Pattern Catalog](#appendix-c-regex-pattern-catalog)
17. [Appendix D: Redaction Strategy Comparison](#appendix-d-redaction-strategy-comparison)
18. [Appendix E: GDPR Compliance Mapping](#appendix-e-gdpr-compliance-mapping)

---

## 1. Project Goals & Non-Goals

### Goals

| # | Goal | Success Metric |
|---|------|----------------|
| G1 | **Input PII never reaches the LLM** — every user prompt is scanned and redacted before forwarding to the model API | Zero raw PII strings in forwarded prompts across the full test corpus (automated + manual) |
| G2 | **Output PII never reaches the user** — every LLM response is scanned for leaked PII before returning | Zero PII leakage in returned responses across the full test corpus |
| G3 | **Three-layer detection** — regex, NER (Presidio + spaCy), and LLM-contextual — so no single layer's blind spot is the system's blind spot | Each layer independently catches ≥1 test case the other two miss; combined recall ≥ 98% on labeled PII corpus |
| G4 | **Configurable redaction strategies** — mask, hash, or synthetic replacement — per PII type and per tenant | Tenant config drives strategy selection; switching strategy changes output without code changes |
| G5 | **Per-tenant rules** — allowlist/denylist entities, allowed PII types, custom regex patterns per tenant | Tenant A can allow emails while Tenant B redacts them; allowlisted entities pass through unredacted |
| G6 | **Immutable compliance audit log** — every redaction event recorded with what was detected, type, action, timestamp, user, tenant | Audit log row per redaction event; UPDATE/DELETE blocked by DB trigger; queryable by tenant/time/type |
| G7 | **Drop-in proxy middleware** — wraps any OpenAI-compatible LLM API call without modifying the calling application | Point client `base_url` at the proxy; existing code works unchanged |
| G8 | **German-specific PII patterns** — Steuer-ID, German IBAN, German phone, PLZ, German address formats | All German PII test cases detected and redacted; `de_core_news_lg` model loaded for German NER |
| G9 | **Compliance dashboard** — Streamlit app showing redaction statistics, PII type distribution, compliance report export | Dashboard renders live stats from audit log; CSV/PDF report export works |
| G10 | **Production-deployable** — containerized, documented, testable end-to-end | `docker compose up` starts proxy + DB + dashboard; health check passes |

### Non-Goals (explicitly out of scope)

- Encrypting PII at rest with reversible decryption (v1 redacts; reversible vault is a v2 concern)
- Real-time streaming response redaction (v1 buffers complete responses; SSE streaming is a v2 concern)
- Fine-tuning custom NER models (v1 uses off-the-shelf Presidio + spaCy pipelines)
- Image/document PII redaction (v1 is text-only; OCR + image redaction is a separate project)
- Multi-language beyond English + German (v1 supports en + de; adding fr/es/it is a config extension)
- Replacing the LLM provider's own safety filters (this is a complement, not a replacement)
- Data loss prevention (DLP) for non-LLM traffic (v1 proxies only LLM API calls)
- Anonymization with differential privacy guarantees (v1 redacts; formal DP is a research concern)

---

## 2. Architecture Overview

### 2.1 System Architecture

```
                              ┌──────────────────────────────────────────────────┐
                              │            Client Application                    │
                              │  (sets base_url=http://proxy:8000/v1             │
                              │   instead of https://api.openai.com/v1)          │
                              └────────────────────┬─────────────────────────────┘
                                                   │ HTTPS
                              ┌────────────────────▼─────────────────────────────┐
                              │           FastAPI Proxy Middleware                │
                              │                                                   │
                              │  ┌──────────────┐    ┌──────────────────────┐    │
                              │  │  Auth +      │    │  Tenant Config       │    │
                              │  │  Tenant      │───▶│  Resolver            │    │
                              │  │  Resolver    │    │  (per-tenant rules)  │    │
                              │  └──────┬───────┘    └──────────┬───────────┘    │
                              │         │                       │                │
                              │  ┌──────▼───────────────────────▼───────────┐    │
                              │  │         INPUT SCANNER (pre-LLM)          │    │
                              │  │                                          │    │
                              │  │  ┌─────────┐  ┌────────┐  ┌───────────┐  │    │
                              │  │  │ Layer 1 │─▶│ Layer 2│─▶│  Layer 3  │  │    │
                              │  │  │ Regex   │  │ NER    │  │  LLM      │  │    │
                              │  │  │ Patterns│  │ Presidio│  │ Contextual│  │    │
                              │  │  └────┬────┘  └───┬────┘  └─────┬─────┘  │    │
                              │  │       └──────────┬┴─────────────┘        │    │
                              │  │              ┌───▼───┐                    │    │
                              │  │              │Merge +│                    │    │
                              │  │              │Dedupe  │                   │    │
                              │  │              └───┬───┘                    │    │
                              │  │             ┌────▼─────┐                  │    │
                              │  │             │ Redactor │                  │    │
                              │  │             │(mask/    │                  │    │
                              │  │             │hash/     │                  │    │
                              │  │             │synthetic)│                  │    │
                              │  │             └────┬─────┘                  │    │
                              │  └──────────────────┼────────────────────────┘    │
                              │                     │ redacted prompt             │
                              │  ┌──────────────────▼────────────────────────┐    │
                              │  │         LLM CLIENT (forward)              │    │
                              │  │  → calls upstream LLM API (OpenAI/Ollama) │    │
                              │  └──────────────────┬────────────────────────┘    │
                              │                     │ raw LLM response            │
                              │  ┌──────────────────▼────────────────────────┐    │
                              │  │         OUTPUT SCANNER (post-LLM)         │    │
                              │  │  (same 3-layer pipeline, output mode)     │    │
                              │  │  → blocks or sanitizes leaked PII         │    │
                              │  └──────────────────┬────────────────────────┘    │
                              │                     │ safe response               │
                              │  ┌──────────────────▼────────────────────────┐    │
                              │  │         COMPLIANCE LOGGER                 │    │
                              │  │  → every redaction event → audit_logs     │    │
                              │  │  → immutable (DB trigger blocks UPDATE/   │    │
                              │  │    DELETE)                                 │    │
                              │  └───────────────────────────────────────────┘    │
                              └────────────────────┬─────────────────────────────┘
                                                   │
              ┌────────────────────────────────────┼──────────────────────────────┐
              │                                    │                              │
     ┌────────▼─────────┐           ┌──────────────▼────────────┐      ┌──────────▼──────────┐
     │  PostgreSQL      │           │  Upstream LLM API         │      │  Streamlit          │
     │                  │           │  (OpenAI / Azure OpenAI   │      │  Dashboard          │
     │  Tables:         │           │   / Ollama / vLLM)        │      │                     │
     │  - redaction_    │           │                           │      │  - Redaction stats  │
     │    events        │           │  Receives REDACTED        │      │  - PII type dist.   │
     │  - tenants       │           │   prompts only            │      │  - Compliance       │
     │  - tenant_config │           │                           │      │    report export    │
     │  - allowlist     │           └───────────────────────────┘      └─────────────────────┘
     │  - audit_logs    │
     └──────────────────┘
```

### 2.2 Three-Layer Detection Architecture (Detail)

```
                         ┌─────────────────────────────────┐
                         │        Input Text               │
                         │  "My name is Anna Müller,       │
                         │   email: a.mueller@gmx.de,      │
                         │   Steuer-ID: 12345678901,       │
                         │   I live at Hauptstr. 42,       │
                         │   24103 Kiel"                   │
                         └──────────────┬──────────────────┘
                                        │
                    ┌───────────────────▼────────────────────┐
                    │          LAYER 1: REGEX                │
                    │  Fast, deterministic, zero ML cost     │
                    │  Catches: emails, phones, SSNs,        │
                    │   IBANs, credit cards, Steuer-ID,      │
                    │   passport, PLZ, German address        │
                    │                                        │
                    │  Matches:                              │
                    │   • a.mueller@gmx.de  → EMAIL          │
                    │   • 12345678901       → STEUER_ID       │
                    │   • 24103             → PLZ             │
                    └───────────────────┬────────────────────┘
                                        │
                    ┌───────────────────▼────────────────────┐
                    │          LAYER 2: NER                  │
                    │  Microsoft Presidio + spaCy            │
                    │  en_core_web_lg + de_core_news_lg      │
                    │  Catches: person names, orgs,          │
                    │   locations, dates, addresses          │
                    │                                        │
                    │  Matches:                              │
                    │   • Anna Müller       → PERSON          │
                    │   • Kiel              → LOCATION        │
                    │   • Hauptstr. 42      → ADDRESS (DE)    │
                    └───────────────────┬────────────────────┘
                                        │
                    ┌───────────────────▼────────────────────┐
                    │       LAYER 3: LLM CONTEXTUAL          │
                    │  Small, fast LLM (gpt-4o-mini)         │
                    │  Catches: implicit PII that regex      │
                    │   and NER miss — "my boss is           │
                    │   the CEO" (role→person), "I work      │
                    │   at the company that makes X"         │
                    │                                        │
                    │  Matches:                              │
                    │   • "I live at" context → confirms     │
                    │     ADDRESS even if NER was unsure     │
                    │   • Implicit employer → ORG            │
                    └───────────────────┬────────────────────┘
                                        │
                    ┌───────────────────▼────────────────────┐
                    │       MERGE + DEDUPLICATE              │
                    │  Combine all detections; resolve       │
                    │   overlapping spans; apply tenant      │
                    │   allowlist/denylist; assign final     │
                    │   PII type + confidence score          │
                    └───────────────────┬────────────────────┘
                                        │
                    ┌───────────────────▼────────────────────┐
                    │       REDACTION ENGINE                 │
                    │  For each detected entity:             │
                    │   1. Look up tenant config → strategy  │
                    │   2. Apply: mask / hash / synthetic    │
                    │   3. Record event → compliance log     │
                    │   4. Store mapping (original↔token)    │
                    │      for potential re-identification   │
                    └───────────────────┬────────────────────┘
                                        │
                         ┌──────────────▼──────────────┐
                         │   Redacted Text             │
                         │  "My name is [REDACTED_     │
                         │   PERSON], email: [REDACTED_│
                         │   EMAIL], Steuer-ID: [HASH_ │
                         │   a3f9...], I live at       │
                         │   [SYNTHETIC_ADDRESS],      │
                         │   [REDACTED_PLZ]"            │
                         └─────────────────────────────┘
```

### 2.3 Request Flow (Proxy Intercept)

```
1. Client sends POST /v1/chat/completions {messages: [...], model: "..."} + JWT
2. Auth middleware decodes JWT → {user_id, tenant_id, roles}
3. Tenant config resolver loads rules for tenant_id:
   a. Enabled PII types (e.g. tenant allows EMAIL but redacts SSN)
   b. Redaction strategy per PII type (mask / hash / synthetic)
   c. Allowlist entities (e.g. "support@company.de" is not PII)
   d. Custom regex patterns
4. Input scanner runs on each message content:
   a. Layer 1: Regex → list of (span, pii_type, pattern_name)
   b. Layer 2: NER (Presidio + spaCy, language auto-detected en/de) → list of (span, pii_type, score)
   c. Layer 3: LLM contextual (only if Layer 1+2 found < N entities or text has PII cues) → list of (span, pii_type, reasoning)
   d. Merge + deduplicate overlapping spans
   e. Apply allowlist: remove allowlisted entities from detection list
   f. Redact each entity per tenant strategy
   g. Log every redaction event to compliance log
5. Forward redacted request to upstream LLM API
6. Receive LLM response
7. Output scanner runs on response content:
   a. Same 3-layer pipeline in output mode
   b. If PII detected in output → sanitize (mask) or block response (configurable)
   c. Log output redaction events
8. Return safe response to client
9. Return metadata header: X-PII-Redacted: true, X-Redaction-Count: 7
```

---

## 3. Tech Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **Language** | Python 3.11+ | Ecosystem maturity for AI/ML, async support, type hints |
| **Web Framework** | FastAPI | Async middleware support, auto OpenAPI docs, Pydantic validation; ideal for proxy pattern |
| **PII Detection — Layer 1** | Python `regex` library (PCRE2) | More powerful than `re`; supports lookbehind, named groups, Unicode; needed for German character classes |
| **PII Detection — Layer 2** | Microsoft Presidio + spaCy | Presidio is the industry standard for PII detection; spaCy provides NER with `en_core_web_lg` + `de_core_news_lg` for German |
| **PII Detection — Layer 3** | `gpt-4o-mini` (OpenAI) or local Llama 3.1 8B via Ollama | Small, fast, cheap model for contextual detection; catches implicit PII that regex/NER miss |
| **Database** | PostgreSQL 16 | Compliance audit log, tenant config, allowlist storage; append-only triggers for immutability |
| **Dashboard** | Streamlit | Rapid internal tooling; charts + tables + report export with minimal code |
| **Containerization** | Docker + Docker Compose | Reproducible dev + prod; multi-service (proxy, DB, dashboard) |
| **Testing** | pytest + pytest-asyncio + httpx | Async test support, FastAPI-native test client |
| **Linting** | ruff | Fast, replaces flake8 + isort + black |
| **Hashing** | hashlib (SHA-256 + HMAC) | Deterministic, salted hashes for re-identification without exposing PII |
| **Synthetic Data** | Faker (with `de_DE` locale) | Realistic German fake names, addresses, phone numbers for synthetic redaction |

### Why Presidio + spaCy over a single approach?

**No single detection method is sufficient.** Regex is fast and deterministic but misses names and contextual PII. NER catches names and locations but misses format-based identifiers like IBANs and Steuer-IDs. LLM-contextual catches implicit PII ("my doctor is Dr. Schmidt" → PERSON) but is slow, costly, and can hallucinate. The three-layer pipeline ensures each layer compensates for the others' blind spots:

| Layer | Strengths | Weaknesses | Latency | Cost |
|-------|-----------|------------|---------|------|
| **Regex** | Exact formats (email, IBAN, SSN, Steuer-ID); deterministic; zero ML cost | Misses names, contextual PII; false positives on similar-looking strings | <1ms | Free |
| **NER (Presidio+spaCy)** | Person names, orgs, locations, dates; multilingual (en+de); contextual entity linking | Misses non-standard formats; can misclassify common words as entities | 10-50ms | Free (local model) |
| **LLM Contextual** | Implicit PII ("my boss" → PERSON); semantic understanding; catches novel patterns | Slow; costly per call; can hallucinate; needs prompt engineering | 200-800ms | $0.001-0.01/call |

The pipeline runs Layer 1 → Layer 2 always; Layer 3 runs only when the first two layers found fewer entities than expected given the text's PII cue density (heuristic: presence of words like "name", "address", "phone", "mein", "Adresse", "Telefon" with low detection count).

### Why a proxy middleware instead of a library?

**The proxy is the security boundary.** If PII redaction is a library that the calling application must invoke, a developer can forget to call it, call it in the wrong order, or bypass it under pressure. A proxy middleware sits between the client and the LLM API — there is no path to the LLM that doesn't go through the scanner. This is the same principle as a firewall: the security control is in the path of the traffic, not optional in the application.

---

## 4. Database Schema

### 4.1 Extensions

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";   -- for gen_random_uuid()
```

### 4.2 Tables

```sql
-- =====================================================
-- TENANTS: organizations using the PII guardrail proxy
-- =====================================================
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name            TEXT NOT NULL UNIQUE,             -- e.g. 'acme-corp'
    description     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- =====================================================
-- TENANT_CONFIG: per-tenant redaction rules
-- One row per tenant — JSONB columns for flexibility
-- =====================================================
CREATE TABLE tenant_config (
    tenant_id               UUID PRIMARY KEY REFERENCES tenants(id) ON DELETE CASCADE,
    -- PII types this tenant wants detected/redacted
    -- e.g. ["EMAIL", "PHONE", "PERSON", "STEUER_ID", "IBAN", "ADDRESS_DE"]
    enabled_pii_types       TEXT[] NOT NULL DEFAULT '{}',
    -- Strategy per PII type: {"EMAIL": "mask", "PERSON": "synthetic", "IBAN": "hash"}
    redaction_strategies    JSONB NOT NULL DEFAULT '{}',
    -- Whether Layer 3 (LLM contextual) is enabled for this tenant
    llm_contextual_enabled  BOOLEAN NOT NULL DEFAULT TRUE,
    -- Output mode: 'sanitize' (mask PII in response) or 'block' (return error)
    output_mode             TEXT NOT NULL DEFAULT 'sanitize'
        CHECK (output_mode IN ('sanitize', 'block')),
    -- Language preference: 'auto' (detect), 'en', 'de'
    language                TEXT NOT NULL DEFAULT 'auto',
    -- Custom regex patterns: [{"name": "employee_id", "pattern": "EMP-\\d{6}", "pii_type": "EMPLOYEE_ID"}]
    custom_regex_patterns   JSONB NOT NULL DEFAULT '[]',
    created_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- =====================================================
-- ALLOWLIST: entities that should NOT be redacted per tenant
-- e.g. "support@company.de" is a public email, not PII
-- e.g. "Max Mustermann" is a test persona, not real PII
-- =====================================================
CREATE TABLE allowlist (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    entity_value    TEXT NOT NULL,                     -- the literal string to allow
    pii_type        TEXT NOT NULL,                     -- which PII type it would match
    -- Match type: 'exact' (string equality) or 'regex' (pattern match)
    match_type      TEXT NOT NULL DEFAULT 'exact'
        CHECK (match_type IN ('exact', 'regex')),
    reason          TEXT,                              -- why is this allowlisted? (audit)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    UNIQUE(tenant_id, entity_value, pii_type)
);

CREATE INDEX idx_allowlist_tenant ON allowlist(tenant_id);
CREATE INDEX idx_allowlist_tenant_type ON allowlist(tenant_id, pii_type);

-- =====================================================
-- REDACTION_EVENTS: every PII detection + redaction action
-- This is the core compliance log. One row per entity redacted.
-- =====================================================
CREATE TABLE redaction_events (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    timestamp           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    user_id             TEXT NOT NULL,                 -- from JWT sub
    request_id          UUID NOT NULL,                 -- correlates all events in one request

    -- Direction: was this input (pre-LLM) or output (post-LLM)?
    direction           TEXT NOT NULL CHECK (direction IN ('input', 'output')),

    -- What was detected
    pii_type            TEXT NOT NULL,                 -- 'EMAIL', 'PERSON', 'STEUER_ID', etc.
    detection_layer     TEXT NOT NULL CHECK (detection_layer IN ('regex', 'ner', 'llm_contextual')),
    detection_score     REAL,                          -- confidence 0.0-1.0 (NER/LLM layers)
    detected_text       TEXT,                          -- the original PII text (for audit — see note)
    detected_span_start INT,                           -- character offset in original text
    detected_span_end   INT,

    -- What action was taken
    redaction_strategy  TEXT NOT NULL CHECK (redaction_strategy IN ('mask', 'hash', 'synthetic', 'block')),
    redacted_text       TEXT NOT NULL,                 -- the replacement text
    hash_value          TEXT,                          -- if strategy=hash, the HMAC-SHA256 value

    -- Context
    model_name          TEXT,                          -- which LLM was called (for input events)
    language            TEXT,                          -- detected language 'en'/'de'

    -- Append-only: enforced by trigger (see below)
    CONSTRAINT no_update CHECK (true)
);

CREATE INDEX idx_redaction_tenant_time ON redaction_events(tenant_id, timestamp DESC);
CREATE INDEX idx_redaction_user_time ON redaction_events(user_id, timestamp DESC);
CREATE INDEX idx_redaction_request ON redaction_events(request_id);
CREATE INDEX idx_redaction_type ON redaction_events(pii_type);
CREATE INDEX idx_redaction_direction ON redaction_events(direction);
CREATE INDEX idx_redaction_layer ON redaction_events(detection_layer);

-- Trigger: prevent UPDATE and DELETE on redaction_events
CREATE OR REPLACE FUNCTION prevent_redaction_modification()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'redaction_events is append-only; UPDATE and DELETE are forbidden';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER no_redaction_update
    BEFORE UPDATE ON redaction_events
    FOR EACH ROW EXECUTE FUNCTION prevent_redaction_modification();

CREATE TRIGGER no_redaction_delete
    BEFORE DELETE ON redaction_events
    FOR EACH ROW EXECUTE FUNCTION prevent_redaction_modification();

-- =====================================================
-- AUDIT_LOGS: high-level request-level audit trail
-- One row per proxied LLM API call (not per entity)
-- =====================================================
CREATE TABLE audit_logs (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    timestamp           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    user_id             TEXT NOT NULL,
    request_id          UUID NOT NULL UNIQUE,          -- correlates with redaction_events

    -- Request metadata
    endpoint            TEXT NOT NULL,                 -- '/v1/chat/completions'
    model_name          TEXT NOT NULL,                 -- 'gpt-4o-mini'
    direction           TEXT NOT NULL CHECK (direction IN ('proxy', 'blocked')),

    -- Redaction summary
    input_redaction_count   INT NOT NULL DEFAULT 0,
    output_redaction_count  INT NOT NULL DEFAULT 0,
    pii_types_detected      TEXT[] NOT NULL DEFAULT '{}',

    -- Performance
    input_scan_ms       INT,
    llm_call_ms         INT,
    output_scan_ms      INT,
    total_latency_ms    INT,

    -- Outcome
    success             BOOLEAN NOT NULL DEFAULT TRUE,
    error_message       TEXT,

    -- Append-only
    CONSTRAINT no_audit_update CHECK (true)
);

CREATE INDEX idx_audit_tenant_time ON audit_logs(tenant_id, timestamp DESC);
CREATE INDEX idx_audit_user_time ON audit_logs(user_id, timestamp DESC);
CREATE INDEX idx_audit_request ON audit_logs(request_id);

CREATE TRIGGER no_audit_update
    BEFORE UPDATE ON audit_logs
    FOR EACH ROW EXECUTE FUNCTION prevent_redaction_modification();

CREATE TRIGGER no_audit_delete
    BEFORE DELETE ON audit_logs
    FOR EACH ROW EXECUTE FUNCTION prevent_redaction_modification();
```

### 4.3 Design Note: Storing Detected PII in the Compliance Log

The `redaction_events.detected_text` column stores the **original PII text**. This is a deliberate decision for audit completeness — you need to know *what* was redacted to investigate incidents. However, this creates a tension: the compliance log itself contains PII.

**Mitigations:**
1. The database is encrypted at rest (PostgreSQL TDE or disk-level LUKS encryption).
2. Access to the compliance log is restricted to compliance officers (separate DB role).
3. The `detected_text` column can be encrypted at the application level using a column-level encryption key stored in a vault (v2 enhancement).
4. A retention policy auto-expires `redaction_events` rows after a configurable period (default: 2 years), replacing `detected_text` with `[EXPIRED]` while keeping all metadata.

```sql
-- Retention policy: expire detected_text after 2 years
-- (Run as a cron job; UPDATE is blocked by trigger, so this uses a dedicated
--  maintenance role with the trigger temporarily disabled for this operation)
-- CREATE OR REPLACE FUNCTION expire_old_redaction_text()
-- RETURNS void AS $$
-- BEGIN
--   ALTER TABLE redaction_events DISABLE TRIGGER no_redaction_update;
--   UPDATE redaction_events
--     SET detected_text = '[EXPIRED]'
--     WHERE timestamp < NOW() - INTERVAL '2 years'
--       AND detected_text != '[EXPIRED]';
--   ALTER TABLE redaction_events ENABLE TRIGGER no_redaction_update;
-- END;
-- $$ LANGUAGE plpgsql;
```

### 4.4 Row-Level Security (Defense in Depth)

```sql
-- Enable RLS on redaction_events and audit_logs
ALTER TABLE redaction_events ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_logs ENABLE ROW LEVEL SECURITY;

-- Policy: a tenant can only see its own redaction events
CREATE POLICY tenant_isolation_redaction ON redaction_events
    USING (tenant_id::text = current_setting('app.current_tenant_id', true));

CREATE POLICY tenant_isolation_audit ON audit_logs
    USING (tenant_id::text = current_setting('app.current_tenant_id', true));
```

---

## 5. Project Structure

```
pii-guardrails/
├── docker-compose.yml
├── .env.example
├── pyproject.toml
├── README.md
├── IMPLEMENTATION_PLAN.md          ← this file
├── agent.md                      # Herdr multi-agent orchestration guide
│
├── src/
│   ├── __init__.py
│   ├── main.py                     # FastAPI app factory + lifespan + middleware registration
│   ├── config.py                   # Pydantic Settings (env vars)
│   │
│   ├── auth/
│   │   ├── __init__.py
│   │   ├── jwt_handler.py          # JWT decode + verify
│   │   ├── models.py               # User, Tenant pydantic models
│   │   └── middleware.py           # AuthMiddleware: extract user + tenant from JWT
│   │
│   ├── db/
│   │   ├── __init__.py
│   │   ├── connection.py           # asyncpg pool
│   │   ├── schema.sql              # full DDL (tables, indexes, RLS, triggers)
│   │   └── migrations/
│   │       ├── 001_initial.sql
│   │       └── 002_rls.sql
│   │
│   ├── detection/
│   │   ├── __init__.py
│   │   ├── models.py               # PIIDetection, RedactionResult, PIIType enum
│   │   ├── regex_detector.py       # Layer 1: regex pattern matching
│   │   ├── ner_detector.py         # Layer 2: Presidio + spaCy NER
│   │   ├── llm_detector.py         # Layer 3: LLM-based contextual detection
│   │   ├── merger.py               # Merge + deduplicate detections from all layers
│   │   └── pipeline.py             # Orchestrates 3-layer detection pipeline
│   │
│   ├── redaction/
│   │   ├── __init__.py
│   │   ├── strategies.py           # Mask, Hash, Synthetic strategies
│   │   ├── engine.py               # Redaction engine: applies strategy per entity
│   │   └── synthetic_data.py       # Faker-based synthetic data generation (de_DE locale)
│   │
│   ├── scanner/
│   │   ├── __init__.py
│   │   ├── input_scanner.py        # Pre-LLM scan: detect + redact in user prompts
│   │   └── output_scanner.py       # Post-LLM scan: detect + sanitize in LLM responses
│   │
│   ├── tenant/
│   │   ├── __init__.py
│   │   ├── config_resolver.py      # Load tenant config, allowlist, custom patterns
│   │   └── models.py               # TenantConfig pydantic model
│   │
│   ├── compliance/
│   │   ├── __init__.py
│   │   ├── logger.py               # Append-only compliance log writer
│   │   └── report.py               # Compliance report generator (CSV/PDF export)
│   │
│   ├── proxy/
│   │   ├── __init__.py
│   │   ├── middleware.py           # PIIGuardrailMiddleware: the core proxy intercept
│   │   ├── llm_client.py           # Forward requests to upstream LLM API
│   │   └── language_detector.py    # Detect en/de for NER model selection
│   │
│   └── api/
│       ├── __init__.py
│       ├── routes/
│       │   ├── __init__.py
│       │   ├── proxy.py            # POST /v1/chat/completions, /v1/completions (proxy)
│       │   ├── admin.py            # GET /admin/tenants, POST /admin/tenants/{id}/config
│       │   ├── audit.py            # GET /audit/redactions, /audit/summary (admin only)
│       │   └── health.py           # GET /health
│       └── dependencies.py         # FastAPI dependency injection
│
├── dashboard/
│   ├── __init__.py
│   ├── app.py                      # Streamlit main app
│   ├── charts.py                   # PII type distribution, redaction trend charts
│   ├── reports.py                  # Compliance report export (CSV/PDF)
│   └── queries.py                  # SQL queries against redaction_events + audit_logs
│
├── patterns/
│   ├── regex_catalog.yaml          # All regex patterns (en + de) with metadata
│   └── german_patterns.yaml        # German-specific PII patterns (Steuer-ID, IBAN, etc.)
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py                 # pytest fixtures: test DB, test client, seed data
│   ├── test_regex_detector.py      # Layer 1: all regex patterns, German-specific
│   ├── test_ner_detector.py        # Layer 2: Presidio + spaCy en/de
│   ├── test_llm_detector.py        # Layer 3: LLM contextual detection
│   ├── test_merger.py              # Overlap resolution, dedup, confidence scoring
│   ├── test_redaction.py           # Mask, hash, synthetic strategies
│   ├── test_input_scanner.py       # Full input scan pipeline
│   ├── test_output_scanner.py      # Full output scan + sanitize/block
│   ├── test_tenant_config.py       # Per-tenant rules, allowlist, custom patterns
│   ├── test_compliance.py          # Audit log immutability, completeness
│   ├── test_proxy.py               # End-to-end proxy: client → scan → LLM → scan → client
│   ├── test_german_pii.py          # German-specific PII: Steuer-ID, IBAN, PLZ, phone, address
│   ├── test_security.py            # PII leakage tests (critical: no raw PII reaches LLM/user)
│   └── test_dashboard.py           # Dashboard data queries
│
├── scripts/
│   ├── init_db.py                  # Run schema.sql against fresh database
│   ├── seed_test_data.py           # Insert test tenants, configs, allowlists
│   ├── generate_jwt.py             # Helper to create test JWT tokens
│   ├── benchmark_detection.py      # Measure detection latency per layer
│   └── evaluate_recall.py          # Compute recall/precision on labeled PII corpus
│
└── docker/
    ├── Dockerfile                  # Proxy app image
    ├── Dockerfile.dashboard        # Streamlit dashboard image
    └── postgres/
        └── init.sql                # Extensions on first boot
```

---

## 6. Implementation Phases

### Phase 1: Foundation + Proxy Skeleton (Day 1)

**Goal:** Running Postgres + FastAPI proxy skeleton with auth and tenant resolution. Proxy forwards requests to upstream LLM without scanning (passthrough).

| Step | Task | Deliverable |
|------|------|-------------|
| 1.1 | Write `docker-compose.yml` with Postgres, proxy app, dashboard service | `docker compose up` starts all three |
| 1.2 | Write `docker/postgres/init.sql` (extensions) + `src/db/schema.sql` (full DDL) | DB initializes on first boot |
| 1.3 | Write `src/config.py` (Pydantic Settings) | Env-driven config |
| 1.4 | Write `src/db/connection.py` (asyncpg pool) | DB connection pool works |
| 1.5 | Write `src/auth/jwt_handler.py` + `middleware.py` | JWT decode, user + tenant extraction |
| 1.6 | Write `src/tenant/config_resolver.py` | Load tenant config from DB |
| 1.7 | Write `src/proxy/llm_client.py` (forward to upstream) | Passthrough proxy works |
| 1.8 | Write `src/proxy/middleware.py` (skeleton: auth → forward → return) | Proxy forwards requests |
| 1.9 | Write `src/main.py` (FastAPI app factory) | App starts, health check responds |
| 1.10 | Write `scripts/generate_jwt.py` + `scripts/seed_test_data.py` | Can create test tokens + seed tenants |
| 1.11 | Write `tests/test_auth.py` + `tests/test_proxy.py` (passthrough) | Auth + passthrough tests pass |

**Verification:** `docker compose up` → send a chat completion request via proxy → get response from upstream LLM. `curl localhost:8000/health` → 200 OK. Auth tests green.

---

### Phase 2: Layer 1 — Regex Detection + German Patterns (Day 2)

**Goal:** Regex-based PII detection for all pattern types including German-specific patterns. No redaction yet — just detection.

| Step | Task | Deliverable |
|------|------|-------------|
| 2.1 | Write `patterns/regex_catalog.yaml` (all patterns with metadata) | Catalog with 15+ pattern types |
| 2.2 | Write `patterns/german_patterns.yaml` (Steuer-ID, IBAN-DE, phone-DE, PLZ, address-DE) | German-specific patterns |
| 2.3 | Write `src/detection/models.py` (PIIDetection, PIIType enum, RedactionResult) | Data models |
| 2.4 | Write `src/detection/regex_detector.py` (load YAML, match, return detections) | Regex detector works |
| 2.5 | Write `tests/test_regex_detector.py` (all patterns, edge cases, German-specific) | Regex tests pass |
| 2.6 | Write `tests/test_german_pii.py` (Steuer-ID, IBAN, PLZ, phone, address) | German PII detection tests pass |

**Verification:** Feed `"My email is test@example.com and my Steuer-ID is 12345678901"` → detector returns 2 detections: EMAIL (span 12-28) and STEUER_ID (span 48-59). All German pattern tests green.

---

### Phase 3: Layer 2 — NER (Presidio + spaCy) + Layer 3 — LLM Contextual (Day 3)

**Goal:** NER detection via Presidio + spaCy (en + de models) and LLM-based contextual detection. Three layers independently functional.

| Step | Task | Deliverable |
|------|------|-------------|
| 3.1 | Install Presidio + spaCy; download `en_core_web_lg` + `de_core_news_lg` | Models available in container |
| 3.2 | Write `src/proxy/language_detector.py` (detect en/de) | Language detection works |
| 3.3 | Write `src/detection/ner_detector.py` (Presidio analyzer with en/de NLP engines) | NER detects persons, orgs, locations, dates |
| 3.4 | Register custom Presidio recognizers for German patterns (Steuer-ID, IBAN-DE, PLZ) | Presidio uses our German regex patterns |
| 3.5 | Write `src/detection/llm_detector.py` (LLM-based contextual detection) | LLM catches implicit PII |
| 3.6 | Write `src/detection/merger.py` (merge + dedup overlapping spans) | Overlapping detections resolved |
| 3.7 | Write `src/detection/pipeline.py` (orchestrate 3 layers + merge) | Full detection pipeline works |
| 3.8 | Write `tests/test_ner_detector.py` (en + de test cases) | NER tests pass |
| 3.9 | Write `tests/test_llm_detector.py` (contextual PII test cases) | LLM detector tests pass |
| 3.10 | Write `tests/test_merger.py` (overlap resolution, confidence scoring) | Merger tests pass |

**Verification:** Feed `"My name is Anna Müller, I live in Kiel"` → Layer 2 detects PERSON (Anna Müller) + LOCATION (Kiel). Feed `"My boss told me to email the CEO"` → Layer 3 detects implicit PERSON references. Merger resolves overlaps when regex and NER both match the same span.

---

### Phase 4: Redaction Engine + Tenant Config (Day 4)

**Goal:** Redaction strategies (mask, hash, synthetic) working with per-tenant configuration and allowlist.

| Step | Task | Deliverable |
|------|------|-------------|
| 4.1 | Write `src/redaction/strategies.py` (MaskStrategy, HashStrategy, SyntheticStrategy) | All 3 strategies implemented |
| 4.2 | Write `src/redaction/synthetic_data.py` (Faker with de_DE locale) | German synthetic names, addresses, phones |
| 4.3 | Write `src/redaction/engine.py` (apply strategy per entity per tenant config) | Engine applies correct strategy per PII type |
| 4.4 | Write `src/tenant/models.py` (TenantConfig pydantic model) | Config model |
| 4.5 | Extend `src/tenant/config_resolver.py` (load allowlist, custom patterns, strategies) | Full tenant config resolution |
| 4.6 | Implement allowlist filtering in merger (remove allowlisted entities) | Allowlisted entities pass through |
| 4.7 | Write `tests/test_redaction.py` (all strategies, per-type config) | Redaction tests pass |
| 4.8 | Write `tests/test_tenant_config.py` (per-tenant rules, allowlist, custom patterns) | Tenant config tests pass |

**Verification:** Tenant A (strategy: mask) → `"test@mail.de"` becomes `"[REDACTED_EMAIL]"`. Tenant B (strategy: hash) → becomes `"[HASH_a3f9b2c1]"`. Tenant C (strategy: synthetic) → becomes `"max.mustermann@example.de"`. Allowlisted `"support@company.de"` is not redacted.

---

### Phase 5: Input/Output Scanners + Proxy Integration (Day 5)

**Goal:** Full proxy pipeline: input scan → redact → forward to LLM → output scan → sanitize → return to client. Compliance logging wired in.

| Step | Task | Deliverable |
|------|------|-------------|
| 5.1 | Write `src/scanner/input_scanner.py` (detect + redact in user prompts) | Input scanner works |
| 5.2 | Write `src/scanner/output_scanner.py` (detect + sanitize/block in LLM responses) | Output scanner works |
| 5.3 | Write `src/compliance/logger.py` (append-only log writer) | Compliance log writer works |
| 5.4 | Wire compliance logging into input + output scanners | Every redaction event logged |
| 5.5 | Integrate scanners into `src/proxy/middleware.py` (full pipeline) | Proxy scans input + output |
| 5.6 | Add `X-PII-Redacted` + `X-Redaction-Count` response headers | Metadata headers present |
| 5.7 | Write `src/api/routes/audit.py` (admin-only GET redaction events + summary) | Audit query endpoint works |
| 5.8 | Write `tests/test_input_scanner.py` + `tests/test_output_scanner.py` | Scanner tests pass |
| 5.9 | Write `tests/test_compliance.py` (immutability, completeness) | Compliance tests pass |
| 5.10 | Write `tests/test_security.py` (critical: no raw PII in forwarded prompt or returned response) | **Security tests pass** |

**Verification:** Send prompt with PII via proxy → inspect forwarded request (mock upstream) → no raw PII present. Mock upstream returns PII in response → client receives sanitized response. Every redaction event appears in `redaction_events` table. Attempt to UPDATE `redaction_events` → exception.

---

### Phase 6: Streamlit Dashboard + Admin API (Day 6)

**Goal:** Dashboard showing redaction statistics, PII type distribution, compliance report export. Admin API for tenant management.

| Step | Task | Deliverable |
|------|------|-------------|
| 6.1 | Write `dashboard/queries.py` (SQL queries for stats) | Dashboard data queries work |
| 6.2 | Write `dashboard/charts.py` (PII type distribution, redaction trend over time) | Charts render |
| 6.3 | Write `dashboard/app.py` (Streamlit main: stats, charts, tables, filters) | Dashboard renders live data |
| 6.4 | Write `dashboard/reports.py` (CSV + PDF compliance report export) | Report export works |
| 6.5 | Write `src/api/routes/admin.py` (GET/POST tenants, tenant config) | Admin API works |
| 6.6 | Write `docker/Dockerfile.dashboard` | Dashboard containerizes |
| 6.7 | Write `tests/test_dashboard.py` (data query correctness) | Dashboard tests pass |

**Verification:** `docker compose up` → open `localhost:8501` → dashboard shows redaction stats from seeded data. Export CSV report → file downloads with correct columns. Create a new tenant via admin API → query as that tenant → per-tenant rules apply.

---

### Phase 7: Hardening, Evaluation, Documentation (Day 7)

**Goal:** Production-ready, evaluated, documented, edge cases handled.

| Step | Task | Deliverable |
|------|------|-------------|
| 7.1 | Write `scripts/evaluate_recall.py` (compute recall/precision on labeled PII corpus) | Evaluation script works |
| 7.2 | Write `scripts/benchmark_detection.py` (measure latency per layer) | Benchmark results documented |
| 7.3 | Add rate limiting (slowapi) + request size limits | Prevents abuse |
| 7.4 | Add structured logging (structlog) | JSON logs for observability |
| 7.5 | Add health check + readiness probe | `/health` and `/ready` endpoints |
| 7.6 | Add error handling: upstream LLM down, scanner timeout, DB down | Clean error responses |
| 7.7 | Write `.env.example` with all env vars | Configuration documented |
| 7.8 | Run full test suite end-to-end | All tests green |
| 7.9 | Security review: re-run `test_security.py` + manual PII leakage audit | No leaks found |
| 7.10 | Run recall evaluation on labeled corpus | Recall ≥ 98% documented |

**Verification:** Fresh `docker compose up` → run `scripts/seed_test_data.py` → send prompt with mixed en/de PII via proxy → verify redacted prompt forwarded → verify sanitized response returned → check dashboard shows events → export compliance report → all works without manual intervention.

---

## 7. Component Specifications

### 7.1 Regex Detector — Layer 1 (`src/detection/regex_detector.py`)

```python
"""
Layer 1: Regex-based PII detection.

Loads patterns from YAML catalog (patterns/regex_catalog.yaml +
patterns/german_patterns.yaml). Returns list of PIIDetection objects.

Fast, deterministic, zero ML cost. Catches format-based PII:
emails, phone numbers, SSNs, credit cards, IBANs, Steuer-IDs,
passport numbers, postal codes, German address patterns.
"""

import regex as re
from pathlib import Path
import yaml
from src.detection.models import PIIDetection, PIIType

class RegexDetector:
    def __init__(self, catalog_path: str = "patterns/regex_catalog.yaml",
                 german_path: str = "patterns/german_patterns.yaml"):
        self.patterns: list[CompiledPattern] = []
        self._load_catalog(catalog_path)
        self._load_catalog(german_path)

    def _load_catalog(self, path: str):
        with open(path) as f:
            catalog = yaml.safe_load(f)
        for entry in catalog["patterns"]:
            self.patterns.append(CompiledPattern(
                name=entry["name"],
                pii_type=PIIType(entry["pii_type"]),
                pattern=re.compile(entry["pattern"], entry.get("flags", 0)),
                validator=entry.get("validator"),  # e.g. "luhn" for credit cards
                priority=entry.get("priority", 0),
            ))

    def detect(self, text: str, tenant_config: TenantConfig | None = None) -> list[PIIDetection]:
        detections = []
        for pat in self.patterns:
            # Skip patterns for PII types the tenant has disabled
            if tenant_config and pat.pii_type.value not in tenant_config.enabled_pii_types:
                continue
            # Also check tenant custom patterns
            for match in pat.pattern.finditer(text):
                matched_text = match.group()
                # Optional validation (e.g. Luhn check for credit cards)
                if pat.validator == "luhn" and not luhn_check(matched_text):
                    continue
                if pat.validator == "iban_checksum" and not iban_validate(matched_text):
                    continue
                detections.append(PIIDetection(
                    pii_type=pat.pii_type,
                    text=matched_text,
                    start=match.start(),
                    end=match.end(),
                    layer="regex",
                    score=1.0,  # regex is deterministic → confidence 1.0
                    pattern_name=pat.name,
                ))
        # Add tenant custom regex patterns
        if tenant_config and tenant_config.custom_regex_patterns:
            for custom in tenant_config.custom_regex_patterns:
                for match in re.finditer(custom["pattern"], text):
                    detections.append(PIIDetection(
                        pii_type=PIIType(custom["pii_type"]),
                        text=match.group(),
                        start=match.start(),
                        end=match.end(),
                        layer="regex",
                        score=1.0,
                        pattern_name=custom["name"],
                    ))
        return detections


def luhn_check(number: str) -> bool:
    """Luhn algorithm checksum for credit card validation."""
    digits = [int(d) for d in number if d.isdigit()]
    if len(digits) < 13:
        return False
    checksum = 0
    parity = len(digits) % 2
    for i, d in enumerate(digits):
        if i % 2 == parity:
            d *= 2
            if d > 9:
                d -= 9
        checksum += d
    return checksum % 2 == 0


def iban_validate(iban: str) -> bool:
    """IBAN checksum validation (mod-97)."""
    iban = iban.replace(" ", "").upper()
    if len(iban) < 15:
        return False
    # Move first 4 chars to end, substitute letters
    rearranged = iban[4:] + iban[:4]
    numeric = ""
    for ch in rearranged:
        if ch.isdigit():
            numeric += ch
        elif ch.isalpha():
            numeric += str(ord(ch) - ord('A') + 10)
        else:
            return False
    return int(numeric) % 97 == 1
```

### 7.2 NER Detector — Layer 2 (`src/detection/ner_detector.py`)

```python
"""
Layer 2: Named Entity Recognition via Microsoft Presidio + spaCy.

Uses en_core_web_lg for English and de_core_news_lg for German.
Presidio orchestrates spaCy NER + custom recognizers for
format-based PII (registered as Presidio PatternRecognizers
using our regex catalog).

Catches: person names, organizations, locations, dates, addresses.
"""

from presidio_analyzer import AnalyzerEngine, RecognizerRegistry
from presidio_analyzer.nlp_engine import NlpEngineProvider
from presidio_anonymizer import AnonymizerEngine
from src.detection.models import PIIDetection, PIIType

# Map Presidio entity types to our PIIType enum
PRESIDIO_TO_PII = {
    "PERSON": PIIType.PERSON,
    "ORGANIZATION": PIIType.ORGANIZATION,
    "LOCATION": PIIType.LOCATION,
    "DATE_TIME": PIIType.DATE,
    "EMAIL_ADDRESS": PIIType.EMAIL,
    "PHONE_NUMBER": PIIType.PHONE,
    "IBAN_CODE": PIIType.IBAN,
    "CREDIT_CARD": PIIType.CREDIT_CARD,
    "US_SSN": PIIType.SSN,
    "DE_STEUER_ID": PIIType.STEUER_ID,    # custom recognizer
    "DE_PLZ": PIIType.PLZ,                # custom recognizer
    "DE_ADDRESS": PIIType.ADDRESS_DE,     # custom recognizer
}

class NERDetector:
    def __init__(self):
        # Configure bilingual NLP engine (en + de)
        self.nlp_engine = NlpEngineProvider({
            "nlp_engine_name": "spacy",
            "models": [
                {"lang_code": "en", "model_name": "en_core_web_lg"},
                {"lang_code": "de", "model_name": "de_core_news_lg"},
            ],
        }).create_engine()

        self.registry = RecognizerRegistry()
        self.registry.load_predefined_recognizers(
            languages=["en", "de"], nlp_engine=self.nlp_engine
        )

        # Register custom German recognizers (Steuer-ID, PLZ, address)
        self._register_german_recognizers()

        self.analyzer = AnalyzerEngine(
            nlp_engine=self.nlp_engine,
            registry=self.registry,
            supported_languages=["en", "de"],
        )

    def _register_german_recognizers(self):
        """Register custom Presidio PatternRecognizers for German PII."""
        from presidio_analyzer import Pattern, PatternRecognizer

        # German Steuer-ID (11 digits, format: 12345678901)
        steuer_pattern = Pattern(
            "steuer_id", r"\b\d{11}\b", 0.85
        )
        self.registry.add_recognizer(PatternRecognizer(
            supported_entity="DE_STEUER_ID",
            patterns=[steuer_pattern],
            supported_language="de",
        ))

        # German postal code (5 digits)
        plz_pattern = Pattern(
            "plz", r"\b\d{5}\b", 0.6  # lower confidence — 5 digits are common
        )
        self.registry.add_recognizer(PatternRecognizer(
            supported_entity="DE_PLZ",
            patterns=[plz_pattern],
            supported_language="de",
        ))

        # German address: StreetName + number + PLZ + city
        address_pattern = Pattern(
            "de_address",
            r"\b[A-ZÄÖÜ][a-zäöüß]+(?:str\.|straße|weg|gasse|allee|platz)\.?\s+\d+[a-z]?,?\s+\d{5}\s+[A-ZÄÖÜ][a-zäöüß]+",
            0.75,
        )
        self.registry.add_recognizer(PatternRecognizer(
            supported_entity="DE_ADDRESS",
            patterns=[address_pattern],
            supported_language="de",
        ))

    def detect(self, text: str, language: str = "en",
               tenant_config: TenantConfig | None = None) -> list[PIIDetection]:
        # Determine which Presidio entities to look for based on tenant config
        entities = self._map_enabled_types(tenant_config)

        results = self.analyzer.analyze(
            text=text,
            entities=entities,
            language=language,
            score_threshold=0.5,
        )

        detections = []
        for result in results:
            pii_type = PRESIDIO_TO_PII.get(result.entity_type)
            if pii_type is None:
                continue
            detections.append(PIIDetection(
                pii_type=pii_type,
                text=text[result.start:result.end],
                start=result.start,
                end=result.end,
                layer="ner",
                score=result.score,
                pattern_name=result.entity_type,
            ))
        return detections

    def _map_enabled_types(self, tenant_config) -> list[str]:
        """Map tenant's enabled PII types to Presidio entity names."""
        if not tenant_config:
            return list(PRESIDIO_TO_PII.keys())
        reverse_map = {v: k for k, v in PRESIDIO_TO_PII.items()}
        return [
            reverse_map[PIIType(t)]
            for t in tenant_config.enabled_pii_types
            if PIIDetection(t) in reverse_map
        ]
```

### 7.3 LLM Contextual Detector — Layer 3 (`src/detection/llm_detector.py`)

```python
"""
Layer 3: LLM-based contextual PII detection.

Uses a small, fast LLM (gpt-4o-mini) to catch implicit PII that
regex and NER miss. Runs only when:
  1. Layer 1 + 2 found fewer entities than expected given PII cue words
  2. Or always (if tenant config sets llm_contextual_enabled=true and
     the text length is below a threshold to control cost)

The LLM is prompted to identify PII spans and return structured JSON.
It does NOT see the raw text with PII — it sees the text with Layer 1+2
detections already masked, plus the original text. The LLM's job is to
find ADDITIONAL PII that the first two layers missed.
"""

import json
from src.detection.models import PIIDetection, PIIType
from src.proxy.llm_client import LLMClient

CONTEXTUAL_DETECTION_PROMPT = """\
You are a PII (personally identifiable information) detection assistant.
Your task is to identify ANY personally identifiable information in the
text below that has NOT already been detected.

Already detected (masked): {existing_detections}

Text to analyze:
"{text}"

Identify additional PII that was missed. Look for:
- Person names (including implicit references like "my boss", "Dr. X")
- Email addresses, phone numbers, physical addresses
- Government IDs (SSN, Steuer-ID, passport, tax IDs)
- Financial information (credit cards, IBANs, account numbers)
- Employer/organization names that identify a specific person
- Any other information that could identify a specific individual

Respond in JSON format:
{
  "detections": [
    {
      "text": "the exact PII text as it appears",
      "pii_type": "PERSON|EMAIL|PHONE|ADDRESS|STEUER_ID|IBAN|CREDIT_CARD|ORG|DATE|OTHER",
      "reasoning": "why this is PII"
    }
  ]
}

If no additional PII is found, return: {"detections": []}
"""

PII_TYPE_MAP = {
    "PERSON": PIIType.PERSON,
    "EMAIL": PIIType.EMAIL,
    "PHONE": PIIType.PHONE,
    "ADDRESS": PIIType.ADDRESS_DE,
    "STEUER_ID": PIIType.STEUER_ID,
    "IBAN": PIIType.IBAN,
    "CREDIT_CARD": PIIType.CREDIT_CARD,
    "ORG": PIIType.ORGANIZATION,
    "DATE": PIIType.DATE,
    "OTHER": PIIType.OTHER,
}

class LLMContextualDetector:
    def __init__(self, llm_client: LLMClient, max_text_length: int = 2000):
        self.llm_client = llm_client
        self.max_text_length = max_text_length

    def should_run(self, text: str, existing_detections: list[PIIDetection]) -> bool:
        """Heuristic: run Layer 3 if text has PII cue words but few detections."""
        if len(text) > self.max_text_length:
            return False  # skip for very long texts (cost control)
        cue_words = [
            "name", "address", "phone", "email", "ssn", "passport",
            "mein", "adresse", "telefon", "name", "wohnhaft",
            "I live", "I work", "my boss", "my doctor", "call me",
            "ich heiße", "ich wohne", "ich arbeite",
        ]
        text_lower = text.lower()
        cue_count = sum(1 for w in cue_words if w in text_lower)
        # Run if there are cue words but few detections, or always if >3 cues
        return cue_count > 0 and len(existing_detections) < cue_count or cue_count > 3

    async def detect(self, text: str,
                     existing_detections: list[PIIDetection],
                     language: str = "en") -> list[PIIDetection]:
        if not self.should_run(text, existing_detections):
            return []

        existing_summary = ", ".join(
            f"[{d.pii_type.value}: masked]" for d in existing_detections
        ) or "None"

        prompt = CONTEXTUAL_DETECTION_PROMPT.format(
            existing_detections=existing_summary,
            text=text,
        )

        response = await self.llm_client.generate(
            prompt=prompt,
            model="gpt-4o-mini",
            temperature=0.0,
            response_format={"type": "json_object"},
        )

        try:
            result = json.loads(response)
        except json.JSONDecodeError:
            return []

        detections = []
        for det in result.get("detections", []):
            pii_type = PII_TYPE_MAP.get(det.get("pii_type", "OTHER"), PIIType.OTHER)
            # Find the span of the detected text in the original
            idx = text.find(det["text"])
            if idx == -1:
                continue  # LLM hallucinated text not in original
            detections.append(PIIDetection(
                pii_type=pii_type,
                text=det["text"],
                start=idx,
                end=idx + len(det["text"]),
                layer="llm_contextual",
                score=0.7,  # LLM confidence — lower than regex/NER
                pattern_name=f"llm:{det.get('reasoning', '')[:50]}",
            ))
        return detections
```

### 7.4 Detection Merger (`src/detection/merger.py`)

```python
"""
Merges detections from all three layers, resolves overlapping spans,
deduplicates, and applies the tenant allowlist.

Overlap resolution rules:
  1. If two detections cover the exact same span → keep the one with
     higher confidence (regex score=1.0 > NER > LLM)
  2. If one detection's span contains another → keep the larger span
     if the PII types are compatible (e.g. ADDRESS_DE contains PLZ);
     otherwise keep both (they may be different entities)
  3. Adjacent detections of the same type → merge into one span
"""

from src.detection.models import PIIDetection

class DetectionMerger:
    def merge(self, detections: list[PIIDetection],
              allowlist: list[AllowlistEntry] | None = None) -> list[PIIDetection]:
        # 1. Sort by start position, then by span length (descending)
        sorted_dets = sorted(detections, key=lambda d: (d.start, -(d.end - d.start)))

        # 2. Resolve overlaps
        merged = []
        for det in sorted_dets:
            if self._is_contained(det, merged):
                continue  # skip — already covered by a larger span
            if self._overlaps_incompatible(det, merged):
                merged.append(det)
            else:
                merged.append(det)

        # 3. Deduplicate exact matches (same span + same type)
        seen = set()
        deduped = []
        for det in merged:
            key = (det.start, det.end, det.pii_type)
            if key not in seen:
                seen.add(key)
                deduped.append(det)

        # 4. Apply allowlist: remove allowlisted entities
        if allowlist:
            deduped = [
                d for d in deduped
                if not self._is_allowlisted(d, allowlist)
            ]

        return deduped

    def _is_contained(self, det: PIIDetection, existing: list[PIIDetection]) -> bool:
        for ex in existing:
            if det.start >= ex.start and det.end <= ex.end:
                # Contained — skip unless it's a different type and not fully covered
                if det.pii_type == ex.pii_type:
                    return True
        return False

    def _overlaps_incompatible(self, det: PIIDetection,
                                existing: list[PIIDetection]) -> bool:
        for ex in existing:
            if det.start < ex.end and det.end > ex.start:
                if det.pii_type != ex.pii_type:
                    return True  # different types overlapping → keep both
        return False

    def _is_allowlisted(self, det: PIIDetection,
                        allowlist: list[AllowlistEntry]) -> bool:
        for entry in allowlist:
            if entry.pii_type != det.pii_type.value:
                continue
            if entry.match_type == "exact" and entry.entity_value == det.text:
                return True
            if entry.match_type == "regex":
                import regex as re
                if re.search(entry.entity_value, det.text):
                    return True
        return False
```

### 7.5 Redaction Engine (`src/redaction/engine.py`)

```python
"""
Applies the configured redaction strategy to each detected PII entity.
Strategies: mask, hash, synthetic.

The engine processes detections from right to left (highest span offset
first) so that replacements don't shift the character offsets of
remaining detections.
"""

import hashlib
import hmac
from src.detection.models import PIIDetection
from src.redaction.strategies import MaskStrategy, HashStrategy, SyntheticStrategy
from src.redaction.synthetic_data import SyntheticDataGenerator

class RedactionEngine:
    def __init__(self, hash_salt: str, synthetic_gen: SyntheticDataGenerator):
        self.hash_salt = hash_salt
        self.synthetic_gen = synthetic_gen
        self.strategies = {
            "mask": MaskStrategy(),
            "hash": HashStrategy(salt=hash_salt),
            "synthetic": SyntheticStrategy(synthetic_gen),
        }

    def redact(self, text: str, detections: list[PIIDetection],
               tenant_config: TenantConfig) -> RedactionResult:
        # Sort detections by start position descending (right to left)
        sorted_dets = sorted(detections, key=lambda d: d.start, reverse=True)

        redacted_text = text
        redaction_events = []

        for det in sorted_dets:
            # Determine strategy for this PII type from tenant config
            strategy_name = tenant_config.redaction_strategies.get(
                det.pii_type.value, "mask"  # default: mask
            )
            strategy = self.strategies[strategy_name]

            # Generate replacement
            replacement, event_meta = strategy.apply(det)

            # Replace in text (right to left preserves offsets)
            redacted_text = (
                redacted_text[:det.start] + replacement + redacted_text[det.end:]
            )

            redaction_events.append(RedactionEvent(
                pii_type=det.pii_type.value,
                detection_layer=det.layer,
                detection_score=det.score,
                detected_text=det.text,
                detected_span_start=det.start,
                detected_span_end=det.end,
                redaction_strategy=strategy_name,
                redacted_text=replacement,
                hash_value=event_meta.get("hash"),
            ))

        return RedactionResult(
            original_text=text,
            redacted_text=redacted_text,
            events=redaction_events,
        )
```

### 7.6 Redaction Strategies (`src/redaction/strategies.py`)

```python
"""
Three redaction strategies:

1. Mask: Replace with [REDACTED_TYPE], e.g. [REDACTED_EMAIL]
   - Pro: Clear to LLM that something was redacted
   - Con: Breaks sentence structure; LLM may behave differently

2. Hash: Replace with [HASH_a3f9b2c1] (HMAC-SHA256, salted)
   - Pro: Deterministic — same PII → same hash → allows audit
          re-identification without exposing PII
   - Con: Not human-readable; LLM sees opaque token

3. Synthetic: Replace with fake data (Faker, de_DE locale)
   - Pro: Preserves sentence structure; LLM processes naturally
   - Con: Fake data may confuse if LLM references it in response
"""

import hashlib
import hmac
from src.detection.models import PIIDetection
from src.redaction.synthetic_data import SyntheticDataGenerator

class MaskStrategy:
    def apply(self, detection: PIIDetection) -> tuple[str, dict]:
        replacement = f"[REDACTED_{detection.pii_type.value}]"
        return replacement, {}

class HashStrategy:
    def __init__(self, salt: str):
        self.salt = salt.encode()

    def apply(self, detection: PIIDetection) -> tuple[str, dict]:
        # HMAC-SHA256 with salt → deterministic, non-reversible
        h = hmac.new(self.salt, detection.text.encode(), hashlib.sha256)
        hash_hex = h.hexdigest()[:8]  # short hash for readability
        replacement = f"[HASH_{hash_hex}]"
        return replacement, {"hash": h.hexdigest()}

class SyntheticStrategy:
    def __init__(self, generator: SyntheticDataGenerator):
        self.generator = generator

    def apply(self, detection: PIIDetection) -> tuple[str, dict]:
        synthetic = self.generator.generate(detection.pii_type, detection.text)
        return synthetic, {"synthetic_original_hash": hashlib.sha256(
            detection.text.encode()
        ).hexdigest()[:8]}
```

### 7.7 Synthetic Data Generator (`src/redaction/synthetic_data.py`)

```python
"""
Generates realistic fake PII for synthetic redaction strategy.
Uses Faker with de_DE locale for German-appropriate fake data.

Maintains a per-request mapping so the same original PII always
maps to the same synthetic value within a single request (consistency).
"""

from faker import Faker
from src.detection.models import PIIType

class SyntheticDataGenerator:
    def __init__(self, locale: str = "de_DE"):
        self.fake = Faker(locale)
        self._cache: dict[str, str] = {}  # original → synthetic

    def generate(self, pii_type: PIIType, original: str) -> str:
        # Consistency: same original → same synthetic within request
        cache_key = f"{pii_type.value}:{original}"
        if cache_key in self._cache:
            return self._cache[cache_key]

        generators = {
            PIIType.PERSON: lambda: self.fake.name(),
            PIIType.EMAIL: lambda: self.fake.email(),
            PIIType.PHONE: lambda: self.fake.phone_number(),
            PIIType.ADDRESS_DE: lambda: f"{self.fake.street_name()} {self.fake.building_number()}, {self.fake.postcode()} {self.fake.city()}",
            PIIType.PLZ: lambda: self.fake.postcode(),
            PIIType.IBAN: lambda: self.fake.iban(),
            PIIType.CREDIT_CARD: lambda: self.fake.credit_card_number(),
            PIIType.STEUER_ID: lambda: f"{self.fake.random_number(digits=11)}",
            PIIType.ORGANIZATION: lambda: self.fake.company(),
            PIIType.DATE: lambda: self.fake.date(),
            PIIType.SSN: lambda: self.fake.ssn(),
        }

        gen = generators.get(pii_type, lambda: "[REDACTED]")
        synthetic = gen()
        self._cache[cache_key] = synthetic
        return synthetic

    def reset_cache(self):
        """Clear the per-request mapping."""
        self._cache.clear()
```

### 7.8 Input Scanner (`src/scanner/input_scanner.py`)

```python
"""
Pre-LLM input scanner. Runs the full 3-layer detection pipeline on
each message in the request, redacts detected PII, and logs every
redaction event to the compliance log.

This is the critical security boundary: no PII passes through to
the LLM API.
"""

from src.detection.pipeline import DetectionPipeline
from src.redaction.engine import RedactionEngine
from src.compliance.logger import ComplianceLogger

class InputScanner:
    def __init__(self, pipeline: DetectionPipeline,
                 engine: RedactionEngine, logger: ComplianceLogger):
        self.pipeline = pipeline
        self.engine = engine
        self.logger = logger

    async def scan(self, messages: list[dict], tenant_config: TenantConfig,
                   user_id: str, request_id: UUID, model_name: str) -> list[dict]:
        scanned_messages = []

        for msg in messages:
            if msg.get("role") == "system":
                # System prompts are trusted — don't scan
                scanned_messages.append(msg)
                continue

            content = msg.get("content", "")
            if not isinstance(content, str) or not content:
                scanned_messages.append(msg)
                continue

            # 1. Detect PII (3-layer pipeline)
            detections = await self.pipeline.detect(
                text=content, tenant_config=tenant_config
            )

            # 2. Redact
            result = self.engine.redact(content, detections, tenant_config)

            # 3. Log every redaction event
            for event in result.events:
                await self.logger.log_redaction(
                    tenant_id=tenant_config.tenant_id,
                    user_id=user_id,
                    request_id=request_id,
                    direction="input",
                    event=event,
                    model_name=model_name,
                    language=tenant_config.detected_language,
                )

            # 4. Replace content with redacted version
            scanned_msg = {**msg, "content": result.redacted_text}
            scanned_messages.append(scanned_msg)

        return scanned_messages
```

### 7.9 Output Scanner (`src/scanner/output_scanner.py`)

```python
"""
Post-LLM output scanner. Scans the LLM's response for PII leakage
(hallucinated PII, training data memorization, or PII that slipped
through if the LLM "reconstructed" it from context).

Two modes (per tenant config):
  - 'sanitize': mask detected PII in the response
  - 'block': return an error instead of the response
"""

from src.detection.pipeline import DetectionPipeline
from src.redaction.engine import RedactionEngine
from src.compliance.logger import ComplianceLogger

class OutputScanner:
    def __init__(self, pipeline: DetectionPipeline,
                 engine: RedactionEngine, logger: ComplianceLogger):
        self.pipeline = pipeline
        self.engine = engine
        self.logger = logger

    async def scan(self, response: dict, tenant_config: TenantConfig,
                   user_id: str, request_id: UUID,
                   model_name: str) -> dict:
        # Extract text from the response (OpenAI format)
        choices = response.get("choices", [])
        pii_found_in_output = False

        for choice in choices:
            content = choice.get("message", {}).get("content", "")
            if not content:
                continue

            # 1. Detect PII in output
            detections = await self.pipeline.detect(
                text=content, tenant_config=tenant_config
            )

            if detections:
                pii_found_in_output = True

                if tenant_config.output_mode == "block":
                    # Block the entire response
                    for event in self._build_events(detections, content):
                        await self.logger.log_redaction(
                            tenant_id=tenant_config.tenant_id,
                            user_id=user_id, request_id=request_id,
                            direction="output", event=event,
                            model_name=model_name,
                            language=tenant_config.detected_language,
                        )
                    raise PIILeakageError(
                        "LLM response contained PII; response blocked per tenant policy"
                    )

                # Sanitize: mask PII in output (always use mask for output)
                # Force mask strategy regardless of tenant input strategy
                mask_config = TenantConfig(
                    redaction_strategies={t.value: "mask" for t in tenant_config.enabled_pii_types}
                )
                result = self.engine.redact(content, detections, mask_config)

                # Log output redaction events
                for event in result.events:
                    await self.logger.log_redaction(
                        tenant_id=tenant_config.tenant_id,
                        user_id=user_id, request_id=request_id,
                        direction="output", event=event,
                        model_name=model_name,
                        language=tenant_config.detected_language,
                    )

                # Replace content with sanitized version
                choice["message"]["content"] = result.redacted_text

        return response, pii_found_in_output
```

### 7.10 Proxy Middleware — Core Integration (`src/proxy/middleware.py`)

```python
"""
The core PII guardrail proxy middleware.

Intercepts every /v1/chat/completions and /v1/completions request:
  1. Authenticate + resolve tenant
  2. Scan input (user messages) → redact PII
  3. Forward redacted request to upstream LLM API
  4. Scan output (LLM response) → sanitize/block PII
  5. Log all redaction events to compliance log
  6. Return safe response with X-PII-Redacted headers

This is the security boundary. No PII crosses this middleware.
"""

import time
import uuid
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import JSONResponse, Response

class PIIGuardrailMiddleware(BaseHTTPMiddleware):
    def __init__(self, app, input_scanner, output_scanner, llm_client,
                 tenant_resolver, compliance_logger):
        super().__init__(app)
        self.input_scanner = input_scanner
        self.output_scanner = output_scanner
        self.llm_client = llm_client
        self.tenant_resolver = tenant_resolver
        self.compliance_logger = compliance_logger

    async def dispatch(self, request: Request, call_next):
        # Skip non-LLM endpoints
        path = request.url.path
        if path not in ("/v1/chat/completions", "/v1/completions"):
            return await call_next(request)

        request_id = uuid.uuid4()
        start_time = time.monotonic()

        # 1. Authenticate + resolve tenant
        token = request.headers.get("Authorization", "").replace("Bearer ", "")
        if not token:
            return JSONResponse({"detail": "Missing token"}, status_code=401)

        try:
            user = verify_jwt(token)
            tenant_config = await self.tenant_resolver.resolve(user.tenant_id)
        except (ExpiredTokenError, InvalidTokenError):
            return JSONResponse({"detail": "Invalid token"}, status_code=401)
        except TenantNotFoundError:
            return JSONResponse({"detail": "Tenant not found"}, status_code=403)

        # 2. Read request body
        body = await request.body()
        request_data = json.loads(body)

        # 3. Input scan: detect + redact PII in messages
        input_scan_start = time.monotonic()
        messages = request_data.get("messages", [])
        if path == "/v1/completions":
            # Completions API: single prompt field
            prompt = request_data.get("prompt", "")
            messages = [{"role": "user", "content": prompt}]

        scanned_messages = await self.input_scanner.scan(
            messages=messages,
            tenant_config=tenant_config,
            user_id=user.user_id,
            request_id=request_id,
            model_name=request_data.get("model", "unknown"),
        )
        input_scan_ms = int((time.monotonic() - input_scan_start) * 1000)

        # Replace messages in request body
        if path == "/v1/chat/completions":
            request_data["messages"] = scanned_messages
        else:
            request_data["prompt"] = scanned_messages[0].get("content", "")

        # 4. Forward to upstream LLM API
        llm_call_start = time.monotonic()
        try:
            llm_response = await self.llm_client.forward(
                path=path, body=request_data, headers=request.headers
            )
        except UpstreamLLMError as e:
            await self.compliance_logger.log_request(
                tenant_id=tenant_config.tenant_id, user_id=user.user_id,
                request_id=request_id, endpoint=path,
                model_name=request_data.get("model", "unknown"),
                direction="blocked", success=False, error_message=str(e),
                input_redaction_count=len(input_events),
                total_latency_ms=int((time.monotonic() - start_time) * 1000),
            )
            return JSONResponse({"detail": f"Upstream LLM error: {e}"}, status_code=502)
        llm_call_ms = int((time.monotonic() - llm_call_start) * 1000)

        # 5. Output scan: detect + sanitize PII in LLM response
        output_scan_start = time.monotonic()
        try:
            safe_response, pii_in_output = await self.output_scanner.scan(
                response=llm_response,
                tenant_config=tenant_config,
                user_id=user.user_id,
                request_id=request_id,
                model_name=request_data.get("model", "unknown"),
            )
        except PIILeakageError as e:
            # Output mode = block: return error to client
            await self.compliance_logger.log_request(
                tenant_id=tenant_config.tenant_id, user_id=user.user_id,
                request_id=request_id, endpoint=path,
                model_name=request_data.get("model", "unknown"),
                direction="blocked", success=False, error_message=str(e),
                input_redaction_count=input_redaction_count,
                output_redaction_count=output_redaction_count,
                total_latency_ms=int((time.monotonic() - start_time) * 1000),
            )
            return JSONResponse(
                {"detail": "Response blocked: PII detected in LLM output"},
                status_code=422,
            )
        output_scan_ms = int((time.monotonic() - output_scan_start) * 1000)

        # 6. Log request-level audit
        total_ms = int((time.monotonic() - start_time) * 1000)
        await self.compliance_logger.log_request(
            tenant_id=tenant_config.tenant_id, user_id=user.user_id,
            request_id=request_id, endpoint=path,
            model_name=request_data.get("model", "unknown"),
            direction="proxy", success=True,
            input_redaction_count=input_redaction_count,
            output_redaction_count=output_redaction_count,
            input_scan_ms=input_scan_ms, llm_call_ms=llm_call_ms,
            output_scan_ms=output_scan_ms, total_latency_ms=total_ms,
        )

        # 7. Return safe response with metadata headers
        response = JSONResponse(safe_response, status_code=200)
        response.headers["X-PII-Redacted"] = "true"
        response.headers["X-Redaction-Count"] = str(
            input_redaction_count + output_redaction_count
        )
        response.headers["X-Request-Id"] = str(request_id)
        return response
```

### 7.11 Compliance Logger (`src/compliance/logger.py`)

```python
"""
Append-only compliance logger. Every redaction event and every
proxied request is recorded.

The redaction_events and audit_logs tables have DB triggers that
prevent UPDATE and DELETE, making them tamper-evident. This is
critical for GDPR Article 30 (records of processing activities)
and SOC 2 CC7.2 (system monitoring).
"""

import uuid

class ComplianceLogger:
    def __init__(self, db_pool):
        self.db_pool = db_pool

    async def log_redaction(self, tenant_id, user_id, request_id,
                            direction, event, model_name, language):
        """Log a single PII redaction event."""
        async with self.db_pool.acquire() as conn:
            await conn.execute(
                """INSERT INTO redaction_events
                   (id, tenant_id, user_id, request_id, direction,
                    pii_type, detection_layer, detection_score,
                    detected_text, detected_span_start, detected_span_end,
                    redaction_strategy, redacted_text, hash_value,
                    model_name, language)
                   VALUES ($1,$2,$3,$4,$5,$6,$7,$8,$9,$10,$11,$12,$13,$14,$15,$16)""",
                uuid.uuid4(), tenant_id, user_id, request_id, direction,
                event.pii_type, event.detection_layer, event.detection_score,
                event.detected_text, event.detected_span_start,
                event.detected_span_end, event.redaction_strategy,
                event.redacted_text, event.hash_value,
                model_name, language,
            )

    async def log_request(self, tenant_id, user_id, request_id, endpoint,
                          model_name, direction, success,
                          input_redaction_count, output_redaction_count,
                          input_scan_ms=None, llm_call_ms=None,
                          output_scan_ms=None, total_latency_ms=None,
                          error_message=None, pii_types_detected=None):
        """Log a request-level audit entry."""
        async with self.db_pool.acquire() as conn:
            await conn.execute(
                """INSERT INTO audit_logs
                   (id, tenant_id, user_id, request_id, endpoint,
                    model_name, direction, input_redaction_count,
                    output_redaction_count, pii_types_detected,
                    input_scan_ms, llm_call_ms, output_scan_ms,
                    total_latency_ms, success, error_message)
                   VALUES ($1,$2,$3,$4,$5,$6,$7,$8,$9,$10,$11,$12,$13,$14,$15,$16)""",
                uuid.uuid4(), tenant_id, user_id, request_id, endpoint,
                model_name, direction, input_redaction_count,
                output_redaction_count, pii_types_detected or [],
                input_scan_ms, llm_call_ms, output_scan_ms,
                total_latency_ms, success, error_message,
            )
```

### 7.12 Streamlit Dashboard (`dashboard/app.py`)

```python
"""
Streamlit dashboard for PII redaction monitoring and compliance reporting.

Shows:
  - Redaction statistics (total events, by type, by layer, by direction)
  - PII type distribution (pie chart)
  - Redaction trend over time (line chart)
  - Per-tenant breakdown
  - Compliance report export (CSV, PDF)
"""

import streamlit as st
import pandas as pd
from dashboard.queries import (
    get_redaction_stats, get_pii_type_distribution,
    get_redaction_trend, get_tenant_breakdown,
)
from dashboard.reports import export_csv_report, export_pdf_report

st.set_page_config(page_title="PII Guardrails Dashboard", page_icon="🛡️")

st.title("PII Redaction Guardrails — Compliance Dashboard")

# --- Filters ---
col1, col2, col3 = st.columns(3)
with col1:
    tenant = st.selectbox("Tenant", ["All"] + get_tenant_list())
with col2:
    date_from = st.date_input("From", value=(datetime.now() - timedelta(days=30)))
with col3:
    date_to = st.date_input("To", value=datetime.now())

# --- Summary Metrics ---
stats = get_redaction_stats(tenant, date_from, date_to)
col1, col2, col3, col4 = st.columns(4)
col1.metric("Total Redactions", stats["total"])
col2.metric("Input Redactions", stats["input_count"])
col3.metric("Output Redactions", stats["output_count"])
col4.metric("Blocked Responses", stats["blocked_count"])

# --- PII Type Distribution ---
st.subheader("PII Type Distribution")
type_dist = get_pii_type_distribution(tenant, date_from, date_to)
fig = px.pie(type_dist, values="count", names="pii_type",
             title="Detected PII by Type")
st.plotly_chart(fig)

# --- Redaction Trend ---
st.subheader("Redaction Trend (Last 30 Days)")
trend = get_redaction_trend(tenant, date_from, date_to)
fig = px.line(trend, x="date", y="count", color="direction",
              title="Redactions Over Time")
st.plotly_chart(fig)

# --- Recent Redaction Events ---
st.subheader("Recent Redaction Events")
events = get_recent_events(tenant, date_from, date_to, limit=100)
st.dataframe(events)

# --- Compliance Report Export ---
st.subheader("Compliance Report Export")
col1, col2 = st.columns(2)
with col1:
    if st.button("Export CSV"):
        csv = export_csv_report(tenant, date_from, date_to)
        st.download_button("Download CSV", csv, "compliance_report.csv", "text/csv")
with col2:
    if st.button("Export PDF"):
        pdf = export_pdf_report(tenant, date_from, date_to)
        st.download_button("Download PDF", pdf, "compliance_report.pdf", "application/pdf")
```

---

## 8. Security & Safety Model

### 8.1 Threat Model

| Threat | Mitigation | Layer |
|--------|-----------|-------|
| **Raw PII reaches the LLM** (input scanner fails or is bypassed) | 1. Proxy middleware is the only path to LLM API  2. Three-layer detection (regex + NER + LLM)  3. Security test suite verifies no PII in forwarded requests | App |
| **Leaked PII reaches the user** (LLM hallucinates or reveals training data) | 1. Output scanner runs same 3-layer pipeline on responses  2. Configurable: sanitize (mask) or block  3. Security test suite verifies no PII in returned responses | App |
| **Single-layer blind spot** (regex misses names, NER misses formats) | Three-layer pipeline: each layer compensates for others' weaknesses; Layer 3 (LLM) catches implicit/contextual PII | App |
| **Audit log tampering** (someone edits/deletes redaction events) | DB triggers block UPDATE/DELETE on `redaction_events` and `audit_logs`; only INSERT allowed | DB |
| **Tenant config bypass** (someone disables redaction for a tenant) | Config changes are admin-only (RBAC); config changes are themselves audit-logged; default-deny (if no config, all PII types are redacted) | App |
| **Allowlist abuse** (someone allowlists all emails, defeating redaction) | Allowlist entries require a reason field; dashboard flags tenants with large allowlists; admin review | App + Process |
| **Layer 3 LLM sees PII** (the detection LLM itself receives PII) | Layer 3 receives text with Layer 1+2 detections already masked; the detection LLM is a different call from the target LLM; detection LLM is a trusted internal component | App |
| **Replay attack** (captured redacted request replayed) | JWT expiry; request_id is unique per request; idempotency key optional | App |
| **Regex ReDoS** (malicious input causes catastrophic backtracking) | Use `regex` library (PCRE2) with timeout; patterns are pre-compiled and tested for backtracking; input length limits | App |
| **DB compromise exposes PII in compliance log** | DB encrypted at rest; `detected_text` column can be app-level encrypted; retention policy expires PII after 2 years; separate DB role for compliance officers | DB + Process |

### 8.2 Safety Principles

```
PRINCIPLE 1: Default-deny
  If no tenant config is found, ALL PII types are redacted with mask strategy.
  A tenant must explicitly opt-in to allow PII types, not opt-out.

PRINCIPLE 2: Defense in depth
  Layer 1 (regex) → Layer 2 (NER) → Layer 3 (LLM) → Merge → Redact → Log
  No single layer is trusted alone. The system is safe only if ALL layers
  agree there is no PII, or if detected PII is redacted.

PRINCIPLE 3: No PII crosses the boundary
  Input: user → [scanner] → LLM (redacted only)
  Output: LLM → [scanner] → user (sanitized only)
  The scanner is in the path, not optional.

PRINCIPLE 4: Auditability
  Every redaction event is logged immutably. Every request is logged.
  The compliance log is the source of truth for "what happened."
  Tampering is prevented at the DB level (triggers).

PRINCIPLE 5: Fail safe
  If the scanner crashes or times out, the request is BLOCKED (not forwarded
  with PII). A 503 error is safer than a PII leak.
  If the upstream LLM is down, the user gets a 502 (no PII leaked).
  If the DB is down, redaction still works (events are buffered and retried).
```

### 8.3 Defense in Depth

```
Layer 1: Regex Detection     → Fast, deterministic format matching
Layer 2: NER Detection       → ML-based entity recognition (Presidio + spaCy)
Layer 3: LLM Contextual      → Semantic understanding of implicit PII
Layer 4: Merger + Allowlist  → Resolve overlaps, apply tenant-specific exceptions
Layer 5: Redaction Engine    → Apply mask/hash/synthetic per tenant config
Layer 6: Output Scanner      → Post-LLM scan for leaked PII
Layer 7: Compliance Log      → Immutable audit trail (DB triggers)
Layer 8: DB Encryption       → At-rest encryption for compliance log
Layer 9: RBAC                → Admin-only config changes, compliance-only log access
```

---

## 9. API Specification

### 9.1 `POST /v1/chat/completions` (Proxy Endpoint)

OpenAI-compatible chat completions endpoint. The proxy intercepts, scans, redacts, forwards, scans output, and returns.

```
Headers:
  Authorization: Bearer <JWT>

Body (OpenAI-compatible):
{
  "model": "gpt-4o-mini",
  "messages": [
    {"role": "user", "content": "My name is Anna Müller, email a.mueller@gmx.de"}
  ],
  "temperature": 0.7
}

Response 200 (with redaction metadata headers):
  X-PII-Redacted: true
  X-Redaction-Count: 2
  X-Request-Id: uuid

{
  "id": "chatcmpl-...",
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "Hello! I see you're [REDACTED_PERSON]..."
      }
    }
  ]
}

Response 422: LLM output contained PII and tenant policy is 'block'
{
  "detail": "Response blocked: PII detected in LLM output"
}

Response 401: Missing or invalid JWT
Response 403: Tenant not found or inactive
Response 502: Upstream LLM API error
Response 503: Scanner error (fail-safe: request blocked, not forwarded)
```

### 9.2 `POST /v1/completions` (Proxy Endpoint)

OpenAI-compatible legacy completions endpoint. Same scanning pipeline.

```
Body:
{
  "model": "gpt-4o-mini",
  "prompt": "Translate: My Steuer-ID is 12345678901"
}

Response 200:
  X-PII-Redacted: true
  X-Redaction-Count: 1

{
  "choices": [{"text": "Translate: My Steuer-ID is [HASH_a3f9b2c1]"}]
}
```

### 9.3 `GET /admin/tenants` (Admin)

List all tenants. Requires `admin` role.

```
Headers:
  Authorization: Bearer <ADMIN_JWT>

Response 200:
{
  "tenants": [
    {
      "id": "uuid",
      "name": "acme-corp",
      "is_active": true,
      "enabled_pii_types": ["EMAIL", "PHONE", "PERSON", "STEUER_ID"],
      "output_mode": "sanitize",
      "created_at": "2025-01-15T10:30:00Z"
    }
  ]
}
```

### 9.4 `POST /admin/tenants/{id}/config` (Admin)

Update tenant configuration. Requires `admin` role.

```
Body:
{
  "enabled_pii_types": ["EMAIL", "PHONE", "PERSON", "STEUER_ID", "IBAN", "ADDRESS_DE"],
  "redaction_strategies": {
    "EMAIL": "mask",
    "PERSON": "synthetic",
    "STEUER_ID": "hash",
    "IBAN": "hash",
    "ADDRESS_DE": "synthetic"
  },
  "llm_contextual_enabled": true,
  "output_mode": "sanitize",
  "language": "auto"
}

Response 200: Updated config
Response 403: Not admin
```

### 9.5 `POST /admin/tenants/{id}/allowlist` (Admin)

Add an entity to a tenant's allowlist.

```
Body:
{
  "entity_value": "support@company.de",
  "pii_type": "EMAIL",
  "match_type": "exact",
  "reason": "Public support email, not personal PII"
}

Response 201: Allowlist entry created
```

### 9.6 `GET /audit/redactions` (Admin/Compliance)

Query redaction events. Requires `admin` or `compliance` role.

```
Query params:
  ?tenant_id=uuid          # filter by tenant
  ?user_id=string          # filter by user
  ?direction=input         # input | output
  ?pii_type=EMAIL          # filter by PII type
  ?from=2025-01-01         # date range start
  ?to=2025-01-31           # date range end
  ?page=1&limit=50

Response 200:
{
  "events": [
    {
      "id": "uuid",
      "timestamp": "2025-01-15T10:30:00Z",
      "tenant_id": "uuid",
      "user_id": "user-123",
      "request_id": "uuid",
      "direction": "input",
      "pii_type": "EMAIL",
      "detection_layer": "regex",
      "detection_score": 1.0,
      "detected_text": "[REDACTED]",     # or actual text if compliance role
      "redaction_strategy": "mask",
      "redacted_text": "[REDACTED_EMAIL]",
      "model_name": "gpt-4o-mini",
      "language": "de"
    }
  ],
  "total": 5432,
  "page": 1
}
```

### 9.7 `GET /audit/summary` (Admin/Compliance)

Aggregate redaction statistics for a date range.

```
Query params:
  ?tenant_id=uuid
  ?from=2025-01-01&to=2025-01-31

Response 200:
{
  "total_redactions": 5432,
  "input_redactions": 4890,
  "output_redactions": 542,
  "blocked_responses": 3,
  "pii_type_distribution": {
    "EMAIL": 2100,
    "PERSON": 1800,
    "STEUER_ID": 450,
    "PHONE": 320,
    "ADDRESS_DE": 280,
    "IBAN": 150,
    "OTHER": 332
  },
  "detection_layer_distribution": {
    "regex": 3200,
    "ner": 1900,
    "llm_contextual": 332
  },
  "avg_latency_ms": {
    "input_scan": 45,
    "llm_call": 620,
    "output_scan": 38,
    "total": 703
  }
}
```

### 9.8 `GET /health`

```
Response 200:
{
  "status": "healthy",
  "database": "connected",
  "spacy_models": ["en_core_web_lg", "de_core_news_lg"],
  "upstream_llm": "reachable",
  "version": "1.0.0"
}
```

---

## 10. Testing Strategy

### 10.1 Test Categories

| Category | What It Proves | Priority |
|----------|---------------|----------|
| **Security tests** | No raw PII in forwarded prompts; no PII in returned responses | P0 — must pass before any merge |
| **German PII tests** | Steuer-ID, IBAN-DE, PLZ, German phone, German address detected + redacted | P0 |
| **Regex tests** | All patterns match correctly; no false positives on benign text; no ReDoS | P0 |
| **Integration tests** | Full proxy pipeline: client → scan → LLM → scan → client | P0 |
| **Compliance tests** | Audit log immutability; every redaction event logged; completeness | P0 |
| **Tenant config tests** | Per-tenant rules, allowlist, custom patterns work correctly | P1 |
| **Redaction tests** | Mask, hash, synthetic strategies produce correct output | P1 |
| **NER tests** | Presidio + spaCy detect persons, orgs, locations in en + de | P1 |
| **LLM detector tests** | Layer 3 catches implicit PII that regex/NER miss | P1 |
| **Dashboard tests** | Dashboard queries return correct data | P2 |

### 10.2 Critical Security Tests (`tests/test_security.py`)

```python
"""
These tests are the backbone of the security claim.
If ANY of these fail, the system is not safe to deploy — PII is leaking.
"""

import json
from unittest.mock import AsyncMock, patch

async def test_no_raw_pii_in_forwarded_prompt():
    """Send a prompt with PII via proxy. Inspect what gets forwarded
    to the upstream LLM API. The forwarded prompt MUST NOT contain
    any raw PII."""
    pii_prompt = (
        "My name is Anna Müller, email: a.mueller@gmx.de, "
        "Steuer-ID: 12345678901, IBAN: DE89370400440532013000, "
        "I live at Hauptstr. 42, 24103 Kiel"
    )
    # Mock the upstream LLM client to capture what gets forwarded
    forwarded_body = {}
    async def mock_forward(path, body, headers):
        forwarded_body.update(body)
        return {"choices": [{"message": {"content": "OK"}}]}

    with patch.object(llm_client, "forward", mock_forward):
        response = await client.post("/v1/chat/completions",
            json={"model": "gpt-4o-mini",
                  "messages": [{"role": "user", "content": pii_prompt}]},
            headers={"Authorization": f"Bearer {token}"})

    forwarded_content = forwarded_body["messages"][0]["content"]
    # Critical assertions: no raw PII in forwarded content
    assert "Anna Müller" not in forwarded_content
    assert "a.mueller@gmx.de" not in forwarded_content
    assert "12345678901" not in forwarded_content
    assert "DE89370400440532013000" not in forwarded_content
    assert "Hauptstr. 42" not in forwarded_content
    assert "24103" not in forwarded_content
    # Redaction markers should be present
    assert "[REDACTED_" in forwarded_content or "[HASH_" in forwarded_content


async def test_no_pii_in_returned_response():
    """Mock the upstream LLM to return PII in its response.
    The client MUST NOT receive the raw PII."""
    pii_response = {
        "choices": [{
            "message": {
                "content": "Sure, I can help. Your email is a.mueller@gmx.de "
                           "and your Steuer-ID is 12345678901."
            }
        }]
    }
    with patch.object(llm_client, "forward", return_value=pii_response):
        response = await client.post("/v1/chat/completions",
            json={"model": "gpt-4o-mini",
                  "messages": [{"role": "user", "content": "Tell me my data"}]},
            headers={"Authorization": f"Bearer {token}"})

    content = response.json()["choices"][0]["message"]["content"]
    assert "a.mueller@gmx.de" not in content
    assert "12345678901" not in content
    assert "[REDACTED_" in content  # PII was sanitized


async def test_output_block_mode_returns_422():
    """Tenant with output_mode='block': LLM returns PII → 422."""
    tenant_config.output_mode = "block"
    pii_response = {
        "choices": [{"message": {"content": "Your SSN is 123-45-6789"}}]
    }
    with patch.object(llm_client, "forward", return_value=pii_response):
        response = await client.post("/v1/chat/completions",
            json={"model": "gpt-4o-mini",
                  "messages": [{"role": "user", "content": "What is my SSN?"}]},
            headers={"Authorization": f"Bearer {token}"})
    assert response.status_code == 422


async def test_redaction_event_logged_for_every_detection():
    """Every PII detection must produce a redaction_events row."""
    pii_prompt = "Email: test@mail.de, Phone: +49 30 12345678"
    await client.post("/v1/chat/completions",
        json={"model": "gpt-4o-mini",
              "messages": [{"role": "user", "content": pii_prompt}]},
        headers={"Authorization": f"Bearer {token}"})

    events = await db.fetch(
        "SELECT * FROM redaction_events WHERE direction='input'"
    )
    assert len(events) >= 2  # at least EMAIL + PHONE
    assert all(e["pii_type"] in ("EMAIL", "PHONE") for e in events)


async def test_audit_log_is_immutable():
    """Attempt to UPDATE and DELETE redaction_events → must raise."""
    with pytest.raises(Exception):
        await db.execute("UPDATE redaction_events SET redacted_text = 'tampered'")
    with pytest.raises(Exception):
        await db.execute("DELETE FROM redaction_events")


async def test_allowlisted_entity_not_redacted():
    """Tenant allowlists 'support@company.de'. That email must pass through."""
    await add_allowlist(tenant_id, "support@company.de", "EMAIL")
    forwarded_body = {}
    async def mock_forward(path, body, headers):
        forwarded_body.update(body)
        return {"choices": [{"message": {"content": "OK"}}]}

    with patch.object(llm_client, "forward", mock_forward):
        await client.post("/v1/chat/completions",
            json={"model": "gpt-4o-mini",
                  "messages": [{"role": "user",
                   "content": "Email support@company.de for help"}]},
            headers={"Authorization": f"Bearer {token}"})

    forwarded = forwarded_body["messages"][0]["content"]
    assert "support@company.de" in forwarded  # NOT redacted


async def test_scanner_failure_blocks_request():
    """If the scanner crashes, the request must NOT be forwarded with PII.
    Fail-safe: block, don't leak."""
    with patch.object(input_scanner, "scan", side_effect=RuntimeError("crash")):
        response = await client.post("/v1/chat/completions",
            json={"model": "gpt-4o-mini",
                  "messages": [{"role": "user", "content": "My SSN is 123-45-6789"}]},
            headers={"Authorization": f"Bearer {token}"})
    assert response.status_code == 503  # fail-safe block
```

### 10.3 German PII Tests (`tests/test_german_pii.py`)

```python
"""
German-specific PII detection tests. All must pass for German market deployment.
"""

class TestGermanSteuerID:
    def test_valid_steuer_id(self, detector):
        """Steuer-ID is 11 digits."""
        detections = detector.detect("Meine Steuer-ID ist 12345678901")
        assert any(d.pii_type == PIIType.STEUER_ID for d in detections)

    def test_steuer_id_not_confused_with_other_11_digits(self, detector):
        """11 digits in a non-Steuer-ID context should have lower confidence."""
        detections = detector.detect("Order number: 12345678901")
        # Should detect but with lower confidence or not at all
        steuer = [d for d in detections if d.pii_type == PIIType.STEUER_ID]
        if steuer:
            assert steuer[0].score < 0.9  # uncertain


class TestGermanIBAN:
    def test_german_iban(self, detector):
        """German IBAN: DE + 22 characters."""
        detections = detector.detect("IBAN: DE89370400440532013000")
        assert any(d.pii_type == PIIType.IBAN for d in detections)
        assert iban_validate("DE89370400440532013000")  # valid checksum

    def test_german_iban_with_spaces(self, detector):
        detections = detector.detect("IBAN: DE89 3704 0044 0532 0130 00")
        assert any(d.pii_type == PIIType.IBAN for d in detections)


class TestGermanPhone:
    def test_german_phone_with_country_code(self, detector):
        detections = detector.detect("Call me at +49 30 12345678")
        assert any(d.pii_type == PIIType.PHONE for d in detections)

    def test_german_mobile(self, detector):
        detections = detector.detect("My mobile: +49 151 12345678")
        assert any(d.pii_type == PIIType.PHONE for d in detections)

    def test_german_phone_no_country_code(self, detector):
        detections = detector.detect("Tel: 030 12345678")
        assert any(d.pii_type == PIIType.PHONE for d in detections)


class TestGermanAddress:
    def test_full_german_address(self, detector):
        detections = detector.detect("Ich wohne in der Hauptstraße 42, 24103 Kiel")
        assert any(d.pii_type == PIIType.ADDRESS_DE for d in detections)

    def test_german_postal_code(self, detector):
        detections = detector.detect("PLZ: 24103")
        assert any(d.pii_type == PIIType.PLZ for d in detections)

    def test_german_postal_code_range(self, detector):
        """German PLZ range: 01001–99998."""
        detections = detector.detect("PLZ: 80331")
        assert any(d.pii_type == PIIType.PLZ for d in detections)

    def test_non_german_postal_code_not_matched(self, detector):
        """US ZIP code 90210 should not match German PLZ pattern in de context."""
        # In English context, 90210 should not be PLZ
        detections = detector.detect("The ZIP is 90210", language="en")
        plz = [d for d in detections if d.pii_type == PIIType.PLZ]
        # In en context, PLZ should not trigger; or if it does, low confidence
        if plz:
            assert plz[0].score < 0.7
```

### 10.4 Test Fixtures (`tests/conftest.py`)

```python
"""
Test setup:
  - Spin up a test Postgres (docker or testcontainers)
  - Run schema.sql
  - Seed two tenants (acme-corp, globex-gmbh) with different configs
  - Provide JWT tokens for each tenant
  - Provide a FastAPI TestClient with mocked upstream LLM
  - Provide a RegexDetector, NERDetector, RedactionEngine
"""

@pytest.fixture
async def db():
    """Fresh database for each test module."""
    conn = await asyncpg.connect(TEST_DB_URL)
    await conn.execute(open("src/db/schema.sql").read())
    yield conn
    await conn.close()

@pytest.fixture
def tenant_acme_config():
    """Tenant that redacts all PII with mask strategy."""
    return TenantConfig(
        tenant_id="acme-uuid",
        enabled_pii_types=["EMAIL", "PHONE", "PERSON", "STEUER_ID", "IBAN",
                           "CREDIT_CARD", "ADDRESS_DE", "PLZ", "SSN", "DATE"],
        redaction_strategies={"EMAIL": "mask", "PERSON": "mask",
                              "STEUER_ID": "hash", "IBAN": "hash"},
        llm_contextual_enabled=True,
        output_mode="sanitize",
        language="auto",
    )

@pytest.fixture
def tenant_globex_config():
    """Tenant that uses synthetic redaction and allows emails."""
    return TenantConfig(
        tenant_id="globex-uuid",
        enabled_pii_types=["PERSON", "STEUER_ID", "IBAN", "ADDRESS_DE", "PLZ"],
        redaction_strategies={"PERSON": "synthetic", "ADDRESS_DE": "synthetic",
                              "STEUER_ID": "hash", "IBAN": "hash"},
        llm_contextual_enabled=False,
        output_mode="block",
        language="de",
    )

@pytest.fixture
def token_acme():
    return generate_jwt(tenant_id="acme-uuid", user_id="user-123", roles=["user"])

@pytest.fixture
def token_globex():
    return generate_jwt(tenant_id="globex-uuid", user_id="user-456", roles=["user"])

@pytest.fixture
def admin_token():
    return generate_jwt(tenant_id="acme-uuid", user_id="admin-1", roles=["admin"])

@pytest.fixture
def regex_detector():
    return RegexDetector()

@pytest.fixture
def ner_detector():
    return NERDetector()

@pytest.fixture
def redaction_engine():
    return RedactionEngine(
        hash_salt="test-salt",
        synthetic_gen=SyntheticDataGenerator(locale="de_DE"),
    )
```

### 10.5 Evaluation Script (`scripts/evaluate_recall.py`)

```python
"""
Evaluates detection recall and precision on a labeled PII corpus.
The corpus is a set of text samples with manually annotated PII spans.

Usage:
    python scripts/evaluate_recall.py --corpus tests/fixtures/labeled_pii.jsonl

Output:
    Per-layer and combined recall/precision/F1 scores.
    Must achieve combined recall >= 0.98 for production readiness.
"""

import json
from pathlib import Path
from src.detection.pipeline import DetectionPipeline

def evaluate(corpus_path: str, pipeline: DetectionPipeline):
    tp = fp = fn = 0  # true positives, false positives, false negatives
    per_layer_stats = {"regex": {"tp": 0, "fp": 0, "fn": 0},
                       "ner": {"tp": 0, "fp": 0, "fn": 0},
                       "llm_contextual": {"tp": 0, "fp": 0, "fn": 0}}

    with open(corpus_path) as f:
        for line in f:
            sample = json.loads(line)
            text = sample["text"]
            expected = sample["annotations"]  # [{start, end, pii_type}, ...]

            detections = pipeline.detect_sync(text)
            detected_spans = {(d.start, d.end, d.pii_type.value) for d in detections}
            expected_spans = {(a["start"], a["end"], a["pii_type"]) for a in expected}

            tp += len(detected_spans & expected_spans)
            fp += len(detected_spans - expected_spans)
            fn += len(expected_spans - detected_spans)

    precision = tp / (tp + fp) if (tp + fp) > 0 else 0
    recall = tp / (tp + fn) if (tp + fn) > 0 else 0
    f1 = 2 * precision * recall / (precision + recall) if (precision + recall) > 0 else 0

    print(f"Combined:  Precision={precision:.4f}  Recall={recall:.4f}  F1={f1:.4f}")
    assert recall >= 0.98, f"Recall {recall:.4f} below 0.98 threshold"
```

---

## 11. Deployment

### 11.1 `docker-compose.yml`

```yaml
version: "3.9"

services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: pii_guardrails
      POSTGRES_USER: guardrail_user
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-changeme}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./docker/postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U guardrail_user -d pii_guardrails"]
      interval: 5s
      timeout: 5s
      retries: 5

  proxy:
    build:
      context: .
      dockerfile: docker/Dockerfile
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgres://guardrail_user:${POSTGRES_PASSWORD:-changeme}@postgres:5432/pii_guardrails
      JWT_SECRET: ${JWT_SECRET:-dev-secret-change-in-prod}
      UPSTREAM_LLM_BASE_URL: ${UPSTREAM_LLM_BASE_URL:-https://api.openai.com}
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      LLM_CONTEXTUAL_MODEL: gpt-4o-mini
      HASH_SALT: ${HASH_SALT:-dev-salt-change-in-prod}
      SPACY_MODELS: "en_core_web_lg,de_core_news_lg"
      LOG_LEVEL: ${LOG_LEVEL:-info}
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 10s
      timeout: 5s
      retries: 3
    volumes:
      - ./patterns:/app/patterns:ro

  dashboard:
    build:
      context: .
      dockerfile: docker/Dockerfile.dashboard
    ports:
      - "8501:8501"
    environment:
      DATABASE_URL: postgres://guardrail_user:${POSTGRES_PASSWORD:-changeme}@postgres:5432/pii_guardrails
    depends_on:
      postgres:
        condition: service_healthy

volumes:
  postgres_data:
```

### 11.2 `docker/Dockerfile`

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install system deps
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential libpq-dev curl \
    && rm -rf /var/lib/apt/lists/*

# Install Python deps
COPY pyproject.toml .
RUN pip install --no-cache-dir -e .

# Download spaCy models during build (not at runtime)
RUN python -m spacy download en_core_web_lg \
    && python -m spacy download de_core_news_lg

# Copy source
COPY src/ ./src/
COPY patterns/ ./patterns/
COPY scripts/ ./scripts/

EXPOSE 8000

CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 11.3 `docker/Dockerfile.dashboard`

```dockerfile
FROM python:3.12-slim

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential libpq-dev \
    && rm -rf /var/lib/apt/lists/*

COPY pyproject.toml .
RUN pip install --no-cache-dir streamlit plotly psycopg2-binary pandas reportlab

COPY dashboard/ ./dashboard/

EXPOSE 8501

CMD ["streamlit", "run", "dashboard/app.py", "--server.port=8501", "--server.address=0.0.0.0"]
```

### 11.4 `docker/postgres/init.sql`

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
-- Schema tables are created by scripts/init_db.py on first startup
```

### 11.5 `.env.example`

```env
# Database
POSTGRES_PASSWORD=changeme-in-production

# JWT
JWT_SECRET=your-256-bit-secret
JWT_ALGORITHM=HS256
JWT_EXPIRY_HOURS=24

# Upstream LLM API (where the proxy forwards redacted requests)
UPSTREAM_LLM_BASE_URL=https://api.openai.com
OPENAI_API_KEY=sk-...

# Layer 3 LLM (for contextual detection — can be same as upstream or different)
LLM_CONTEXTUAL_MODEL=gpt-4o-mini

# Hash salt for hash redaction strategy
HASH_SALT=your-hmac-salt-change-in-prod

# spaCy models
SPACY_MODELS=en_core_web_lg,de_core_news_lg

# App
LOG_LEVEL=info
MAX_REQUEST_SIZE_MB=10
SCANNER_TIMEOUT_SECONDS=30

# Local LLM alternative (for Layer 3, instead of OpenAI)
# LLM_CONTEXTUAL_MODEL=llama3.1:8b
# OLLAMA_HOST=http://localhost:11434
```

### 11.6 Production Considerations

| Concern | Recommendation |
|---------|---------------|
| **TLS** | Terminate TLS at reverse proxy (nginx/traefik); proxy listens on HTTP |
| **DB encryption** | PostgreSQL TDE or disk-level LUKS; `detected_text` column contains PII |
| **Secrets** | Use Docker secrets or a vault; never bake secrets into images |
| **DB user privileges** | App uses limited-privilege user (INSERT on redaction_events/audit_logs; SELECT/INSERT on tenants/config/allowlist); compliance officer has separate read-only role |
| **Rate limiting** | slowapi middleware (e.g. 20 requests/min per user); protects against DoS |
| **Scanner timeout** | If scanner exceeds `SCANNER_TIMEOUT_SECONDS`, request is blocked (fail-safe) |
| **Monitoring** | Structured JSON logs → Loki/ELK; `/health` endpoint for k8s probes; alert on output PII detection rate spike |
| **Retention** | `redaction_events.detected_text` expires after 2 years (retention cron); metadata kept indefinitely |
| **Upstream LLM redundancy** | Configure multiple upstream endpoints; failover on 5xx |
| **Layer 3 cost control** | Layer 3 only runs when heuristic triggers (PII cue words + low detection count); cap text length at 2000 chars |

---

## 12. Roadmap & Milestones

```
Week 1 (Days 1-4)
├── Phase 1: Foundation + Proxy Skeleton    [█░░░░░░] Day 1
├── Phase 2: Layer 1 — Regex + German       [░█░░░░░] Day 2
├── Phase 3: Layer 2+3 — NER + LLM          [░░█░░░░] Day 3
└── Phase 4: Redaction + Tenant Config      [░░░█░░░] Day 4

Week 1.5 (Days 5-7)
├── Phase 5: Scanners + Proxy Integration   [░░░░█░░] Day 5
├── Phase 6: Dashboard + Admin API          [░░░░░█░] Day 6
└── Phase 7: Hardening + Evaluation + Docs  [░░░░░░█] Day 7

Future (Post-v1)
├── Streaming response redaction (SSE)
├── Reversible PII vault (encrypt at rest with key management)
├── Image/document PII redaction (OCR + image masking)
├── Additional languages (fr, es, it)
├── Custom NER model fine-tuning for domain-specific PII
├── Kubernetes deployment manifests + Helm chart
├── OAuth2/OIDC integration (replace simple JWT)
├── Real-time alerting on PII leakage spikes (Prometheus alerts)
├── Differential privacy guarantees for aggregate statistics
└── Integration with external SIEM (Splunk, Elastic SIEM)
```

### Milestone Summary

| Milestone | Day | Deliverable |
|-----------|-----|-------------|
| M1: Proxy skeleton | Day 1 | Docker Compose up, proxy forwards to LLM, auth works, health check passes |
| M2: Regex detection | Day 2 | All regex patterns (en + de) detect correctly; German PII tests green |
| M3: Three-layer pipeline | Day 3 | Regex + NER + LLM contextual all functional; merger resolves overlaps |
| M4: Redaction engine | Day 4 | Mask/hash/synthetic strategies work; per-tenant config drives strategy |
| M5: Full proxy pipeline | Day 5 | Input scan → redact → LLM → output scan → sanitize; compliance log wired; security tests green |
| M6: Dashboard | Day 6 | Streamlit shows live stats; CSV/PDF report export; admin API works |
| M7: Production-ready | Day 7 | Full test suite green; recall ≥ 98% on labeled corpus; documented; deployable |

---

## 13. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| **False negatives — PII not detected** | Medium | Critical | Three-layer pipeline compensates for individual blind spots; Layer 3 (LLM) catches implicit PII; recall evaluation on labeled corpus (≥ 98% threshold); continuous improvement with missed-detection analysis |
| **False positives — benign text redacted** | Medium | Medium | Confidence scoring (NER/LLM layers); allowlist for known-safe entities; tenant can disable specific PII types; monitor false positive rate in dashboard |
| **Layer 3 LLM hallucinates PII that isn't there** | Medium | Medium | LLM detector only runs when heuristic triggers; LLM output validated against original text (span must exist); lower confidence score (0.7) for LLM detections |
| **Regex ReDoS — catastrophic backtracking** | Low | High | Use `regex` library (PCRE2) with timeout; patterns pre-tested for backtracking; input length limits; patterns use atomic groups where possible |
| **Compliance log contains PII (detected_text column)** | High | High | DB encrypted at rest; separate compliance officer DB role; retention policy expires `detected_text` after 2 years; app-level column encryption in v2 |
| **Upstream LLM API downtime** | Medium | Medium | Retry with exponential backoff; configure multiple upstream endpoints; failover; 502 error to client (no PII leaked) |
| **Scanner timeout under high load** | Medium | High | Fail-safe: timeout → block request (503), never forward with PII; async processing; horizontal scaling of proxy instances |
| **Tenant config misconfiguration disables redaction** | Low | Critical | Default-deny: no config → all PII redacted; config changes require admin role + are audit-logged; dashboard flags tenants with unusual configs |
| **Allowlist abuse (allowlist all emails)** | Low | High | Allowlist entries require reason field; dashboard flags large allowlists; periodic admin review; limit allowlist size per tenant |
| **spaCy model unavailable in container** | Low | Medium | Models downloaded during Docker build (not at runtime); health check verifies model availability; fallback to smaller models |
| **Layer 3 LLM cost spirals** | Medium | Low | Heuristic gating (only runs when cue words present + low detection count); text length cap (2000 chars); per-tenant enable/disable; cost monitoring in dashboard |
| **GDPR right to erasure — user requests all their PII deleted** | Medium | Medium | `redaction_events` and `audit_logs` are immutable (triggers block DELETE); GDPR Article 17(3)(e) exemption for compliance logs; retention policy auto-expires `detected_text`; metadata can be pseudonymized |

---

## Appendix A: Quick Start

```bash
# 1. Clone
git clone <repo-url> pii-guardrails
cd pii-guardrails

# 2. Configure
cp .env.example .env
# Edit .env: set OPENAI_API_KEY, JWT_SECRET, POSTGRES_PASSWORD, HASH_SALT,
#            UPSTREAM_LLM_BASE_URL

# 3. Start
docker compose up -d

# 4. Wait for health
curl http://localhost:8000/health
# → {"status": "healthy", "database": "connected",
#    "spacy_models": ["en_core_web_lg", "de_core_news_lg"], ...}

# 5. Initialize database (creates schema, triggers, RLS)
python scripts/init_db.py

# 6. Seed test data (creates tenants, configs, allowlists)
python scripts/seed_test_data.py

# 7. Generate a test JWT
python scripts/generate_jwt.py --tenant acme-corp --user user-123 --roles user

# 8. Send a request through the proxy (PII gets redacted)
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [
      {"role": "user", "content": "My name is Anna Müller, email a.mueller@gmx.de, Steuer-ID 12345678901. Summarize my data."}
    ]
  }'

# Response headers will include:
#   X-PII-Redacted: true
#   X-Redaction-Count: 3
# The forwarded prompt (to OpenAI) will contain:
#   "My name is [REDACTED_PERSON], email [REDACTED_EMAIL], Steuer-ID [HASH_a3f9b2c1]..."

# 9. Open the dashboard
open http://localhost:8501
# → See redaction statistics, PII type distribution, export compliance report

# 10. Query audit trail (admin only)
curl http://localhost:8000/audit/redactions?pii_type=EMAIL \
  -H "Authorization: Bearer <ADMIN_TOKEN>"
```

### Client Integration (Drop-in Proxy)

```python
# Before: client calls OpenAI directly
from openai import OpenAI
client = OpenAI(base_url="https://api.openai.com/v1", api_key="sk-...")

# After: client calls through PII guardrail proxy
from openai import OpenAI
client = OpenAI(
    base_url="http://localhost:8000/v1",   # ← point at the proxy
    api_key="<JWT_TOKEN>",                  # ← JWT instead of API key
)

# That's it. No other code changes. The proxy handles PII redaction.
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "My name is Anna Müller, email a.mueller@gmx.de"}],
)
# The LLM never sees "Anna Müller" or "a.mueller@gmx.de".
```

---

## Appendix B: Key Design Decisions

| Decision | Chosen | Rejected Alternative | Why |
|----------|--------|----------------------|-----|
| Architecture | Proxy middleware | Library/SDK | Proxy is in the path of traffic — no path to LLM bypasses the scanner; library can be forgotten or skipped |
| Detection | Three-layer (regex + NER + LLM) | Single-layer (regex only or NER only) | No single layer is sufficient; regex misses names, NER misses formats, LLM catches implicit PII; layers compensate for each other |
| NER engine | Presidio + spaCy | Standalone spaCy, Stanford NER, AWS Comprehend | Presidio is purpose-built for PII; orchestrates spaCy + custom recognizers; open-source; supports custom patterns; industry standard |
| Redaction strategies | Mask + Hash + Synthetic | Encrypt/decrypt (reversible) | Mask is clear to LLM; hash allows audit re-identification; synthetic preserves sentence structure; reversible encryption adds key management complexity and risk (v2 concern) |
| Hash method | HMAC-SHA256 with salt | SHA-256 without salt, MD5 | HMAC with salt prevents rainbow table attacks; deterministic for audit correlation; non-reversible without salt |
| Synthetic data | Faker with de_DE locale | Fixed placeholders ("John Doe") | Faker generates realistic, varied German names/addresses; preserves sentence structure better; locale-appropriate |
| Compliance log | DB table with triggers | External log file / SIEM | DB table is queryable via API; triggers guarantee immutability; structured data for dashboard; SIEM integration is a v2 concern |
| Tenant config | DB-driven (JSONB columns) | YAML config files | DB-driven allows runtime changes via admin API; per-tenant isolation; no restart needed; audit trail of config changes |
| Layer 3 gating | Heuristic (cue words + low detection count) | Always on / Always off | Always on is too costly; always off misses implicit PII; heuristic balances cost and coverage |
| Language detection | Auto-detect (en/de) via heuristic | Manual per-tenant config | Auto-detect handles mixed-language prompts; tenant can override with `language: "de"` if all prompts are German |
| Output mode | Configurable (sanitize or block) | Always sanitize / Always block | Some tenants need the response (sanitize); others have strict policy (block); configurable per tenant |
| Dashboard | Streamlit | Grafana, custom React app | Streamlit is rapid to build; internal tooling doesn't need polish; charts + tables + export with minimal code; Grafana requires separate infra |

---

## Appendix C: Regex Pattern Catalog

### C.1 `patterns/regex_catalog.yaml` (English + Universal Patterns)

```yaml
# PII Regex Pattern Catalog
# Each pattern has: name, pii_type, pattern (PCRE2), optional validator, priority
# Validators: "luhn" (credit card), "iban_checksum" (IBAN mod-97)

patterns:
  # --- Email ---
  - name: email
    pii_type: EMAIL
    pattern: '\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b'
    priority: 10

  # --- Phone (international, US, UK formats) ---
  - name: phone_international
    pii_type: PHONE
    pattern: '\+?\d{1,3}[\s.-]?\(?\d{1,4}\)?[\s.-]?\d{3,4}[\s.-]?\d{3,4}\b'
    priority: 5

  - name: phone_us
    pii_type: PHONE
    pattern: '\b\(?\d{3}\)?[\s.-]?\d{3}[\s.-]?\d{4}\b'
    priority: 7

  # --- US SSN ---
  - name: us_ssn
    pii_type: SSN
    pattern: '\b(?!000|666|9\d{2})\d{3}-(?!00)\d{2}-(?!0000)\d{4}\b'
    priority: 10

  # --- Credit Card (Visa, Mastercard, Amex, Discover) ---
  - name: credit_card_visa
    pii_type: CREDIT_CARD
    pattern: '\b4\d{3}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b'
    validator: luhn
    priority: 10

  - name: credit_card_mastercard
    pii_type: CREDIT_CARD
    pattern: '\b(?:5[1-5]\d{2}|2[2-7]\d{3})[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b'
    validator: luhn
    priority: 10

  - name: credit_card_amex
    pii_type: CREDIT_CARD
    pattern: '\b3[47]\d{2}[\s-]?\d{6}[\s-]?\d{5}\b'
    validator: luhn
    priority: 10

  # --- IBAN (generic, validated with mod-97) ---
  - name: iban_generic
    pii_type: IBAN
    pattern: '\b[A-Z]{2}\d{2}[\s-]?(?:\d{4}[\s-]?){4,7}\d{1,4}\b'
    validator: iban_checksum
    priority: 10

  # --- Passport (US, UK) ---
  - name: passport_us
    pii_type: PASSPORT
    pattern: '\b(?!\d{9})[A-Z]\d{8}\b'
    priority: 7

  - name: passport_uk
    pii_type: PASSPORT
    pattern: '\b\d{9}[A-Z]{2}\b'
    priority: 7

  # --- IP Address ---
  - name: ip_address
    pii_type: IP_ADDRESS
    pattern: '\b(?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\b'
    priority: 3

  # --- Date of Birth (various formats) ---
  - name: date_iso
    pii_type: DATE
    pattern: '\b(?:19|20)\d{2}-(?:0[1-9]|1[0-2])-(?:0[1-9]|[12]\d|3[01])\b'
    priority: 5

  - name: date_us
    pii_type: DATE
    pattern: '\b(?:0[1-9]|1[0-2])/(?:0[1-9]|[12]\d|3[01])/(?:19|20)\d{2}\b'
    priority: 5
```

### C.2 `patterns/german_patterns.yaml` (German-Specific Patterns)

```yaml
# German-Specific PII Regex Patterns
# Critical for GDPR compliance in the German market

patterns:
  # --- German Steuer-ID (Tax Identification Number) ---
  # Format: 11 digits (e.g. 12345678901)
  # Issued by Bundeszentralamt für Steuern, lifelong unique
  - name: german_steuer_id
    pii_type: STEUER_ID
    pattern: '\b\d{11}\b'
    priority: 8
    # Note: 11-digit number is ambiguous; NER context disambiguates
    # Lower confidence (0.85) unless preceded by "Steuer-ID" or "Steuernummer"

  # --- German IBAN ---
  # Format: DE + 22 characters (2 country + 2 checksum + 22 BBAN)
  # Example: DE89 3704 0044 0532 0130 00
  - name: german_iban
    pii_type: IBAN
    pattern: '\bDE\d{2}[\s-]?(?:\d{4}[\s-]?){4}\d{2}\b'
    validator: iban_checksum
    priority: 10

  # --- German Phone Numbers ---
  # Format: +49 (country code) + area code + number
  # Examples: +49 30 12345678, +49 151 12345678, 030 12345678
  - name: german_phone_international
    pii_type: PHONE
    pattern: '\+49[\s.-]?\d{2,5}[\s.-]?\d{3,8}\b'
    priority: 9

  - name: german_phone_domestic
    pii_type: PHONE
    pattern: '\b0\d{2,5}[\s.-]?\d{3,8}\b'
    priority: 6
    # Note: domestic format (0XX ...) is common; lower priority to avoid
    # false positives on numbers that aren't phone numbers

  - name: german_mobile
    pii_type: PHONE
    pattern: '\+49[\s.-]?1[5-7]\d[\s.-]?\d{6,8}\b'
    priority: 9

  # --- German Postal Code (PLZ) ---
  # Format: 5 digits (01001–99998)
  # Example: 24103 (Kiel), 80331 (Munich)
  - name: german_plz
    pii_type: PLZ
    pattern: '\b\d{5}\b'
    priority: 4
    # Note: 5-digit number is ambiguous; use context (preceded by "PLZ" or
    # followed by city name) to increase confidence

  # --- German Address (full) ---
  # Format: StreetName + number + PLZ + City
  # Example: Hauptstraße 42, 24103 Kiel
  # German street suffixes: straße, str., weg, gasse, allee, platz, ring, ufer
  - name: german_address_full
    pii_type: ADDRESS_DE
    pattern: '\b[A-ZÄÖÜ][a-zäöüß]+(?:straße|str\.|weg|gasse|allee|platz|ring|ufer|damm|chaussee)\.?\s+\d+[a-zA-Z]?,?\s+\d{5}\s+[A-ZÄÖÜ][a-zäöüß]+'
    priority: 8

  # --- German Street (without PLZ/City) ---
  - name: german_street_only
    pii_type: ADDRESS_DE
    pattern: '\b[A-ZÄÖÜ][a-zäöüß]+(?:straße|str\.|weg|gasse|allee|platz|ring|ufer|damm|chaussee)\.?\s+\d+[a-zA-Z]?\b'
    priority: 5

  # --- German Passport Number ---
  # Format: C, F, G, H, J, K, L, N, P, R, T, V, X, Y, Z + 8 digits
  # Example: C01X00T47 (new format since 2017)
  - name: german_passport
    pii_type: PASSPORT
    pattern: '\b[CFGHJKLMNPRTVXYZ]\d{2}[A-Z]\d{5}\b'
    priority: 9

  # --- German Identity Card (Personalausweis) ---
  # Format: 10 alphanumeric characters (new format since 2021)
  # Example: L01X00T472
  - name: german_id_card
    pii_type: ID_CARD_DE
    pattern: '\bL\d{2}[A-Z]\d{6}\b'
    priority: 9

  # --- German Health Insurance Number (Krankenversichertennummer) ---
  # Format: Letter + 9 digits (old) or 10 alphanumeric (new)
  - name: german_health_insurance
    pii_type: HEALTH_INSURANCE_DE
    pattern: '\b[A-Z]\d{9}\b'
    priority: 6

  # --- German Driver's License Number (Führerscheinnummer) ---
  # Format: varies by state, generally alphanumeric
  - name: german_drivers_license
    pii_type: DRIVERS_LICENSE_DE
    pattern: '\b[A-ZÄÖÜ]{1,3}[\s.-]?\d{5,7}[\s.-]?\d{1,5}\b'
    priority: 4

  # --- German Tax Number (Steuernummer) ---
  # Format: varies by federal state, typically 10-13 digits
  # Different from Steuer-ID (which is 11 digits, lifelong)
  - name: german_tax_number
    pii_type: TAX_NUMBER_DE
    pattern: '\b\d{2}[\s.-]?\d{3}[\s.-]?\d{3}[\s.-]?\d{3}\b'
    priority: 5
```

### C.3 Pattern Priority and Confidence Notes

| PII Type | Pattern | Priority | Confidence | Notes |
|----------|---------|----------|------------|-------|
| EMAIL | RFC 5322 simplified | 10 | 1.0 | Very reliable; few false positives |
| STEUER_ID | 11 digits | 8 | 0.85 | Ambiguous (any 11-digit number); context required; NER + LLM layers disambiguate |
| IBAN (DE) | DE + 22 chars + mod-97 | 10 | 1.0 | Checksum validation makes this very reliable |
| PHONE (DE) | +49 or 0XX format | 6-9 | 0.8 | Many format variations; false positives on non-phone numbers |
| PLZ | 5 digits | 4 | 0.6 | Very ambiguous (any 5-digit number); context required |
| ADDRESS_DE | Street + number + PLZ + city | 8 | 0.75 | Complex pattern; NER layer provides backup |
| PASSPORT (DE) | Letter + 8 chars | 9 | 0.9 | Specific format; low false positive rate |
| CREDIT_CARD | Luhn-validated | 10 | 1.0 | Luhn checksum eliminates false positives |
| SSN (US) | 3-2-4 format | 10 | 0.95 | Area/group/serial validation rules |

---

## Appendix D: Redaction Strategy Comparison

| Strategy | Example Output | Pros | Cons | Best For | Reversible? |
|----------|---------------|------|------|----------|-------------|
| **Mask** | `[REDACTED_EMAIL]` | Clear to LLM that data was removed; simple; no external deps | Breaks sentence structure; LLM may behave differently; no audit correlation | Emails, SSNs, Steuer-IDs (where LLM doesn't need the value) | No |
| **Hash** | `[HASH_a3f9b2c1]` | Deterministic — same PII → same hash; allows audit re-identification without exposing PII; non-reversible without salt | Not human-readable; LLM sees opaque token; hash collisions (extremely rare with SHA-256) | IBANs, credit cards, Steuer-IDs (where audit correlation is needed) | No (without salt) |
| **Synthetic** | `Max Mustermann` | Preserves sentence structure; LLM processes naturally; realistic data; locale-appropriate (de_DE) | Fake data may confuse if LLM references it in response; requires Faker dependency; consistency within request | Person names, addresses, organizations (where sentence structure matters) | No |
| **Block** | `422 Unprocessable Entity` | Guarantees no PII in output; strictest policy; clear signal to user | User gets no response; poor UX; may block legitimate requests if false positive | Output mode for strict tenants (e.g. healthcare, finance) | N/A |

### Strategy Selection Guide

```
┌─────────────────────────────────────────────────────────────┐
│  When to use which strategy?                                │
│                                                             │
│  EMAIL      → mask (LLM doesn't need the actual email)      │
│  PHONE      → mask (LLM doesn't need the actual phone)      │
│  PERSON     → synthetic (LLM needs a name to process text)  │
│  ADDRESS    → synthetic (LLM needs an address for context)  │
│  STEUER_ID  → hash (audit correlation needed; LLM doesn't   │
│               need the value)                               │
│  IBAN       → hash (audit correlation; LLM doesn't need it) │
│  CREDIT_CARD→ hash (audit correlation; LLM doesn't need it) │
│  SSN        → mask (LLM never needs an SSN)                 │
│  ORG        → synthetic (LLM may need org name for context) │
│  DATE       → mask (dates are rarely needed by LLM)         │
│  PLZ        → mask (LLM rarely needs postal code)           │
│                                                             │
│  OUTPUT (all types) → mask (always mask in output mode;     │
│    synthetic in output would introduce fake data to user)   │
└─────────────────────────────────────────────────────────────┘
```

### Default Tenant Configuration

```json
{
  "enabled_pii_types": [
    "EMAIL", "PHONE", "PERSON", "STEUER_ID", "IBAN", "CREDIT_CARD",
    "SSN", "ADDRESS_DE", "PLZ", "DATE", "PASSPORT", "ORGANIZATION"
  ],
  "redaction_strategies": {
    "EMAIL": "mask",
    "PHONE": "mask",
    "PERSON": "synthetic",
    "STEUER_ID": "hash",
    "IBAN": "hash",
    "CREDIT_CARD": "hash",
    "SSN": "mask",
    "ADDRESS_DE": "synthetic",
    "PLZ": "mask",
    "DATE": "mask",
    "PASSPORT": "mask",
    "ORGANIZATION": "synthetic"
  },
  "llm_contextual_enabled": true,
  "output_mode": "sanitize",
  "language": "auto"
}
```

---

## Appendix E: GDPR Compliance Mapping

### GDPR Articles → System Features

| GDPR Article | Requirement | System Feature | Evidence |
|--------------|-------------|----------------|----------|
| **Art. 5(1)(b)** — Purpose limitation | Personal data collected for specified, explicit purposes | PII is redacted before reaching the LLM; the LLM's purpose is text completion, not PII processing | Proxy middleware intercepts all LLM calls; `redaction_events` log shows what was redacted |
| **Art. 5(1)(c)** — Data minimisation | Personal data adequate, relevant, limited to what is necessary | Only configured PII types are redacted; non-PII passes through; tenant can disable types | Tenant config `enabled_pii_types` array; dashboard shows what types are active |
| **Art. 5(1)(f)** — Integrity and confidentiality | Personal data processed in a manner ensuring appropriate security | PII never reaches the LLM API; compliance log is immutable; DB encrypted at rest | Security test suite (no PII in forwarded prompts); DB triggers; TLS |
| **Art. 6** — Lawfulness of processing | Processing must be lawful (consent, contract, legitimate interest) | Redaction itself is a data protection measure; the system reduces processing of PII by the LLM | Compliance log records all processing; tenant config is explicit |
| **Art. 9** — Special categories of data | Health, racial origin, political opinions, etc. require explicit consent | System can be configured to detect and redact health-related PII; custom regex patterns for medical IDs | German health insurance number pattern; custom patterns per tenant |
| **Art. 12** — Transparent information | Data subject informed about processing | Dashboard provides transparency; compliance report export; `X-PII-Redacted` header informs client | Streamlit dashboard; CSV/PDF report; response headers |
| **Art. 13** — Information to be provided | Identity of controller, purposes, recipients | Compliance log records tenant, user, model (recipient), timestamp | `redaction_events` and `audit_logs` tables |
| **Art. 15** — Right of access | Data subject can access their personal data | Audit API allows querying redaction events by `user_id`; compliance officer can export | `GET /audit/redactions?user_id=X` endpoint |
| **Art. 16** — Right to rectification | Data subject can correct inaccurate data | Redaction is applied to text in transit; original PII is in the compliance log (for audit); rectification is at the source system | Compliance log is the record of what was processed; source system handles rectification |
| **Art. 17** — Right to erasure (right to be forgotten) | Data subject can have personal data erased | Compliance log is immutable (triggers); `detected_text` expires after retention period (2 years); metadata is pseudonymized | Retention policy cron; `detected_text` → `[EXPIRED]`; Art. 17(3)(e) exemption for compliance records |
| **Art. 20** — Right to data portability | Data subject can receive their data in structured format | Compliance report export (CSV) provides structured data per user | `GET /audit/redactions?user_id=X` → CSV export |
| **Art. 25** — Data protection by design and by default | Data protection measures built into the system | Default-deny (all PII redacted if no config); proxy is in the path; three-layer detection | Default tenant config; proxy middleware architecture; security test suite |
| **Art. 30** — Records of processing activities | Controller maintains a record of processing activities | `audit_logs` table is the record of processing activities; one row per LLM API call | `audit_logs` schema with tenant, user, model, timestamp, redaction counts |
| **Art. 32** — Security of processing | Appropriate technical and organisational measures | TLS, DB encryption, JWT auth, RBAC, immutable audit log, fail-safe scanner | Security model (Section 8); DB triggers; TLS at reverse proxy |
| **Art. 33** — Notification of personal data breach | Notify supervisory authority within 72 hours | If PII leak is detected (output scanner finds PII), the event is logged with full context for breach investigation | `redaction_events` with `direction='output'`; dashboard alerts on output PII spike |
| **Art. 35** — Data protection impact assessment | DPIA required for high-risk processing | This system is itself a DPIA mitigation measure; documentation supports DPIA | This implementation plan; security model; risk register |

### SOC 2 Trust Services Criteria Mapping

| SOC 2 Criteria | Requirement | System Feature |
|----------------|-------------|----------------|
| **CC6.1** — Logical and physical access controls | Controls implemented to restrict access | JWT auth, RBAC (admin/compliance/user roles), DB role separation |
| **CC6.2** — User authentication | Authentication mechanisms | JWT verification; token expiry; tenant resolution |
| **CC7.1** — System monitoring | Detection of security events | Compliance log records all redaction events; dashboard monitors PII detection rates |
| **CC7.2** — Anomaly detection | Identification of anomalies | Output PII detection (LLM leaking PII is an anomaly); dashboard alerts on rate spikes |
| **CC7.3** — Incident response | Procedures for responding to incidents | Output PII events are logged with full context for incident investigation; configurable block mode |
| **CC8.1** — Change management | Controls over system changes | Tenant config changes are admin-only and audit-logged; schema migrations are versioned |
| **A1.2** — System availability | System available for operation | Health check endpoint; fail-safe (blocks on scanner failure rather than passing PII) |

---

*End of Implementation Plan*