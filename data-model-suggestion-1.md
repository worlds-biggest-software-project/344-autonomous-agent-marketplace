# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Autonomous Agent Marketplace · Created: 2026-05-25

## Philosophy

This model gives every concept its own table with full referential integrity. Publishers register agents with versioned metadata conforming to A2A Agent Cards, MCP server descriptions, and OpenAPI 3.1 specs. Buyers discover agents through semantic search, deploy them to workspaces, and track usage through metered billing records. The certification pipeline stores test results with per-test-case granularity, linking adversarial probes to OWASP categories. Revenue-share flows through a dedicated royalty attribution table that traces sub-agent calls back to their authors.

The key insight is that an agent marketplace has four distinct data domains that interact: the **registry domain** (publishers, agents, versions, capabilities), the **commerce domain** (subscriptions, usage metering, invoices, payouts, disputes), the **certification domain** (test suites, test runs, vulnerability findings), and the **composition domain** (bundles, workflow graphs, sub-agent dependencies). These domains meet at the agent listing — a certified, priced, composable entity. Keeping them normalized allows independent evolution: a new billing model doesn't require registry schema changes, and a new certification check doesn't affect the commerce layer.

**Best for:** Teams building a production marketplace with complex billing models (usage-based + outcome-based), multi-party revenue-share, regulatory compliance (EU AI Act), and deep certification analytics. Ideal for enterprise deployments where billing disputes, audit trails, and cross-agent dependency analysis require precise relational data.

**Trade-offs:**
- (+) Full referential integrity from publisher → agent → version → certification → listing → subscription → usage → payout
- (+) Revenue-share and royalty attribution with foreign-key-backed sub-agent tracing
- (+) Certification results queryable by OWASP category, severity, and agent version
- (+) SLA metrics as first-class columns enable SQL-native performance dashboards
- (+) EU AI Act risk classification and documentation tracked per agent version
- (-) 22 tables — larger migration surface
- (-) Loading a complete agent listing with certifications, pricing, and reviews requires multiple JOINs
- (-) Usage metering tables grow to billions of rows at marketplace scale
- (-) Revenue-share calculation spans subscriptions, usage_records, and royalty_attributions

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| A2A v1.0 | Agent Cards stored in `agent_versions.a2a_card_json`; signed cards validated at registration |
| MCP | MCP server descriptions in `agent_versions.mcp_manifest_json`; tool schemas in agent capabilities |
| OpenAPI 3.1 | HTTP capability specs in `agent_versions.openapi_spec_json` |
| OAuth 2.0 / PKCE | Auth scopes per agent in `agent_versions.auth_scopes`; buyer delegated auth |
| W3C DIDs | Publisher and agent DIDs in `publishers.did` and `agents.did` |
| W3C Verifiable Credentials | Certification badges as VCs in `certifications.vc_json` |
| OWASP Agentic Top 10 | Certification test categories in `cert_test_cases.owasp_category` |
| OWASP LLM Top 10 | Prompt injection and PII tests in certification pipeline |
| EU AI Act | Risk classification in `agent_versions.ai_act_risk_class`; documentation in `agent_versions.ai_act_docs_json` |
| NIST AI RMF | Risk management metadata in agent certification |
| x402 | Payment protocol option in `pricing_plans.payment_protocol` |
| Stripe Connect | Payout infrastructure tracked in `publishers.stripe_connect_id` |
| CloudEvents 1.0.2 | Audit log events use CloudEvents envelope |
| ISO/IEC 42001 | AI management system evidence in certification records |

---

## Core Tables

### Publisher & Organisation

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT UNIQUE NOT NULL,
    org_type        TEXT NOT NULL DEFAULT 'buyer'
                    CHECK (org_type IN ('buyer','publisher','both','platform')),
    plan            TEXT NOT NULL DEFAULT 'free' CHECK (plan IN ('free','pro','enterprise')),
    settings_json   JSONB NOT NULL DEFAULT '{}',
    billing_email   TEXT,
    stripe_customer_id TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    email           TEXT NOT NULL,
    full_name       TEXT,
    role            TEXT NOT NULL DEFAULT 'member'
                    CHECK (role IN ('owner','admin','developer','reviewer','billing','viewer')),
    auth_provider   TEXT DEFAULT 'local' CHECK (auth_provider IN ('local','google','github','saml')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, email)
);

