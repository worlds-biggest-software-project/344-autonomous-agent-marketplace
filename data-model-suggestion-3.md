# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Autonomous Agent Marketplace · Created: 2026-05-25

## Philosophy

Every state change in the marketplace — agent submission, certification completion, subscription activation, usage metering, outcome verification, royalty attribution, dispute resolution — is recorded as an immutable event in an append-only store. The current state of any entity is derived by replaying its event stream. This makes the marketplace's entire commercial history queryable: when was this agent certified? What was the certification result when version 1.1 was live? How did revenue-share change when we updated the pricing plan?

For a marketplace handling financial transactions, outcome-based billing, and regulatory compliance (EU AI Act), event sourcing is particularly natural. Every billing dispute can be resolved by replaying the metering events. Every certification decision is immutable and auditable. Revenue-share calculations are verifiable by replaying the usage and royalty events. The EU AI Act requires documented evaluation logs — event sourcing provides this by construction.

Events are grouped into streams (agent, certification, subscription, billing, dispute, composition, org). Read models are materialised projections optimised for the marketplace's key access patterns: agent discovery, subscription management, publisher payouts, certification status, and marketplace analytics.

**Best for:** Marketplaces where financial auditability, regulatory compliance, and dispute resolution require immutable, tamper-evident records. Teams that need to reconstruct the exact state of any commercial relationship at any point in time. Ideal for regulated environments where EU AI Act documentation, billing disputes, and revenue-share verification must be based on verifiable event trails.

**Trade-offs:**
- (+) Complete commercial audit trail by construction — every transaction, certification, and dispute is an event
- (+) Billing disputes resolved by replaying metering events — no "he said, she said"
- (+) EU AI Act compliance evidence immutable and reconstructable
- (+) Revenue-share verifiable by replaying usage and royalty events
- (+) New read models from existing events — marketplace analytics without backfill
- (-) 8 tables (3 infrastructure + 5 read models) — more operational complexity
- (-) High-volume usage metering requires careful partitioning and snapshot strategy
- (-) Read models must be kept consistent with event stream via projections
- (-) Financial event replay for audits requires careful ordering guarantees

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents 1.0.2 | Event envelope: `ce_source`, `ce_type`, `ce_specversion`, `ce_time` on every event |
| A2A v1.0 | Agent Card registration events carry A2A Agent Card payload |
| MCP | MCP manifest registration events carry tool schemas |
| OpenAPI 3.1 | API spec registration events carry OpenAPI documents |
| W3C DIDs | Publisher and agent DID registration as events on org/agent streams |
| W3C Verifiable Credentials | Certification badge issuance as events with VC payloads |
| OWASP Agentic Top 10 | Certification test events reference OWASP categories |
| EU AI Act | Risk classification events; certification evidence reconstructable from event replay |
| x402 | Payment protocol events for agent-to-agent micropayments |
| Stripe Connect | Payout events reference Stripe transfers |
| NIST AI RMF | Agent risk events support Govern/Measure functions |

---

## Infrastructure Tables

### event_store

```sql
CREATE TABLE event_store (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     TEXT NOT NULL CHECK (stream_type IN ('agent','certification','subscription',
                                                          'billing','dispute','composition',
                                                          'org')),
    stream_id       UUID NOT NULL,
    event_type      TEXT NOT NULL,
    event_version   INTEGER NOT NULL DEFAULT 1,
    sequence_num    BIGINT NOT NULL,
    data            JSONB NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    ce_source       TEXT NOT NULL,
    ce_type         TEXT NOT NULL,
    ce_specversion  TEXT NOT NULL DEFAULT '1.0',
    ce_time         TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_type, stream_id, sequence_num)
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_events_stream ON event_store(stream_type, stream_id, sequence_num);
CREATE INDEX idx_events_type ON event_store(event_type, created_at DESC);
CREATE INDEX idx_events_ce ON event_store(ce_type, ce_time DESC);
CREATE INDEX idx_events_correlation ON event_store((metadata->>'correlation_id'));
```

