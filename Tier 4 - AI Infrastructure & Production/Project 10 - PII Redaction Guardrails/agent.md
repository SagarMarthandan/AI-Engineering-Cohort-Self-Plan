# PII Redaction Guardrails — Herdr Multi-Agent Orchestration Guide

> **Runbook for orchestrating 4 codex agents + 1 reviewer across 7 phases (7 days).**
> Each phase maps to IMPLEMENTATION_PLAN.md sections. Prompts reference exact files, pseudocode, and plan sections.

---

## 1. Agent Roster

| Agent | Kind | Responsibility | Owns (files/dirs) |
|-------|------|----------------|-------------------|
| **detection** | codex | Three-layer PII detection pipeline: regex patterns (Layer 1), Presidio + spaCy NER (Layer 2), LLM contextual (Layer 3), merger/dedup, language detection, detection models, pattern catalogs, evaluation/benchmark scripts | `src/detection/`, `patterns/`, `src/proxy/language_detector.py`, `scripts/evaluate_recall.py`, `scripts/benchmark_detection.py`, `tests/test_regex_detector.py`, `tests/test_ner_detector.py`, `tests/test_llm_detector.py`, `tests/test_merger.py`, `tests/test_german_pii.py` |
| **middleware** | codex | FastAPI proxy middleware, LLM client forwarding, input/output scanners, redaction engine + strategies + synthetic data, app factory, config, auth, Dockerfile, docker-compose, proxy/health routes, security tests | `src/main.py`, `src/config.py`, `src/auth/`, `src/proxy/middleware.py`, `src/proxy/llm_client.py`, `src/scanner/`, `src/redaction/`, `src/api/routes/proxy.py`, `src/api/routes/health.py`, `src/api/dependencies.py`, `docker/Dockerfile`, `docker-compose.yml`, `tests/test_input_scanner.py`, `tests/test_output_scanner.py`, `tests/test_proxy.py`, `tests/test_security.py`, `tests/test_redaction.py`, `tests/conftest.py` |
| **compliance** | codex | Database schema + migrations + triggers + RLS, tenant config resolver + models, allowlist, compliance logger, audit/admin API routes, seed/init scripts, JWT generation, compliance + tenant config tests | `src/db/`, `src/tenant/`, `src/compliance/`, `src/api/routes/admin.py`, `src/api/routes/audit.py`, `scripts/init_db.py`, `scripts/seed_test_data.py`, `scripts/generate_jwt.py`, `docker/postgres/`, `tests/test_compliance.py`, `tests/test_tenant_config.py` |
| **dashboard** | codex | Streamlit dashboard: queries, charts, reports (CSV/PDF), app main, dashboard Dockerfile, dashboard tests | `dashboard/`, `docker/Dockerfile.dashboard`, `tests/test_dashboard.py` |
| **reviewer** | codex | Read-only code review after each phase: security audit, PII leakage check, test coverage verification, cross-agent integration validation, GDPR compliance check | *(read-only — no file ownership)* |

---

## 2. Pane Layout

```
┌─────────────────────────────────────────────────────────────┐
│                    Herdr Pane Layout                         │
├───────────────────┬───────────────────┬─────────────────────┤
│  Pane 1           │  Pane 2           │  Pane 3             │
│  detection        │  middleware       │  compliance         │
│                   │                   │                     │
│  Regex + NER +    │  FastAPI proxy +  │  DB schema +        │
│  LLM contextual   │  scanners +       │  audit log +        │
│  pipeline         │  redaction engine │  tenant config      │
├───────────────────┼───────────────────┼─────────────────────┤
│  Pane 4           │  Pane 5           │  Pane 6             │
│  dashboard        │  reviewer         │  (shared: tests     │
│                   │                   │   + git commits)    │
│  Streamlit +      │  Code review +    │                     │
│  reports          │  security audit   │                     │
└───────────────────┴───────────────────┴─────────────────────┘
```

---

## 3. Setup Commands

```bash
# --- Initialize Herdr workspace ---
cd "/home/sagar/Projects/AI-Engineering Roadmap/Tier 4 - AI Infrastructure & Production/Project 10 - PII Redaction Guardrails"

# --- Split panes (all use project cwd, no focus steal) ---
herdr pane split --cwd "$PWD" --no-focus          # Pane 2
herdr pane split --cwd "$PWD" --no-focus          # Pane 3
herdr pane split --cwd "$PWD" --no-focus          # Pane 4
herdr pane split --cwd "$PWD" --no-focus          # Pane 5
herdr pane split --cwd "$PWD" --no-focus          # Pane 6

# --- Start agents (codex kind, one per pane) ---
herdr agent start detection   --kind codex --pane 1
herdr agent start middleware  --kind codex --pane 2
herdr agent start compliance  --kind codex --pane 3
herdr agent start dashboard   --kind codex --pane 4
herdr agent start reviewer    --kind codex --pane 5

# --- Verify all agents are running ---
herdr agent list
```

---

## 4. Phase-by-Phase Orchestration

### Phase 1: Foundation + Proxy Skeleton (Day 1)

**Goal:** Running Postgres + FastAPI proxy skeleton with auth and tenant resolution. Proxy forwards requests to upstream LLM without scanning (passthrough). (Plan §6, Phase 1, Steps 1.1–1.11)

#### Parallel Work

**detection:**
```
You are the detection agent for the PII Redaction Guardrails project. Read IMPLEMENTATION_PLAN.md sections 5 (Project Structure) and 7.1 (Regex Detector spec) for context.

Phase 1 task — create detection data models only (other detection work comes in Phase 2+):

1. Create `src/detection/__init__.py`
2. Create `src/detection/models.py` with:
   - `PIIType` enum: EMAIL, PHONE, SSN, IBAN, CREDIT_CARD, STEUER_ID, PLZ, ADDRESS_DE, PERSON, ORGANIZATION, LOCATION, DATE, PASSPORT, IP_ADDRESS, OTHER, ID_CARD_DE, HEALTH_INSURANCE_DE, DRIVERS_LICENSE_DE, TAX_NUMBER_DE
   - `PIIDetection` pydantic model: pii_type (PIIType), text (str), start (int), end (int), layer (str: "regex"|"ner"|"llm_contextual"), score (float), pattern_name (str|None)
   - `RedactionResult` pydantic model: original_text (str), redacted_text (str), events (list[RedactionEvent])
   - `RedactionEvent` pydantic model: pii_type (str), detection_layer (str), detection_score (float|None), detected_text (str), detected_span_start (int), detected_span_end (int), redaction_strategy (str), redacted_text (str), hash_value (str|None)

Reference: Plan §7.1 shows PIIDetection usage. Plan §7.5 shows RedactionResult/RedactionEvent usage. Match those field names exactly.

Do NOT create regex_detector.py, ner_detector.py, or any other detection files yet — those are Phase 2+3.
```

**middleware:**
```
You are the middleware agent for the PII Redaction Guardrails project. Read IMPLEMENTATION_PLAN.md sections 5 (Project Structure), 6 Phase 1 (Steps 1.1–1.11), 7.10 (Proxy Middleware spec), 11.1 (docker-compose.yml), 11.2 (Dockerfile), 11.5 (.env.example).

Phase 1 tasks — proxy skeleton with passthrough (no scanning yet):

1. Write `docker-compose.yml` (Plan §11.1) with three services: postgres (image: postgres:16, healthcheck, init.sql volume), proxy (build from docker/Dockerfile, port 8000, env vars, depends_on postgres healthy), dashboard (build from docker/Dockerfile.dashboard, port 8501, depends_on postgres healthy). Use the exact YAML from §11.1.
2. Write `docker/Dockerfile` (Plan §11.2): python:3.12-slim, install build-essential libpq-dev curl, pip install -e ., download spaCy models en_core_web_lg + de_core_news_lg, copy src/ patterns/ scripts/, CMD uvicorn src.main:app.
3. Write `pyproject.toml` with dependencies: fastapi, uvicorn, asyncpg, pydantic-settings, python-jose[cryptography], regex, pyyaml, presidio-analyzer, presidio-anonymizer, spacy, faker, httpx, structlog, slowapi, pytest, pytest-asyncio. Python 3.11+.
4. Write `src/config.py` — Pydantic Settings reading env vars: DATABASE_URL, JWT_SECRET, JWT_ALGORITHM (default HS256), JWT_EXPIRY_HOURS (default 24), UPSTREAM_LLM_BASE_URL, OPENAI_API_KEY, LLM_CONTEXTUAL_MODEL (default gpt-4o-mini), HASH_SALT, SPACY_MODELS, LOG_LEVEL, MAX_REQUEST_SIZE_MB (default 10), SCANNER_TIMEOUT_SECONDS (default 30). Reference §11.5 for all env vars.
5. Write `src/__init__.py` and `src/auth/__init__.py`
6. Write `src/auth/models.py` — User pydantic model: user_id (str), tenant_id (str), roles (list[str])
7. Write `src/auth/jwt_handler.py` — JWT decode + verify using python-jose. Functions: verify_jwt(token) -> User, create_jwt(tenant_id, user_id, roles) -> str. Use JWT_SECRET and JWT_ALGORITHM from config.
8. Write `src/auth/middleware.py` — AuthMiddleware (Starlette BaseHTTPMiddleware) that extracts Bearer token, calls verify_jwt, attaches user to request.state.
9. Write `src/proxy/__init__.py`
10. Write `src/proxy/llm_client.py` — LLMClient class with async forward(path, body, headers) method. Uses httpx.AsyncClient to POST to UPSTREAM_LLM_BASE_URL + path. Forward Authorization header from original request (use OPENAI_API_KEY if no upstream auth). Return parsed JSON response. Raise UpstreamLLMError on non-2xx.
11. Write `src/proxy/middleware.py` — SKELETON ONLY (Plan §7.10 but without scanning): PIIGuardrailMiddleware that does auth → read body → forward to upstream → return response. No input/output scanner calls yet (they don't exist). Add TODO comments where scanners will be wired in Phase 5. Include request_id generation (uuid4) and timing.
12. Write `src/api/__init__.py`, `src/api/routes/__init__.py`, `src/api/dependencies.py`
13. Write `src/api/routes/proxy.py` — POST /v1/chat/completions and /v1/completions routes that pass through to the middleware (the middleware handles interception).
14. Write `src/api/routes/health.py` — GET /health returning {"status": "healthy", "database": "connected", "spacy_models": [...], "upstream_llm": "reachable", "version": "1.0.0"}. Reference §9.8.
15. Write `src/main.py` — FastAPI app factory: create app, register PIIGuardrailMiddleware, include routers (proxy, health), lifespan handler to init DB pool on startup. Reference §5 and §7.10.
16. Write `.env.example` (Plan §11.5 — copy exactly).
17. Write `tests/conftest.py` — pytest fixtures: async db connection (run schema.sql), two tenant configs (acme-corp with mask, globex-gmbh with synthetic/block), JWT tokens, FastAPI TestClient with mocked upstream LLM. Reference §10.4.
18. Write `tests/test_proxy.py` — passthrough test: send chat completion via proxy with mocked upstream, verify response returned. Auth test: missing token → 401, invalid token → 401.

Do NOT implement scanning, redaction, or compliance logging yet — Phase 5.
```

