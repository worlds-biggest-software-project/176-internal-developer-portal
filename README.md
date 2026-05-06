# Internal Developer Portal

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source internal developer portal that combines a flexible service catalog with LLM-powered documentation, dependency analysis, and incident correlation.

This project is a Backstage-class internal developer portal targeted at platform engineering teams who want the extensibility of open source without the multi-FTE implementation burden. It centralises service ownership, documentation, scorecards, and self-service actions, then layers AI capabilities that today's incumbents do not yet provide.

---

## Why Internal Developer Portal?

- Backstage delivers unmatched extensibility but typically requires 2–3 FTE engineers and 6+ months to stand up, with an estimated hidden cost of ~$150K/year per 20 developers in maintenance time.
- Commercial alternatives (Port, Cortex, OpsLevel) charge $30–$65 per user per month and lock teams into proprietary data models and SaaS-only deployment.
- Gartner's 2025 Market Guide projects that by 2028, 85% of platform engineering teams will provide IDPs, yet no leading platform auto-generates service documentation, supports natural-language catalog queries, or auto-correlates incidents to owners and runbooks.
- Existing scorecards and production-readiness checks are static checklists; none adapt to observed incident patterns.
- The "build vs buy" debate has shifted toward buy, leaving a gap for an open-source option that is turnkey rather than framework-only.

---

## Key Features

### Service Catalog and Ownership

- Customizable entity types and relationships covering services, apps, libraries, and data pipelines
- Ownership tracking with team and domain-based organization
- Git-based auto-discovery of catalog entries
- Multi-tenancy with role-based access control

### Documentation and Standards

- Docs-like-code documentation with Markdown living alongside source
- Scorecards and production-readiness checks for codifying engineering standards
- DORA metrics reporting (deployment frequency, lead time, MTTR, change failure rate)
- Maturity reports with team segmentation

### Self-Service and Automation

- Templated service scaffolding and creation
- Self-service actions for infrastructure provisioning and common developer tasks
- Approval workflows integrated with Slack, email, and PagerDuty
- Webhook and API-based custom integrations

### AI-Powered Capabilities

- AI-generated service documentation derived from code, commit history, and runbooks
- Intelligent dependency graph analysis surfacing blast-radius risks via LLMs
- Natural-language catalog querying (e.g. "which services call this deprecated API?")
- Adaptive production-readiness scoring trained on incident patterns
- Automated incident correlation linking failing services to on-call owners, runbooks, and recent deployments

### Integrations

- Git providers (GitHub, GitLab, Bitbucket)
- Issue trackers, Slack, PagerDuty
- Kubernetes APIs and observability tools (Prometheus, Grafana, OpenTelemetry sources)
- OpenAPI / AsyncAPI ingestion for service documentation

---

## AI-Native Advantage

Incumbents treat AI as a roadmap item; only Port has begun shipping agentic features (2026), and adoption data is limited. This project makes AI core: LLMs auto-generate service docs from code and commits to close the chronic documentation gap, models analyse dependency graphs to surface hidden blast-radius risks that static mapping misses, natural-language queries replace point-and-click navigation, and incident correlation links failures to owners, runbooks, and deployments — capabilities that research identifies as genuinely underserved across the entire incumbent landscape.

---

## Tech Stack & Deployment

Designed for self-hosting (Apache 2.0-style open source positioning) with an optional managed cloud path, following the precedent of Backstage and Plural. Ingests OpenAPI / AsyncAPI for service documentation, surfaces DORA metrics, and integrates with OpenTelemetry-compatible observability sources, OAuth 2.0 / OIDC for SSO, and SLSA for supply-chain scoring. YAML descriptor files (in the style of `catalog-info.yaml`) drive the catalog. SDKs and plugin patterns follow the React-based extensibility model proven in the Backstage ecosystem.

---

## Market Context

Gartner's 2025 Market Guide for Internal Developer Portals projects that 85% of platform engineering teams will provide IDPs by 2028. Commercial IDPs price at $30–$65 per user per month; Backstage's nominal-free model carries an estimated $150K/year per 20 developers in maintenance overhead, making commercial alternatives 8–16x cheaper in TCO for most teams. Primary buyers are platform engineering leads, VPs of Engineering and CTOs at organisations with 50+ developers, and DevOps/SRE leads owning internal tooling standardisation.

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