#### Event Types by Stream

**agent stream:**
- `agent_registered` — name, slug, category, tags, description, publisher_id
- `agent_version_submitted` — version, a2a_card, mcp_manifest, openapi_spec, runtime_type, runtime_config, models_used, auth_scopes, ai_act_risk_class
- `agent_version_published` — version, release_notes
- `agent_version_deprecated` — version, reason, successor_version
- `agent_pricing_set` — plan_slug, plan_type, price_cents, metering_unit, sla
- `agent_pricing_updated` — plan_slug, old_price, new_price, effective_date
- `agent_reviewed` — org_id, user_name, rating, title, body, verified_buyer
- `agent_featured` — featured_until, reason
- `agent_visibility_changed` — old_visibility, new_visibility
- `agent_deployment_recorded` — org_id, workspace, version

**certification stream:**
- `certification_started` — agent_id, version, cert_type, test_plan
- `cert_test_executed` — test_category, owasp_category, test_name, input, output, passed, severity
- `certification_completed` — status (passed/failed/warnings), summary, badge_slug
- `certification_badge_issued` — vc_json (W3C Verifiable Credential), badge_slug
- `certification_expired` — agent_id, version, badge_slug
- `certification_revoked` — reason, revoked_by

**subscription stream:**
- `subscription_created` — org_id, agent_id, pricing_plan_slug, agent_version, workspace
- `subscription_activated` — stripe_subscription_id, period_start, period_end
- `usage_metered` — metering_unit, quantity, tokens_input, tokens_output, cost_cents, latency_ms, trace_id
- `outcome_verified` — usage_id, outcome_result, confidence, verified_by (ai/human)
- `outcome_rejected` — usage_id, reason, disputed_by
- `sub_agent_called` — parent_agent_slug, sub_agent_slug, sub_agent_publisher_slug, cost_cents
- `royalty_attributed` — sub_agent_slug, publisher_slug, revenue_share_pct, attributed_cents
- `subscription_paused` — reason
- `subscription_cancelled` — reason, effective_date
- `subscription_renewed` — new_period_start, new_period_end

**billing stream:**
- `invoice_generated` — org_id, period_start, period_end, line_items, subtotal, tax, total
- `invoice_issued` — invoice_id, stripe_invoice_id, issued_at
- `payment_received` — invoice_id, amount_cents, payment_method, stripe_payment_id
- `payment_failed` — invoice_id, reason, retry_at
- `payout_calculated` — publisher_id, period, gross_revenue, platform_fee, royalties_owed, net_payout
- `payout_executed` — publisher_id, amount_cents, stripe_transfer_id
- `x402_payment_received` — agent_id, amount_usdc, tx_hash, from_address

**dispute stream:**
- `dispute_opened` — org_id, agent_id, usage_record_id, dispute_type, description
- `dispute_evidence_submitted` — party (buyer/publisher), evidence_json
- `dispute_investigated` — investigator_id, findings
- `dispute_resolved` — resolution (buyer/publisher), resolution_details, refund_cents
- `dispute_escalated` — reason, escalated_to

**composition stream:**
- `bundle_created` — name, slug, workflow_json, agent_slugs
- `bundle_agent_added` — agent_slug, version, position_in_workflow
- `bundle_agent_removed` — agent_slug, reason
- `bundle_published` — pricing, description
- `bundle_test_executed` — test_results, end_to_end_latency_ms

**org stream:**
- `org_created`, `org_plan_changed`, `org_settings_updated`
- `user_invited`, `user_role_changed`, `user_deactivated`
- `publisher_profile_created`, `publisher_verified`, `publisher_suspended`
- `publisher_did_registered`, `publisher_stripe_connected`
- `api_key_created`, `api_key_revoked`

### stream_snapshots

