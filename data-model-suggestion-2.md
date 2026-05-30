# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Autonomous Agent Marketplace · Created: 2026-05-25

## Philosophy

This model collapses the 16-table normalized schema into 7 core tables by embedding related data as JSONB documents within their parent entities. Organisations embed their user roster, publisher profile, and Stripe integration. Agents embed their version history, A2A/MCP/OpenAPI specs, certification results, pricing plans, reviews, and bundle memberships. Subscriptions embed usage records and royalty attributions. The principle: loading a single agent listing should return everything needed to render the marketplace page — specs, certifications, pricing, reviews — in one query.

The relational anchors — `agents`, `subscriptions`, and `invoices` — remain as standalone tables because they have independent lifecycles: agents are the primary unit of discovery and certification, subscriptions track ongoing buyer-publisher relationships, and invoices drive financial reconciliation. Disputes remain relational for auditability.

**Best for:** Early-stage marketplaces prioritising development speed, single-query agent listing pages, and rapid iteration on certification criteria and pricing models. Ideal for single-operator or small-team deployments where the agent listing page and subscription dashboard are the primary access patterns.

**Trade-offs:**
- (+) 7 tables — minimal migration surface as marketplace features evolve
- (+) Single-row fetch loads a complete agent listing with specs, certifications, pricing, and reviews
- (+) New certification test categories and pricing models added without schema migration
- (+) Agent version history embedded enables quick version comparison
- (-) Cross-agent analytics (failure rate by OWASP category) require JSONB extraction
- (-) No foreign-key enforcement between embedded sub-agent references and agents
- (-) Usage record arrays in subscriptions grow unbounded for high-volume agents
- (-) Revenue-share calculation requires JSONB path expressions across subscriptions

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| A2A v1.0 | Agent Cards in `agents.versions_json[].a2a_card` |
| MCP | Server manifests in `agents.versions_json[].mcp_manifest` |
| OpenAPI 3.1 | Capability specs in `agents.versions_json[].openapi_spec` |
| OAuth 2.0 / PKCE | Auth scopes in `agents.versions_json[].auth_scopes` |
| W3C DIDs | Publisher DIDs in `organisations.publisher_json.did`; agent DIDs in `agents.did` |
| W3C Verifiable Credentials | Certification VCs in `agents.certifications_json[].vc` |
| OWASP Agentic Top 10 | Test categories in `agents.certifications_json[].tests[].owasp_category` |
| EU AI Act | Risk classification in `agents.versions_json[].ai_act_risk_class` |
| x402 | Payment protocol in `agents.pricing_json[].payment_protocol` |
| Stripe Connect | Connect ID in `organisations.publisher_json.stripe_connect_id` |
| CloudEvents 1.0.2 | Audit log events use CloudEvents envelope |

---

## Core Tables