CREATE TABLE publishers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) UNIQUE,
    display_name    TEXT NOT NULL,
    slug            TEXT UNIQUE NOT NULL,
    description     TEXT,
    website_url     TEXT,
    logo_url        TEXT,
    did             TEXT,
    stripe_connect_id TEXT,
    payout_schedule TEXT NOT NULL DEFAULT 'monthly'
                    CHECK (payout_schedule IN ('weekly','biweekly','monthly')),
    verification_status TEXT NOT NULL DEFAULT 'unverified'
                    CHECK (verification_status IN ('unverified','pending','verified','suspended')),
    verified_at     TIMESTAMPTZ,
    soc2_certified  BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_publishers_verification ON publishers(verification_status);
```

### Agent Registry

```sql
CREATE TABLE agents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publisher_id    UUID NOT NULL REFERENCES publishers(id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    did             TEXT,
    tagline         TEXT,
    description     TEXT,
    category        TEXT NOT NULL CHECK (category IN ('customer_service','sales','marketing',
                                                       'engineering','data','finance','hr',
                                                       'legal','operations','security',
                                                       'research','creative','general')),
    tags            TEXT[] NOT NULL DEFAULT '{}',
    logo_url        TEXT,
    is_published    BOOLEAN NOT NULL DEFAULT false,
    is_featured     BOOLEAN NOT NULL DEFAULT false,
    visibility      TEXT NOT NULL DEFAULT 'public'
                    CHECK (visibility IN ('public','private','unlisted')),
    total_deployments INTEGER NOT NULL DEFAULT 0,
    avg_rating      NUMERIC(3,2),
    review_count    INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (publisher_id, slug)
);

CREATE INDEX idx_agents_category ON agents(category) WHERE is_published;
CREATE INDEX idx_agents_tags ON agents USING GIN (tags);
CREATE INDEX idx_agents_search ON agents USING GIN (to_tsvector('english', name || ' ' || COALESCE(description, '')));

CREATE TABLE agent_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_id        UUID NOT NULL REFERENCES agents(id),
    version         TEXT NOT NULL,
    version_ordinal INTEGER NOT NULL,
    release_notes   TEXT,
    status          TEXT NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft','submitted','certifying','certified',
                                      'published','deprecated','rejected')),

    a2a_card_json   JSONB,
    -- A2A Agent Card with signed capabilities
    mcp_manifest_json JSONB,
    -- MCP server description with tool schemas
    openapi_spec_json JSONB,
    -- OpenAPI 3.1 capability spec

    runtime_type    TEXT NOT NULL CHECK (runtime_type IN ('docker','kubernetes','lambda',
                                                           'cloud_run','azure_container',
                                                           'mcp_server','a2a_endpoint','saas')),
    runtime_config_json JSONB NOT NULL DEFAULT '{}',
    models_used     TEXT[] NOT NULL DEFAULT '{}',
    model_providers TEXT[] NOT NULL DEFAULT '{}',
    auth_scopes     TEXT[] NOT NULL DEFAULT '{}',
    data_residency  TEXT[] NOT NULL DEFAULT '{}',

    ai_act_risk_class TEXT CHECK (ai_act_risk_class IN ('minimal','limited','high','unacceptable')),
    ai_act_docs_json JSONB,

    source_code_hash TEXT,
    is_current      BOOLEAN NOT NULL DEFAULT false,
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (agent_id, version)
);

CREATE INDEX idx_agent_versions_agent ON agent_versions(agent_id, version_ordinal DESC);
CREATE INDEX idx_agent_versions_status ON agent_versions(status);
```

### Certification & Safety

```sql
CREATE TABLE certifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_version_id UUID NOT NULL REFERENCES agent_versions(id),
    cert_type       TEXT NOT NULL CHECK (cert_type IN ('automated','manual','third_party')),
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending','running','passed','failed',
                                      'passed_with_warnings','expired')),
    badge_slug      TEXT,
    vc_json         JSONB,
    -- W3C Verifiable Credential for the certification badge
    total_tests     INTEGER NOT NULL DEFAULT 0,
    passed_tests    INTEGER NOT NULL DEFAULT 0,
    failed_tests    INTEGER NOT NULL DEFAULT 0,
    warning_tests   INTEGER NOT NULL DEFAULT 0,
    critical_findings INTEGER NOT NULL DEFAULT 0,
    reviewer_id     UUID REFERENCES users(id),
    review_notes    TEXT,
    expires_at      TIMESTAMPTZ,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_certs_version ON certifications(agent_version_id);

