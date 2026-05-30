# Autonomous Agent Marketplace — Phased Development Plan

> Project: 344-autonomous-agent-marketplace · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the three data-model suggestions. The database design adopts **data-model-suggestion-1 (Entity-Centric Normalized Relational)** as the canonical schema: it gives full referential integrity across publisher → agent → version → certification → listing → subscription → usage → payout, which is required for the marketplace's multi-party revenue-share, outcome-based billing, certification analytics, and EU AI Act compliance. JSONB columns (suggestion 2's strength) are retained only where the payload is genuinely schemaless (A2A cards, MCP manifests, OpenAPI specs, SLA terms). An append-only `audit_log` (suggestion 3's core idea) provides the immutable financial/compliance trail without forcing full event sourcing on every entity.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | **Python 3.12** | The product is LLM-heavy: semantic discovery, LLM-based adversarial certification, outcome verification, and composition recommendations all call models. Python has first-party SDKs for Anthropic, the A2A Python SDK, the Google ADK, the MCP Python SDK, and `stripe` — every integration in `standards.md` ships a Python client. |
| API framework | **FastAPI** | Native async (needed for fan-out LLM calls and webhook handling), automatic **OpenAPI 3.1** generation (a marketplace listing requirement — we both consume and emit OpenAPI), and Pydantic v2 request/response validation that maps cleanly to JSON Schema 2020-12. |
| Data validation | **Pydantic v2** | Single source of truth for API schemas, A2A Agent Card validation, MCP manifest validation, and config parsing. JSON Schema export aligns with standards.md. |
| Database | **PostgreSQL 16** | The schema needs foreign keys, partial/GIN indexes, full-text search (`tsvector`), `JSONB` for protocol specs, `PARTITION BY RANGE` for `usage_records` and `audit_log`, and `PERCENTILE_CONT` for SLA dashboards. All of these are used directly in data-model-suggestion-1. |
| Vector search | **pgvector** (Postgres extension) | Semantic/intent discovery needs embeddings. Keeping vectors in Postgres avoids a second datastore for the MVP and lets discovery JOIN against `agents` filters (category, certification status). |
| ORM / migrations | **SQLAlchemy 2.0 + Alembic** | Async ORM with typed models; Alembic gives versioned, reviewable migrations (Definition of Done requires migrations per phase). |
| Task queue | **Celery + Redis** | Certification runs (sandboxed adversarial testing), webhook processing, payout batch jobs, and SLA aggregation are long-running and must not block API requests. Redis doubles as cache and rate-limit store. |
| Sandboxing | **Docker (via Docker SDK for Python) with gVisor runtime** | Certification executes untrusted agent code/probes. Each test runs in a network-egress-restricted, resource-capped container. gVisor (`runsc`) adds a syscall-interception boundary appropriate for untrusted workloads. |
| LLM provider | **Anthropic Claude (claude-opus-4 / claude-sonnet-4)** via `anthropic` SDK, behind a provider abstraction | Used for discovery intent inference, adversarial probing, PII detection, outcome verification, and composition recommendations. Abstraction allows OpenAI/Gemini fallback. Prompt caching enabled on all system prompts. |
| Payments | **Stripe Connect + Agent Toolkit**; **x402** behind a feature flag | Stripe Connect natively supports multi-party splitting and connected-account payouts (revenue-share). x402 (`standards.md`) is the agent-native micropayment path, gated off by default pending the legal review the research flags. |
| Auth | **OAuth 2.1 / OIDC** (Authlib) with mandatory **PKCE**; SAML for enterprise SSO | `standards.md` mandates OAuth 2.1 + PKCE for MCP remote servers and headless agents. OIDC for human login; SAML for enterprise IAM. |
| Agent identity | **W3C DIDs (`did:web`) + Verifiable Credentials** | Publisher/agent DIDs and certification badges issued as VCs, per data-model-suggestion-1 standards alignment. Enables portable, third-party-verifiable certification. |
| Frontend | **Next.js 15 (App Router) + TypeScript + shadcn/ui + Tailwind** | The marketplace needs a buyer-facing catalogue, publisher console, certification report viewer, composition (workflow) editor, and billing dashboards. Server components for SEO-friendly listing pages; client components for the workflow editor. |
| Workflow editor | **React Flow** | The agent composition graph (bundles) is a DAG editor; React Flow is the standard node/edge canvas library. |
| Error format | **RFC 7807 Problem Details** | `standards.md` requires machine-readable errors for agent runtimes consuming the API. |
| Event format | **CloudEvents 1.0.2** | `audit_log` rows and the public event stream use the CloudEvents envelope (`ce_source`, `ce_type`). |
| Testing | **pytest + pytest-asyncio + testcontainers** (backend); **Vitest + Playwright** (frontend) | testcontainers spins real Postgres/Redis for integration tests; Playwright for E2E buyer/publisher flows. |
| Code quality | **Ruff** (lint+format), **mypy --strict**, **pre-commit** (Python); **ESLint + Prettier + tsc** (frontend) | Enforced in Definition of Done. |
| Packaging | **uv** (Python deps + venv), **pnpm** (frontend) | Fast, lockfile-based, reproducible. |
| Containerisation | **Docker + docker-compose** | Self-hosted, cloud, and hybrid deployment per README. compose wires api, worker, postgres, redis, frontend. |
| Observability | **OpenTelemetry → Prometheus + Grafana** | `trace_id` on usage records ties metering to traces; SLA/performance dashboards need metrics. |

### Project Structure

```
autonomous-agent-marketplace/
├── pyproject.toml
├── uv.lock
├── docker-compose.yml
├── Dockerfile.api
├── Dockerfile.worker
├── .env.example
├── alembic.ini
├── README.md
├── migrations/                      # Alembic revisions
│   └── versions/
├── src/
│   └── marketplace/
│       ├── __init__.py
│       ├── main.py                  # FastAPI app factory, RFC 7807 handlers
│       ├── config.py                # Pydantic Settings (env-driven)
│       ├── db.py                    # async engine, session, base
│       ├── deps.py                  # FastAPI dependencies (auth, db, rbac)
│       ├── errors.py                # ProblemDetail model + exception mapping
│       ├── models/                  # SQLAlchemy ORM models (one file per domain)
│       │   ├── org.py               # organisations, users, publishers
│       │   ├── registry.py          # agents, agent_versions
│       │   ├── certification.py     # certifications, cert_test_cases
│       │   ├── commerce.py          # pricing_plans, subscriptions, usage_records, invoices, royalty_attributions, disputes
│       │   ├── composition.py       # agent_bundles, reviews
│       │   └── audit.py             # audit_log
│       ├── schemas/                 # Pydantic request/response + protocol schemas
│       │   ├── a2a.py               # AgentCard models (A2A v1.0)
│       │   ├── mcp.py               # MCP manifest models
│       │   ├── openapi_spec.py      # OpenAPI 3.1 wrapper validation
│       │   └── ...                  # agent.py, billing.py, cert.py, etc.
│       ├── api/                     # FastAPI routers
│       │   ├── auth.py
│       │   ├── publishers.py
│       │   ├── agents.py
│       │   ├── versions.py
│       │   ├── discovery.py
│       │   ├── certification.py
│       │   ├── subscriptions.py
│       │   ├── usage.py             # metering ingest + webhooks
│       │   ├── billing.py
│       │   ├── bundles.py
│       │   ├── reviews.py
│       │   └── admin.py
│       ├── services/                # business logic (no FastAPI imports)
│       │   ├── registry_service.py
│       │   ├── validation_service.py   # A2A/MCP/OpenAPI validation + signature checks
│       │   ├── discovery_service.py    # embeddings + LLM intent inference
│       │   ├── certification_service.py
│       │   ├── billing_service.py      # metering, invoicing, royalty calc
│       │   ├── payout_service.py       # Stripe Connect splits
│       │   ├── composition_service.py  # DAG validation + recommendations
│       │   └── outcome_service.py      # LLM outcome verification
│       ├── llm/
│       │   ├── client.py            # provider abstraction + prompt caching
│       │   └── prompts/             # system/user prompt templates
│       ├── sandbox/
│       │   ├── runner.py            # Docker/gVisor execution wrapper
│       │   └── probes/              # adversarial probe definitions
│       ├── identity/
│       │   ├── did.py               # did:web resolution + issuance
│       │   └── vc.py                # Verifiable Credential issue/verify
│       ├── workers/
│       │   ├── celery_app.py
│       │   └── tasks.py             # cert runs, payouts, SLA aggregation
│       └── events/
│           └── cloudevents.py       # CloudEvents envelope helpers
├── tests/
│   ├── conftest.py                  # testcontainers fixtures
│   ├── fixtures/                    # sample agent cards, MCP manifests, probes
│   ├── unit/
│   ├── integration/
│   └── e2e/
└── frontend/
    ├── package.json
    ├── app/                         # Next.js App Router
    │   ├── (catalogue)/             # buyer discovery + listing pages
    │   ├── (console)/               # publisher console
    │   ├── (dashboard)/             # buyer subscriptions + billing
    │   └── api/                     # BFF route handlers
    ├── components/
    │   ├── ui/                      # shadcn/ui
    │   ├── workflow-editor/         # React Flow DAG editor
    │   └── cert-report/             # certification report viewer
    └── lib/
        └── api-client.ts            # generated from OpenAPI spec
```

The structure groups by **concern** (models, schemas, api, services, llm, sandbox, identity, workers). Every phase adds files within these existing groups; no phase restructures.

---

## Phase 1: Foundation & Core Domain

### Purpose
Stand up the runnable skeleton: configuration, async database, migrations, the organisation/user/publisher domain, RFC 7807 error handling, and the auth layer with OAuth 2.1 + PKCE. After this phase the application boots, connects to Postgres and Redis, serves an auto-generated OpenAPI 3.1 spec, and supports authenticated, role-scoped requests. Everything later builds on these primitives.

### Tasks

#### 1.1 — Project scaffold, config, and app factory

**What**: Create the `uv`-managed project, Docker/compose setup, FastAPI app factory, and environment-driven settings.

**Design**:
- `config.py` using `pydantic-settings`:
  ```python
  class Settings(BaseSettings):
      database_url: PostgresDsn
      redis_url: RedisDsn
      anthropic_api_key: SecretStr
      llm_default_model: str = "claude-sonnet-4"
      jwt_secret: SecretStr
      jwt_algorithm: str = "RS256"
      oidc_issuer: str
      stripe_secret_key: SecretStr | None = None
      stripe_connect_client_id: str | None = None
      x402_enabled: bool = False
      sandbox_runtime: str = "runsc"          # "runc" for dev
      sandbox_cpu_limit: float = 1.0
      sandbox_mem_limit_mb: int = 512
      sandbox_timeout_s: int = 30
      environment: Literal["dev", "staging", "prod"] = "dev"
      model_config = SettingsConfigDict(env_file=".env", env_prefix="MKT_")
  ```
- `main.py`: `create_app() -> FastAPI` registers routers, exception handlers, OTel middleware, and a `/healthz` (liveness) + `/readyz` (checks DB + Redis) endpoint.
- `docker-compose.yml` services: `postgres` (pgvector image `pgvector/pgvector:pg16`), `redis`, `api`, `worker`, `frontend`.

**Testing**:
- `Unit: Settings loads from env with prefix MKT_ → correct types, SecretStr masks value in repr`
- `Unit: missing required DATABASE_URL → ValidationError naming the field`
- `Integration: GET /healthz → 200 {"status":"ok"}`
- `Integration (testcontainers): GET /readyz with DB+Redis up → 200; with DB down → 503`

#### 1.2 — Async DB layer and Alembic baseline

**What**: Async SQLAlchemy engine/session and the first Alembic migration creating the organisation domain.

**Design**:
- `db.py`: `create_async_engine`, `async_sessionmaker`, `Base(DeclarativeBase)`, `get_session()` dependency yielding a transaction-scoped session.
- ORM models for `organisations`, `users`, `publishers` exactly per data-model-suggestion-1 §"Publisher & Organisation" (UUID PKs, `org_type`/`plan`/`role` CHECK enums mapped to SQLAlchemy `Enum`, `settings_json`/`config_json` as `JSONB`, unique `(org_id, email)`, `idx_publishers_verification`).
- Alembic env configured for async; `alembic revision --autogenerate` baseline.

**Testing**:
- `Integration (testcontainers): run all migrations up then down → no errors, schema empty after downgrade`
- `Unit: inserting user with role outside enum → IntegrityError/CHECK violation`
- `Unit: two users same email same org → UniqueViolation; same email different org → ok`

#### 1.3 — RFC 7807 error model

**What**: Uniform machine-readable errors for all endpoints.

**Design**:
- `errors.py`:
  ```python
  class ProblemDetail(BaseModel):
      type: str = "about:blank"
      title: str
      status: int
      detail: str | None = None
      instance: str | None = None
      errors: list[dict] | None = None     # field-level validation errors
  ```
- Register handlers for `RequestValidationError`, `HTTPException`, and a base `MarketplaceError` hierarchy (`NotFoundError`, `ConflictError`, `AuthzError`, `CertificationError`, `BillingError`). Content-Type `application/problem+json`.

**Testing**:
- `Unit: NotFoundError("agent", id) → ProblemDetail(status=404, type="/errors/not-found")`
- `Integration: POST with invalid body → 422 application/problem+json with errors[] listing field paths`

#### 1.4 — Auth: OIDC login, OAuth 2.1 + PKCE, RBAC

**What**: Human login via OIDC and machine/agent access via OAuth 2.1 authorization-code-with-PKCE; role-based dependencies.

**Design**:
- Authlib-backed `/auth/login`, `/auth/callback` (OIDC), `/auth/token` (OAuth 2.1, **PKCE required** — reject any auth-code grant lacking `code_challenge`), `/auth/introspect`.
- JWT (RS256) access tokens carry `sub`, `org_id`, `role`, `scopes[]`.
- `deps.py`: `current_user()`, `require_role(*roles)`, `require_scope(*scopes)`. Roles per data-model `users.role` enum: `owner, admin, developer, reviewer, billing, viewer`.
- API keys for agent runtimes: hashed (`argon2`) `api_keys` table (add migration) with scopes.

**Testing**:
- `Unit: token endpoint, auth-code grant without code_challenge → 400 invalid_request`
- `Unit: PKCE S256 verifier mismatch → 400 invalid_grant`
- `Integration: viewer calls publisher-only endpoint → 403 ProblemDetail`
- `Integration (mocked OIDC): /auth/callback with valid code → session + JWT issued`
- `Unit: expired JWT → 401; tampered signature → 401`

---

## Phase 2: Agent Registry & Protocol Validation

### Purpose
Implement the heart of the catalogue: publishers register agents and submit versioned metadata conforming to **A2A v1.0 Agent Cards**, **MCP server manifests**, and **OpenAPI 3.1** specs. This phase makes the marketplace a registry — agents can be created, versioned, and validated for protocol compliance and signed-Agent-Card trust, but not yet discovered, certified, or sold.

### Tasks

#### 2.1 — Agent and AgentVersion models + CRUD

**What**: Persist `agents` and `agent_versions` per data-model-suggestion-1 §"Agent Registry".

**Design**:
- ORM models with all columns: `category` CHECK enum, `tags TEXT[]` (GIN index), full-text `idx_agents_search`, `agent_versions` with `a2a_card_json`, `mcp_manifest_json`, `openapi_spec_json`, `runtime_type` enum, `runtime_config_json`, `models_used[]`, `auth_scopes[]`, `data_residency[]`, `ai_act_risk_class`, `version_ordinal`, `is_current`, status state machine.
- Version status state machine: `draft → submitted → certifying → certified → published`; plus `deprecated` (from published) and `rejected` (from certifying). `is_current` flips to the newest published version atomically.
- Endpoints:
  - `POST /publishers/{pid}/agents` → 201 Agent
  - `GET /agents/{id}` → Agent + current version
  - `POST /agents/{id}/versions` → 201 AgentVersion (status=draft)
  - `PATCH /agents/{id}/versions/{vid}` (draft only)
  - `POST /agents/{id}/versions/{vid}/submit` → status=submitted (triggers Phase 4 certification)

**Testing**:
- `Unit: version status transition draft→published directly → InvalidTransitionError`
- `Unit: publishing v2 sets v2.is_current=true and v1.is_current=false`
- `Integration: create agent with category not in enum → 422`
- `Integration: duplicate (publisher_id, slug) → 409 ConflictError`

#### 2.2 — A2A Agent Card validation & signature verification

**What**: Validate submitted A2A Agent Cards and verify signed-card domain ownership.

**Design**:
- `schemas/a2a.py`: Pydantic `AgentCard` per A2A v1.0 (`name`, `description`, `url`, `version`, `capabilities`, `skills[]`, `securitySchemes`, `provider`, signature block).
- `validation_service.validate_a2a_card(card_json) -> ValidationReport`.
- Signature verification: A2A signed Agent Cards use JWS; verify the signature against the key resolved from the card's declared domain (`did:web` / `.well-known`). Domain must match publisher's verified domain.
- `ValidationReport`:
  ```python
  class ValidationReport(BaseModel):
      valid: bool
      protocol: Literal["a2a","mcp","openapi"]
      errors: list[ValidationIssue]
      warnings: list[ValidationIssue]
      signature_verified: bool | None = None
  ```

**Testing**:
- `Fixture: valid signed Agent Card (tests/fixtures/a2a/valid_signed.json) → valid=True, signature_verified=True`
- `Fixture: card missing required "capabilities" → valid=False, error path "capabilities"`
- `Fixture: signature signed by non-matching domain → signature_verified=False`
- `Unit: unsigned card → signature_verified=None, valid=True (signature optional, flagged as warning)`

#### 2.3 — MCP manifest & OpenAPI 3.1 validation

**What**: Validate MCP server manifests and OpenAPI 3.1 capability specs; auto-derive an MCP server description from an OpenAPI spec when only the latter is supplied.

**Design**:
- `schemas/mcp.py`: `MCPManifest` (`name`, `version`, `tools[]` each with `name`, `description`, `inputSchema` (JSON Schema 2020-12), `transport` ∈ {stdio, http_sse, websocket}).
- OpenAPI 3.1 validation via `openapi-spec-validator`.
- `validation_service.openapi_to_mcp(spec) -> MCPManifest` — map each operation to an MCP tool (operationId → tool name, requestBody/params → inputSchema).
- On `submit`, require at least one of {A2A card, MCP manifest, OpenAPI spec}; record per-protocol `ValidationReport` on the version.

**Testing**:
- `Fixture: valid MCP manifest with 3 tools → valid, 3 tools parsed`
- `Fixture: MCP tool inputSchema invalid JSON Schema → valid=False`
- `Fixture: valid OpenAPI 3.1 → openapi_to_mcp yields one tool per operationId`
- `Unit: submit version with no specs → 422 "at least one capability spec required"`

#### 2.4 — Publisher DID & domain verification

**What**: Verify publisher domain ownership and issue/resolve `did:web` identifiers.

**Design**:
- `identity/did.py`: `resolve_did_web(did) -> DIDDocument`, `verify_domain_ownership(publisher, method)` where method ∈ {`dns_txt`, `well_known`}.
- On success set `publishers.verification_status='verified'`, `verified_at`, and `publishers.did = did:web:<domain>`.
- Endpoint `POST /publishers/{pid}/verify` → starts challenge; `POST /publishers/{pid}/verify/confirm`.

**Testing**:
- `Integration (mocked DNS): correct TXT challenge → verification_status='verified'`
- `Integration (mocked HTTP): /.well-known/did.json reachable & matching → verified`
- `Unit: mismatched challenge token → status stays 'pending'`

---

## Phase 3: Semantic & Intent-Based Discovery

### Purpose
Deliver the AI-native discovery layer — the marketplace's primary differentiator for non-technical buyers. Buyers search in natural language; the system infers intent, ranks agents by embedding similarity blended with quality signals, and supports conversational refinement. This phase makes the catalogue usable, not just populated.

### Tasks

#### 3.1 — Embedding index over agents

**What**: Generate and store embeddings for published agents; keep them fresh on publish.

**Design**:
- Add `agent_embeddings(agent_id UUID PK FK, embedding vector(1536), model TEXT, updated_at)` migration; pgvector `ivfflat` index on `embedding vector_cosine_ops`.
- Embedding text = `name + tagline + description + tags + capability/tool names` from current version specs.
- On version publish, enqueue Celery `reindex_agent(agent_id)`.

**Testing**:
- `Integration: publish agent → embedding row created with correct dim (1536)`
- `Unit: embedding text builder concatenates name/desc/tags/tool names deterministically`
- `Integration (mocked embeddings): re-publish updates embedding.updated_at`

#### 3.2 — LLM intent inference + hybrid ranking

**What**: Parse a natural-language query into structured intent, then rank agents by blended vector + keyword + quality score.

**Design**:
- `llm/prompts/intent.md` system prompt → returns structured JSON:
  ```python
  class SearchIntent(BaseModel):
      task: str
      category: AgentCategory | None
      required_capabilities: list[str]
      constraints: dict  # {"data_residency": "eu", "max_price_cents": 100, "certified_only": true}
  ```
- `discovery_service.search(query, filters) -> list[RankedAgent]`:
  1. LLM infers `SearchIntent` (prompt-cached system prompt).
  2. Embed the `task`; pgvector cosine top-K (K=50).
  3. Apply hard filters (category, `certified_only`, `data_residency`, price) in SQL.
  4. Final score = `0.6*cosine + 0.2*norm(avg_rating) + 0.1*norm(total_deployments) + 0.1*cert_bonus`.
- Endpoint `POST /discovery/search` → ranked results with `match_reasons` (LLM one-line justification per top result).

**Testing**:
- `Unit: intent parse "find a cheap EU-hosted lead qualifier" → category=sales, constraints.data_residency='eu', max_price set (mocked LLM)`
- `Integration: certified_only filter excludes uncertified agents`
- `Unit: ranking formula — agent A higher cosine but B higher rating → deterministic order given weights`
- `Integration: empty result set → 200 with [] and a suggestion message`

#### 3.3 — Conversational refinement

**What**: Multi-turn refinement of a search within a session.

**Design**:
- `POST /discovery/refine` with `{session_id, message}`; server keeps prior `SearchIntent` + result IDs in Redis (TTL 30 min), merges the refinement into a new intent, re-ranks.
- Refinement prompt receives prior intent + new message → updated `SearchIntent`.

**Testing**:
- `Integration: search then refine "cheaper" → max_price_cents lowered, results change`
- `Unit: expired session_id → 404 ProblemDetail`

---

## Phase 4: Automated Certification & Safety Pipeline

### Purpose
Build the trust engine — the second core differentiator. On version submission, agents run through a sandboxed pipeline combining static checks, LLM-driven adversarial probing (prompt injection, excessive agency, tool misuse), and contextual PII-leak detection, scored against the **OWASP Top 10 for Agentic Applications 2026** and **OWASP LLM Top 10 2025**. Results are persisted per test case and a pass issues a Verifiable Credential badge. This gates publishing.

### Tasks

#### 4.1 — Certification data model + orchestration

**What**: Persist `certifications` and `cert_test_cases` and orchestrate a run as a Celery task.

**Design**:
- ORM per data-model-suggestion-1 §"Certification & Safety" (`certifications` with summary counts + `vc_json`; `cert_test_cases` with `test_category`, `owasp_category`, `test_input`, `actual_output`, `passed`, `severity`, `latency_ms`).
- Celery `run_certification(agent_version_id)`: status `pending → running`, execute test batteries, aggregate, set final status: `passed` (no critical/high fails), `passed_with_warnings`, or `failed`. On submit (2.1), version goes `submitted → certifying`; on pass `→ certified`.
- Pipeline config (per category enable/disable + thresholds) in `config.py`.

**Testing**:
- `Integration: submit version → certification row created status=pending, version=certifying`
- `Unit: aggregation — any critical failure → status=failed; only low/info warnings → passed_with_warnings`
- `Integration: cert pass → version status=certified, cert.completed_at set`

#### 4.2 — Sandboxed execution runner

**What**: Execute probes against the submitted agent inside a locked-down container.

**Design**:
- `sandbox/runner.py`: `run_probe(version, probe) -> ProbeResult`. Container: image from `runtime_config_json`, `--runtime=runsc`, `--network=none` (or egress-allowlist proxy), CPU/mem limits and `sandbox_timeout_s` from settings, read-only rootfs, dropped capabilities.
- For `mcp_server`/`a2a_endpoint` runtimes, drive the agent over its declared transport from inside the network namespace; capture transcript.
- `ProbeResult{passed, actual_output, latency_ms, raised, logs}`.

**Testing**:
- `Integration (real Docker, marked @sandbox): probe that runs `sleep 999` → killed at timeout, passed=False, reason=timeout`
- `Integration: probe attempting outbound network → blocked, recorded`
- `Unit (mocked Docker SDK): resource limits passed to container.run with correct values`

#### 4.3 — LLM adversarial probes (prompt injection, excessive agency, tool misuse)

**What**: Generate and evaluate adversarial inputs mapped to OWASP categories.

**Design**:
- `sandbox/probes/`: declarative probe library, e.g.
  ```python
  class Probe(BaseModel):
      id: str
      test_category: TestCategory   # prompt_injection|excessive_agency|tool_misuse|...
      owasp_category: str           # "LLM01", "AGENTIC02", ...
      severity: Severity
      generator: Literal["static","llm"]
      template: str | None          # for static
  ```
- For `generator="llm"`: `llm/prompts/adversarial.md` produces injection payloads tailored to the agent's declared tools/skills.
- Evaluation: an LLM judge (`llm/prompts/judge.md`) scores the agent's transcript → `passed` (did the agent resist?), with rubric per category. Maps to `cert_test_cases`.

**Testing**:
- `Fixture: agent that echoes its system prompt when asked → prompt_injection probe LLM01 fails (critical)`
- `Fixture: agent that refuses out-of-scope tool calls → tool_misuse passes`
- `Unit (mocked LLM judge): judge JSON {"resisted":true} → passed=True`
- `Unit: probe library loads, every probe has a valid owasp_category`

#### 4.4 — Contextual PII-leak detection

**What**: Detect PII in agent outputs across probe transcripts.

**Design**:
- Two-stage: Presidio (regex+NER) for recall, then LLM contextual pass (`llm/prompts/pii.md`) to cut false positives (e.g., public example emails) → `pii_leak` test cases (OWASP LLM06), severity by PII class (SSN/health → critical).

**Testing**:
- `Unit: output containing SSN-like string in sensitive context → pii_leak critical`
- `Unit: documentation example email → LLM pass marks not-a-leak → passed`
- `Fixture: transcript with leaked customer phone → fails, severity high`

#### 4.5 — Verifiable Credential badge issuance

**What**: On pass, issue a W3C VC certification badge.

**Design**:
- `identity/vc.py`: `issue_cert_vc(certification) -> dict` — VC 2.0 with `issuer` = marketplace DID, `credentialSubject` = agent DID + `{badge, owaspSummary, expires}`, signed (Ed25519). Store in `certifications.vc_json`, `badge_slug` set, `expires_at` = +12 months. `verify_vc(vc)` for third parties.
- Endpoint `GET /agents/{id}/versions/{vid}/certification` returns report + VC.

**Testing**:
- `Unit: issued VC verifies with issuer key; tampered VC fails verification`
- `Integration: passed cert → vc_json populated, badge_slug='marketplace-verified'`
- `Unit: expired VC → verify returns valid=False reason=expired`

---

## Phase 5: Commerce — Pricing, Subscriptions & Metering

### Purpose
Establish the commercial core: publishers attach usage-based and outcome-based pricing plans; buyers subscribe; agent runtimes report metered usage (including outcomes and sub-agent calls). This phase turns the registry into a marketplace that can charge for value. Billing settlement, royalties, and disputes follow in Phase 6.

### Tasks

#### 5.1 — Pricing plans & subscriptions

**What**: Persist `pricing_plans` and `subscriptions`.

**Design**:
- ORM per data-model-suggestion-1 §"Commerce & Billing": `pricing_plans` (`plan_type` ∈ free/per_request/per_conversation/per_outcome/per_user_month/flat_month/custom, `metering_unit`, `included_units`, `overage_price_cents`, `payment_protocol` ∈ stripe/x402/invoice, `sla_json`); `subscriptions` (status enum, `workspace_slug`, `current_period_*`).
- Endpoints: `POST /agents/{id}/pricing-plans`, `POST /agents/{id}/subscribe` (creates Stripe subscription when plan is recurring), `POST /subscriptions/{id}/cancel`.

**Testing**:
- `Unit: per_outcome plan requires metering_unit → 422 if missing`
- `Integration (mocked Stripe): subscribe to flat_month → stripe_subscription_id stored, status=active`
- `Unit: subscribe to unpublished agent → 409`

#### 5.2 — Usage metering ingest

**What**: Partitioned `usage_records` and an authenticated ingest endpoint for agent runtimes.

**Design**:
- ORM `usage_records` **PARTITION BY RANGE (created_at)** with monthly partitions auto-created by a Celery beat job; columns include `metering_unit`, `quantity`, `outcome_verified`, `outcome_details_json`, `tokens_*`, `cost_cents`, `latency_ms`, `sub_agent_calls JSONB`, `trace_id`.
- `POST /usage` (API-key auth, scope `usage:write`) accepts a batch; idempotent on `(subscription_id, trace_id, metering_unit)`.
- `cost_cents` computed from the subscription's plan at ingest for per_request/per_conversation; per_outcome rows start `outcome_verified=NULL` pending Phase 6 verification.

**Testing**:
- `Integration: ingest batch of 100 → rows in correct monthly partition`
- `Unit: duplicate (subscription, trace_id, unit) → second ingest is no-op (idempotent)`
- `Integration: ingest with API key lacking usage:write scope → 403`
- `Unit: ingest for cancelled subscription → 409`

#### 5.3 — SLA metric capture

**What**: Capture latency/availability for SLA dashboards from usage records.

**Design**:
- Celery `aggregate_sla(subscription_id, period)` computes rolling `total_requests`, `avg_latency_ms`, `p95` (`PERCENTILE_CONT`), `sla_breaches` vs `pricing_plans.sla_json.p95_latency_ms`. Stored in a `sla_snapshots` table (add migration) keyed by subscription+period.
- `GET /subscriptions/{id}/sla` returns the latest snapshot.

**Testing**:
- `Unit: p95 computation over fixed latency set → expected value`
- `Integration: usage with latencies > SLA threshold → sla_breaches counted`

---

## Phase 6: Billing Settlement, Royalties & Disputes

### Purpose
Close the commercial loop with the marketplace's deepest differentiators: outcome verification, multi-party revenue-share/royalty payouts for sub-agent authors, invoice generation and settlement, and an LLM-assisted dispute process. After this phase the marketplace can take money, split it fairly across agent authors, and adjudicate contested outcomes.

### Tasks

#### 6.1 — LLM outcome verification

**What**: Verify whether outcome-priced usage events represent a genuine outcome (e.g., "lead truly qualified", "ticket resolved").

**Design**:
- `outcome_service.verify(usage_record) -> OutcomeVerdict`: `llm/prompts/outcome.md` receives the metering unit definition, the agent transcript/evidence in `outcome_details_json`, and the success criteria from the plan; returns `{verified: bool, confidence: float, rationale: str}`.
- Celery `verify_outcomes(period)` batch-updates `usage_records.outcome_verified` and writes `outcome_details_json.verified_by='ai'`. Low-confidence (<0.7) flagged for human reviewer.

**Testing**:
- `Unit (mocked LLM): transcript showing resolved ticket → verified=True confidence>0.7`
- `Unit: ambiguous transcript → confidence<0.7 → flagged_for_review=True, outcome_verified stays NULL`
- `Fixture: clearly unresolved → verified=False (not billable)`

#### 6.2 — Royalty attribution & revenue-share

**What**: Attribute revenue to sub-agent authors when an agent calls sub-agents from other publishers.

**Design**:
- ORM `royalty_attributions` per data-model-suggestion-1. `billing_service.attribute_royalties(usage_record)`: read `usage_record.sub_agent_calls[]`, resolve each sub-agent's publisher, compute `attributed_cents = parent_cost * revenue_share_pct` (share defined on sub-agent's plan), insert rows with `payout_status='pending'`.
- Guard against attribution cycles (agent → sub → parent) via a visited set on the trace.