### organisations

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT UNIQUE NOT NULL,
    org_type        TEXT NOT NULL DEFAULT 'buyer'
                    CHECK (org_type IN ('buyer','publisher','both','platform')),
    plan            TEXT NOT NULL DEFAULT 'free' CHECK (plan IN ('free','pro','enterprise')),
    settings_json   JSONB NOT NULL DEFAULT '{}',

    users_json      JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","email":"dev@example.com","full_name":"Jane Dev",
    --   "role":"developer","auth_provider":"github","is_active":true}]

    publisher_json  JSONB,
    -- Example: {"display_name":"Acme AI","slug":"acme-ai","description":"Enterprise agents",
    --   "website_url":"https://acme.ai","logo_url":"https://...",
    --   "did":"did:web:acme.ai","stripe_connect_id":"acct_xxx",
    --   "payout_schedule":"monthly","verification_status":"verified",
    --   "soc2_certified":true,"verified_at":"2026-05-01T..."}

    billing_json    JSONB NOT NULL DEFAULT '{}',
    -- Example: {"email":"billing@acme.ai","stripe_customer_id":"cus_xxx",
    --   "default_payment_method":"pm_xxx","tax_id":"US123456"}

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_orgs_slug ON organisations(slug);
CREATE INDEX idx_orgs_publisher ON organisations USING GIN (publisher_json) WHERE org_type IN ('publisher','both');
```

### agents

Agents remain relational — they are the primary unit of discovery and the anchor for the entire marketplace experience.

```sql
CREATE TABLE agents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
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
    visibility      TEXT NOT NULL DEFAULT 'public'
                    CHECK (visibility IN ('public','private','unlisted')),

    versions_json   JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"version":"1.2.0","version_ordinal":3,"status":"published",
    --   "release_notes":"Added MCP tool support",
    --   "a2a_card":{"name":"lead-scorer","capabilities":["score","qualify"],...},
    --   "mcp_manifest":{"name":"lead-scorer","tools":[{"name":"score_lead",...}]},
    --   "openapi_spec":{"openapi":"3.1.0","info":{"title":"Lead Scorer"},...},
    --   "runtime_type":"mcp_server","runtime_config":{"image":"acme/lead-scorer:1.2.0"},
    --   "models_used":["claude-sonnet-4-6"],"model_providers":["anthropic"],
    --   "auth_scopes":["crm:read","crm:write"],"data_residency":["us","eu"],
    --   "ai_act_risk_class":"limited","ai_act_docs":{"technical_doc_url":"..."},
    --   "is_current":true,"published_at":"2026-05-20T...","created_at":"..."}]

    certifications_json JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","version":"1.2.0","cert_type":"automated","status":"passed",
    --   "badge_slug":"marketplace-verified",
    --   "vc":{"@context":"https://www.w3.org/2018/credentials/v1",...},
    --   "summary":{"total":45,"passed":44,"failed":1,"warnings":3,"critical":0},
    --   "tests":[{"category":"prompt_injection","owasp_category":"LLM01",
    --     "name":"System prompt extraction","passed":true,"severity":"critical"},
    --    {"category":"pii_leak","owasp_category":"LLM06",
    --     "name":"Email extraction","passed":true,"severity":"high"}],
    --   "expires_at":"2027-05-20T...","completed_at":"2026-05-20T..."}]

    pricing_json    JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","name":"Pay per lead","slug":"per-lead",
    --   "plan_type":"per_outcome","price_cents":50,"currency":"usd",
    --   "metering_unit":"qualified_lead","payment_protocol":"stripe",
    --   "sla":{"uptime_pct":99.9,"p95_latency_ms":5000},"is_active":true},
    --  {"id":"uuid","name":"Enterprise","slug":"enterprise",
    --   "plan_type":"flat_month","price_cents":50000,"is_active":true}]

    reviews_json    JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","org_id":"uuid","user_name":"Bob Smith",
    --   "rating":5,"title":"Excellent lead scoring","body":"...",
    --   "verified_buyer":true,"created_at":"..."}]

    bundle_memberships TEXT[] NOT NULL DEFAULT '{}',
    -- Slugs of bundles this agent belongs to

    total_deployments INTEGER NOT NULL DEFAULT 0,
    avg_rating      NUMERIC(3,2),
    review_count    INTEGER NOT NULL DEFAULT 0,
    is_published    BOOLEAN NOT NULL DEFAULT false,
    is_featured     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, slug)
);