**compliance:**
```
You are the compliance agent for the PII Redaction Guardrails project. Read IMPLEMENTATION_PLAN.md sections 4 (Database Schema — ALL of it), 5 (Project Structure), 6 Phase 1 (Steps 1.2, 1.6, 1.10), 11.4 (postgres init.sql).

Phase 1 tasks — database schema, tenant config resolver, seed/init scripts:

1. Write `docker/postgres/init.sql` (Plan §11.4): CREATE EXTENSION uuid-ossp, CREATE EXTENSION pgcrypto.
2. Write `src/db/__init__.py`
3. Write `src/db/schema.sql` — Full DDL from Plan §4.2. Include ALL tables: tenants, tenant_config, allowlist, redaction_events, audit_logs. Include ALL indexes. Include the prevent_redaction_modification() function and ALL triggers (no_redaction_update, no_redaction_delete, no_audit_update, no_audit_delete). Include RLS from §4.4 (ENABLE ROW LEVEL SECURITY on redaction_events and audit_logs, tenant_isolation policies). Copy the SQL exactly from the plan.
4. Write `src/db/migrations/001_initial.sql` — same as schema.sql tables/triggers.
5. Write `src/db/migrations/002_rls.sql` — RLS policies from §4.4.
6. Write `src/db/connection.py` — asyncpg connection pool. Class DatabasePool with async create_pool(database_url), acquire() context manager, close(). Use src.config.settings.DATABASE_URL.
7. Write `src/tenant/__init__.py`
8. Write `src/tenant/models.py` — TenantConfig pydantic model matching tenant_config table: tenant_id (UUID), enabled_pii_types (list[str]), redaction_strategies (dict[str, str]), llm_contextual_enabled (bool, default True), output_mode (str: "sanitize"|"block", default "sanitize"), language (str, default "auto"), custom_regex_patterns (list[dict]), detected_language (str|None). Reference §4.2 tenant_config columns and §7.8/7.9 usage.
9. Write `src/tenant/config_resolver.py` — TenantConfigResolver class with async resolve(tenant_id) -> TenantConfig. Queries tenants table (check is_active), joins tenant_config, loads allowlist entries. Raises TenantNotFoundError if tenant doesn't exist or is inactive. Default-deny: if no config row exists, return TenantConfig with ALL PII types enabled and mask strategy (Plan §8.2 Principle 1). Reference §2.3 step 3 and §7.5.
10. Write `src/compliance/__init__.py` (logger.py comes in Phase 5)
11. Write `scripts/init_db.py` — connects to DB, runs schema.sql, prints success.
12. Write `scripts/seed_test_data.py` — inserts two tenants: acme-corp (mask strategy, sanitize output, auto language, all PII types) and globex-gmbh (synthetic for PERSON/ADDRESS_DE, hash for STEUER_ID/IBAN, block output, de language). Insert allowlist entry: support@company.de for acme-corp (reason: "Public support email"). Reference §10.4 fixtures for exact configs.
13. Write `scripts/generate_jwt.py` — CLI tool: --tenant, --user, --roles args. Creates JWT using src.auth.jwt_handler.create_jwt. Prints token.

Do NOT write compliance/logger.py or audit/admin routes yet — Phase 5+6.
```

**dashboard:**
```
You are the dashboard agent for the PII Redaction Guardrails project. Read IMPLEMENTATION_PLAN.md sections 5 (Project Structure — dashboard/ dir), 11.3 (Dockerfile.dashboard).

Phase 1 task — dashboard skeleton only (full dashboard is Phase 6):

1. Write `dashboard/__init__.py`
2. Write `dashboard/app.py` — minimal Streamlit app: st.title("PII Redaction Guardrails — Compliance Dashboard"), st.info("Dashboard will be populated in Phase 6"). This is a placeholder so docker-compose can build the dashboard service.
3. Write `docker/Dockerfile.dashboard` (Plan §11.3): python:3.12-slim, install streamlit plotly psycopg2-binary pandas reportlab, copy dashboard/, CMD streamlit run dashboard/app.py --server.port=8501 --server.address=0.0.0.0.

Do NOT write queries.py, charts.py, or reports.py yet — Phase 6.
```

#### Sequential (after parallel)

**middleware** (after compliance finishes schema.sql + connection.py):
```
compliance agent has completed src/db/schema.sql and src/db/connection.py. Update src/main.py lifespan handler to:
1. Import DatabasePool from src.db.connection
2. Initialize the pool on startup using settings.DATABASE_URL
3. Close the pool on shutdown
4. Pass the pool to TenantConfigResolver and PIIGuardrailMiddleware

Also verify the proxy starts: run `docker compose up postgres` then `python scripts/init_db.py` then start the proxy and curl localhost:8000/health.
```

#### Review

**reviewer:**
```
You are the reviewer agent. Review Phase 1 deliverables for the PII Redaction Guardrails project.

Check:
1. `docker compose up` starts postgres + proxy + dashboard without errors
2. `curl localhost:8000/health` returns 200 with JSON body (Plan §9.8)
3. Proxy passthrough works: send a chat completion request with a valid JWT → response from mocked upstream
4. Auth: missing token → 401, invalid token → 401 (Plan §9.1)
5. DB schema: all 5 tables exist (tenants, tenant_config, allowlist, redaction_events, audit_logs), triggers block UPDATE/DELETE on redaction_events and audit_logs (test by attempting UPDATE — should raise exception)
6. RLS enabled on redaction_events and audit_logs
7. TenantConfigResolver: returns config for seeded tenant, returns default-deny config for unknown tenant
8. src/detection/models.py: PIIType enum has all types, PIIDetection/RedactionResult/RedactionEvent match plan §7.1/7.5 field names
9. .env.example matches Plan §11.5

Report any failures with file path and line number. Do NOT fix issues — report only.
```

#### Commit

```bash
git add -A && git commit -m "Phase 1: Foundation + proxy skeleton with auth, DB schema, tenant resolution, passthrough"
```

---

### Phase 2: Layer 1 — Regex Detection + German Patterns (Day 2)

**Goal:** Regex-based PII detection for all pattern types including German-specific patterns. No redaction yet — just detection. (Plan §6, Phase 2, Steps 2.1–2.6)

#### Parallel Work

