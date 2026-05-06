# Standards & API Reference

> Project: Internal Developer Portal · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

No directly applicable ISO standards govern internal developer portals specifically. The following adjacent ISO standards are relevant to the security, interoperability, and data governance dimensions of an IDP:

- **ISO/IEC 27001** — Information security management systems. Relevant for enterprise IDP deployments that handle access credentials, source code metadata, and sensitive service ownership data. Most SaaS IDP vendors (OpsLevel, Port) hold SOC 2 Type II, which maps to ISO 27001 controls.
  URL: https://www.iso.org/isoiec-27001-information-security.html

- **ISO/IEC 27017** — Cloud security controls. Applicable to IDPs delivered as SaaS; governs data residency, tenant isolation, and cloud-provider responsibilities.
  URL: https://www.iso.org/standard/43757.html

- **ISO/IEC 25010** — Software product quality model (SQuaRE). Provides the quality attribute taxonomy (reliability, maintainability, portability, usability) commonly used to structure IDP scorecard dimensions and service maturity checks.
  URL: https://www.iso.org/standard/35733.html

### W3C & IETF Standards

- **RFC 6749 — OAuth 2.0 Authorization Framework**
  URL: https://datatracker.ietf.org/doc/html/rfc6749
  The authoritative specification for delegated authorization. All major IDPs (Backstage, Port, OpsLevel, Cortex) use OAuth 2.0 for integration with source control, CI/CD, and identity providers. The Authorization Code + PKCE flow is the recommended pattern for user-facing IDP authentication.

- **RFC 6750 — OAuth 2.0 Bearer Token Usage**
  URL: https://datatracker.ietf.org/doc/html/rfc6750
  Defines how bearer tokens are used in HTTP requests. Required by every IDP API client and integration plugin.

- **RFC 7591 — OAuth 2.0 Dynamic Client Registration Protocol**
  URL: https://datatracker.ietf.org/doc/html/rfc7591
  Enables dynamic registration of OAuth clients at runtime. Relevant for IDP plugin ecosystems where third-party integrations register themselves programmatically against an enterprise IdP without manual admin provisioning.

- **RFC 7592 — OAuth 2.0 Dynamic Client Registration Management Protocol**
  URL: https://datatracker.ietf.org/doc/html/rfc7592
  Extends RFC 7591 with management operations (update, delete) for dynamically registered clients. Useful for IDP platforms that expose a self-service integration marketplace.

- **OpenID Connect Core 1.0**
  URL: https://openid.net/specs/openid-connect-core-1_0.html
  Adds the identity layer on top of OAuth 2.0. IDPs use OIDC for SSO integration with enterprise identity providers (Okta, Azure AD, Google Workspace), enabling role-based access to portal features and self-service actions.

- **RFC 7644 — SCIM 2.0 Protocol (System for Cross-domain Identity Management)**
  URL: https://datatracker.ietf.org/doc/html/rfc7644
  Together with RFC 7643 (schema), SCIM 2.0 is the standard for automated user and group provisioning. Enterprise IDP deployments use SCIM to sync team membership and ownership data from the corporate IdP into the software catalog, keeping service ownership current without manual updates.

- **RFC 7643 — SCIM 2.0 Core Schema**
  URL: https://datatracker.ietf.org/doc/html/rfc7643
  Defines the user and group resource schemas that SCIM provisioning endpoints must implement. The enterprise user extension (URN `urn:ietf:params:scim:schemas:extension:enterprise:2.0:User`) provides department, team, and manager attributes directly useful for IDP ownership models.

- **RFC 8927 — JSON Type Definition (JTD)**
  URL: https://www.rfc-editor.org/rfc/rfc8927.html
  A portable, code-generation-friendly alternative to JSON Schema for describing the shape of JSON messages. Relevant for defining IDP entity data models and catalog API payloads; supported by Ajv and other validators used in IDP plugin development.

