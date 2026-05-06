# Standards & API Reference

> Project: Autonomous Agent Marketplace · Generated: 2026-05-04

## Industry Standards & Specifications

### ISO Standards

No ISO standard specifically governs AI agent marketplaces at this time, but the following ISO standards are directly relevant to system design and compliance:

- **ISO/IEC 42001:2023** — AI Management System standard. Provides a framework for establishing, implementing, and improving AI management within organisations. Directly relevant for marketplace operators that must demonstrate responsible AI governance to enterprise buyers. URL: https://www.iso.org/standard/81230.html
- **ISO/IEC 27001:2022** — Information Security Management Systems (ISMS). Required by enterprise procurement teams before onboarding any SaaS platform. A marketplace storing agent IP, buyer credentials, and usage data must comply. URL: https://www.iso.org/isoiec-27001-information-security.html
- **ISO/IEC 27017:2015** — Security controls for cloud services. Extends 27001 with cloud-specific controls applicable to a marketplace running on shared infrastructure. URL: https://www.iso.org/standard/43757.html
- **ISO/IEC 27018:2019** — Protection of PII in public clouds. Relevant because agent inputs and outputs will frequently include personally identifiable information processed via cloud services. URL: https://www.iso.org/standard/76559.html
- **ISO/IEC 25010:2023** — Systems and software quality models. Provides vocabulary and quality characteristics (reliability, performance efficiency, security, maintainability) applicable to marketplace-hosted agent certification and benchmarking. URL: https://www.iso.org/standard/35733.html

---

### W3C & IETF Standards

- **W3C Decentralized Identifiers (DIDs) v1.1** — Specification for cryptographically verifiable, decentralised identifiers for any subject (including software agents). Invited for implementation by W3C in March 2026. Enables agents to carry self-sovereign identities that can be verified without a centralised registry — directly relevant to agent provenance and agent-to-agent trust establishment. URL: https://www.w3.org/TR/did-1.1/
- **W3C Verifiable Credentials Data Model 2.0** — Standard for expressing cryptographically verifiable claims. Used alongside DIDs to allow agents to present certified capabilities, SOC 2 attestations, or safety certifications without manual review. URL: https://www.w3.org/TR/vc-data-model-2.0/
- **RFC 6749 — OAuth 2.0 Authorization Framework** — The baseline standard for delegated authorisation; agents must act on behalf of users and organisations within constrained scopes. URL: https://datatracker.ietf.org/doc/html/rfc6749
- **RFC 7636 — PKCE (Proof Key for Code Exchange)** — Extension to OAuth 2.0 that prevents authorisation code interception. Made mandatory for all public clients in OAuth 2.1 (2025 consolidation). Especially important for headless agents and containerised runtimes that cannot securely store client secrets. URL: https://datatracker.ietf.org/doc/html/rfc7636
- **IETF WIMSE (Workload Identity in Multi-System Environments)** — Active IETF working group defining standards for service/workload identity across cloud platforms. Includes Workload Identity Token (WIT), Workload Identity Certificate (WIC), mutual TLS for workloads, and service-to-service token exchange — the plumbing that lets agents authenticate to each other at the infrastructure layer. Drafts active as of April 2026. URL: https://datatracker.ietf.org/wg/wimse/about/
- **draft-klrc-aiagent-auth-00 (AIMS)** — March 2026 IETF draft composing WIMSE, SPIFFE, and OAuth 2.0 into an Agent Identity Management System (AIMS) framework. The emerging holistic standard for how AI agents authenticate and authorise across multi-system environments. URL: https://datatracker.ietf.org/
- **RFC 7807 — Problem Details for HTTP APIs** — Standard for machine-readable error responses. Relevant for a marketplace API that must return structured errors to agent runtimes that programmatically handle failures. URL: https://datatracker.ietf.org/doc/html/rfc7807
- **CloudEvents v1.0 (CNCF)** — Specification for describing event data in common formats to enable interoperability across event-driven systems. Graduated CNCF project (January 2024). Applicable for marketplace event streams (agent lifecycle events, billing events, certification status changes). URL: https://cloudevents.io/

---

### Data Model & API Specifications