**detection:**
```
You are the detection agent. Read IMPLEMENTATION_PLAN.md sections 7.1 (Regex Detector spec with full pseudocode), Appendix C (Regex Pattern Catalog — C.1 regex_catalog.yaml, C.2 german_patterns.yaml, C.3 priority notes).

Phase 2 tasks — Layer 1 regex detection:

1. Write `patterns/regex_catalog.yaml` — Copy ALL patterns from Appendix C.1 (email, phone_international, phone_us, us_ssn, credit_card_visa, credit_card_mastercard, credit_card_amex, iban_generic, passport_us, passport_uk, ip_address, date_iso, date_us). Each entry: name, pii_type, pattern, priority, optional validator. Use the exact regex from the appendix.
2. Write `patterns/german_patterns.yaml` — Copy ALL patterns from Appendix C.2 (german_steuer_id, german_iban, german_phone_international, german_phone_domestic, german_mobile, german_plz, german_address_full, german_street_only, german_passport, german_id_card, german_health_insurance, german_drivers_license, german_tax_number). Use the exact regex including German character classes [ÄÖÜäöüß].
3. Write `src/detection/regex_detector.py` — Implement RegexDetector class per Plan §7.1 pseudocode:
   - __init__(catalog_path, german_path): load both YAML catalogs, compile patterns with `regex` library (PCRE2)
   - _load_catalog(path): parse YAML, create CompiledPattern objects (name, pii_type, pattern, validator, priority)
   - detect(text, tenant_config=None): iterate patterns, skip disabled PII types, apply validators (luhn_check, iban_validate), return list[PIIDetection] with score=1.0, layer="regex"
   - Support tenant custom_regex_patterns (from tenant_config)
   - Implement luhn_check(number) function (Plan §7.1 — Luhn algorithm)
   - Implement iban_validate(iban) function (Plan §7.1 — mod-97 checksum)
   Reference the exact pseudocode in §7.1.
4. Write `tests/test_regex_detector.py` — Test ALL patterns:
   - Email: "test@example.com" → EMAIL detection
   - US SSN: "123-45-6789" → SSN
   - Credit card: "4111111111111111" → CREDIT_CARD (Luhn valid)
   - Invalid credit card: "4111111111111112" → no CREDIT_CARD (Luhn fails)
   - IBAN: "DE89370400440532013000" → IBAN (mod-97 valid)
   - Invalid IBAN: "DE00370400440532013000" → no IBAN
   - Phone: "+49 30 12345678" → PHONE
   - IP: "192.168.1.1" → IP_ADDRESS
   - Date: "2025-01-15" → DATE
   - No false positives on benign text: "Hello world 123" → no detections
5. Write `tests/test_german_pii.py` — Test ALL German patterns (Plan §10.3):
   - TestGermanSteuerID: valid "12345678901" → STEUER_ID; "Order number: 12345678901" → lower confidence or none
   - TestGermanIBAN: "DE89370400440532013000" → IBAN; with spaces "DE89 3704 0044 0532 0130 00" → IBAN
   - TestGermanPhone: "+49 30 12345678" → PHONE; "+49 151 12345678" → PHONE; "030 12345678" → PHONE
   - TestGermanAddress: "Ich wohne in der Hauptstraße 42, 24103 Kiel" → ADDRESS_DE; "PLZ: 24103" → PLZ; "PLZ: 80331" → PLZ
   - TestGermanPassport: "C01X00T47" → PASSPORT
   - TestGermanIDCard: "L01X00T472" → ID_CARD_DE
   Reference §10.3 for exact test structure.

Run tests: pytest tests/test_regex_detector.py tests/test_german_pii.py -v
```

**compliance:**
```
You are the compliance agent. Phase 2 is primarily detection work, but you have a supporting task:

1. Extend `scripts/seed_test_data.py` to add more diverse test data that will be useful for Phase 2+ testing:
   - Add a third tenant "test-tenant-de" with language="de", all German PII types enabled, hash strategy for STEUER_ID/IBAN, synthetic for PERSON/ADDRESS_DE
   - Add custom_regex_patterns to acme-corp: {"name": "employee_id", "pattern": "EMP-\\d{6}", "pii_type": "EMPLOYEE_ID"}
   - Add allowlist entries: "Max Mustermann" (PERSON, exact, "Test persona") for globex-gmbh

2. Write `tests/test_tenant_config.py` — basic tests (full tests come in Phase 4):
   - test_resolve_existing_tenant: resolve acme-corp → returns TenantConfig with correct enabled_pii_types
   - test_resolve_unknown_tenant_default_deny: resolve non-existent tenant → returns config with ALL PII types + mask strategy (Plan §8.2 Principle 1)
   - test_resolve_inactive_tenant: set tenant is_active=false → TenantNotFoundError
   - test_custom_regex_patterns_loaded: acme-corp config has employee_id custom pattern

Do NOT modify detection files. Coordinate with detection agent if you need PIIDetection models.
```

**middleware:**
```
You are the middleware agent. Phase 2 is primarily detection work. Your task:

1. Write `src/proxy/language_detector.py` — Language detection for en/de (Plan §5, §2.3 step 4b):
   - detect_language(text) -> str: returns "en" or "de"
   - Use a simple heuristic: count German-specific words/characters (ä, ö, ü, ß, "der", "die", "das", "ist", "und", "nicht", "ich", "mein", "Adresse", "Telefon", "Steuer") vs English indicators
   - If German indicators > English → "de", else "en"
   - This is used by NER detector (Phase 3) to select spaCy model

2. Write `tests/test_proxy.py` additions — test that language_detector correctly identifies:
   - "My name is Anna Müller" → "en" (or "de" if Müller triggers — test the threshold)
   - "Ich heiße Anna Müller, ich wohne in Kiel" → "de"
   - "Hello world" → "en"
   - "Meine Steuer-ID ist 12345678901" → "de"

Do NOT modify detection files.
```

**dashboard:** *(no work this phase — idle)*

#### Review

**reviewer:**
```
Review Phase 2 deliverables:

1. patterns/regex_catalog.yaml has 13+ patterns matching Appendix C.1 exactly
2. patterns/german_patterns.yaml has 13+ patterns matching Appendix C.2 exactly (including German character classes)
3. RegexDetector.detect() returns correct PIIDetection objects with accurate spans (start/end offsets)
4. Luhn validation works: valid credit cards detected, invalid ones rejected
5. IBAN mod-97 validation works: valid IBANs detected, invalid rejected
6. German-specific tests all pass: Steuer-ID (11 digits), German IBAN (DE + 22 chars), German phone (+49/0XX), PLZ (5 digits), German address (straße/weg/gasse/etc.)
7. No false positives on benign text ("Hello world", "Order #12345")
8. Language detector correctly identifies en vs de
9. TenantConfigResolver default-deny works for unknown tenants

Run: pytest tests/test_regex_detector.py tests/test_german_pii.py tests/test_tenant_config.py -v
Report failures with file:line. Do NOT fix.
```

#### Commit

```bash
git add -A && git commit -m "Phase 2: Layer 1 regex detection + German PII patterns + language detector"
```

---

### Phase 3: Layer 2 — NER (Presidio + spaCy) + Layer 3 — LLM Contextual (Day 3)

**Goal:** NER detection via Presidio + spaCy (en + de models) and LLM-based contextual detection. Three layers independently functional. (Plan §6, Phase 3, Steps 3.1–3.10)

#### Parallel Work

**detection:**
```
You are the detection agent. Read IMPLEMENTATION_PLAN.md sections 7.2 (NER Detector — full pseudocode), 7.3 (LLM Contextual Detector — full pseudocode), 7.4 (Detection Merger — full pseudocode), 3 (Tech Stack — Presidio + spaCy).

Phase 3 tasks — Layers 2+3 + merger + pipeline:

1. Write `src/detection/ner_detector.py` — Implement NERDetector class per Plan §7.2:
   - __init__: configure bilingual NLP engine via NlpEngineProvider with en_core_web_lg + de_core_news_lg
   - Create RecognizerRegistry, load predefined recognizers for en + de
   - _register_german_recognizers(): register custom Presidio PatternRecognizers for DE_STEUER_ID (11 digits, score 0.85), DE_PLZ (5 digits, score 0.6), DE_ADDRESS (street+number+PLZ+city pattern, score 0.75). Use the exact patterns from §7.2.
   - Create AnalyzerEngine with nlp_engine + registry, supported_languages=["en", "de"]
   - PRESIDIO_TO_PII mapping dict (§7.2): PERSON→PERSON, ORGANIZATION→ORGANIZATION, LOCATION→LOCATION, DATE_TIME→DATE, EMAIL_ADDRESS→EMAIL, PHONE_NUMBER→PHONE, IBAN_CODE→IBAN, CREDIT_CARD→CREDIT_CARD, US_SSN→SSN, DE_STEUER_ID→STEUER_ID, DE_PLZ→PLZ, DE_ADDRESS→ADDRESS_DE
   - detect(text, language="en", tenant_config=None): call analyzer.analyze() with entities mapped from tenant config, score_threshold=0.5. Convert results to PIIDetection objects with layer="ner".
   - _map_enabled_types(tenant_config): reverse-map PIIType to Presidio entity names
   Reference the exact pseudocode in §7.2.

2. Write `src/detection/llm_detector.py` — Implement LLMContextualDetector per Plan §7.3:
   - __init__(llm_client, max_text_length=2000)
   - should_run(text, existing_detections): heuristic — check cue words (name, address, phone, email, ssn, passport, mein, adresse, telefon, wohnhaft, I live, I work, my boss, my doctor, call me, ich heiße, ich wohne, ich arbeite). Run if cue_count > 0 and len(existing) < cue_count, or cue_count > 3. Skip if text > max_text_length.
   - async detect(text, existing_detections, language="en"): build prompt from CONTEXTUAL_DETECTION_PROMPT template (§7.3 — copy exactly), call llm_client.generate(model="gpt-4o-mini", temperature=0.0, response_format json_object), parse JSON, find spans via text.find(det["text"]), skip hallucinated text (idx == -1), return PIIDetection with layer="llm_contextual", score=0.7
   - PII_TYPE_MAP dict (§7.3): PERSON, EMAIL, PHONE, ADDRESS→ADDRESS_DE, STEUER_ID, IBAN, CREDIT_CARD, ORG→ORGANIZATION, DATE, OTHER
   Reference the exact pseudocode in §7.3.

3. Write `src/detection/merger.py` — Implement DetectionMerger per Plan §7.4:
   - merge(detections, allowlist=None): sort by start position then span length descending, resolve overlaps (_is_contained, _overlaps_incompatible), deduplicate exact matches (same start+end+pii_type), apply allowlist filtering
   - _is_contained(det, existing): skip if span is contained in existing same-type detection
   - _overlaps_incompatible(det, existing): keep both if different types overlap
   - _is_allowlisted(det, allowlist): check exact match or regex match per entry
   Reference the exact pseudocode in §7.4.

4. Write `src/detection/pipeline.py` — DetectionPipeline orchestrator:
   - __init__(regex_detector, ner_detector, llm_detector, merger, language_detector)
   - async detect(text, tenant_config=None) -> list[PIIDetection]:
     a. Detect language (auto or from tenant config)
     b. Layer 1: regex_detector.detect(text, tenant_config)
     c. Layer 2: ner_detector.detect(text, language, tenant_config)
     d. Merge Layer 1+2 detections
     e. Layer 3: if tenant_config.llm_contextual_enabled and llm_detector.should_run(text, merged): llm_detector.detect(text, merged, language)
     f. Merge all detections + apply allowlist
     g. Return final list
   - detect_sync(text, tenant_config=None): sync wrapper for evaluation scripts

5. Write `tests/test_ner_detector.py`:
   - English: "My name is Anna Müller, I live in Kiel" → PERSON (Anna Müller), LOCATION (Kiel)
   - English: "John Smith works at Google in New York" → PERSON, ORGANIZATION, LOCATION
   - German: "Anna Müller wohnt in Berlin" → PERSON, LOCATION (using de model)
   - German: "Ich arbeite bei Siemens in München" → ORGANIZATION, LOCATION
   - Date: "I was born on January 15, 1990" → DATE
   - Score threshold: detections below 0.5 are filtered

6. Write `tests/test_llm_detector.py`:
   - Mock LLMClient.generate to return JSON with detections
   - "My boss told me to email the CEO" → LLM detects implicit PERSON
   - should_run returns False for text without cue words
   - should_run returns False for text > 2000 chars
   - Hallucinated text (not in original) is skipped
   - Empty detections: {"detections": []} → returns []

7. Write `tests/test_merger.py`:
   - Exact same span + same type → deduplicated to one
   - Contained span (PLZ inside ADDRESS_DE) → PLZ skipped if same type, kept if different type
   - Overlapping different types → both kept
   - Adjacent same-type detections → merged
   - Allowlist exact match → removed from results
   - Allowlist regex match → removed from results
   - Confidence: regex (1.0) > NER > LLM (0.7) — higher confidence wins on exact overlap

Run: pytest tests/test_ner_detector.py tests/test_llm_detector.py tests/test_merger.py -v
```