**Testing**:
- `Unit: usage with 2 sub-agent calls → 2 royalty rows, cents sum ≤ parent cost`
- `Unit: cyclic sub-agent reference → cycle broken, logged, no double-attribution`
- `Integration: royalty rows queryable by sub_agent_publisher_id + payout_status (matches data-model example query)`

#### 6.3 — Invoice generation & settlement (Stripe Connect)

**What**: Generate monthly invoices and settle via Stripe Connect with multi-party splits.

**Design**:
- ORM `invoices` (status state machine `draft → issued → paid|overdue|disputed|void`, `line_items_json`).
- Celery `generate_invoices(period)`: aggregate verified usage per org into line items; create Stripe invoice; on payment webhook → `paid`, then `payout_service` transfers each publisher's net (gross − royalties owed to others + royalties earned) to their `stripe_connect_id`.
- Stripe webhook endpoint with signature verification (mirrors any HMAC webhook pattern).

**Testing**:
- `Integration (mocked Stripe): generate_invoices → invoice with correct line_items and totals`
- `Integration: Stripe webhook invalid signature → 401, no state change`
- `Integration: payment_succeeded webhook → invoice.status=paid, transfers created per publisher`
- `Unit: payout net = gross − royalties_paid_out + royalties_earned`

#### 6.4 — Disputes with LLM-assisted resolution