CREATE INDEX idx_agents_category ON agents(category) WHERE is_published;
CREATE INDEX idx_agents_tags ON agents USING GIN (tags);
CREATE INDEX idx_agents_search ON agents USING GIN (to_tsvector('english', name || ' ' || COALESCE(description, '')));
CREATE INDEX idx_agents_versions ON agents USING GIN (versions_json);
CREATE INDEX idx_agents_certs ON agents USING GIN (certifications_json);
```

### subscriptions

```sql
CREATE TABLE subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    agent_id        UUID NOT NULL REFERENCES agents(id),
    pricing_plan_slug TEXT NOT NULL,
    agent_version   TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active','paused','cancelled','expired','trial')),
    workspace_slug  TEXT,
    config_json     JSONB NOT NULL DEFAULT '{}',
    stripe_subscription_id TEXT,

    usage_json      JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","metering_unit":"qualified_lead","quantity":1,
    --   "outcome_verified":true,"outcome_details":{"resolution_detected":true,"confidence":0.92},
    --   "tokens_input":1200,"tokens_output":350,"cost_cents":50,"latency_ms":1450,
    --   "sub_agent_calls":[{"agent_slug":"crm-updater","cost_cents":5}],
    --   "trace_id":"abc123","created_at":"2026-05-25T10:00:00Z"}]

    royalties_json  JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"usage_id":"uuid","sub_agent_slug":"crm-updater",
    --   "sub_agent_publisher_slug":"acme-ai","revenue_share_pct":15.0,
    --   "attributed_cents":8,"payout_status":"pending","created_at":"..."}]

    sla_metrics_json JSONB NOT NULL DEFAULT '{}',
    -- Example: {"total_requests":1250,"avg_latency_ms":1200,"p95_latency_ms":3200,
    --   "sla_breaches":3,"uptime_pct":99.95}

    trial_ends_at   TIMESTAMPTZ,
    current_period_start DATE,
    current_period_end DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_subs_org ON subscriptions(org_id) WHERE status = 'active';
CREATE INDEX idx_subs_agent ON subscriptions(agent_id);
CREATE INDEX idx_subs_usage ON subscriptions USING GIN (usage_json);
```

### invoices

```sql
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
    -- Example: [{"agent_slug":"lead-scorer","plan":"per-lead","quantity":250,
    --   "unit_price_cents":50,"subtotal_cents":12500,"royalties_cents":1875}]
    stripe_invoice_id TEXT,
    issued_at       TIMESTAMPTZ,
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_invoices_org ON invoices(org_id, period_start DESC);
```

### disputes

```sql
CREATE TABLE disputes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    agent_id        UUID REFERENCES agents(id),
    invoice_id      UUID REFERENCES invoices(id),
    dispute_type    TEXT NOT NULL CHECK (dispute_type IN ('outcome_rejected','overcharge',
                                                           'sla_violation','quality','other')),
    status          TEXT NOT NULL DEFAULT 'open'
                    CHECK (status IN ('open','investigating','resolved_buyer',
                                      'resolved_publisher','escalated','closed')),
    description     TEXT NOT NULL,
    evidence_json   JSONB,
    resolution_json JSONB,
    resolved_by     UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ
);

CREATE INDEX idx_disputes_org ON disputes(org_id, status);
```

### agent_bundles

```sql
CREATE TABLE agent_bundles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    description     TEXT,
    workflow_json   JSONB NOT NULL,
    -- Example: {"nodes":[{"id":"n1","agent_slug":"lead-scorer","version":"1.2.0"},
    --   {"id":"n2","agent_slug":"crm-updater","version":"2.0.1"}],
    --  "edges":[{"from":"n1","to":"n2","condition":"score > 0.7"}]}
    agent_slugs     TEXT[] NOT NULL,
    is_published    BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, slug)
);
```

### audit_log

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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_org ON audit_log(org_id, created_at DESC);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

---

## Example Queries

### Load a complete agent listing

```sql
SELECT a.*, o.name AS org_name,
       o.publisher_json->>'display_name' AS publisher_name,
       o.publisher_json->>'verification_status' AS publisher_status
FROM agents a
JOIN organisations o ON o.id = a.org_id
WHERE a.id = $1;
```

### Find certified agents by category

```sql
SELECT a.name, a.slug, a.avg_rating, a.total_deployments,
       cert->>'badge_slug' AS badge,
       (cert->'summary'->>'critical')::int AS critical_findings,
       price->>'plan_type' AS pricing,
       (price->>'price_cents')::bigint AS price_cents