**middleware:**
```
You are the middleware agent. Phase 3 requires you to ensure LLMClient is compatible with the LLM detector.

1. Review `src/proxy/llm_client.py` (written in Phase 1). Add an async generate() method if not present:
   - async generate(prompt, model, temperature=0.0, response_format=None) -> str: calls the upstream LLM API with a simple completion request (not chat — this is for Layer 3 detection). Returns the text content of the response.
   - This method is used by LLMContextualDetector (src/detection/llm_detector.py) — the detection agent will import LLMClient from src.proxy.llm_client
   - If response_format is {"type": "json_object"}, pass it through to the API

2. Update `tests/test_proxy.py` — add test for llm_client.generate() with mocked upstream.

Do NOT modify detection files. The detection agent depends on your LLMClient.generate() method signature.
```

**compliance:** *(no work this phase — idle)*

**dashboard:** *(no work this phase — idle)*

#### Review

**reviewer:**
```
Review Phase 3 deliverables:

1. NERDetector loads both spaCy models (en_core_web_lg + de_core_news_lg) successfully
2. Custom German recognizers registered: DE_STEUER_ID, DE_PLZ, DE_ADDRESS
3. NER detects persons, orgs, locations in both English and German text
4. LLMContextualDetector.should_run() heuristic works: triggers on cue words, skips long text
5. LLM detector parses JSON response, finds spans, skips hallucinated text
6. DetectionMerger: exact duplicates removed, contained spans resolved, different-type overlaps kept
7. Allowlist filtering works in merger
8. DetectionPipeline orchestrates all 3 layers in correct order: regex → NER → merge → LLM (conditional) → merge+allowlist
9. LLMClient.generate() method exists and is compatible with llm_detector.py
10. Each layer independently catches ≥1 test case the others miss (Plan G3)

Run: pytest tests/test_ner_detector.py tests/test_llm_detector.py tests/test_merger.py -v
Report failures with file:line. Do NOT fix.
```

#### Commit

```bash
git add -A && git commit -m "Phase 3: Layer 2 NER (Presidio+spaCy) + Layer 3 LLM contextual + merger + pipeline"
```

---

### Phase 4: Redaction Engine + Tenant Config (Day 4)

**Goal:** Redaction strategies (mask, hash, synthetic) working with per-tenant configuration and allowlist. (Plan §6, Phase 4, Steps 4.1–4.8)

#### Parallel Work

**middleware:**
```
You are the middleware agent. Read IMPLEMENTATION_PLAN.md sections 7.5 (Redaction Engine — full pseudocode), 7.6 (Redaction Strategies — full pseudocode), 7.7 (Synthetic Data Generator — full pseudocode), Appendix D (Redaction Strategy Comparison + default config).

Phase 4 tasks — redaction engine + strategies + synthetic data:

1. Write `src/redaction/__init__.py`
2. Write `src/redaction/synthetic_data.py` — SyntheticDataGenerator per Plan §7.7:
   - __init__(locale="de_DE"): self.fake = Faker(locale), self._cache = {}
   - generate(pii_type, original) -> str: check cache (f"{pii_type.value}:{original}"), generate using Faker based on PIIType:
     - PERSON → fake.name()
     - EMAIL → fake.email()
     - PHONE → fake.phone_number()
     - ADDRESS_DE → f"{fake.street_name()} {fake.building_number()}, {fake.postcode()} {fake.city()}"
     - PLZ → fake.postcode()
     - IBAN → fake.iban()
     - CREDIT_CARD → fake.credit_card_number()
     - STEUER_ID → f"{fake.random_number(digits=11)}"
     - ORGANIZATION → fake.company()
     - DATE → fake.date()
     - SSN → fake.ssn()
     - default → "[REDACTED]"
   - Cache result for consistency within request
   - reset_cache(): clear cache
   Reference §7.7 exactly.

3. Write `src/redaction/strategies.py` — Three strategies per Plan §7.6:
   - MaskStrategy.apply(detection) -> (replacement, meta): return f"[REDACTED_{detection.pii_type.value}]", {}
   - HashStrategy.__init__(salt): self.salt = salt.encode()
     HashStrategy.apply(detection) -> (replacement, meta): HMAC-SHA256 with salt, hexdigest[:8] for short hash, return f"[HASH_{hash_hex}]", {"hash": full_hexdigest}
   - SyntheticStrategy.__init__(generator): self.generator = generator
     SyntheticStrategy.apply(detection) -> (replacement, meta): synthetic = generator.generate(detection.pii_type, detection.text), return synthetic, {"synthetic_original_hash": sha256[:8]}
   Reference §7.6 exactly.

4. Write `src/redaction/engine.py` — RedactionEngine per Plan §7.5:
   - __init__(hash_salt, synthetic_gen): create strategies dict {"mask": MaskStrategy(), "hash": HashStrategy(salt), "synthetic": SyntheticStrategy(gen)}
   - redact(text, detections, tenant_config) -> RedactionResult:
     a. Sort detections by start position DESCENDING (right to left — preserves offsets)
     b. For each detection: look up strategy from tenant_config.redaction_strategies.get(pii_type, "mask")
     c. Apply strategy → get replacement + event_meta
     d. Replace in text: redacted_text[:start] + replacement + redacted_text[end:]
     e. Build RedactionEvent with all fields
     f. Return RedactionResult(original_text, redacted_text, events)
   Reference §7.5 exactly.

5. Write `tests/test_redaction.py`:
   - MaskStrategy: "test@mail.de" → "[REDACTED_EMAIL]"
   - HashStrategy: same input → same hash (deterministic); different inputs → different hashes
   - HashStrategy: hash is HMAC-SHA256, 8-char prefix in replacement, full hash in meta
   - SyntheticStrategy: PERSON → realistic German name (Faker de_DE); same input → same synthetic (cache)
   - SyntheticStrategy.reset_cache() → next call generates new synthetic
   - RedactionEngine: multiple detections in one text → all replaced, offsets correct
   - RedactionEngine: right-to-left processing preserves offsets (test with overlapping spans)
   - Per-tenant strategy: tenant A (mask) → "[REDACTED_EMAIL]", tenant B (hash) → "[HASH_...]", tenant C (synthetic) → fake email
   - Default strategy: PII type not in tenant config → mask
   Reference Plan §6 Phase 4 Verification for expected outputs.

Run: pytest tests/test_redaction.py -v
```