**What**: Dispute lifecycle with an LLM-proposed resolution.

**Design**:
- ORM `disputes` (types: outcome_rejected/overcharge/sla_violation/quality/other; status lifecycle open → investigating → resolved_buyer|resolved_publisher|escalated|closed).
- `POST /disputes` (buyer); `llm/prompts/dispute.md` reviews evidence (usage record, transcript, SLA snapshot) and proposes `resolution_json{recommendation, refund_cents, rationale}`. A human reviewer confirms/overrides; confirmation reverses royalties/invoice line as needed.

**Testing**:
- `Unit (mocked LLM): outcome_rejected with transcript showing non-resolution → recommends refund`
- `Integration: resolved_buyer with refund → invoice line credited, related royalty_attributions set payout_status='disputed'`
- `Unit: dispute on already-paid invoice → creates credit note, not mutation of paid invoice`

---

## Phase 7: Composition, Bundles & Reviews

### Purpose
Add the depth differentiator no incumbent exposes: first-class agent composition. Publishers compose multiple agents into workflow DAGs ("bundles"), test them, and publish them; an LLM recommends compatible agents. Reviews close the trust/social-proof loop.

### Tasks

#### 7.1 — Bundle model & DAG validation

**What**: Persist `agent_bundles` and validate the workflow graph.

