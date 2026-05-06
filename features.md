# Autonomous Agent Marketplace — Feature & Functionality Survey

> Candidate #344 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Salesforce AgentExchange | Commercial SaaS | Proprietary / Subscription | https://www.salesforce.com/agentforce/agentexchange/ |
| Google Cloud AI Agent Marketplace | Commercial SaaS | Proprietary / Usage-based | https://cloud.google.com/marketplace/docs/explore-ai-agents |
| Replit Agent Market | Commercial SaaS | Proprietary / Subscription + per-agent | https://replit.com/agent-market |
| Oracle Fusion AI Agent Marketplace | Commercial / Licensed | Proprietary / Bundled | https://www.oracle.com/applications/fusion-ai/ai-agent-marketplace/ |
| Agentverse (Fetch.ai) | Open source + Platform | Apache 2.0 (uAgents) + SaaS | https://agentverse.ai/ |
| CROO Agent Protocol | Protocol + Platform | Open protocol / Token-based | https://croo.network/ |
| Anthropic Agent Marketplace (pilot) | Commercial / Pilot | Proprietary / TBD | Not publicly available |

## Feature Analysis by Solution

### Salesforce AgentExchange

**Core features**
- Unified discovery across 10,000+ Salesforce apps, 2,600 Slack apps/agents, and 1,000+ Agentforce agents, sub-agents, tools, and MCP servers
- Semantic search powered by Data 360 with intent-based discovery
- Conversational search (launching fall 2026) for refined results and solution comparison
- Automated provisioning and one-click activation for deployment
- Customer Private Offers via AgentExchange Go-to-Market App (reduces contract negotiation timelines)
- Integrated billing and consolidated vendor management

**Differentiating features**
- Deep integration with Salesforce Agentforce and Slack ecosystem — agents can leverage 6,000 Slack apps through MCP
- Semantic search optimised for enterprise buyers without technical training
- Conversational discovery layer allows follow-up refinement
- Private Offers automation reduces sales cycles

**UX patterns**
- Marketplace-first discovery (semantic + conversational)
- One-click provisioning reduces friction
- Private Offers embedded in marketplace reduce procurement cycles
- Integrated permissions and access control

**Integration points**
- Salesforce Agentforce APIs and agent runtime
- Slack Marketplace and Slack MCP integration (6,000+ apps available through MCP)
- Salesforce IAM for access control and provisioning
- Consolidated billing across Salesforce products

**Known gaps**
- Locked to Salesforce ecosystem; limited integration with AWS, Azure, Google Cloud agents
- No public information on outcome-based pricing support or agent composition/bundling at marketplace level
- Limited transparency on agent certification and safety testing processes

**Licence / IP notes**
- Proprietary; all integrations leverage open standards (MCP, OpenAPI)
- No known patent conflicts; MCP is open standard backed by Anthropic

---

### Google Cloud AI Agent Marketplace

**Core features**
- Agent discovery with Gemini-powered natural language search across thousands of partner agents
- A2A (Agent2Agent) protocol support as first-class integration standard
- Automated agent registration via Agent Card (JSON format based on A2A spec)
- Metadata ingestion and capability discovery without manual listing effort
- IAM-based governance and access control for agent deployment
- Private Marketplace functionality for cost control and compliance
- Ecosystem partners: Atlassian, Box, Lovable, Oracle, ServiceNow, Workday, and 50+ others

**Differentiating features**
- Native A2A protocol support (agents can delegate to other agents seamlessly)
- Integration with Gemini Enterprise for agent-to-LLM coordination
- Automatic capability discovery from Agent Card eliminates manual feature entry
- Unified registry spanning multiple vendors (not siloed to one platform)

**UX patterns**
- Gemini-powered natural language discovery
- Agent Card (JSON) as lingua franca for agent metadata
- Direct integration to Gemini Enterprise workspace
- Administrator-controlled governance via IAM and Private Marketplace

**Integration points**
- Agent2Agent (A2A) protocol for agent-to-agent communication
- Gemini Enterprise LLM integration
- Google Cloud IAM for access and cost control
- OpenAPI for agent capability description
- OAuth 2.0 / PKCE for agent authorization

**Known gaps**
- Limited information on agent certification / safety scanning processes
- No public outcome-based pricing support
- Agent composition and bundling not explicitly mentioned
- Tied to Google Cloud infrastructure; limited offline or on-premises agent support