- **RFC 7807 — Problem Details for HTTP APIs**
  URL: https://datatracker.ietf.org/doc/html/rfc7807
  Standard format for machine-readable error responses in HTTP APIs. Any IDP REST API should follow this convention for consistent error handling across integrations and plugin SDKs.

### Data Model & API Specifications

- **OpenAPI Specification 3.1**
  URL: https://spec.openapis.org/oas/v3.1.0.html
  The primary standard for describing RESTful APIs. IDPs consume OpenAPI specs to populate service catalog API documentation and use OAS to expose their own management APIs. Backstage, Port, Cortex, and OpsLevel all provide or consume OpenAPI-formatted specifications. Version 3.1 aligns with JSON Schema draft 2020-12.

- **OpenAPI Specification 3.2 (latest)**
  URL: https://spec.openapis.org/oas/v3.2.0.html
  The most recent release of OAS at time of research, introducing refinements to webhooks and the schema object model.

- **AsyncAPI Specification 3.1**
  URL: https://www.asyncapi.com/docs/reference/specification/v3.1.0
  The event-driven counterpart to OpenAPI. Describes message-driven APIs over Kafka, AMQP, MQTT, WebSockets, and other protocols. IDPs managing microservices must catalog both REST and event-driven interfaces; Roadie and Backstage's api-docs plugin support AsyncAPI natively.

- **GraphQL Specification (June 2018 edition)**
  URL: https://spec.graphql.org/June2018/
  Used natively by OpsLevel as its primary API interface. Also supported as a catalogued API type in Backstage and Roadie. An IDP should be capable of rendering GraphQL schema documentation alongside OpenAPI and AsyncAPI specs.

- **Backstage Catalog Descriptor Format (catalog-info.yaml)**
  URL: https://backstage.io/docs/features/software-catalog/descriptor-format/
  The de-facto standard for machine-readable service descriptors in the IDP category. Defines entity kinds: `Component`, `API`, `Resource`, `System`, `Domain`, `Group`, `User`, `Template`, and `Location`. Each YAML document carries `apiVersion`, `kind`, `metadata`, and `spec` envelope fields. This format has become the industry reference point; any competing IDP should either support it natively or provide a migration path.

- **JSON Schema (draft 2020-12 / draft-07)**
  URL: https://json-schema.org/specification
  Used for validating IDP entity descriptor payloads, custom blueprint/property schemas (Port, Cortex), and API request/response bodies. Draft-07 remains widely supported; draft 2020-12 is aligned with OAS 3.1.

- **Score Specification (CNCF sandbox project)**
  URL: https://docs.score.dev/
  Platform-agnostic workload specification (YAML) that describes container-based workloads independently of the target platform. Humanitec and other IDP vendors support Score as a developer-facing abstraction, enabling portability across Kubernetes, Docker Compose, and serverless environments.

### Security & Authentication Standards

- **OAuth 2.0 (RFC 6749) + PKCE (RFC 7636)**
  URL: https://datatracker.ietf.org/doc/html/rfc7636
  Proof Key for Code Exchange — the recommended hardening of the Authorization Code flow for browser and native apps. Required for any IDP frontend performing OAuth-based SSO or plugin authentication.

- **OpenID Connect Core 1.0 + Discovery**
  URL: https://openid.net/specs/openid-connect-discovery-1_0.html
  Discovery allows IDP instances to automatically configure themselves against enterprise IdPs by fetching the provider metadata document from `/.well-known/openid-configuration`.

- **OWASP Top 10 (latest edition)**
  URL: https://owasp.org/www-project-top-ten/
  Developer portals aggregate secrets, tokens, source code links, and infrastructure metadata. The OWASP Top 10 injection, broken access control, and security misconfiguration categories are directly applicable. IDP scorecard checks often track OWASP compliance at the service level.

- **SLSA (Supply Chain Levels for Software Artifacts) v1.0**
  URL: https://slsa.dev/spec/v1.0/
  An emerging CNCF-backed framework for supply chain integrity. IDP scorecards increasingly surface SLSA level per service (provenance attestation, build integrity). Not a formal standards body publication but widely adopted.