**Design**:
- ORM `agent_bundles` per data-model-suggestion-1 (`workflow_json` = `{nodes[], edges[], entry_node}`, `agent_ids[]`).
- `composition_service.validate_workflow(workflow)`: DAG acyclicity, every node references a published+certified agent version, edge `condition` expressions parse, capability compatibility (an edge from A→B requires B's input schema satisfiable from A's output).
- Endpoints: `POST /publishers/{pid}/bundles`, `POST /bundles/{id}/validate`, `POST /bundles/{id}/publish`.

**Testing**:
- `Unit: workflow with cycle → ValidationError "cycle detected"`
- `Unit: node referencing uncertified agent → ValidationError`
- `Unit: incompatible A→B schemas → warning with field mismatch`
- `Integration: publish valid bundle → is_published=true`

#### 7.2 — Bundle test execution

**What**: Run a bundle end-to-end against sample input in the sandbox.

**Design**:
- `composition_service.test_run(bundle, input)`: topological execution from `entry_node`, evaluating edge conditions on each node's output, each agent invoked via the Phase 4 sandbox runner; returns per-node outputs + trace.

**Testing**:
- `Integration (mocked agents): linear A→B bundle → B receives A's output`
- `Unit: edge condition "score > 0.7" false → downstream node skipped`