**Licence / IP notes**
- Proprietary; built on open standards (A2A protocol is open, contributed to Linux Foundation)
- A2A protocol reached 150 production installations as of May 2026

---

### Replit Agent Market

**Core features**
- App-store style marketplace for AI agents built and deployed on Replit
- Agent framework (Agent 3 and Agent 4) with autonomous code generation and repair
- Parallel sub-agent spawning for task decomposition
- Integrated development environment (IDE) for agent development
- Agents can be built by any developer on Replit (low barrier to entry)
- Testing and debugging tools with real-time browser execution
- Web search integration for current information
- Built-in services: Authentication, Database, Hosting, Monitoring
- Third-party integrations (Stripe, OpenAI, custom APIs)

**Differentiating features**
- Closest to a traditional app store for agents (vs. enterprise marketplace)
- Parallel agent execution (agents work simultaneously on independent sub-tasks)
- Autonomous code repair and testing (agent tests and fixes its own code)
- Very low barrier to entry for developers (Replit IDE + templates)
- Effort modes (Economy, Power, Turbo) for cost/quality trade-off

**UX patterns**
- Freemium + premium tiers (Replit Core $20/mo)
- Individual agent pricing varies
- Developer-self-service discovery and listing
- Collaborative agent building (agents can spawn sub-agents for specialised tasks)

**Integration points**
- Replit hosting and runtime infrastructure
- Built-in Database and Auth services
- Third-party API integrations (Stripe, OpenAI, etc.)
- Browser execution for web interaction (testing and live agent operations)
- Web search API for autonomous information retrieval

**Known gaps**
- Limited enterprise governance and compliance features
- No explicit outcome-based pricing or revenue-share infrastructure
- No mention of agent certification or safety scanning
- Primarily focused on hobbyist and early-stage developer adoption; less traction in enterprise

**Licence / IP notes**
- Proprietary platform; agents built on Replit are owned by developer
- No IP conflicts identified

---

### Oracle Fusion AI Agent Marketplace

**Core features**
- Marketplace inside Oracle Fusion Cloud Applications for partner-built agent templates
- 21-point validation and certification process for all templates
- Pre-built templates covering HR, Finance, Supply Chain, and other Fusion modules
- Certified partner ecosystem (Alithya, Apex IT, Argano, CLOUDSUFI, Infosys, Wipro, and Big Four consultancies)
- Deep integration with Oracle Fusion Cloud Applications
- 63,000 certified experts and SI partnerships

**Differentiating features**
- Rigorous 21-point certification process (vs. other marketplaces with lighter gatekeeping)
- Exclusively focused on Fusion Cloud use cases (narrow but deep specialisation)
- Backed by large SIs (Accenture, Deloitte, KPMG, PwC)
- Templates are pre-tested for Fusion data models and workflows

**UX patterns**
- Templates delivered as part of Fusion Cloud environment (no separate provisioning)
- Certification ensures quality and support from Oracle
- SI partnerships provide implementation and support services

**Integration points**
- Fusion Cloud APIs and data models
- Oracle SaaS ecosystem (HCM, Finance, SCM, CX modules)
- SI-provided professional services

**Known gaps**
- Extremely narrow addressable market (Fusion customers only)
- Limited visibility on agent composition, bundling, or outcome-based pricing
- No public information on MCP or A2A protocol support
- No independent developer ecosystem (templates come from certified SIs only)

**Licence / IP notes**
- Proprietary; templates provided under Oracle licensing terms
- No known patent conflicts

---

### Agentverse (Fetch.ai)

**Core features**
- Decentralised registry for autonomous agents with blockchain-based discovery
- Cloud-based IDE for agent development (write, edit, run agent code in browser)
- Almanac service for agent registration and discoverability
- Agent hosting infrastructure (no self-managed servers required)
- Mailroom Service allowing agents to receive messages offline and retrieve on reconnect
- Blockchain-based identity and wallet support for agents (can hold tokens, query balances, interact with smart contracts)
- Integration with ASI:One Web3-native LLM for agent discovery via natural language queries

**Differentiating features**
- Decentralised architecture (agents registered on blockchain, not central database)
- Native crypto/token support (agents have wallets, can transact autonomously)
- Integration with ASI:One LLM for natural language agent discovery and invocation
- Mailroom Service unique among surveyed platforms (allows offline agent messaging)
- Agent composition via blockchain-based identity (agents can be composed/nested)