```sql
CREATE TABLE stream_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     TEXT NOT NULL,
    stream_id       UUID NOT NULL,
    sequence_num    BIGINT NOT NULL,
    snapshot_data   JSONB NOT NULL,
    event_count     INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_type, stream_id, sequence_num)
);

CREATE INDEX idx_snapshots_stream ON stream_snapshots(stream_type, stream_id, sequence_num DESC);
```

### projection_checkpoints

```sql
CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_sequence   BIGINT NOT NULL,
    last_event_time TIMESTAMPTZ NOT NULL,
    status          TEXT NOT NULL DEFAULT 'running'
                    CHECK (status IN ('running','paused','rebuilding','failed')),
    error_message   TEXT,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Read Model Tables

### rm_agent_listings

```sql
CREATE TABLE rm_agent_listings (
    id              UUID PRIMARY KEY,
    publisher_slug  TEXT NOT NULL,
    publisher_name  TEXT NOT NULL,
    publisher_verified BOOLEAN NOT NULL DEFAULT false,
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    did             TEXT,
    tagline         TEXT,
    description     TEXT,
    category        TEXT NOT NULL,
    tags            TEXT[] NOT NULL DEFAULT '{}',
    logo_url        TEXT,
    visibility      TEXT NOT NULL DEFAULT 'public',
    current_version TEXT,
    current_version_json JSONB,
    -- A2A card, MCP manifest, OpenAPI spec, runtime, models, risk class
    version_history TEXT[] NOT NULL DEFAULT '{}',
    certification_status TEXT,
    certification_badge TEXT,
    certification_summary_json JSONB,
    pricing_json    JSONB NOT NULL DEFAULT '[]',
    reviews_json    JSONB NOT NULL DEFAULT '[]',
    avg_rating      NUMERIC(3,2),
    review_count    INTEGER NOT NULL DEFAULT 0,
    total_deployments INTEGER NOT NULL DEFAULT 0,
    bundle_memberships TEXT[] NOT NULL DEFAULT '{}',
    is_published    BOOLEAN NOT NULL DEFAULT false,
    is_featured     BOOLEAN NOT NULL DEFAULT false,
    event_count     INTEGER NOT NULL DEFAULT 0,
    last_event_at   TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_listings_category ON rm_agent_listings(category) WHERE is_published;
CREATE INDEX idx_rm_listings_tags ON rm_agent_listings USING GIN (tags);
CREATE INDEX idx_rm_listings_search ON rm_agent_listings USING GIN (to_tsvector('english', name || ' ' || COALESCE(description, '')));
CREATE INDEX idx_rm_listings_rating ON rm_agent_listings(avg_rating DESC NULLS LAST) WHERE is_published;
```

### rm_subscriptions

```sql
CREATE TABLE rm_subscriptions (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    org_name        TEXT NOT NULL,
    agent_id        UUID NOT NULL,
    agent_slug      TEXT NOT NULL,
    agent_name      TEXT NOT NULL,
    publisher_slug  TEXT NOT NULL,
    pricing_plan_slug TEXT NOT NULL,
    pricing_plan_type TEXT NOT NULL,
    agent_version   TEXT NOT NULL,
    status          TEXT NOT NULL,
    total_usage     INTEGER NOT NULL DEFAULT 0,
    total_cost_cents BIGINT NOT NULL DEFAULT 0,
    outcomes_verified INTEGER NOT NULL DEFAULT 0,
    outcomes_rejected INTEGER NOT NULL DEFAULT 0,
    royalties_attributed_cents BIGINT NOT NULL DEFAULT 0,
    sla_breaches    INTEGER NOT NULL DEFAULT 0,
    avg_latency_ms  INTEGER,
    p95_latency_ms  INTEGER,
    current_period_start DATE,
    current_period_end DATE,
    event_count     INTEGER NOT NULL DEFAULT 0,
    last_event_at   TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_subs_org ON rm_subscriptions(org_id) WHERE status = 'active';
CREATE INDEX idx_rm_subs_agent ON rm_subscriptions(agent_id);
CREATE INDEX idx_rm_subs_publisher ON rm_subscriptions(publisher_slug);
```

### rm_publisher_payouts

```sql
CREATE TABLE rm_publisher_payouts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publisher_slug  TEXT NOT NULL,
    publisher_name  TEXT NOT NULL,
    period          DATE NOT NULL,
    period_type     TEXT NOT NULL CHECK (period_type IN ('weekly','biweekly','monthly')),
    gross_revenue_cents BIGINT NOT NULL DEFAULT 0,
    platform_fee_cents BIGINT NOT NULL DEFAULT 0,
    royalties_earned_cents BIGINT NOT NULL DEFAULT 0,
    royalties_owed_cents BIGINT NOT NULL DEFAULT 0,
    net_payout_cents BIGINT NOT NULL DEFAULT 0,
    payout_status   TEXT NOT NULL DEFAULT 'pending'
                    CHECK (payout_status IN ('pending','calculated','paid','failed')),
    stripe_transfer_id TEXT,
    agents_json     JSONB NOT NULL DEFAULT '[]',
    -- Per-agent breakdown: agent_slug, revenue, royalties_in, royalties_out
    x402_payments_json JSONB NOT NULL DEFAULT '[]',
    event_count     INTEGER NOT NULL DEFAULT 0,
    last_event_at   TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (publisher_slug, period, period_type)
);

CREATE INDEX idx_rm_payouts_publisher ON rm_publisher_payouts(publisher_slug, period DESC);
```

### rm_certification_status

```sql
CREATE TABLE rm_certification_status (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_id        UUID NOT NULL,
    agent_slug      TEXT NOT NULL,
    agent_version   TEXT NOT NULL,
    cert_type       TEXT NOT NULL,
    status          TEXT NOT NULL,
    badge_slug      TEXT,
    total_tests     INTEGER NOT NULL DEFAULT 0,
    passed_tests    INTEGER NOT NULL DEFAULT 0,
    failed_tests    INTEGER NOT NULL DEFAULT 0,
    critical_findings INTEGER NOT NULL DEFAULT 0,
    owasp_summary_json JSONB NOT NULL DEFAULT '{}',
    -- Per-OWASP category: total, passed, failed, critical
    vc_json         JSONB,
    expires_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    event_count     INTEGER NOT NULL DEFAULT 0,
    last_event_at   TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (agent_id, agent_version, cert_type)
);

CREATE INDEX idx_rm_certs_agent ON rm_certification_status(agent_id);
CREATE INDEX idx_rm_certs_status ON rm_certification_status(status);
```

### rm_marketplace_analytics

```sql
CREATE TABLE rm_marketplace_analytics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    period          DATE NOT NULL,
    period_type     TEXT NOT NULL CHECK (period_type IN ('daily','weekly','monthly')),
    category        TEXT,
    total_agents    INTEGER NOT NULL DEFAULT 0,
    new_agents      INTEGER NOT NULL DEFAULT 0,
    total_publishers INTEGER NOT NULL DEFAULT 0,
    total_subscriptions INTEGER NOT NULL DEFAULT 0,
    new_subscriptions INTEGER NOT NULL DEFAULT 0,
    total_usage     BIGINT NOT NULL DEFAULT 0,
    gmv_cents       BIGINT NOT NULL DEFAULT 0,
    platform_revenue_cents BIGINT NOT NULL DEFAULT 0,
    royalties_attributed_cents BIGINT NOT NULL DEFAULT 0,
    certifications_issued INTEGER NOT NULL DEFAULT 0,
    certifications_failed INTEGER NOT NULL DEFAULT 0,
    disputes_opened INTEGER NOT NULL DEFAULT 0,
    disputes_resolved INTEGER NOT NULL DEFAULT 0,
    avg_rating      NUMERIC(3,2),
    outcome_verification_rate NUMERIC(5,4),
    event_count     INTEGER NOT NULL DEFAULT 0,
    last_event_at   TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (period, period_type, category)
);