CREATE TABLE cert_test_cases (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    certification_id UUID NOT NULL REFERENCES certifications(id) ON DELETE CASCADE,
    test_category   TEXT NOT NULL CHECK (test_category IN ('prompt_injection','pii_leak',
                                                            'malicious_command','excessive_agency',
                                                            'tool_misuse','data_exfiltration',
                                                            'goal_misalignment','trust_delegation',
                                                            'memory_poisoning','functional',
                                                            'performance','compliance')),
    owasp_category  TEXT,
    -- e.g., 'LLM01' (prompt injection), 'AGENTIC03' (tool misuse)
    test_name       TEXT NOT NULL,
    test_input      JSONB NOT NULL,
    expected_behaviour TEXT,
    actual_output   JSONB,
    passed          BOOLEAN,
    severity        TEXT CHECK (severity IN ('critical','high','medium','low','info')),
    details_json    JSONB,
    latency_ms      INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cert_tests_cert ON cert_test_cases(certification_id);
CREATE INDEX idx_cert_tests_category ON cert_test_cases(test_category, passed);
```

### Commerce & Billing

```sql
CREATE TABLE pricing_plans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_id        UUID NOT NULL REFERENCES agents(id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    plan_type       TEXT NOT NULL CHECK (plan_type IN ('free','per_request','per_conversation',
                                                        'per_outcome','per_user_month',
                                                        'flat_month','custom')),
    price_cents     BIGINT,
    currency        TEXT NOT NULL DEFAULT 'usd',
    metering_unit   TEXT,
    -- e.g., 'request', 'conversation', 'resolution', 'qualified_lead'
    included_units  INTEGER,
    overage_price_cents BIGINT,
    payment_protocol TEXT NOT NULL DEFAULT 'stripe'
                    CHECK (payment_protocol IN ('stripe','x402','invoice')),
    sla_json        JSONB,
    -- Example: {"uptime_pct":99.9,"p95_latency_ms":5000,"support_response_hours":4}
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (agent_id, slug)
);

CREATE TABLE subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    agent_id        UUID NOT NULL REFERENCES agents(id),
    pricing_plan_id UUID NOT NULL REFERENCES pricing_plans(id),
    agent_version   TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active','paused','cancelled','expired','trial')),
    workspace_slug  TEXT,
    config_json     JSONB NOT NULL DEFAULT '{}',
    stripe_subscription_id TEXT,
    trial_ends_at   TIMESTAMPTZ,
    current_period_start DATE,
    current_period_end DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_subs_org ON subscriptions(org_id) WHERE status = 'active';
CREATE INDEX idx_subs_agent ON subscriptions(agent_id);

CREATE TABLE usage_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id UUID NOT NULL REFERENCES subscriptions(id),
    org_id          UUID NOT NULL,
    agent_id        UUID NOT NULL,
    metering_unit   TEXT NOT NULL,
    quantity        INTEGER NOT NULL DEFAULT 1,
    outcome_verified BOOLEAN,
    outcome_details_json JSONB,
    -- Example: {"resolution_detected":true,"confidence":0.92,"verified_by":"ai"}
    tokens_input    BIGINT,
    tokens_output   BIGINT,
    cost_cents      BIGINT,
    latency_ms      INTEGER,
    sub_agent_calls JSONB,
    -- Example: [{"agent_id":"uuid","agent_slug":"lead-scorer","usage_record_id":"uuid","cost_cents":5}]
    trace_id        TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_usage_sub ON usage_records(subscription_id, created_at DESC);
CREATE INDEX idx_usage_agent ON usage_records(agent_id, created_at DESC);

CREATE TABLE invoices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft','issued','paid','overdue','disputed','void')),
    subtotal_cents  BIGINT NOT NULL DEFAULT 0,
    tax_cents       BIGINT NOT NULL DEFAULT 0,
    total_cents     BIGINT NOT NULL DEFAULT 0,
    currency        TEXT NOT NULL DEFAULT 'usd',
    line_items_json JSONB NOT NULL DEFAULT '[]',
    stripe_invoice_id TEXT,
    issued_at       TIMESTAMPTZ,
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_invoices_org ON invoices(org_id, period_start DESC);

CREATE TABLE royalty_attributions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    usage_record_id UUID NOT NULL REFERENCES usage_records(id),
    parent_agent_id UUID NOT NULL REFERENCES agents(id),
    sub_agent_id    UUID NOT NULL REFERENCES agents(id),
    sub_agent_publisher_id UUID NOT NULL REFERENCES publishers(id),
    revenue_share_pct NUMERIC(5,2) NOT NULL,
    attributed_cents BIGINT NOT NULL,
    payout_status   TEXT NOT NULL DEFAULT 'pending'
                    CHECK (payout_status IN ('pending','included','paid','disputed')),
    payout_id       UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_royalty_sub_agent ON royalty_attributions(sub_agent_id, created_at DESC);
CREATE INDEX idx_royalty_publisher ON royalty_attributions(sub_agent_publisher_id, payout_status);

CREATE TABLE disputes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    usage_record_id UUID REFERENCES usage_records(id),
    invoice_id      UUID REFERENCES invoices(id),
    dispute_type    TEXT NOT NULL CHECK (dispute_type IN ('outcome_rejected','overcharge',
                                                           'sla_violation','quality','other')),
    status          TEXT NOT NULL DEFAULT 'open'
                    CHECK (status IN ('open','investigating','resolved_buyer',
                                      'resolved_publisher','escalated','closed')),
    description     TEXT NOT NULL,
    evidence_json   JSONB,
    resolution_json JSONB,
    resolved_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ
);