**UX patterns**
- Blockchain-based identity and trust model
- Natural language agent discovery via ASI:One LLM
- Token-based interactions and settlement
- Developer-centric interface; low enterprise adoption

**Integration points**
- Blockchain networks (Fetch.ai network, Ethereum, or other compatible chains)
- ASI:One LLM for agent query interpretation
- Agent-to-agent micropayment protocol (token-based)
- uAgents Python framework (open source)

**Known gaps**
- Limited enterprise adoption and trust signals (no SOC 2 certification visible)
- No explicit outcome-based pricing infrastructure visible
- Limited information on agent certification or safety scanning
- Blockchain-first model may not appeal to risk-averse enterprise buyers

**Licence / IP notes**
- uAgents framework: Apache 2.0 (open source)
- Platform infrastructure: proprietary
- Blockchain registrations create immutable audit trail but no IP protection beyond on-chain records

---

### CROO Agent Protocol

**Core features**
- Open protocol (CROO Agent Protocol / CAP) for agent discovery, coordination, and micropayment
- CROO Agent Store (marketplace launching May 2026 Beta) for listing services
- Agent Asset Exchange (Q3 2026) enabling agents to be bought/sold as digital assets
- Full-stack commercial layer: standardized coordination (CAP), service discovery (Agent Store), assetization (Exchange)
- Developer-retained agent ownership
- Participation in CROO V1 Pioneers Program (early access)

**Differentiating features**
- On-chain agent-to-agent payments (novel primitive in 2026)
- Assetization of profitable agents (Agent Asset Exchange) — buy/sell agents like SaaS businesses
- Developer ownership retained (not siloed to CROO platform)
- Full-stack commercial infrastructure (coordinated rollout of discovery, payments, and asset trading)

**UX patterns**
- Protocol-first approach (agents not locked to single platform)
- Three-layered ecosystem: Protocol (CAP) → Discovery (Agent Store) → Assetization (Exchange)
- Token-based transactions and settlement
- Developer participation in pioneering cohort

**Integration points**
- CROO Agent Protocol (open, launched April 2026 on Base blockchain)
- Agent Store (SaaS marketplace, May 2026 Beta)
- Agent Asset Exchange (Q3 2026, planned)
- Base blockchain for settlement and identity

**Known gaps**
- Early stage (launched April 2026, Store Beta in May, Exchange in Q3)
- Regulatory uncertainty around on-chain agent commerce and asset trading
- No certification or safety scanning processes documented
- Limited adoption or production deployments visible as of May 2026

**Licence / IP notes**
- CROO Agent Protocol: Open protocol (governance model TBD; compare to A2A contributed to Linux Foundation)
- Regulatory concerns: Token-based settlement and asset trading may face regulatory scrutiny in some jurisdictions; independent legal review recommended before production use

---

### Anthropic Agent Marketplace (Pilot)

**Core features**
- Pilot marketplace where Claude-powered agents can be discovered and purchased
- Designed for Claude-native agents
- Backed by Anthropic (frontier model provider)

**Differentiating features**
- First-party support from Anthropic (frontier model provider)
- Likely MCP-first integration (Anthropic standard)
- Potential for outcome-based billing integration

**UX patterns**
- Expected to follow agent-marketplace norms once launched

**Integration points**
- Claude API and models
- Likely MCP integration
- Unknown: A2A protocol support

**Known gaps**
- **Not yet public**; pilot stage only
- No confirmed details on certification, safety scanning, revenue share, or pricing models
- No information on agent composition/bundling support
- Launch timeline and target audience not disclosed

**Licence / IP notes**
- Proprietary; details unavailable
- All Anthropic API usage governed by standard Anthropic terms

---

## Cross-Cutting Feature Themes

### Table-Stakes Features

These are capabilities present in nearly every solution and mandatory for marketplace viability:

- **Agent discovery and search** — all platforms provide some form of browsing/search; semantic and natural language search increasingly expected
- **One-click deployment or provisioning** — agents must be deployable with minimal setup
- **Interoperability standards** — MCP (Anthropic, 9,400+ servers by April 2026) and A2A (Google, 150+ production deployments) have become de facto requirements
- **Vendor/marketplace trust** — marketplace curators provide some level of validation (certification, review, or testing)
- **API or service specification** — OpenAPI or equivalent for describing agent capabilities
- **Billing and payment integration** — usage-based or outcome-based; no marketplace thrives without clear pricing and settlement
- **Access control and governance** — IAM, role-based access, or equivalent for enterprise deployments
- **Agent metadata and capability declaration** — standardised way to express what an agent does (Agent Card in A2A, MCP server description, OpenAPI spec)