#### 7.3 — LLM composition recommendations

**What**: Suggest agents that compose well with a given agent.

**Design**:
- `composition_service.recommend(agent_id) -> list[Recommendation]`: candidate set from embedding neighbours + complementary categories; LLM (`llm/prompts/compose.md`) ranks by capability complementarity (does candidate's input consume this agent's output?). Endpoint `GET /agents/{id}/recommendations`.

**Testing**:
- `Unit (mocked LLM): lead-scorer → recommends crm-updater (consumes score), not another scorer`
- `Integration: recommendations exclude same-publisher duplicates and uncertified agents`

#### 7.4 — Reviews

**What**: Verified-buyer reviews with rating rollups.

**Design**:
- ORM `reviews` (1 per org per agent, `verified_buyer` derived from an active/past subscription, rating 1–5). On insert, recompute `agents.avg_rating`, `review_count`.
- Endpoints `POST /agents/{id}/reviews`, `GET /agents/{id}/reviews`.

**Testing**:
- `Unit: second review by same org → 409`
- `Unit: reviewer with no subscription → verified_buyer=false`
- `Integration: 3 reviews (5,4,3) → avg_rating=4.00, review_count=3`

---

## Phase 8: Governance, Audit & Compliance

### Purpose
Make the marketplace enterprise-deployable: immutable audit logging (CloudEvents), private marketplaces, EU AI Act conformity gating for high-risk listings, and IAM cost controls. These features are procurement prerequisites for the enterprise buyers identified in `research.md`.

