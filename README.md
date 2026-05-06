# Autonomous Agent Marketplace

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> A vendor-neutral, multi-cloud marketplace for deploying, discovering, and monetizing autonomous AI agents.

The Autonomous Agent Marketplace is a registry and commercial platform where developers can publish AI agents and enterprises can discover, certify, and deploy them across cloud providers. It targets the gap between hyperscaler-locked marketplaces (Salesforce AgentExchange, Google Cloud AI Agent Marketplace, Oracle Fusion) and developer-centric platforms (Replit, Agentverse) by offering a neutral catalogue with first-class interoperability via MCP and A2A.

---

## Why Autonomous Agent Marketplace?

- **Hyperscaler lock-in**: Salesforce AgentExchange is restricted to the Salesforce ecosystem, Google Cloud's marketplace requires Gemini Enterprise, and Oracle's marketplace only serves Fusion customers — none works as a neutral catalogue across AWS, Azure, Google Cloud, and on-premises.
- **Weak certification**: existing platforms rely on manual review (Oracle's 21-point process) or lightweight keyword scanning; sandboxed adversarial testing, prompt-injection probing, and PII-leak detection are largely absent from submission pipelines.
- **No outcome-based billing primitives**: Salesforce has experimented with per-conversation ($2), per-action ($0.10), and per-user ($125/mo) models, but no marketplace provides metering, dispute resolution, and SLA tracking for pay-per-result agents as core infrastructure.
- **Missing composition layer**: enterprises increasingly want agent bundles (e.g., lead-to-pipeline workflows that orchestrate scoring, qualification, and CRM-update sub-agents), yet no surveyed marketplace exposes composition graphs as a first-class feature.
- **Royalty attribution gap**: as agents call sub-agents from other authors, automated revenue-share and royalty tracking is missing across all surveyed platforms.

---

## Key Features

### Vendor-Neutral Registry and Discovery

- Multi-cloud agent registry spanning AWS, Azure, Google Cloud, Salesforce, and on-premises deployments
- LLM-powered semantic and intent-based discovery with conversational refinement for non-technical buyers
- Standardised agent metadata via Agent Card (A2A), MCP server descriptions, and OpenAPI 3.x specifications
- Private marketplace functionality for cost control and compliance

### Automated Certification and Safety

- Submission pipeline with sandboxed test execution
- Static and LLM-based adversarial testing for prompt injection and malicious commands
- Contextual PII-leak scanning
- Optional third-party certification badges (e.g. SOC 2, Marketplace Verified)

### Interoperability and Standards Compliance

- Native Model Context Protocol (MCP) support for tool and context hand-offs
- Agent2Agent (A2A) protocol for cross-organisation agent delegation
- OpenAPI 3.x for callable HTTP capability descriptions
- OAuth 2.0 / PKCE for delegated authorisation across third-party services

### Commercial Infrastructure

- Usage-based and outcome-based billing with metering and settlement
- SLA tracking and automated dispute resolution
- Revenue-share and royalty attribution for agents that call sub-agents from other authors
- IAM, role-based access, audit logs, and cost controls suitable for enterprise deployment

### Composition and Performance

- Agent composition and bundling with a workflow editor and testing environment
- Outcome verification (e.g. resolution detection for support agents)
- Performance benchmarking dashboards covering success rates, latency, cost, and user feedback
- Regression detection and author alerts for behaviour shifts

---

## AI-Native Advantage

LLMs are used inside the marketplace itself, not just within the agents listed on it. Discovery is driven by intent inference rather than keyword matching; certification uses LLM-powered adversarial agents to probe submissions for prompt-injection and PII-leak vulnerabilities; outcome verification (e.g. "was this lead truly qualified?") is automated rather than manually adjudicated; and composition recommendations suggest which agents work well together based on declared capabilities. This positions the marketplace ahead of incumbents that rely on static rule sets, manual review, and keyword search.

---

## Tech Stack & Deployment

The marketplace is designed to be deployable in self-hosted, cloud, and hybrid configurations, with no dependency on a single hyperscaler. Integration is anchored on three open standards: MCP (9,400+ public servers as of April 2026), A2A (150+ production deployments as of May 2026, governed by the Linux Foundation), and OpenAPI 3.x. Authorisation uses OAuth 2.0 with PKCE. Optional decentralised settlement (following the CROO / Agentverse model) is on the backlog for developer communities that prefer on-chain payments.

---

## Market Context

Gartner projected up to 40% of enterprise applications would include task-specific AI agents by 2026, and Deloitte estimated enterprise generative-AI deployments of agents would rise from 25% in 2025 to 50% by 2027. Analyst estimates embed agent marketplaces within a broader agentic AI platform market projected to exceed USD 9 billion by 2027. Incumbent pricing ranges from Agentforce at $2/conversation or $125/user/month, through Replit Core at $20/month, to bundled Oracle Fusion licensing. Primary buyers are enterprises consolidating agent procurement, independent developers and AI consultancies seeking distribution, and SaaS vendors extending their products with agent capabilities.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