- **OpenAPI Specification 3.1** — De facto standard for describing agent capabilities as callable HTTP services. OpenAPI 3.1 achieves full JSON Schema Draft-07 compatibility. Marketplace listings in 2026 increasingly require an OpenAPI spec to describe what an agent does and what parameters it accepts. The spec can be automatically converted into an MCP server. URL: https://www.openapis.org/
- **Model Context Protocol (MCP) — Specification 2025-11-25** — Open standard originally authored by Anthropic, now governed by the Agentic AI Foundation (a directed fund under the Linux Foundation, co-founded by Anthropic, Block, and OpenAI). Defines a standard for tool and context hand-offs between LLM hosts and servers (agents, data sources). Over 9,400 public MCP servers as of April 2026. MCP communication uses JSON-RPC 2.0 over STDIO, HTTP+SSE, or WebSocket transports. The 2026 roadmap addresses enterprise readiness: SSO-integrated auth, audit trails, gateway behaviour, and configuration portability. URL: https://modelcontextprotocol.io/specification/2025-11-25
- **Agent2Agent (A2A) Protocol v1.0** — Open protocol donated by Google to the Linux Foundation (June 2025). Enables communication and interoperability between opaque agentic applications. Key features: signed Agent Cards (cryptographic domain-owner verification), multi-tenant agent endpoints (SaaS providers serve different agents per tenant), JSON-RPC and gRPC dual transport with version negotiation. A2A v1.0 is production-grade with 150+ adopting organisations including Microsoft, AWS, Salesforce, SAP, and ServiceNow. URL: https://a2a-protocol.org/latest/specification/
- **JSON-RPC 2.0** — The underlying RPC protocol used by both MCP and A2A. Stateless, transport-agnostic, lightweight. Defines request, response, notification, and batch call semantics. Both MCP and A2A agents exchange messages as strict JSON-RPC 2.0 objects. URL: https://www.jsonrpc.org/specification
- **JSON Schema Draft-07 / 2020-12** — Standard for describing and validating JSON data structures. Used for Agent Card validation (A2A), MCP tool parameter schemas, and OpenAPI 3.1 request/response validation. URL: https://json-schema.org/

---

### Security & Authentication Standards