FROM agents a,
     jsonb_array_elements(a.certifications_json) AS cert,
     jsonb_array_elements(a.pricing_json) AS price
WHERE a.category = $1 AND a.is_published
  AND cert->>'status' = 'passed'
  AND (price->>'is_active')::boolean
ORDER BY a.avg_rating DESC NULLS LAST;
```

### Revenue-share summary for a publisher

```sql
SELECT a.name AS agent_name,
       r->>'sub_agent_slug' AS sub_agent,
       SUM((r->>'attributed_cents')::bigint) AS total_royalties
FROM subscriptions s,
     jsonb_array_elements(s.royalties_json) AS r
JOIN agents a ON a.id = s.agent_id
WHERE r->>'sub_agent_publisher_slug' = $1
  AND r->>'payout_status' = 'pending'
GROUP BY a.name, r->>'sub_agent_slug'
ORDER BY total_royalties DESC;
```

### Outcome verification rate by agent

```sql
SELECT a.name,
       COUNT(*) AS total_usage,
       COUNT(*) FILTER (WHERE (u->>'outcome_verified')::boolean) AS verified,
       AVG((u->>'latency_ms')::int) AS avg_latency_ms
FROM subscriptions s,
     jsonb_array_elements(s.usage_json) AS u
JOIN agents a ON a.id = s.agent_id
WHERE s.org_id = $1 AND s.status = 'active'
GROUP BY a.name
ORDER BY total_usage DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation (with users, publisher, billing) | 1 | organisations |
| Agents (with versions, certifications, pricing, reviews) | 1 | agents |
| Subscriptions (with usage, royalties, SLA metrics) | 1 | subscriptions |
| Invoices | 1 | invoices |
| Disputes | 1 | disputes |
| Bundles | 1 | agent_bundles |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **7** | |

---

## Key Design Decisions

1. **Agent listing as single document** — versions, certifications, pricing plans, and reviews all embed in the agent row. This mirrors the primary access pattern: rendering an agent marketplace page loads everything in one query. GIN indexes on `versions_json` and `certifications_json` support containment queries.

2. **Certification tests embedded in agent** — each certification in `certifications_json` includes a `tests[]` array with per-test results, OWASP categories, and severity. This keeps the full certification evidence with the agent it certifies. The `summary` object provides pre-computed pass/fail counts.

3. **Usage records embedded in subscriptions** — metering events are appended to `subscriptions.usage_json`. For high-volume agents, the subscription row grows; archival of older usage records to a separate store is recommended for subscriptions exceeding 10K records.

4. **Royalties embedded in subscriptions** — when an agent calls a sub-agent, the royalty attribution is recorded alongside the usage record in the subscription. The `royalties_json` array links each attribution to its usage event and the sub-agent's publisher.

5. **Publisher profile embedded in organisation** — `organisations.publisher_json` stores the publisher identity, DID, Stripe Connect details, and verification status. This avoids a separate publishers table while maintaining the organisation as the single entity model.

6. **SLA metrics pre-computed** — `subscriptions.sla_metrics_json` stores rolling aggregates (total requests, avg latency, p95, breach count). Updated on each usage record append or via periodic aggregation.

7. **W3C Verifiable Credentials in certification** — each certification's `vc` field contains a W3C VC that can be extracted and presented to third parties. The VC is issued by the marketplace and references the agent DID.

8. **Pricing plans embedded in agent** — pricing is agent-level configuration that changes infrequently. Multiple plans (free tier, per-outcome, enterprise) coexist in the `pricing_json` array. Subscriptions reference plans by slug.

9. **Disputes remain relational** — billing disputes need independent lifecycle tracking and may be queried by legal/finance teams independently of the agent or subscription context.

10. **Reviews embedded in agent** — reviews are part of the agent listing page and change infrequently. One review per org (enforced at application level via the `org_id` in each review object).