CREATE INDEX idx_disputes_org ON disputes(org_id, status);
```

### Composition & Reviews

```sql
CREATE TABLE agent_bundles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publisher_id    UUID NOT NULL REFERENCES publishers(id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    description     TEXT,
    workflow_json   JSONB NOT NULL,
    -- Example: {"nodes":[{"id":"n1","agent_slug":"lead-scorer","version":"1.2.0"},
    --   {"id":"n2","agent_slug":"crm-updater","version":"2.0.1"}],
    --  "edges":[{"from":"n1","to":"n2","condition":"score > 0.7"}],
    --  "entry_node":"n1"}
    agent_ids       UUID[] NOT NULL,
    is_published    BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (publisher_id, slug)
);

CREATE TABLE reviews (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_id        UUID NOT NULL REFERENCES agents(id),
    reviewer_org_id UUID NOT NULL REFERENCES organisations(id),
    reviewer_user_id UUID NOT NULL REFERENCES users(id),
    rating          INTEGER NOT NULL CHECK (rating >= 1 AND rating <= 5),
    title           TEXT,
    body            TEXT,
    verified_buyer  BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (agent_id, reviewer_org_id)
);

CREATE INDEX idx_reviews_agent ON reviews(agent_id, rating);
```

### Audit

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID,
    actor_id        UUID,
    actor_type      TEXT NOT NULL CHECK (actor_type IN ('user','agent','system','api_key','ai')),
    action          TEXT NOT NULL,
    entity_type     TEXT NOT NULL,
    entity_id       UUID,
    changes_json    JSONB,
    ip_address      INET,
    ce_source       TEXT,
    ce_type         TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_org ON audit_log(org_id, created_at DESC);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

---

## Example Queries

### Agent listing with certification and pricing

```sql
SELECT a.name, a.category, a.avg_rating, a.total_deployments,
       p.display_name AS publisher, p.verification_status,
       av.version, av.ai_act_risk_class,
       c.status AS cert_status, c.badge_slug,
       pp.plan_type, pp.price_cents, pp.metering_unit
FROM agents a
JOIN publishers p ON p.id = a.publisher_id
JOIN agent_versions av ON av.agent_id = a.id AND av.is_current
LEFT JOIN certifications c ON c.agent_version_id = av.id AND c.status = 'passed'
LEFT JOIN pricing_plans pp ON pp.agent_id = a.id AND pp.is_active
WHERE a.is_published AND a.category = $1
ORDER BY a.avg_rating DESC NULLS LAST;
```

### Revenue-share payouts by publisher this month

```sql
SELECT pub.display_name, a.name AS agent_name,
       SUM(ra.attributed_cents) AS total_royalties_cents,
       COUNT(*) AS attribution_count