### Tasks

#### 8.1 — Audit log (CloudEvents) & event stream

**What**: Append-only audit trail for every state change.

**Design**:
- ORM `audit_log` (partitioned) per data-model-suggestion-1 with `ce_source`, `ce_type`. `events/cloudevents.py` wraps changes in a CloudEvents 1.0.2 envelope. A SQLAlchemy event listener (or service-layer decorator) records create/update/delete on key entities with `actor_type` ∈ user/agent/system/api_key/ai.
- `GET /audit` (admin/reviewer, org-scoped) with filters by entity, actor, date.

**Testing**:
- `Integration: publish agent → audit row type="com.marketplace.agent.published", actor recorded`
- `Unit: audit rows are never updated/deleted (no UPDATE/DELETE path)`

#### 8.2 — Private marketplaces & cost controls

**What**: Org-scoped private catalogues and spend caps.

**Design**:
- `agents.visibility='private'` + an `org_marketplace_allowlist` table; discovery filters by allowlist for the requesting org. Cost controls: per-org monthly budget; ingest/subscribe checks against budget → soft-warn then hard-block at threshold.

**Testing**:
- `Integration: private agent invisible to non-allowlisted org discovery; visible to allowlisted`
- `Unit: usage pushing org over hard budget → 402 PaymentRequired ProblemDetail`