**compliance:**
```
You are the compliance agent. Read IMPLEMENTATION_PLAN.md sections 4.2 (tenant_config + allowlist tables), 7.4 (merger allowlist — AllowlistEntry), 7.5 (RedactionEngine uses tenant_config.redaction_strategies), Appendix D (default tenant config).

Phase 4 tasks — extend tenant config + full tenant tests:

1. Extend `src/tenant/config_resolver.py` (written in Phase 1):
   - Load allowlist entries from DB (allowlist table) alongside tenant config
   - Return AllowlistEntry objects: entity_value, pii_type, match_type ("exact"|"regex"), reason
   - Load custom_regex_patterns from tenant_config JSONB column
   - detected_language: if tenant config language="auto", leave as None (set by pipeline); if "en"/"de", set directly
   Reference §2.3 step 3 and §7.4 _is_allowlisted usage.

2. Extend `src/tenant/models.py`:
   - Add AllowlistEntry pydantic model: entity_value (str), pii_type (str), match_type (str), reason (str|None)
   - Add allowlist field to TenantConfig: list[AllowlistEntry]
   - Ensure TenantConfig is compatible with RedactionEngine (§7.5) and DetectionMerger (§7.4)

3. Write full `tests/test_tenant_config.py`:
   - test_resolve_existing_tenant: acme-corp → correct enabled_pii_types, redaction_strategies, output_mode
   - test_resolve_unknown_tenant_default_deny: non-existent → ALL PII types + mask strategy
   - test_resolve_inactive_tenant: is_active=false → TenantNotFoundError
   - test_custom_regex_patterns: acme-corp has employee_id custom pattern
   - test_allowlist_loaded: acme-corp has support@company.de in allowlist
   - test_allowlist_exact_match: "support@company.de" → allowlisted
   - test_allowlist_regex_match: regex pattern in allowlist → matching text allowlisted
   - test_per_tenant_strategy: acme-corp EMAIL=mask, globex-gmbh PERSON=synthetic
   - test_output_mode: acme-corp=sanitize, globex-gmbh=block
   - test_language_config: globex-gmbh language="de", acme-corp language="auto"

4. Implement allowlist filtering in merger — coordinate with detection agent:
   The detection agent owns src/detection/merger.py. The merger already has _is_allowlisted() from Phase 3.
   Verify that TenantConfig.allowlist is passed to DetectionMerger.merge() as the allowlist parameter.
   If the merger's AllowlistEntry interface doesn't match your model, update src/tenant/models.py AllowlistEntry to match what merger expects (entity_value, pii_type, match_type).

Do NOT modify detection files (merger.py). If interface mismatch, update your models to match merger's expectations.
```

**detection:**
```
You are the detection agent. Phase 4 is primarily redaction work, but you have a verification task:

1. Verify that src/detection/merger.py _is_allowlisted() method is compatible with the compliance agent's AllowlistEntry model. The merger expects: entry.pii_type (str), entry.entity_value (str), entry.match_type ("exact"|"regex"). If the compliance agent's AllowlistEntry uses different field names, coordinate via hub messaging to align.

2. Write a quick integration test in `tests/test_merger.py` (append to existing):
   - test_allowlist_with_tenant_config: create TenantConfig with allowlist containing "support@company.de" (EMAIL, exact), run merger with detections including that email → email is NOT in results
   - test_allowlist_regex: allowlist with regex pattern ".*@company\.de" → all company.de emails removed

3. Verify DetectionPipeline passes tenant_config.allowlist to merger.merge(). If not, update pipeline.py to pass it.

Do NOT modify redaction or tenant files.
```

**dashboard:** *(no work this phase — idle)*

#### Review

**reviewer:**
```
Review Phase 4 deliverables:

1. MaskStrategy: produces "[REDACTED_TYPE]" format
2. HashStrategy: HMAC-SHA256 with salt, deterministic (same input → same hash), 8-char prefix in output
3. SyntheticStrategy: Faker de_DE locale, German names/addresses/phones, per-request cache consistency
4. RedactionEngine: processes right-to-left, offsets preserved, correct strategy per tenant config
5. Per-tenant strategy selection: mask vs hash vs synthetic produces different outputs for same input (Plan §6 Phase 4 Verification)
6. Allowlisted entities pass through unredacted ("support@company.de" not redacted)
7. TenantConfigResolver loads allowlist + custom patterns + strategies from DB
8. Default-deny: unknown tenant gets all PII types + mask
9. AllowlistEntry model compatible between compliance and detection agents

Run: pytest tests/test_redaction.py tests/test_tenant_config.py tests/test_merger.py -v
Report failures with file:line. Do NOT fix.
```

#### Commit

```bash
git add -A && git commit -m "Phase 4: Redaction engine (mask/hash/synthetic) + per-tenant config + allowlist"
```

---

### Phase 5: Input/Output Scanners + Proxy Integration (Day 5)

**Goal:** Full proxy pipeline: input scan → redact → forward to LLM → output scan → sanitize → return to client. Compliance logging wired in. (Plan §6, Phase 5, Steps 5.1–5.10)

#### Parallel Work

**compliance:**
```
You are the compliance agent. Read IMPLEMENTATION_PLAN.md sections 7.11 (Compliance Logger — full pseudocode), 9.6 (GET /audit/redactions), 9.7 (GET /audit/summary).

Phase 5 tasks — compliance logger + audit API:

1. Write `src/compliance/logger.py` — ComplianceLogger per Plan §7.11:
   - __init__(db_pool): store pool
   - async log_redaction(tenant_id, user_id, request_id, direction, event, model_name, language): INSERT into redaction_events with all fields (id, timestamp default NOW(), tenant_id, user_id, request_id, direction, pii_type, detection_layer, detection_score, detected_text, detected_span_start, detected_span_end, redaction_strategy, redacted_text, hash_value, model_name, language). Use parameterized query ($1, $2, ...) with asyncpg.
   - async log_request(tenant_id, user_id, request_id, endpoint, model_name, direction, success, input_redaction_count, output_redaction_count, input_scan_ms, llm_call_ms, output_scan_ms, total_latency_ms, error_message, pii_types_detected): INSERT into audit_logs.
   Reference §7.11 exactly. Match the SQL column order.

2. Write `src/api/routes/audit.py`:
   - GET /audit/redactions: query redaction_events with filters (tenant_id, user_id, direction, pii_type, from, to, page, limit). Requires admin or compliance role. If compliance role: show detected_text. If admin role: show "[REDACTED]" for detected_text (Plan §9.6). Paginated response with events + total + page.
   - GET /audit/summary: aggregate stats for date range (total_redactions, input_redactions, output_redactions, blocked_responses, pii_type_distribution, detection_layer_distribution, avg_latency_ms). Requires admin or compliance role. Reference §9.7 for response shape.
   - Use FastAPI dependency for role check (admin or compliance).

3. Write `tests/test_compliance.py`:
   - test_log_redaction_creates_row: log a redaction event → query redaction_events → row exists with correct fields
   - test_log_request_creates_row: log a request → query audit_logs → row exists
   - test_audit_log_immutable: attempt UPDATE redaction_events → exception (Plan §10.2 test_audit_log_is_immutable)
   - test_audit_log_delete_blocked: attempt DELETE redaction_events → exception
   - test_audit_logs_immutable: attempt UPDATE/DELETE audit_logs → exception
   - test_every_redaction_logged: send prompt with 2 PII entities via proxy → 2 rows in redaction_events with direction='input'
   - test_audit_redactions_endpoint: GET /audit/redactions with admin token → 200 with events list
   - test_audit_summary_endpoint: GET /audit/summary with admin token → 200 with aggregate stats
   - test_audit_requires_admin: GET /audit/redactions with user token → 403
   - test_compliance_role_sees_detected_text: GET /audit/redactions with compliance token → detected_text visible
   - test_admin_role_sees_redacted: GET /audit/redactions with admin token → detected_text is "[REDACTED]"

Run: pytest tests/test_compliance.py -v
```

**middleware:**
```
You are the middleware agent. Read IMPLEMENTATION_PLAN.md sections 7.8 (Input Scanner — full pseudocode), 7.9 (Output Scanner — full pseudocode), 7.10 (Proxy Middleware — full pseudocode), 9.1-9.2 (API spec), 10.2 (Critical Security Tests).

Phase 5 tasks — scanners + full proxy integration + security tests:

1. Write `src/scanner/__init__.py`
2. Write `src/scanner/input_scanner.py` — InputScanner per Plan §7.8:
   - __init__(pipeline, engine, logger): store DetectionPipeline, RedactionEngine, ComplianceLogger
   - async scan(messages, tenant_config, user_id, request_id, model_name) -> list[dict]:
     a. For each message: skip system messages (trusted), skip empty/non-string content
     b. Detect PII via pipeline.detect(content, tenant_config)
     c. Redact via engine.redact(content, detections, tenant_config)
     d. Log every redaction event via logger.log_redaction(direction="input", ...)
     e. Replace content with redacted version
     f. Return scanned messages
   Reference §7.8 exactly.

3. Write `src/scanner/output_scanner.py` — OutputScanner per Plan §7.9:
   - __init__(pipeline, engine, logger)
   - async scan(response, tenant_config, user_id, request_id, model_name) -> (dict, bool):
     a. Extract choices from response (OpenAI format)
     b. For each choice: get message.content
     c. Detect PII via pipeline.detect(content, tenant_config)
     d. If detections and output_mode=="block": log events, raise PIILeakageError
     e. If detections and output_mode=="sanitize": force mask strategy (create mask_config with all types → "mask"), redact, log events, replace content
     f. Return (response, pii_found_in_output)
   - Define PIILeakageError exception
   Reference §7.9 exactly. Note: output ALWAYS uses mask strategy regardless of tenant input strategy.

4. Update `src/proxy/middleware.py` — FULL implementation per Plan §7.10:
   - __init__(app, input_scanner, output_scanner, llm_client, tenant_resolver, compliance_logger)
   - async dispatch(request, call_next):
     a. Skip non-LLM endpoints (only /v1/chat/completions and /v1/completions)
     b. Generate request_id (uuid4), start timing
     c. Auth: extract Bearer token, verify_jwt, resolve tenant config
     d. Read request body, parse JSON
     e. Input scan: call input_scanner.scan(messages, tenant_config, ...)
     f. Replace messages in request body
     g. Forward to upstream LLM via llm_client.forward()
     h. Output scan: call output_scanner.scan(response, tenant_config, ...)
     i. Handle PIILeakageError → 422 response
     j. Handle UpstreamLLMError → 502 response
     k. Handle scanner crash → 503 response (FAIL SAFE — Plan §8.2 Principle 5)
     l. Log request-level audit via compliance_logger.log_request()
     m. Add response headers: X-PII-Redacted: true, X-Redaction-Count, X-Request-Id
     n. Return safe response
   Reference §7.10 exactly. This is the CRITICAL security boundary.

5. Update `src/main.py` — wire all components:
   - Create DetectionPipeline (regex + ner + llm detectors, merger, language detector)
   - Create RedactionEngine (hash_salt from config, synthetic gen)
   - Create ComplianceLogger (db_pool)
   - Create InputScanner, OutputScanner
   - Create TenantConfigResolver
   - Register PIIGuardrailMiddleware with all dependencies
   - Include audit router

6. Write `tests/test_input_scanner.py`:
   - System messages are not scanned (pass through)
   - Empty content passes through
   - PII in user message → redacted, event logged
   - Multiple messages → each scanned independently
   - No PII → message unchanged, no events

7. Write `tests/test_output_scanner.py`:
   - No PII in response → returned unchanged, pii_found=False
   - PII in response + sanitize mode → PII masked, pii_found=True
   - PII in response + block mode → PIILeakageError raised
   - Output always uses mask strategy (not tenant's input strategy)

8. Write `tests/test_security.py` — CRITICAL security tests per Plan §10.2:
   - test_no_raw_pii_in_forwarded_prompt: send PII prompt, mock upstream to capture forwarded body, assert NO raw PII (Anna Müller, a.mueller@gmx.de, 12345678901, DE89370400440532013000, Hauptstr. 42, 24103) in forwarded content. Assert [REDACTED_] or [HASH_] present.
   - test_no_pii_in_returned_response: mock upstream returns PII, assert client receives sanitized response (no raw PII)
   - test_output_block_mode_returns_422: tenant with block mode, upstream returns PII → 422
   - test_redaction_event_logged_for_every_detection: send prompt with EMAIL + PHONE → 2+ rows in redaction_events
   - test_audit_log_is_immutable: UPDATE redaction_events → exception
   - test_allowlisted_entity_not_redacted: allowlisted "support@company.de" passes through to upstream
   - test_scanner_failure_blocks_request: scanner crashes → 503 (NOT forwarded with PII)
   Reference §10.2 — copy test logic exactly. These are P0 tests.

Run: pytest tests/test_input_scanner.py tests/test_output_scanner.py tests/test_security.py -v

CRITICAL: All security tests MUST pass. If any fail, the system is not safe to deploy.
```