FROM royalty_attributions ra
JOIN publishers pub ON pub.id = ra.sub_agent_publisher_id
JOIN agents a ON a.id = ra.sub_agent_id
WHERE ra.created_at >= date_trunc('month', now())
  AND ra.payout_status = 'pending'
GROUP BY pub.display_name, a.name
ORDER BY total_royalties_cents DESC;
```

### Certification results by OWASP category

```sql
SELECT tc.owasp_category, tc.test_category,
       COUNT(*) AS total, COUNT(*) FILTER (WHERE tc.passed) AS passed,
       COUNT(*) FILTER (WHERE NOT tc.passed AND tc.severity = 'critical') AS critical_fails
FROM cert_test_cases tc
JOIN certifications c ON c.id = tc.certification_id
WHERE c.agent_version_id = $1
GROUP BY tc.owasp_category, tc.test_category
ORDER BY critical_fails DESC;
```

### SLA compliance check

```sql
SELECT a.name, pp.sla_json->>'uptime_pct' AS sla_uptime,
       COUNT(*) AS total_requests,
       AVG(ur.latency_ms) AS avg_latency_ms,
       PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY ur.latency_ms) AS p95_latency_ms,
       COUNT(*) FILTER (WHERE ur.latency_ms > (pp.sla_json->>'p95_latency_ms')::int) AS sla_breaches
FROM usage_records ur
JOIN subscriptions s ON s.id = ur.subscription_id
JOIN agents a ON a.id = ur.agent_id
JOIN pricing_plans pp ON pp.id = s.pricing_plan_id
WHERE s.org_id = $1 AND ur.created_at >= date_trunc('month', now())
GROUP BY a.name, pp.sla_json;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Publisher & Organisation | 3 | organisations, users, publishers |
| Agent Registry | 2 | agents, agent_versions |
| Certification & Safety | 2 | certifications, cert_test_cases |
| Commerce & Billing | 6 | pricing_plans, subscriptions, usage_records (partitioned), invoices, royalty_attributions, disputes |
| Composition & Reviews | 2 | agent_bundles, reviews |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **16** | |

---

## Key Design Decisions

1. **Agent versions with A2A/MCP/OpenAPI specs** — each agent version carries its A2A Agent Card, MCP server manifest, and OpenAPI spec as JSONB columns. This enables protocol-specific discovery ("find all agents with MCP tools matching 'search'") and version-pinned deployments.

2. **Certification as structured test results** — `cert_test_cases` stores individual test results with OWASP categories, severity levels, and actual outputs. This enables per-category analytics ("which OWASP category has the highest failure rate across all agents?") and regression tracking between versions.

3. **Revenue-share via royalty_attributions** — when an agent calls a sub-agent from another publisher, `royalty_attributions` records the attributed revenue share with foreign keys to both agents and the sub-agent's publisher. This enables automated payout calculation and dispute resolution.

4. **Outcome-based billing with verification** — `usage_records.outcome_verified` and `outcome_details_json` track whether an outcome (resolution, qualified lead) was verified by AI or human review. This is the foundation for pay-per-result pricing models.

5. **Disputes as first-class entities** — billing disputes have their own lifecycle (open → investigating → resolved) with evidence and resolution records. This is essential for a marketplace where outcome verification can be contested.

6. **W3C Verifiable Credentials for certification badges** — `certifications.vc_json` stores the certification as a W3C VC, enabling agents to present their certification to third parties (other marketplaces, enterprise buyers) without relying on the marketplace as a verification intermediary.

7. **EU AI Act risk classification per version** — `agent_versions.ai_act_risk_class` tracks the AI Act classification (minimal/limited/high/unacceptable), and `ai_act_docs_json` stores the required technical documentation. High-risk agents require certification before publishing.

8. **Agent bundles as workflow graphs** — `agent_bundles.workflow_json` stores the composition DAG with agent references, edge conditions, and entry points. Bundles reference specific agent versions for reproducibility.

9. **SLA tracking via pricing plan metadata** — SLA terms (uptime, latency, support response) are stored in `pricing_plans.sla_json`. SLA compliance is calculated from `usage_records` at query time, avoiding separate SLA tracking tables.

10. **Publisher verification and Stripe Connect** — publishers have a verification lifecycle (unverified → pending → verified) and a Stripe Connect account for automated payouts. SOC 2 certification is tracked as a boolean flag for trust signalling.