- **NIST SP 800-53 Rev 5 — Security and Privacy Controls**
  URL: https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final
  The federal baseline for security controls. Enterprise IDP deployments in regulated industries (finance, government) are assessed against NIST 800-53 control families, particularly AC (Access Control), AU (Audit), CM (Configuration Management), and SA (System and Services Acquisition).

### MCP Server Specifications

- **Model Context Protocol (MCP) — Anthropic**
  URL: https://modelcontextprotocol.io/specification
  MCP is a JSON-RPC 2.0-based protocol that standardises how AI agents access context from external systems. An AI-native IDP is a natural MCP server: it can expose the software catalog, scorecards, runbooks, and self-service action definitions as structured context for AI coding assistants and agents. This is a significant differentiator opportunity over legacy IDPs. MCP servers expose `resources`, `tools`, and `prompts`; an IDP MCP server would surface catalog entities as resources and self-service actions as tools.
  URL: https://spec.modelcontextprotocol.io/

### Observability & Metrics Standards

- **OpenTelemetry (OTel) — CNCF graduated project**
  URL: https://opentelemetry.io/
  Vendor-neutral standard for distributed tracing, metrics, and logs. IDPs pull deployment events, error rates, and latency metrics from OTel-compatible observability backends (Datadog, Grafana, Honeycomb) to populate service health dashboards. The CNCF CI/CD Observability SIG is extending OTel semantic conventions to cover CI/CD pipeline telemetry, directly applicable to DORA metric collection within an IDP.

- **DORA Metrics Framework**
  URL: https://dora.dev/
  The four key DORA metrics (Deployment Frequency, Lead Time for Changes, Mean Time to Restore, Change Failure Rate) are the dominant engineering performance framework displayed in IDP dashboards. Not an IETF/ISO standard, but the authoritative reference for developer productivity measurement.

- **CNCF Platform Engineering Maturity Model**
  URL: https://tag-app-delivery.cncf.io/whitepapers/platform-eng-maturity-model/
  Five-level maturity model defining progressive standardisation, automation, and user-centricity for internal developer platforms. Used by platform engineering teams to plan IDP roadmaps and benchmark portal capabilities.

---

## Similar Products — Developer Documentation & APIs

### Backstage (Spotify / CNCF)

- **Description:** Open-source framework for building developer portals, powering the software catalog and plugin ecosystem; donated to CNCF, the most widely adopted IDP foundation.
- **API Documentation:** https://backstage.io/docs/features/software-catalog/software-catalog-api/
- **Plugin API (api-docs plugin):** https://github.com/backstage/backstage/blob/master/plugins/api-docs/README.md
- **Catalog Descriptor Format:** https://backstage.io/docs/features/software-catalog/descriptor-format/
- **GitHub Repository:** https://github.com/backstage/backstage
- **Developer Guide:** https://backstage.io/docs/
- **Standards:** REST/JSON catalog API; YAML entity descriptors; OpenAPI, AsyncAPI, GraphQL, gRPC API type support
- **Authentication:** OAuth 2.0 / OIDC (configurable); supports Okta, Auth0, GitHub, GitLab, Azure AD, Google

### Port

- **Description:** No-code internal developer portal with customisable blueprints, self-service actions, and an agentic AI layer; positions as "agentic IDP."
- **API Documentation:** https://docs.port.io/api-reference/port-api/
- **Developer Docs:** https://docs.port.io/
- **Ocean Framework (integrations):** https://ocean.port.io/
- **Standards:** REST/JSON (OpenAPI-documented); entity blueprints use a JSON Schema-like property model
- **Authentication:** API key (Bearer token); OAuth 2.0 / OIDC for portal SSO

### OpsLevel

- **Description:** Commercial IDP with automated service maturity checks, scorecards, self-service actions, and a developer knowledge center.
- **API Documentation:** https://docs.opslevel.com/docs/graphql
- **General API Docs:** https://docs.opslevel.com/docs/api-docs
- **Standards:** GraphQL API (primary); REST used for pushing API doc artifacts; OpenAPI spec ingestion for service documentation
- **Authentication:** API token (Bearer); SSO via OIDC / SAML