### Differentiating Features

Capabilities that set some solutions apart and provide competitive advantage:

- **Outcome-based pricing infrastructure** — Zendesk, Intercom, Salesforce, HubSpot have implemented pay-per-result models; a marketplace with outcome-based metering and settlement differentiation
- **Agent composition and sub-agent orchestration** — Replit supports parallel agents; Google/Microsoft frameworks support hierarchical agent trees; no marketplace explicitly advertises this yet
- **Conversational / semantic discovery** — Salesforce (semantic), Google (Gemini-powered), Agentverse (ASI:One) lead; standard keyword search is table-stakes
- **Decentralised / blockchain-based identity and settlement** — Agentverse and CROO are exploring this; enterprises favour centralised but this appeals to developer-first platforms
- **Agent composition and bundling** — ability to group multiple agents into workflows or orchestrated bundles (e.g., pipeline agent → lead-scoring → CRM-update)
- **Certification and safety scanning** — ClawSecure (8-point security checklist), Cisco IDE scanner (Skill Scanner + MCP Scanner), Agensi (8-point OWASP ASI Top 10 verification); differentiates on trust
- **Revenue-share and royalty automation** — no marketplace surveyed explicitly exposes this; but outcome-based pricing requires robust attribution and payout tracking

### Underserved Areas / Opportunities

Gaps that represent genuine opportunities for a new entrant:

- **Vendor-neutral multi-cloud agent registry** — Salesforce is locked to Salesforce, Oracle to Fusion, Google to Google Cloud; a marketplace agnostic to the underlying cloud infrastructure (AWS, Azure, Google, on-premises) is a gap
- **Automated agent certification and safety scanning** — current approaches are manual (21-point review at Oracle) or lightweight (keyword scanning at ClawSecure); sandboxed execution, PII-leak scanning, prompt-injection testing as part of submission pipeline is underserved
- **Agent composition and bundling as first-class feature** — no surveyed marketplace exposes composition graphs or agent bundling; enterprises want pre-composed, tested pipelines (e.g., "lead-to-pipeline" bundle)
- **Outcome-based billing infrastructure (metering + settlement)** — outcome-based pricing is proven (Zendesk, Intercom, HubSpot) but requires robust metering, dispute resolution, and SLA tracking; only specialist platforms (Nevermined Pay) address this
- **Revenue-share and royalty tracking for agent authors** — as agents become IP, clear attribution and automated payouts for agents calling sub-agents from other authors is a missing piece
- **Independent certification and trust signals** — Agentforce agents undergo Salesforce review; Fusion agents undergo Oracle review; a neutral, third-party certification body (similar to AWS Partner or Salesforce ISV Certification) would build trust across platforms
- **Agent provenance and supply-chain integrity** — no surveyed platform addresses traceability of agent origins, model versions, or code genealogy; relevant for compliance and IP attribution
- **SLA and performance guarantees** — no marketplace surveyed exposes SLAs, uptime guarantees, or performance benchmarking; relevant for production critical use cases

### AI-Augmentation Candidates

Features that existing tools implement with manual/rule-based approaches but where AI could provide meaningfully better results:

- **Agent discovery and relevance ranking** — current platforms (Google, Salesforce) use semantic search or rule-based ranking; LLMs could improve relevance through deeper task understanding and user intent inference
- **Agent certification and safety scanning** — current approaches (ClawSecure, Agensi) use static rule sets; LLM-powered agents could perform dynamic adversarial testing, prompt-injection attempts, and contextual PII detection
- **Agent composition and recommendation** — no platform suggests agent bundles or compositions; an LLM could infer which agents work well together, generate composition graphs, and optimise orchestration
- **Outcome tracking and dispute resolution** — outcome-based pricing requires manual SLA tracking and dispute resolution; LLMs could automate outcome verification (e.g., "was this lead truly qualified?") and suggest fair resolutions
- **Agent performance and regression detection** — LLMs could monitor agent behavior, detect performance regressions, and alert authors to breaking changes or quality shifts
- **Royalty and revenue attribution** — as agents call sub-agents, LLMs could trace contribution flows and automate proportional payouts to sub-agent authors

---

## Legal & IP Summary