**detection:** *(no work this phase — idle, but available for pipeline integration questions)*

**dashboard:** *(no work this phase — idle)*

#### Sequential (after parallel)

**middleware** (after compliance finishes logger.py):
```
compliance agent has completed src/compliance/logger.py. Verify integration:
1. InputScanner and OutputScanner correctly call ComplianceLogger.log_redaction() with all required parameters
2. PIIGuardrailMiddleware correctly calls ComplianceLogger.log_request() for audit_logs
3. Run full end-to-end test: docker compose up → seed data → send PII prompt → verify:
   a. Forwarded prompt has no raw PII
   b. Response has no raw PII
   c. redaction_events table has rows for each detection
   d. audit_logs table has a row for the request
   e. X-PII-Redacted header is "true"
   f. X-Redaction-Count header matches number of redactions
```

#### Review

**reviewer:**
```
Review Phase 5 deliverables — THIS IS THE CRITICAL SECURITY PHASE.

Security checks (ALL must pass):
1. No raw PII in forwarded prompts (test_security.py::test_no_raw_pii_in_forwarded_prompt)
2. No PII in returned responses (test_security.py::test_no_pii_in_returned_response)
3. Block mode returns 422 when PII in output (test_security.py::test_output_block_mode_returns_422)
4. Every redaction event logged (test_security.py::test_redaction_event_logged_for_every_detection)
5. Audit log immutable — UPDATE/DELETE blocked by trigger (test_security.py::test_audit_log_is_immutable)
6. Allowlisted entities not redacted (test_security.py::test_allowlisted_entity_not_redacted)
7. Scanner failure → 503, NOT forwarded with PII (test_security.py::test_scanner_failure_blocks_request)

Integration checks:
8. Full proxy pipeline works: client → input scan → redact → forward → output scan → sanitize → client
9. X-PII-Redacted and X-Redaction-Count headers present in response
10. ComplianceLogger.log_redaction() inserts into redaction_events correctly
11. ComplianceLogger.log_request() inserts into audit_logs correctly
12. Audit API: GET /audit/redactions and /audit/summary work with admin/compliance roles
13. Output scanner always uses mask strategy (never synthetic/hash in output)
14. Fail-safe: scanner timeout/crash → 503 (Plan §8.2 Principle 5)

Run: pytest tests/test_security.py tests/test_input_scanner.py tests/test_output_scanner.py tests/test_compliance.py -v
Report ANY security test failure as CRITICAL. Do NOT fix.
```

#### Commit

```bash
git add -A && git commit -m "Phase 5: Input/output scanners + full proxy pipeline + compliance logging + security tests"
```

---

### Phase 6: Streamlit Dashboard + Admin API (Day 6)

**Goal:** Dashboard showing redaction statistics, PII type distribution, compliance report export. Admin API for tenant management. (Plan §6, Phase 6, Steps 6.1–6.7)

#### Parallel Work

**dashboard:**
```
You are the dashboard agent. Read IMPLEMENTATION_PLAN.md sections 7.12 (Streamlit Dashboard — full pseudocode), 5 (dashboard/ dir structure), 9.3-9.7 (API specs for data sources).

Phase 6 tasks — full Streamlit dashboard:

1. Write `dashboard/queries.py` — SQL queries against redaction_events + audit_logs:
   - get_redaction_stats(tenant, date_from, date_to) -> dict: total, input_count, output_count, blocked_count
   - get_pii_type_distribution(tenant, date_from, date_to) -> DataFrame: pii_type, count
   - get_redaction_trend(tenant, date_from, date_to) -> DataFrame: date, count, direction
   - get_tenant_breakdown(tenant, date_from, date_to) -> DataFrame: tenant_name, redaction_count
   - get_recent_events(tenant, date_from, date_to, limit=100) -> DataFrame: timestamp, pii_type, direction, detection_layer, redaction_strategy
   - get_tenant_list() -> list[str]: tenant names from tenants table
   - get_detection_layer_distribution(tenant, date_from, date_to) -> DataFrame: layer, count
   Use psycopg2 to connect to DB (DATABASE_URL env var). All queries filter by tenant (or all tenants) and date range.

2. Write `dashboard/charts.py` — Chart functions using plotly:
   - plot_pii_type_distribution(dist_df) -> plotly Figure: pie chart of PII types
   - plot_redaction_trend(trend_df) -> plotly Figure: line chart over time, colored by direction
   - plot_detection_layer_distribution(layer_df) -> plotly Figure: bar chart by layer
   - plot_tenant_breakdown(tenant_df) -> plotly Figure: horizontal bar chart

3. Write `dashboard/reports.py` — Compliance report export:
   - export_csv_report(tenant, date_from, date_to) -> str: CSV string with columns: timestamp, tenant, user_id, direction, pii_type, detection_layer, redaction_strategy, model_name, language
   - export_pdf_report(tenant, date_from, date_to) -> bytes: PDF using reportlab with summary stats + PII type distribution table + date range

4. Write `dashboard/app.py` — Full Streamlit app per Plan §7.12:
   - st.set_page_config(page_title="PII Guardrails Dashboard", page_icon="🛡️")
   - Title: "PII Redaction Guardrails — Compliance Dashboard"
   - Filters: tenant selectbox (All + tenant list), date_from, date_to
   - Summary metrics: Total Redactions, Input Redactions, Output Redactions, Blocked Responses (4 columns)
   - PII Type Distribution: pie chart (plotly)
   - Redaction Trend: line chart over time (plotly)
   - Detection Layer Distribution: bar chart
   - Recent Redaction Events: st.dataframe with latest 100 events
   - Compliance Report Export: CSV button + PDF button (st.download_button)
   Reference §7.12 exactly.

5. Update `docker/Dockerfile.dashboard` — ensure all deps installed: streamlit, plotly, psycopg2-binary, pandas, reportlab. Copy dashboard/ dir. CMD streamlit run.

6. Write `tests/test_dashboard.py`:
   - test_get_redaction_stats: seed data → query → correct counts
   - test_get_pii_type_distribution: seed data → correct type counts
   - test_get_redaction_trend: seed data → correct daily counts
   - test_export_csv_report: export → CSV has correct columns, rows match data
   - test_export_pdf_report: export → PDF is non-empty bytes
   - test_tenant_filter: filter by tenant → only that tenant's events
   - test_date_filter: filter by date range → only events in range

Run: pytest tests/test_dashboard.py -v
```