#### 8.3 — EU AI Act conformity gating

**What**: Enforce documentation for high-risk listings before publish.

**Design**:
- On publish, if `agent_versions.ai_act_risk_class='high'`, require `ai_act_docs_json` to contain the mandated fields (risk-management ref, data-governance ref, technical-doc URL, human-oversight description, logging attestation). Block publish (`unacceptable` → reject outright). Surface compliance status on the listing.

**Testing**:
- `Unit: high-risk version missing technical_doc_url → publish blocked, ProblemDetail lists missing fields`
- `Unit: risk_class='unacceptable' → version auto-rejected`
- `Integration: high-risk with complete docs → publish succeeds`

---

## Phase 9: Frontend — Catalogue, Console & Dashboards

### Purpose
Ship the human-facing surfaces: the buyer catalogue with conversational discovery, the publisher console (submit/version/certification report), the composition (workflow) editor, and the billing/SLA dashboards. The API is the contract; the frontend consumes the generated OpenAPI client.

### Tasks

#### 9.1 — App shell, auth, and generated API client

**What**: Next.js shell with OIDC login and a typed client generated from the OpenAPI spec.

**Design**:
- `lib/api-client.ts` generated via `openapi-typescript` from `/openapi.json`. Auth via NextAuth (OIDC) storing the marketplace JWT. Layout with role-aware nav (buyer vs publisher vs admin).

**Testing**:
- `E2E (Playwright): login flow → lands on catalogue, session persists on reload`
- `Unit: client regeneration matches committed types (CI check)`

#### 9.2 — Catalogue & conversational discovery

**What**: Buyer browse/search with the conversational refinement panel.

**Design**:
- Server-component listing pages (SEO) for `/agents` and `/agents/[slug]` showing specs, certification badge + report link, pricing, reviews, SLA. Client `SearchChat` calls `/discovery/search` and `/discovery/refine`, rendering `match_reasons`.

**Testing**:
- `E2E: search "qualify EU leads cheaply" → results render with badges; refine "even cheaper" updates list`
- `E2E: agent detail page shows certification report and pricing`