**Copyright and Licensing:**
All surveyed commercial platforms (Salesforce, Google, Oracle, Replit) operate under proprietary licensing; no reproductions of proprietary UI copy or documentation were included in this survey.

Open-source tools and standards carry permissive licences compatible with most enterprise use:
- **uAgents (Fetch.ai)** operates under Apache 2.0; freely usable and modifiable.
- **Agent2Agent (A2A)** protocol is open and contributed to the Linux Foundation; freely usable.
- **Model Context Protocol (MCP)** is open source under Anthropic's governance; 9,400+ public servers exist.

**Patents:**
No known software patents encumber the core capabilities surveyed. Agent orchestration, semantic discovery, and outcome-based pricing are practised across the industry without evidence of patent licensing requirements. However, blockchain-based agent settlement (CROO) may face regulatory rather than patent concerns.

**Regulatory & Compliance Concerns:**
- **Token-based settlement (CROO)**: Agent Asset Exchange and CROO Agent Protocol's token-based payments may trigger securities or money transmission regulation in jurisdictions like the US and EU; independent legal review strongly recommended before production use.
- **Cross-border data flows (Google, Fetch.ai)**: Cloud-hosted marketplaces may trigger GDPR, UK DPA, or other data residency regulations depending on agent data and compute locations.
- **A2A and MCP governance**: Both protocols are open, but governance models differ. A2A is Linux Foundation–governed; MCP governance is Anthropic-led. Evaluate lock-in risk if adopting a proprietary agent framework.

**IP Uncertainties:**
- **Anthropic Agent Marketplace**: Not yet public; terms, certification processes, and revenue-share model are unknown.
- **CROO Agent Asset Exchange**: Early-stage (Q3 2026 launch); regulatory treatment of "agent assets" as securities or intellectual property is unsettled.

No material was omitted due to copyright or licensing uncertainty.

---

## Recommended Feature Scope

Based on the research, here is a prioritised feature scope for a new autonomous agent marketplace project:

### Must-Have (MVP)

These are essential capabilities required to launch a viable marketplace:

1. **Multi-cloud, vendor-neutral agent registry** — support agents from AWS, Azure, Google Cloud, Salesforce, and on-premises; use MCP and A2A as first-class integration standards to avoid platform lock-in
2. **Semantic / intent-based discovery** — implement LLM-powered search (not keyword-only) to help non-technical buyers find agents; include conversational refinement for better relevance
3. **Automated agent certification** — implement a submission pipeline that scans for security issues (prompt injection, PII leaks, malicious commands) using both rule-based checks and LLM-based adversarial testing
4. **OpenAPI + MCP + A2A compliance** — ensure all marketplace agents expose capability specifications in these formats for interoperability
5. **IAM and governance controls** — provide role-based access, audit logs, and cost controls suitable for enterprise deployments
6. **Usage-based + outcome-based billing infrastructure** — support both consumption-based and result-based pricing models; provide metering, settlement, and dispute resolution

### Should-Have (v1.1)

These capabilities significantly enhance the marketplace but are not strictly required for launch:

1. **Agent composition and bundling** — allow developers to group agents into pre-composed workflows; provide a composition editor and testing environment
2. **Outcome verification and SLA tracking** — automated outcome detection (e.g., "was the support ticket truly resolved?") and SLA metrics
3. **Revenue-share and royalty tracking** — attribute usage/outcomes to sub-agents and automate payouts to agent authors
4. **Third-party agent certification** — partner with independent auditors or create a formal certification badge (e.g., "Marketplace Verified," "SOC 2 Certified") to build trust
5. **Agent performance benchmarking and monitoring** — dashboard showing agent success rates, latency, cost, and user feedback; detect regressions

### Nice-to-Have (Backlog)

These features provide unique differentiation or address advanced use cases:

1. **Agent provenance and code genealogy tracking** — version control and traceability for agent code, model versions, and dependencies
2. **Decentralised settlement (blockchain option)** — optional on-chain payment and identity (following CROO model) for developer communities that prefer it
3. **Agent composition recommendations** — LLM-powered suggestions for which agents work well together and optimal orchestration patterns
4. **Cross-marketplace agent federation** — enable agents listed on one marketplace to discover and invoke agents from others (via standardised A2A / MCP protocols)
5. **Outcome-based insurance and escrow** — offer optional escrow or insurance services for high-value agent transactions or outcome guarantees