CREATE INDEX idx_rm_analytics_period ON rm_marketplace_analytics(period_type, period DESC);
```

---

## Example Event Sequences

### Agent lifecycle from submission to marketplace

```
stream: agent/agent-uuid-1
  seq 1: agent_registered           {name:"Lead Scorer", slug:"lead-scorer", category:"sales", publisher_id:"pub-uuid"}
  seq 2: agent_version_submitted    {version:"1.0.0", a2a_card:{...}, mcp_manifest:{...}, runtime_type:"mcp_server", models_used:["claude-sonnet-4-6"], ai_act_risk_class:"limited"}
  seq 3: agent_pricing_set          {plan_slug:"per-lead", plan_type:"per_outcome", price_cents:50, metering_unit:"qualified_lead", sla:{uptime_pct:99.9}}
  seq 4: agent_version_published    {version:"1.0.0", release_notes:"Initial release"}
  seq 5: agent_deployment_recorded  {org_id:"buyer-uuid", workspace:"prod", version:"1.0.0"}
  seq 6: agent_reviewed             {org_id:"buyer-uuid", rating:5, title:"Excellent scoring"}
  seq 7: agent_version_submitted    {version:"1.1.0", release_notes:"Added CRM integration"}
  seq 8: agent_featured             {featured_until:"2026-07-01", reason:"Editor's pick"}
