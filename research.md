# Autonomous Agent Marketplace

> Candidate #344 · Researched: 2026-05-02

## Existing Products and Software Packages

| Tool | Description | Type | Pricing | Strengths / Weaknesses |
|------|-------------|------|---------|------------------------|
| Salesforce AgentExchange | Marketplace for deploying and discovering Agentforce-compatible agents and templates within Salesforce | Commercial | Agentforce from $2/conversation or $125/user/mo; marketplace listings vary | Strength: direct integration with largest CRM; Weakness: locked to Salesforce ecosystem |
| Google Cloud AI Agent Marketplace | Google Cloud platform connecting enterprises to partner-built AI agents validated for Gemini Enterprise | Commercial | Usage-based via Google Cloud billing | Strength: hyperscaler distribution, enterprise trust; Weakness: Google Cloud-only, limited independent developers |
| Replit Agent Market | App-store-style marketplace for AI agents built and deployed on Replit | SaaS | Replit Core from $20/mo; individual agent pricing varies | Strength: closest analogue to a traditional app store for agents; Weakness: skews developer/hobbyist rather than enterprise |
| Oracle AI Agent Marketplace | Marketplace inside Oracle Fusion where certified partners publish validated agent templates | Commercial | Bundled within Oracle Fusion licensing | Strength: deep enterprise integration; Weakness: extremely narrow addressable market |
| Agentverse (Fetch.ai) | Decentralised registry for autonomous agents using blockchain-based discovery and micropayment protocols | Open source + platform | Free beta; post-beta pricing TBD | Strength: decentralised, interoperable; Weakness: developer-centric, low enterprise adoption |
| CROO Agent Protocol | Autonomous agent commerce protocol enabling on-chain agent-to-agent transactions | Protocol / platform | Token-based; launched April 2026 | Strength: novel agent-to-agent payment primitive; Weakness: early stage, regulatory uncertainty |
| Anthropic Agent Marketplace (pilot) | Reported internal test of a marketplace where Claude-powered agents can be purchased and run | Pilot | Not yet public | Strength: backed by frontier model provider; Weakness: pre-launch, no confirmed details |

## Relevant Industry Standards or Protocols

- **Model Context Protocol (MCP)** — Anthropic-originated open standard for tool and context hand-offs; becoming a de-facto interoperability layer that marketplace agents must support to be broadly usable
- **Agent2Agent (A2A)** — Google-backed protocol for cross-organisational agent delegation and discovery; relevant for marketplaces enabling business-to-business agent transactions
- **OpenAPI 3.x** — standard for describing agent capabilities as callable HTTP services; marketplace listings increasingly require an OpenAPI spec
- **OAuth 2.0 / PKCE** — standard for authorising agents to act on behalf of users across third-party services

## Available Research Materials

1. AI Business Review (2026). *Anthropic Tests AI Agent Commerce Marketplace*. https://www.aibusinessreview.org/2026/04/26/anthropic-ai-agent-commerce-marketplace/
2. Chainwire (2026). *CROO Launches CROO Agent Protocol: Powering the Autonomous Agent Commerce*. https://chainwire.org/2026/04/15/croo-launches-croo-agent-protocol-powering-the-autonomous-agent-commerce/
3. Google Cloud (2026). *Google Cloud AI Agent Marketplace*. Google Cloud Blog. https://cloud.google.com/blog/topics/partners/google-cloud-ai-agent-marketplace
4. Rapidclaw (2026). *[2026 Guide] AI Agent Marketplace — List, Buy & Monetize*. https://rapidclaw.dev/blog/ai-agent-marketplace-guide-2026
5. Poniak Times (2026). *Why AI Agent Marketplaces Are Emerging as the Next Software Distribution Layer*. https://www.poniaktimes.com/ai-agent-marketplaces-software-distribution/
6. Nevermined (2026). *How to Monetize AI Agents in 2026*. https://nevermined.ai/blog/monetize-ai-agents
7. Digital Applied (2026). *AI Agent Marketplaces 2026: Discovery and Distribution*. https://www.digitalapplied.com/blog/ai-agent-marketplaces-2026-discovery-distribution
8. Landbase (2026). *Best AI Agent Monetization Platforms*. https://www.landbase.com/blog/best-ai-agent-monetization-platforms

## Market Research

**Market Size:** Gartner projected that up to 40% of enterprise applications would include task-specific AI agents by 2026. Deloitte estimated that 25% of enterprises using generative AI would deploy AI agents in 2025, rising to 50% by 2027. No firm market-size figure exists for agent marketplaces specifically; analyst estimates embed it within the broader agentic AI platform market, projected to exceed USD 9 billion by 2027.

**Funding:** Fetch.ai raised over $40 M to build decentralised agent infrastructure. The broader agent marketplace space is nascent; most activity is from hyperscalers (Google, Oracle, Salesforce) investing internal capital rather than funding independent startups. Anthropic's pilot signals intent from a frontier model provider to own the distribution layer.

**Pricing Landscape:** Monetisation models are still being established. Salesforce has experimented with per-conversation ($2), per-action (Flex Credits at $0.10/action), and per-user/month ($125) pricing within 18 months of launch. Replit-style marketplaces lean toward subscription or one-time purchase. Outcome-based pricing (pay-per-resolution) is an emerging model suited to customer-service agents.

**Key Buyer Personas:** Enterprises seeking pre-built, certified agents to accelerate deployment; independent developers and boutique AI consultancies wanting a distribution channel for agent IP; platform operators (SaaS vendors) wanting to extend their product with AI agent capabilities; CIOs consolidating agent procurement through a vetted catalogue.

**Notable Trends:** Interoperability via MCP and A2A has shifted from a technical nicety to a commercial requirement — agents that cannot be plugged into existing systems do not sell. Safety, certification, and trust signals (SOC 2, privacy attestation) are becoming key listing requirements as enterprise buyers grow risk-aware. Outcome-based pricing is gaining traction as it aligns vendor incentives with buyer value.

## AI-Native Opportunity

- **Neutral, multi-cloud agent registry**: the biggest marketplace gap is a vendor-neutral discovery layer that works across AWS, Azure, Google Cloud, and Salesforce; an independent catalogue with standardised interoperability metadata (MCP manifests, A2A specs) could become the canonical agent registry
- **Automated agent certification and safety scanning**: buyers will not deploy untrusted agents at scale; automated capability verification, sandboxed test execution, and PII-leak scanning as part of the submission pipeline would differentiate a marketplace on trust
- **Outcome-based billing infrastructure**: the payments and metering layer for "pay-per-task-completed" is technically complex; a marketplace that abstracts billing, dispute resolution, and SLA tracking for outcome-priced agents would unlock a new commercial model
- **Agent composition and bundling**: rather than deploying single agents, enterprises want orchestrated bundles (e.g., a sales pipeline agent that delegates to a lead-scoring agent and a CRM-update agent); a marketplace that supports agent composition graphs has a meaningful depth advantage
- **Revenue-share and royalty tracking for agent authors**: as agents become intellectual property, clear attribution and automated royalty distribution for agents that call sub-agents from other authors becomes necessary infrastructure