- **OWASP Top 10 for Agentic Applications 2026** — Globally peer-reviewed framework identifying the most critical security risks facing autonomous and agentic AI systems. Developed by 100+ industry experts. Covers: goal misalignment, tool misuse, delegated trust failures, inter-agent communication vulnerabilities, persistent memory risks, and emergent autonomous behaviour. Directly applicable to marketplace certification pipelines. URL: https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/
- **OWASP Top 10 for LLM Applications 2025** — Companion framework covering the LLM-specific threat landscape: prompt injection (#1), sensitive information disclosure (#2), RAG poisoning, and system prompt exposure. Relevant for any marketplace agent that processes external user input. URL: https://genai.owasp.org/resource/owasp-top-10-for-llm-applications-2025/
- **NIST AI Risk Management Framework (AI RMF 1.0)** — Voluntary framework (NIST AI 100-1) for trustworthiness considerations across the AI lifecycle. The 2026 GOVERN function procedural manual provides expanded implementation guidance. NIST is preparing an AI Agent Interoperability Profile (planned Q4 2026) and has launched the AI Agent Standards Initiative through CAISI. URL: https://www.nist.gov/itl/ai-risk-management-framework
- **OAuth 2.1 (Draft)** — 2025 consolidation of OAuth 2.0 that mandates PKCE for all public clients, deprecates implicit grant and resource owner password credentials, and simplifies the security model. The MCP specification requires OAuth 2.1 for remote server authentication. URL: https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/
- **OpenID Connect Core 1.0** — Identity layer on top of OAuth 2.0 providing ID tokens with user identity claims. Used for marketplace user authentication and agent-level identity assertions in enterprise SSO deployments. URL: https://openid.net/specs/openid-connect-core-1_0.html
- **Self-Issued OpenID Provider v2 (SIOP v2)** — Extension to OIDC enabling agents or users to act as their own identity provider, issuing verifiable identity tokens without relying on a centralised IdP. Relevant for marketplace scenarios where agents must assert identity autonomously. URL: https://openid.net/specs/openid-connect-self-issued-v2-1_0.html

---

### Regulatory Frameworks

- **EU AI Act (2026 enforcement)** — The remaining provisions of the EU AI Act become applicable on 2 August 2026. AI agents classified as high-risk systems require: risk management systems, data governance documentation, technical documentation, logging, human oversight, robustness, accuracy, and cybersecurity controls. Any marketplace placing agents on the EU market must enforce conformity assessments for high-risk listings. URL: https://digital-strategy.ec.europa.eu/en/policies/regulatory-framework-ai
- **GDPR (EU 2016/679)** — Data protection regulation with direct implications for agent data processing. Cloud-hosted marketplaces processing EU-resident data must address data residency, controller/processor agreements, and agent data retention policies. URL: https://gdpr.eu/
- **SOC 2 Type II** — AICPA Trust Services Criteria certification. The de facto enterprise trust signal for SaaS platforms. The 2026 AICPA updates introduce AI-specific controls for AI-driven systems. Over 60% of enterprises are more likely to partner with SOC 2-compliant vendors; most enterprise B2B procurement requires a recent Type II report. URL: https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-greater-than-soc-2

---

### Payment & Commerce Standards

- **x402 Protocol** — Open, internet-native payment standard built on top of the HTTP 402 status code. Developed by the Coinbase Developer Platform (launched May 2025). Enables AI agents to autonomously pay for API access using USDC micropayments on-chain, without human intervention or account setup. Settlement in under 2 seconds at ~$0.0001 per transaction. Supported by Cloudflare, Google, Vercel, and Nevermined. Directly applicable to outcome-based and per-request billing in an agent marketplace. URL: https://www.x402.org/
- **Stripe Connect API** — Mature platform-and-marketplace payments API used by Shopify, DoorDash, and other marketplaces. Supports multi-party payment splitting, revenue sharing, and connected account onboarding. Stripe also provides an Agent Toolkit enabling OpenAI Agents SDK, LangChain, CrewAI, and Vercel AI SDK to call Stripe APIs natively. A hosted MCP server is available at https://mcp.stripe.com. URL: https://docs.stripe.com/connect

---

### MCP Server Specifications

- **Model Context Protocol — Official Specification and GitHub** — The canonical MCP specification repository. URL: https://github.com/modelcontextprotocol/modelcontextprotocol
- **MCP 2026 Roadmap** — Covers transport scalability, agent communication, governance maturation, enterprise readiness, SEP prioritisation, and the SEP-1865 formalisation of MCP Apps (formerly mcp-ui). URL: https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/
- **Agentic AI Foundation (AAIF)** — Linux Foundation directed fund, co-founded by Anthropic, Block, and OpenAI (December 2025). Governs MCP under neutral industry stewardship. URL: https://www.linuxfoundation.org/

---

## Similar Products — Developer Documentation & APIs

### Salesforce AgentExchange

- **Description:** Unified discovery marketplace for 10,000+ Salesforce apps, 2,600+ Slack apps, and 1,000+ Agentforce agents/tools/MCP servers. Announced at TrailblazerDX 2026 (April 15, 2026). Supports semantic search, one-click provisioning, and private offers.
- **API Documentation:** https://developer.salesforce.com/docs/agentforce (Agentforce developer documentation; full AgentExchange publisher docs accessible via Salesforce Developer Center)
- **SDKs/Libraries:** Salesforce SDK (Apex, JavaScript, Python) via https://developer.salesforce.com/
- **Developer Guide:** https://appexchange.salesforce.com/learn/agentexchange-explained
- **Standards:** MCP, A2A, OpenAPI 3.x, OAuth 2.0
- **Authentication:** Salesforce IAM, OAuth 2.0 / PKCE, Salesforce Identity

---

### Google Cloud AI Agent Marketplace + A2A

- **Description:** Enterprise marketplace for partner-built AI agents validated for A2A protocol and Gemini Enterprise integration. Agents are registered via JSON Agent Cards. Supports Gemini-powered natural language discovery and IAM-based governance.
- **API Documentation:** https://docs.cloud.google.com/marketplace/docs/partners/ai-agents (publisher guide); https://docs.cloud.google.com/marketplace/docs/explore-ai-agents (buyer guide)
- **SDKs/Libraries:** Google Agent Development Kit (ADK) — Python, Go, Java, TypeScript: https://google.github.io/adk-docs/; A2A Python SDK: https://a2a-protocol.org/latest/
- **Developer Guide:** https://codelabs.developers.google.com/intro-a2a-purchasing-concierge
- **Standards:** A2A v1.0 (Linux Foundation), OpenAPI, JSON Schema, OAuth 2.0
- **Authentication:** Google Cloud IAM, OAuth 2.0 / PKCE, Service Account credentials

---

### Agentverse (Fetch.ai)

- **Description:** Decentralised registry for autonomous agents with blockchain-based discovery, a cloud IDE, hosted infrastructure, and an Almanac service for agent registration. Agents can hold wallets, transact autonomously, and communicate offline via a Mailroom service.
- **API Documentation:** https://docs.agentverse.ai/documentation/getting-started/overview (Agentverse docs); https://uagents.fetch.ai/docs (uAgents SDK)
- **SDKs/Libraries:** uAgents Python framework (Apache 2.0): https://github.com/fetchai/uAgents
- **Developer Guide:** https://uagents.fetch.ai/docs/getting-started/create
- **Standards:** JSON-RPC (agent communication), REST/JSON (Agentverse API); blockchain-native identity (Fetch.ai network)
- **Authentication:** Blockchain wallet / cryptographic key pairs; Almanac on-chain registration

---

### OpenAI Agents SDK

- **Description:** Production-grade agent framework (released March 2025, replacing Swarm). Core abstraction is handoffs — agents transfer control to each other explicitly with conversation context. Supports tool use, multi-agent orchestration, and MCP server integration.
- **API Documentation:** https://developers.openai.com/api/docs/guides/agents
- **SDKs/Libraries:** Python SDK: https://github.com/openai/openai-agents-python; TypeScript SDK available
- **Developer Guide:** https://developers.openai.com/apps-sdk/concepts/mcp-server (MCP integration guide)
- **Standards:** MCP (tool integration), OpenAPI (tool schemas), JSON-RPC 2.0
- **Authentication:** OpenAI API key; OAuth 2.0 for third-party tool access

---

### LangChain / LangGraph

- **Description:** LangChain is the leading open-source framework for LLM application development (Python and JavaScript). LangGraph is its graph-based agent orchestration layer, modelling agents as explicit state machines over a directed graph. 27,100 monthly searches (highest of any framework, per Langfuse 2026 data).
- **API Documentation:** https://python.langchain.com/docs/ (LangChain); https://langchain-ai.github.io/langgraph/ (LangGraph)
- **SDKs/Libraries:** Python: `langchain`, `langgraph`; JavaScript: `langchain`, `@langchain/langgraph`
- **Developer Guide:** https://langchain-ai.github.io/langgraph/tutorials/
- **Standards:** MCP, OpenAPI, JSON Schema (tool definitions)
- **Authentication:** API key delegation to underlying model providers; OAuth 2.0 for connected services

---

### CrewAI

- **Description:** Open-source multi-agent framework that composes agents as role-driven crews with declarative tasks. Provides a marketplace of pre-built crews for content pipelines, research, and customer service workflows. Leads on MCP integration depth in 2026.
- **API Documentation:** https://docs.crewai.com/
- **SDKs/Libraries:** Python: `crewai`; JavaScript/TypeScript support in development
- **Developer Guide:** https://docs.crewai.com/quickstart
- **Standards:** MCP (inline server declaration with automatic tool discovery), OpenAPI
- **Authentication:** API key per tool/service; OAuth for external integrations

---

### Nevermined Payments

- **Description:** Billing and payments infrastructure layer for AI agents. Supports tokenisation of AI agent APIs, plans with fiat/crypto/ERC20/free pricing, real-time metering, and x402 protocol integration. Settlement via Stripe today; PayPal Braintree and Cybersource forthcoming.
- **API Documentation:** https://nevermined.ai/docs/api-reference/ (TypeScript and Python SDKs)
- **SDKs/Libraries:** TypeScript and Python SDKs; MCP server integration available
- **Developer Guide:** https://docs.nevermined.app/docs/getting-started/
- **Standards:** x402 (HTTP 402 payment protocol), REST/JSON, ERC20 token standards
- **Authentication:** Stripe OAuth for fiat settlement; blockchain wallet keys for crypto settlement

---

### Stripe Connect + Agent Toolkit

- **Description:** Platform-and-marketplace payments API used by major marketplaces (Shopify, DoorDash). Supports multi-party payment splitting, revenue sharing, connected account onboarding, and metered billing. Stripe Agent Toolkit enables popular agent frameworks to invoke Stripe APIs natively via function calling; a hosted MCP server is available at https://mcp.stripe.com.
- **API Documentation:** https://docs.stripe.com/connect; https://docs.stripe.com/api
- **SDKs/Libraries:** Python: `stripe`; Node.js: `stripe`; Agent Toolkit (Python + TypeScript): https://github.com/stripe/ai
- **Developer Guide:** https://docs.stripe.com/connect/marketplace
- **Standards:** REST/JSON, OpenAPI 3.0, OAuth 2.0 (Connect OAuth flow), MCP (via hosted MCP server)
- **Authentication:** Stripe API keys; OAuth 2.0 for connected accounts

---

## Notes

**Emerging and evolving standards:**
- **AIMS (Agent Identity Management System)** — draft-klrc-aiagent-auth-00 (March 2026) composes WIMSE, SPIFFE, and OAuth 2.0 into a holistic agent identity framework. Not yet an RFC; monitor IETF datatracker for progress.
- **NIST AI Agent Interoperability Profile** — Planned for Q4 2026. Will provide governance and risk management guidance specifically for agentic systems.
- **EU AI Act enforcement (August 2026)** — The high-risk AI system requirements become enforceable on 2 August 2026; marketplace operators must decide which listed agents qualify as high-risk and enforce corresponding documentation and certification requirements before that deadline.
- **x402 and outcome-based billing** — x402 is a 2025/2026 protocol; while Coinbase, Cloudflare, and Google support it, regulatory treatment of stablecoin payments for API services differs by jurisdiction. Independent legal review is recommended before enabling x402-based billing for EU customers.
- **A2A v1.0 Signed Agent Cards** — The signed Agent Card mechanism (cryptographic domain-owner verification) is the trust primitive underpinning decentralised agent discovery. Marketplace architecture should plan to validate Agent Card signatures as a baseline trust check during agent registration.