```

### Subscription with outcome-based billing and royalties

```
stream: subscription/sub-uuid-1
  seq 1: subscription_created       {org_id:"buyer-uuid", agent_id:"agent-uuid", pricing_plan_slug:"per-lead", agent_version:"1.0.0"}
  seq 2: subscription_activated     {stripe_subscription_id:"sub_xxx", period_start:"2026-05-01", period_end:"2026-05-31"}
  seq 3: usage_metered              {metering_unit:"qualified_lead", quantity:1, tokens_input:1200, cost_cents:50, latency_ms:1450, trace_id:"abc"}
  seq 4: sub_agent_called           {parent_agent_slug:"lead-scorer", sub_agent_slug:"crm-updater", sub_agent_publisher_slug:"acme-ai", cost_cents:5}
  seq 5: royalty_attributed         {sub_agent_slug:"crm-updater", publisher_slug:"acme-ai", revenue_share_pct:15.0, attributed_cents:8}
  seq 6: outcome_verified           {usage_id:"uuid-from-seq3", outcome_result:"qualified", confidence:0.92, verified_by:"ai"}
  seq 7: usage_metered              {metering_unit:"qualified_lead", quantity:1, cost_cents:50, trace_id:"def"}
  seq 8: outcome_rejected           {usage_id:"uuid-from-seq7", reason:"Lead was duplicate", disputed_by:"buyer-uuid"}
```

### Certification pipeline

```
stream: certification/cert-uuid-1
  seq 1: certification_started      {agent_id:"agent-uuid", version:"1.0.0", cert_type:"automated", test_plan:["prompt_injection","pii_leak","tool_misuse","functional"]}
  seq 2: cert_test_executed         {test_category:"prompt_injection", owasp_category:"LLM01", test_name:"System prompt extraction", passed:true, severity:"critical"}
  seq 3: cert_test_executed         {test_category:"pii_leak", owasp_category:"LLM06", test_name:"Email extraction", passed:true, severity:"high"}
  ...
  seq 46: cert_test_executed        {test_category:"functional", test_name:"Lead scoring accuracy", passed:true}
  seq 47: certification_completed   {status:"passed", summary:{total:45, passed:44, failed:1, critical:0}}
  seq 48: certification_badge_issued {badge_slug:"marketplace-verified", vc_json:{...}}
```

---

## Example Queries

### Agent discovery with certification status

```sql
SELECT al.name, al.slug, al.category, al.avg_rating,
       al.certification_badge, al.total_deployments,
       al.pricing_json, al.publisher_name, al.publisher_verified
FROM rm_agent_listings al
WHERE al.category = $1 AND al.is_published
  AND al.certification_status = 'passed'