**compliance:**
```
You are the compliance agent. Read IMPLEMENTATION_PLAN.md sections 9.3 (GET /admin/tenants), 9.4 (POST /admin/tenants/{id}/config), 9.5 (POST /admin/tenants/{id}/allowlist).

Phase 6 tasks — admin API routes:

1. Write `src/api/routes/admin.py`:
   - GET /admin/tenants: list all tenants with their config summary (id, name, is_active, enabled_pii_types, output_mode, created_at). Requires admin role. Reference §9.3.
   - POST /admin/tenants: create a new tenant. Body: name, description. Requires admin role.
   - POST /admin/tenants/{id}/config: update tenant configuration. Body: enabled_pii_types, redaction_strategies, llm_contextual_enabled, output_mode, language, custom_regex_patterns. Upsert into tenant_config table. Requires admin role. Reference §9.4.
   - POST /admin/tenants/{id}/allowlist: add allowlist entry. Body: entity_value, pii_type, match_type, reason. Insert into allowlist table. Requires admin role. Reference §9.5.
   - GET /admin/tenants/{id}/allowlist: list allowlist entries for a tenant. Requires admin role.
   - DELETE /admin/tenants/{id}/allowlist/{entry_id}: remove allowlist entry. Requires admin role.
   - Use FastAPI dependency for admin role check.

2. Add tests to `tests/test_tenant_config.py` (append):
   - test_admin_list_tenants: GET /admin/tenants with admin token → 200 with tenant list
   - test_admin_create_tenant: POST /admin/tenants → 201, new tenant exists
   - test_admin_update_config: POST /admin/tenants/{id}/config → 200, config updated in DB
   - test_admin_add_allowlist: POST /admin/tenants/{id}/allowlist → 201, entry in DB
   - test_admin_list_allowlist: GET /admin/tenants/{id}/allowlist → 200 with entries
   - test_admin_delete_allowlist: DELETE → 200, entry removed
   - test_admin_requires_admin_role: all admin endpoints with user token → 403

3. Register admin router in `src/main.py` (coordinate with middleware agent if they own main.py — if so, send them a hub message with the router import line to add).

Run: pytest tests/test_tenant_config.py -v
```

**detection:** *(no work this phase — idle)*

**middleware:** *(no work this phase — idle, but available for main.py router registration)*

#### Review

**reviewer:**
```
Review Phase 6 deliverables:

1. Dashboard renders at localhost:8501 with live data from seeded redaction_events
2. Summary metrics show correct counts (total, input, output, blocked)
3. PII type distribution pie chart renders with correct proportions
4. Redaction trend line chart shows data over time
5. Recent events table shows latest 100 events with correct columns
6. CSV export: downloads file with correct columns (timestamp, tenant, user_id, direction, pii_type, detection_layer, redaction_strategy, model_name, language)
7. PDF export: downloads non-empty PDF file
8. Tenant filter works: selecting a tenant shows only that tenant's data
9. Date filter works: date range filters events correctly
10. Admin API: GET /admin/tenants returns tenant list (admin token)
11. Admin API: POST /admin/tenants/{id}/config updates config (admin token)
12. Admin API: POST /admin/tenants/{id}/allowlist adds entry (admin token)
13. Admin API: non-admin token → 403 on all admin endpoints
14. docker/Dockerfile.dashboard builds successfully

Run: pytest tests/test_dashboard.py tests/test_tenant_config.py -v
Report failures with file:line. Do NOT fix.
```

#### Commit

```bash
git add -A && git commit -m "Phase 6: Streamlit dashboard + compliance reports + admin API"
```

---

### Phase 7: Hardening, Evaluation, Documentation (Day 7)

**Goal:** Production-ready, evaluated, documented, edge cases handled. (Plan §6, Phase 7, Steps 7.1–7.10)

#### Parallel Work

**detection:**
```
You are the detection agent. Read IMPLEMENTATION_PLAN.md sections 10.5 (Evaluation Script — full pseudocode), Plan §6 Phase 7 Steps 7.1-7.2, 7.10.

Phase 7 tasks — evaluation + benchmarking:

1. Write `scripts/evaluate_recall.py` — per Plan §10.5:
   - evaluate(corpus_path, pipeline): read JSONL corpus (each line: {text, annotations: [{start, end, pii_type}]})
   - Run pipeline.detect_sync() on each sample
   - Compare detected spans vs expected spans (set intersection)
   - Compute per-layer and combined TP, FP, FN
   - Compute precision, recall, F1
   - Assert recall >= 0.98 (Plan G3)
   - CLI: --corpus path argument
   - Print per-layer and combined scores
   Reference §10.5 exactly.

2. Write `scripts/benchmark_detection.py`:
   - Measure latency per layer: regex, NER, LLM contextual
   - Run detection on a set of sample texts (varying lengths: 100, 500, 1000, 2000 chars)
   - Measure: Layer 1 time, Layer 2 time, Layer 3 time, total pipeline time
   - Print results table: text_length, regex_ms, ner_ms, llm_ms, total_ms
   - Run 100 iterations per sample for stable measurements
   - Document expected latencies: regex <1ms, NER 10-50ms, LLM 200-800ms (Plan §3)

3. Create `tests/fixtures/labeled_pii.jsonl` — labeled PII corpus for evaluation:
   - 20+ samples with mixed en/de text
   - Each sample has manually annotated PII spans (start, end, pii_type)
   - Include: emails, phone numbers, German Steuer-IDs, IBANs, person names, addresses, PLZ, credit cards, SSNs, dates
   - Include edge cases: no PII (should have empty annotations), multiple PII types in one text, German text with umlauts

4. Run evaluation: python scripts/evaluate_recall.py --corpus tests/fixtures/labeled_pii.jsonl
   Document the recall score. If < 0.98, analyze false negatives and report which PII types are missed.

Run: python scripts/evaluate_recall.py --corpus tests/fixtures/labeled_pii.jsonl
```

**middleware:**
```
You are the middleware agent. Read IMPLEMENTATION_PLAN.md sections 6 Phase 7 (Steps 7.3-7.6), 8.2 (Safety Principles — fail safe), 11.6 (Production Considerations).

Phase 7 tasks — hardening:

1. Add rate limiting using slowapi:
   - Configure slowapi Limiter with default limit (e.g., 20 requests/min per user)
   - Apply to proxy endpoints (/v1/chat/completions, /v1/completions)
   - Reference §11.6 Production Considerations

2. Add request size limits:
   - Reject requests larger than MAX_REQUEST_SIZE_MB (from config, default 10MB)
   - Return 413 Payload Too Large

3. Add structured logging using structlog:
   - Configure structlog with JSON output
   - Log: request_id, tenant_id, user_id, endpoint, model, redaction_count, latency_ms
   - Log scanner events, errors, upstream LLM status
   - Reference §11.6

4. Add health check + readiness probe:
   - GET /health: check DB connection, spaCy models loaded, upstream LLM reachable (Plan §9.8)
   - GET /ready: return 200 if all dependencies healthy, 503 if any down
   - Reference §7.10 and §11.6

5. Add error handling for edge cases:
   - Upstream LLM down → 502 with clear error message (no PII leaked)
   - Scanner timeout (SCANNER_TIMEOUT_SECONDS) → 503 (fail-safe: block, don't forward with PII)
   - DB down → redaction still works (buffer events in memory, retry logging)
   - Malformed request body → 400 Bad Request
   - Missing model field → 400
   - Reference §8.2 Principle 5 (fail safe) and §11.6

6. Update `tests/test_proxy.py` with hardening tests:
   - test_rate_limit: send 25 requests rapidly → 429 after limit
   - test_request_size_limit: send >10MB body → 413
   - test_health_check: GET /health → 200 with all fields
   - test_ready_check: GET /ready → 200 when healthy
   - test_upstream_down: mock upstream to raise → 502
   - test_scanner_timeout: mock scanner to timeout → 503
   - test_malformed_body: send invalid JSON → 400

Run: pytest tests/test_proxy.py -v
```

**compliance:**
```
You are the compliance agent. Read IMPLEMENTATION_PLAN.md sections 4.3 (Design Note: Storing PII), 4.4 (RLS), 11.6 (Production Considerations), Appendix E (GDPR Compliance Mapping).

Phase 7 tasks — compliance hardening + documentation:

1. Implement retention policy stub (Plan §4.3):
   - Write `scripts/expire_old_redactions.py` — cron job that:
     a. Temporarily disables trigger
     b. UPDATE redaction_events SET detected_text = '[EXPIRED]' WHERE timestamp < NOW() - INTERVAL '2 years'
     c. Re-enables trigger
   - Reference §4.3 commented SQL. This is a maintenance script, not run automatically.

2. Verify RLS is working:
   - Write test in `tests/test_compliance.py` (append):
     - test_rls_tenant_isolation: set app.current_tenant_id to tenant A, query redaction_events → only tenant A's events visible
     - test_rls_no_setting: without setting app.current_tenant_id → no events visible (RLS denies)

3. Write `README.md` — project documentation:
   - Overview: what the system does (from Plan §1 elevator pitch)
   - Architecture: three-layer detection + proxy middleware (from §2.1)
   - Quick Start: copy from Appendix A (§A)
   - Configuration: env vars from §11.5
   - API Reference: summarize §9 endpoints
   - Testing: how to run tests
   - Deployment: docker compose instructions from §11
   - GDPR Compliance: summarize Appendix E mapping
   - Security Model: summarize §8

4. Verify all compliance tests pass:
   - Audit log immutability
   - RLS tenant isolation
   - Every redaction event logged
   - Audit API role-based access

Run: pytest tests/test_compliance.py -v
```

**dashboard:**
```
You are the dashboard agent. Read IMPLEMENTATION_PLAN.md sections 7.12 (Dashboard), 11.6 (Production Considerations).

Phase 7 tasks — dashboard polish + alerting:

1. Add alert indicators to `dashboard/app.py`:
   - If output PII detection rate spikes (>5% of requests in last hour) → show red warning banner
   - If blocked response count > threshold → show alert
   - Display detection layer distribution (regex vs NER vs LLM) as a bar chart — shows system health

2. Add per-tenant detail view to dashboard:
   - Select a specific tenant → show that tenant's redaction stats, config summary, allowlist entries
   - Show tenant's output_mode (sanitize/block) and enabled PII types

3. Add latency metrics to dashboard:
   - Average input_scan_ms, llm_call_ms, output_scan_ms, total_latency_ms
   - Display as a breakdown chart

4. Update `tests/test_dashboard.py` with new feature tests:
   - test_alert_threshold: seed data with high output PII rate → alert flag set
   - test_tenant_detail: query per-tenant stats → correct data
   - test_latency_metrics: query avg latencies → correct values

Run: pytest tests/test_dashboard.py -v
```