### Cortex

- **Description:** Engineering operations platform centred on a service catalog, scorecards, and developer self-service workflows.
- **API Documentation:** https://docs.cortex.io/api
- **Developer Guide:** https://docs.cortex.io/
- **Standards:** REST/JSON API; supports OpenAPI and AsyncAPI spec display in catalog; GitOps integration via YAML descriptor files
- **Authentication:** Bearer API key; SSO via OIDC / SAML 2.0

### Roadie (Managed Backstage)

- **Description:** Managed SaaS offering of Backstage with pre-installed plugins, automatic upgrades, and additional security controls.
- **API Documentation:** https://roadie.io/docs/
- **OpenAPI Specs Guide:** https://roadie.io/docs/catalog/openapi-specs/
- **Standards:** Inherits Backstage catalog REST API; supports OpenAPI, AsyncAPI, GraphQL, gRPC, and custom API types
- **Authentication:** Backstage-compatible OAuth 2.0 / OIDC

### Humanitec Platform Orchestrator

- **Description:** Platform orchestrator focused on dynamic configuration management and the Score workload specification; acts as the engine layer beneath an IDP portal.
- **API Documentation:** https://api-docs.humanitec.com/
- **Developer Docs:** https://developer.humanitec.com/
- **Score Spec Docs:** https://developer.humanitec.com/app-humanitec-io/docs/score/overview/
- **GitHub (Score):** https://github.com/score-spec/score-humanitec
- **Standards:** REST/JSON API; YAML-based Score specification (platform-agnostic workload spec)
- **Authentication:** Bearer API key; OAuth 2.0 for enterprise

### DX (GetDX)

- **Description:** Developer intelligence platform focused on developer experience measurement, DORA metrics, and survey-based sentiment tracking.
- **API Documentation:** https://docs.getdx.com/webapi/overview/
- **Data Cloud API:** https://docs.getdx.com/datacloudapi/overview/
- **Developer Docs:** https://docs.getdx.com/
- **Standards:** REST/JSON (Web API); 1000 req/min rate limit; JSON responses with `ok` status pattern
- **Authentication:** API key (Bearer token)

### Atlassian Compass

- **Description:** Developer experience platform from Atlassian with component catalog, DevOps metrics, and deep Jira/Confluence integration.
- **API Documentation:** https://developer.atlassian.com/cloud/compass/
- **Standards:** REST/JSON; Atlassian Forge extensibility platform; YAML component descriptors
- **Authentication:** OAuth 2.0 (Atlassian platform); API token support

---

## Notes

### Emerging & Evolving Areas

- **AI-native IDP APIs:** No standard yet exists for AI-augmented IDP capabilities (natural-language catalog search, AI-generated runbooks, automated scorecard remediation). The MCP specification is the strongest candidate for standardising how AI agents interact with IDP data. This is a genuine open gap in the standards landscape.

- **Federated catalog standards:** No cross-vendor standard exists for federating software catalogs across organizations or tools. Backstage's YAML descriptor format is the de-facto baseline, but it is not governed by a formal standards body. An interoperability specification analogous to OpenAPI for service catalogs would be a significant contribution.

- **Developer experience metrics:** DORA is the dominant framework but is not an IETF or ISO standard. The SPACE framework (Satisfaction, Performance, Activity, Communication, Efficiency) is an academic/industry complement that is gaining adoption in IDP dashboards but lacks a formal specification.

- **GitOps integration:** The OpenGitOps principles (https://opengitops.dev/) and the Argo CD / Flux CD ecosystem inform how IDP self-service actions trigger infrastructure changes via Git commits. No formal standard governs the IDP-to-GitOps handoff, representing an integration specification gap.

- **Plugin / extension registries:** Backstage's plugin marketplace is the largest, but there is no cross-platform plugin specification. An open plugin manifest format (similar to VS Code's extension schema) for IDP plugins would reduce vendor lock-in and lower the cost of building integrations.