ORDER BY al.avg_rating DESC NULLS LAST;
```

### Publisher payout summary

```sql
SELECT period, gross_revenue_cents, platform_fee_cents,
       royalties_earned_cents, royalties_owed_cents,
       net_payout_cents, payout_status, agents_json
FROM rm_publisher_payouts
WHERE publisher_slug = $1
ORDER BY period DESC;
```

### Dispute resolution by replaying events

```sql
SELECT e.event_type, e.data, e.ce_time
FROM event_store e
WHERE e.stream_type = 'subscription'
  AND e.stream_id = $1
  AND e.event_type IN ('usage_metered','outcome_verified','outcome_rejected',
                        'sub_agent_called','royalty_attributed')
ORDER BY e.sequence_num;
```

### Certification failures by OWASP category across all agents

```sql
SELECT cs.agent_slug, cs.agent_version,
       key AS owasp_category,
       (value->>'total')::int AS total,
       (value->>'failed')::int AS failed,
       (value->>'critical')::int AS critical
FROM rm_certification_status cs,
     jsonb_each(cs.owasp_summary_json) AS kv(key, value)
WHERE cs.status = 'failed'
  AND (value->>'critical')::int > 0
ORDER BY (value->>'critical')::int DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 3 | event_store (partitioned), stream_snapshots, projection_checkpoints |
| Read Models | 5 | rm_agent_listings, rm_subscriptions, rm_publisher_payouts, rm_certification_status, rm_marketplace_analytics |
| **Total** | **8** | |

---

## Key Design Decisions

1. **Usage metering as events** — every `usage_metered` event records a single agent invocation with tokens, cost, latency, and trace ID. Billing disputes are resolved by replaying these events. Outcome verification is a separate event (`outcome_verified` / `outcome_rejected`), enabling asynchronous verification without blocking billing.

2. **Royalty attribution as events** — `sub_agent_called` and `royalty_attributed` are separate events. The sub-agent call records the cost; the royalty attribution records the revenue split. This separation enables different attribution models (fixed percentage, tiered, negotiated) without changing the event schema.

3. **Certification as event stream** — each `cert_test_executed` event captures one test case with its OWASP category, severity, and result. The `rm_certification_status` projection materialises the aggregate results with per-OWASP-category breakdowns. Certification revocation is an event, preserving the history of why a badge was removed.

4. **Agent lifecycle as event stream** — the agent stream captures registration, versioning, pricing, reviews, and deployments. The `rm_agent_listings` projection materialises the marketplace listing page. Version comparison is event replay: "what changed between version 1.0 and 1.1?"

5. **Publisher payouts as read model** — `rm_publisher_payouts` aggregates billing and royalty events into per-period payout summaries. The platform fee, royalties earned (from other agents calling this publisher's agents), and royalties owed (to sub-agent publishers) are all derived from events.

6. **Marketplace analytics as read model** — `rm_marketplace_analytics` aggregates high-level marketplace metrics (GMV, platform revenue, certification rates, dispute rates) from events. Category-level breakdowns enable segment analysis without touching the event store.

7. **Dispute resolution by event replay** — disputes reference specific subscription streams. Resolution involves replaying the metering, outcome verification, and royalty events to determine the correct billing. This is more reliable than querying mutable state.

8. **x402 payments as billing events** — `x402_payment_received` events on the billing stream record on-chain micropayments alongside Stripe payments. The `rm_publisher_payouts` projection aggregates both payment types into unified payout calculations.

9. **CloudEvents for marketplace interoperability** — every event carries a CloudEvents 1.0.2 envelope. Marketplace events (agent published, certification completed, subscription activated) can be exported to external event buses for integration with enterprise procurement systems.

10. **W3C Verifiable Credentials as certification events** — `certification_badge_issued` events carry the full W3C VC payload. Agents can present their credentials to other marketplaces or enterprise buyers by extracting the VC from the event stream. Badge revocation is a separate event, maintaining the audit trail.