#### Sequential (after parallel)

**reviewer** (full end-to-end audit):
```
You are the reviewer agent. Conduct the FINAL security and quality audit for the PII Redaction Guardrails project.

Run the FULL test suite:
  pytest tests/ -v --tb=short

Security audit (Plan §8, §10.2):
1. Re-run ALL security tests — every one MUST pass
2. Manual PII leakage audit: send 10 prompts with mixed en/de PII through the proxy, verify:
   a. No raw PII in forwarded prompts (inspect mock upstream)
   b. No PII in returned responses
   c. Every detection logged in redaction_events
   d. X-PII-Redacted and X-Redaction-Count headers present
3. Test fail-safe: kill scanner mid-request → 503 (not forwarded with PII)
4. Test audit log immutability: attempt UPDATE/DELETE → exception
5. Test RLS: tenant A cannot see tenant B's events

German PII audit:
6. All German patterns detected: Steuer-ID (11 digits), German IBAN (DE+22), German phone (+49/0XX), PLZ (5 digits), German address (straße/weg/gasse/etc.), German passport, German ID card
7. German NER: de_core_news_lg model loaded, detects German names and locations

Evaluation audit:
8. Recall >= 98% on labeled corpus (scripts/evaluate_recall.py)
9. Benchmark results documented (scripts/benchmark_detection.py)

Compliance audit:
10. GDPR mapping (Appendix E): verify each Article has a corresponding system feature
11. SOC 2 mapping: verify CC6.1, CC6.2, CC7.1, CC7.2, CC7.3, CC8.1, A1.2 features implemented
12. Retention policy script exists (scripts/expire_old_redactions.py)

Dashboard audit:
13. Dashboard renders with live data
14. CSV/PDF export works
15. Alert indicators work

Production readiness:
16. docker compose up starts all services without errors
17. Health check passes
18. Rate limiting works
19. Request size limits work
20. Structured logging outputs JSON

Report a comprehensive summary with:
- PASS/FAIL for each check
- Any critical issues (security test failures = CRITICAL)
- Overall production readiness verdict
```

#### Commit

```bash
git add -A && git commit -m "Phase 7: Hardening (rate limit, structured logging, error handling) + evaluation + dashboard polish + documentation"
```

---

## 5. Coordination Rules

### File Ownership Boundaries

| Agent | Can Write | Cannot Touch |
|-------|-----------|--------------|
| **detection** | `src/detection/*`, `patterns/*`, `src/proxy/language_detector.py`, `scripts/evaluate_recall.py`, `scripts/benchmark_detection.py`, `tests/test_regex_detector.py`, `tests/test_ner_detector.py`, `tests/test_llm_detector.py`, `tests/test_merger.py`, `tests/test_german_pii.py`, `tests/fixtures/*` | `src/proxy/middleware.py`, `src/scanner/*`, `src/redaction/*`, `src/db/*`, `src/tenant/*`, `src/compliance/*`, `dashboard/*`, `src/api/*` |
| **middleware** | `src/main.py`, `src/config.py`, `src/auth/*`, `src/proxy/middleware.py`, `src/proxy/llm_client.py`, `src/scanner/*`, `src/redaction/*`, `src/api/routes/proxy.py`, `src/api/routes/health.py`, `src/api/dependencies.py`, `docker/Dockerfile`, `docker-compose.yml`, `pyproject.toml`, `.env.example`, `tests/conftest.py`, `tests/test_input_scanner.py`, `tests/test_output_scanner.py`, `tests/test_proxy.py`, `tests/test_security.py`, `tests/test_redaction.py` | `src/detection/*`, `patterns/*`, `src/db/*`, `src/tenant/*`, `src/compliance/*`, `dashboard/*`, `src/api/routes/admin.py`, `src/api/routes/audit.py` |
| **compliance** | `src/db/*`, `src/tenant/*`, `src/compliance/*`, `src/api/routes/admin.py`, `src/api/routes/audit.py`, `scripts/init_db.py`, `scripts/seed_test_data.py`, `scripts/generate_jwt.py`, `scripts/expire_old_redactions.py`, `docker/postgres/*`, `tests/test_compliance.py`, `tests/test_tenant_config.py`, `README.md` | `src/detection/*`, `patterns/*`, `src/proxy/*`, `src/scanner/*`, `src/redaction/*`, `dashboard/*`, `src/auth/*`, `src/main.py`, `src/config.py` |
| **dashboard** | `dashboard/*`, `docker/Dockerfile.dashboard`, `tests/test_dashboard.py` | Everything outside `dashboard/` |
| **reviewer** | *(nothing — read-only)* | *(everything)* |

### Conflict Avoidance Rules

1. **Shared files**: `src/main.py` is owned by middleware, but compliance needs to register admin/audit routers. **Rule**: compliance sends a hub message to middleware with the exact import + include_router lines. Middleware adds them. Compliance does NOT edit main.py directly.
2. **Interface contracts**: If an agent needs to change an interface another agent depends on (e.g., AllowlistEntry fields, LLMClient.generate signature), they MUST coordinate via hub messaging BEFORE making the change.
3. **Test fixtures**: `tests/conftest.py` is owned by middleware. Other agents who need fixtures should request them via hub message or define local fixtures in their own test files.
4. **pyproject.toml**: Owned by middleware. If detection needs presidio/spacy deps or dashboard needs streamlit/plotly, send a hub message with the dependency name. Middleware adds it.
5. **No simultaneous edits to the same file**: If two agents need to edit the same file (e.g., scripts/seed_test_data.py), the owner does it. The other agent sends requirements via hub.

### Parallel vs Sequential

- **Parallel**: Phases 1-4 and 6-7 have parallel work because agents own separate directories with no file conflicts.
- **Sequential within a phase**: Phase 1 has middleware depending on compliance's DB schema. Phase 5 has middleware depending on compliance's logger. These are marked as "Sequential (after parallel)" sections.
- **Cross-phase dependencies**: Phase 3's LLM detector depends on middleware's LLMClient.generate() — middleware must complete that in Phase 3 parallel work before detection can test llm_detector.py.

### Blocked Agent Protocol

1. If an agent is blocked waiting for another agent's deliverable, it sends a hub message to that agent: "Blocked: need <file/interface> for <task>. ETA?"
2. The blocked agent continues with any other independent tasks while waiting.
3. If the dependency is not delivered within the phase, the blocked agent reports the blocker in its output and yields with `result.error` describing the missing dependency.
4. The reviewer checks for cross-agent integration issues at each phase boundary.

---

## 6. Quick Reference

### Herdr Commands

```bash
# --- Agent management ---
herdr agent list                          # List all running agents
herdr agent start <name> --kind codex --pane <id>  # Start agent in pane
herdr agent stop <name>                   # Stop agent
herdr agent restart <name>                # Restart agent

# --- Pane management ---
herdr pane split --cwd "$PWD" --no-focus  # Split pane
herdr pane list                           # List panes
herdr pane focus <id>                     # Focus a pane

# --- Messaging ---
herdr send <agent-name> "<message>"       # Send message to agent
herdr send all "<broadcast>"              # Broadcast to all agents

# --- Monitoring ---
herdr logs <agent-name>                   # View agent logs
herdr status                              # Overall status
```

### Phase Summary

| Phase | Day | Focus | Lead Agents | Parallel? |
|-------|-----|-------|-------------|-----------|
| 1 | 1 | Foundation + proxy skeleton | middleware, compliance, detection, dashboard | Yes (4 agents) |
| 2 | 2 | Regex detection + German patterns | detection | Yes (3 agents) |
| 3 | 3 | NER + LLM contextual + merger | detection, middleware | Yes (2 agents) |
| 4 | 4 | Redaction engine + tenant config | middleware, compliance, detection | Yes (3 agents) |
| 5 | 5 | Scanners + proxy integration + security | middleware, compliance | Parallel then sequential |
| 6 | 6 | Dashboard + admin API | dashboard, compliance | Yes (2 agents) |
| 7 | 7 | Hardening + evaluation + docs | all 4 agents | Yes (4 agents) |

### Critical Security Tests (P0 — must pass before any merge)

```bash
# Run security tests — ALL must pass
pytest tests/test_security.py -v

# Key tests:
# - test_no_raw_pii_in_forwarded_prompt
# - test_no_pii_in_returned_response
# - test_output_block_mode_returns_422
# - test_redaction_event_logged_for_every_detection
# - test_audit_log_is_immutable
# - test_allowlisted_entity_not_redacted
# - test_scanner_failure_blocks_request
```

### End-to-End Verification

```bash
# Full system verification (Plan §A Quick Start)
docker compose up -d
curl http://localhost:8000/health
python scripts/init_db.py
python scripts/seed_test_data.py
python scripts/generate_jwt.py --tenant acme-corp --user user-123 --roles user

# Send PII through proxy
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-4o-mini", "messages": [{"role": "user", "content": "My name is Anna Müller, email a.mueller@gmx.de, Steuer-ID 12345678901"}]}'

# Check headers: X-PII-Redacted: true, X-Redaction-Count: 3
# Check forwarded prompt: no raw PII
# Check dashboard: localhost:8501
```