#### 9.3 — Publisher console & certification report viewer

**What**: Agent submission, versioning, and a drill-down certification report.

**Design**:
- Forms upload A2A card/MCP manifest/OpenAPI spec (client-side schema lint before submit). `cert-report/` renders `cert_test_cases` grouped by OWASP category with severity colouring and the VC badge.

**Testing**:
- `E2E: submit version with valid specs → status transitions to certifying then certified (mocked worker)`
- `E2E: failed cert → report shows critical findings by OWASP category`

#### 9.4 — Composition (workflow) editor

**What**: React Flow DAG editor for bundles.

**Design**:
- Drag agents onto canvas, draw edges with condition expressions, run `validate`/`test`. Recommendation sidebar from `GET /agents/{id}/recommendations`.

**Testing**:
- `E2E: build A→B bundle, validate (pass), test run shows per-node output`
- `E2E: introduce cycle → validation error surfaced on canvas`

#### 9.5 — Billing, SLA & payout dashboards

**What**: Buyer spend/usage/SLA views and publisher revenue/royalty/payout views.

**Design**:
- Buyer: invoices, usage trends, SLA snapshots, dispute filing. Publisher: revenue, royalties earned/owed, payout schedule, dispute queue. Charts from the SLA and billing endpoints.

**Testing**:
- `E2E: publisher sees royalty earnings; files no dispute → totals match API`
- `E2E: buyer files a dispute from a usage row → dispute appears with LLM-proposed resolution`

---

## Phase 10: Hardening, Observability & Deployment

### Purpose
Production-readiness: rate limiting, OpenTelemetry tracing tied to `trace_id`, performance benchmarking dashboards with regression detection, security review against the OWASP frameworks, and reproducible self-hosted/cloud/hybrid deployment.

### Tasks

#### 10.1 — Rate limiting, security headers, secrets

**What**: Protect the API and runtime ingest path.

**Design**:
- Redis token-bucket rate limits per API key/IP (stricter on `/discovery` and `/usage`). Security headers, CORS allowlist, secret management via env/Vault. `x402` path remains flag-off.

**Testing**:
- `Integration: exceeding rate limit → 429 ProblemDetail with Retry-After`
- `Integration: CORS preflight from disallowed origin → blocked`

#### 10.2 — Observability & performance/regression dashboards

**What**: Tracing/metrics and agent performance benchmarking with regression alerts.

**Design**:
- OTel spans across api→service→sandbox/LLM; `trace_id` propagated into `usage_records`. Grafana dashboards: success rate, p50/p95 latency, cost, ratings per agent. Celery `detect_regression(agent_id)` compares current-window metrics to baseline; on significant drop, notify publisher.

**Testing**:
- `Integration: a request emits a connected trace (api→service span present)`
- `Unit: regression detector — 30% success-rate drop vs baseline → alert fired; noise within threshold → no alert`

#### 10.3 — Deployment, packaging & docs

**What**: Reproducible deploy in all three modes.

**Design**:
- Production `docker-compose.prod.yml` (api+worker+beat+postgres+redis+frontend+otel-collector); optional Helm chart for Kubernetes (cloud/hybrid). `.env.example` documents every setting. Operator runbook: backups, partition rotation, migration procedure.

**Testing**:
- `E2E (smoke, CI): docker-compose up → /readyz green, seed publisher can submit and certify a sample agent end-to-end`
- `Integration: alembic upgrade head on a fresh DB → all phases' schema present`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Core Domain          ─── required by everything
    │
Phase 2: Registry & Protocol Validation    ─── requires 1
    │
    ├── Phase 3: Semantic Discovery         ─── requires 2  ┐ can parallel
    ├── Phase 4: Certification & Safety      ─── requires 2  ┘ (3 & 4 independent)
    │        │
    │   Phase 5: Commerce (pricing/metering) ─── requires 2 (gating uses 4's certified status)
    │        │
    │   Phase 6: Billing/Royalties/Disputes  ─── requires 5 (outcome verify reuses 4's LLM layer)
    │
    ├── Phase 7: Composition/Bundles/Reviews ─── requires 4 (certified agents) + 2; reviews need 5
    │
    └── Phase 8: Governance/Audit/Compliance ─── requires 2 (audit hooks span all; can build alongside 5–7)

Phase 9: Frontend                           ─── requires 2–8 endpoints (build incrementally as APIs land)
Phase 10: Hardening/Observability/Deploy    ─── requires all; finalises production readiness
```

**Parallelism opportunities:**
- Phases **3 (Discovery)** and **4 (Certification)** are independent given Phase 2 — different teams/sessions.
- Phase **8 (Governance/Audit)** audit hooks and EU AI Act gating can be developed alongside Phases 5–7.
- Phase **9 (Frontend)** can begin per-surface as soon as the matching backend phase's endpoints exist (catalogue after 3, console after 4, billing after 6).

---

## Definition of Done (per phase)

Every phase must satisfy all of the following before it is considered complete:

1. All tasks implemented as specified.
2. All unit and integration tests pass (`pytest`); sandbox/real-dependency tests marked and runnable.
3. Frontend tests pass where applicable (`vitest`, `playwright`).
4. Linting and formatting pass (`ruff check`, `ruff format --check`; `eslint`, `prettier --check`).
5. Type checking passes (`mypy --strict`; `tsc --noEmit`).
6. Docker images build (`Dockerfile.api`, `Dockerfile.worker`, frontend) and `docker-compose up` reaches `/readyz` green.
7. The phase's feature works end-to-end (demonstrable via the listed E2E or integration test).
8. New config options documented in `.env.example` with defaults.
9. New/changed API endpoints appear in the auto-generated OpenAPI 3.1 spec and validate.
10. Database changes ship as a reviewed Alembic migration that upgrades and downgrades cleanly.
11. State changes emit an `audit_log` CloudEvents entry (from Phase 8 onward; hooks added retroactively where missing).
12. Errors return RFC 7807 `application/problem+json`.
```
