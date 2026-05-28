# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Internal Developer Portal · Created: 2026-05-20

## Philosophy

This model follows the traditional relational approach where every concept in the domain receives its own dedicated table with strongly-typed columns, foreign key constraints, and junction tables for many-to-many relationships. It mirrors how Backstage conceptually organises its catalog (Component, API, Resource, System, Domain, Group, User) but stores everything in PostgreSQL rather than YAML files, enabling complex cross-entity queries and enforcing referential integrity at the database level.

The normalised approach is well-suited to an IDP where data integrity matters more than schema flexibility. Service ownership, team membership, scorecard definitions, and DORA metrics all have well-defined shapes that rarely change. By modelling each concept explicitly, the schema serves as living documentation of the domain and enables the query planner to optimise joins across entity types.

This is the architecture that most enterprise software teams will be familiar with. It aligns with how products like OpsLevel and Cortex structure their backends — dedicated tables for services, teams, scorecards, and integrations, with referential integrity enforced by the database.

**Best for:** Teams that value data integrity, need complex cross-entity reporting, and operate in regulated environments where schema clarity and constraint enforcement are paramount.

**Trade-offs:**
- (+) Strong referential integrity — the database prevents orphaned records and broken relationships
- (+) Excellent query performance for known access patterns with proper indexing
- (+) Schema serves as documentation — new developers can understand the domain from the DDL
- (+) Standard PostgreSQL tooling for backup, replication, and monitoring
- (-) Schema migrations required for every new entity type or property — slower iteration
- (-) Junction tables proliferate for many-to-many relationships (service-to-team, entity-to-tag)
- (-) Less flexible for jurisdiction-specific or user-defined custom properties
- (-) Adding a new "kind" of catalog entity requires DDL changes and a deployment

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Backstage Catalog Descriptor Format | Entity kinds (Component, API, Resource, System, Domain, Group, User, Template) map 1:1 to dedicated tables |
| OpenAPI 3.1 / AsyncAPI 3.1 | `api_specs` table stores parsed specification metadata with type discriminator |
| DORA Metrics Framework | Dedicated `dora_metrics` table with columns for each of the four key metrics |
| ISO/IEC 25010 (SQuaRE) | Scorecard dimensions align with SQuaRE quality attributes (reliability, maintainability, etc.) |
| OAuth 2.0 / OIDC (RFC 6749) | `identity_providers` and `oauth_tokens` tables store SSO configuration |
| SCIM 2.0 (RFC 7644) | `users` and `teams` tables align with SCIM User and Group resource schemas |
| SLSA v1.0 | `supply_chain_assessments` table tracks SLSA level per component |
| OpenTelemetry | `service_health_snapshots` table stores OTel-derived metrics |
| Score Specification | `workload_specs` table can store Score YAML references |

---

## Core Catalog Tables

### Organizations and Tenancy

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE domains (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    owner_team_id   UUID,  -- FK added after teams table
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);
```

### Teams and Users

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    avatar_url      TEXT,
    scim_external_id VARCHAR(255),  -- SCIM 2.0 externalId (RFC 7643)
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'inactive', 'suspended')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, email)
);

CREATE TABLE teams (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    parent_team_id  UUID REFERENCES teams(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);

CREATE TABLE team_members (
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member'
                    CHECK (role IN ('owner', 'maintainer', 'member')),
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (team_id, user_id)
);

CREATE INDEX idx_team_members_user ON team_members(user_id);

-- Add deferred FK for domains.owner_team_id
ALTER TABLE domains ADD CONSTRAINT fk_domains_owner_team
    FOREIGN KEY (owner_team_id) REFERENCES teams(id);
```

### Components (Services, Libraries, Websites)

```sql
CREATE TABLE components (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    type            VARCHAR(50) NOT NULL
                    CHECK (type IN ('service', 'website', 'library', 'data-pipeline', 'mobile-app', 'cli')),
    lifecycle       VARCHAR(30) NOT NULL DEFAULT 'production'
                    CHECK (lifecycle IN ('experimental', 'development', 'production', 'deprecated')),
    owner_team_id   UUID NOT NULL REFERENCES teams(id),
    system_id       UUID,  -- FK added after systems table
    domain_id       UUID REFERENCES domains(id),
    repo_url        TEXT,
    language        VARCHAR(50),
    framework       VARCHAR(100),
    tier            VARCHAR(20) DEFAULT 'tier-3'
                    CHECK (tier IN ('tier-1', 'tier-2', 'tier-3', 'tier-4')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);

CREATE INDEX idx_components_owner ON components(owner_team_id);
CREATE INDEX idx_components_type ON components(type);
CREATE INDEX idx_components_lifecycle ON components(lifecycle);
```

### Systems

```sql
CREATE TABLE systems (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    owner_team_id   UUID NOT NULL REFERENCES teams(id),
    domain_id       UUID REFERENCES domains(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);

ALTER TABLE components ADD CONSTRAINT fk_components_system
    FOREIGN KEY (system_id) REFERENCES systems(id);
```

### APIs

```sql
CREATE TABLE apis (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    type            VARCHAR(30) NOT NULL
                    CHECK (type IN ('openapi', 'asyncapi', 'graphql', 'grpc')),
    lifecycle       VARCHAR(30) NOT NULL DEFAULT 'production',
    owner_team_id   UUID NOT NULL REFERENCES teams(id),
    system_id       UUID REFERENCES systems(id),
    spec_url        TEXT,
    spec_version    VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);

CREATE TABLE api_specs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_id          UUID NOT NULL REFERENCES apis(id) ON DELETE CASCADE,
    version         VARCHAR(50) NOT NULL,
    spec_content    TEXT NOT NULL,        -- Raw OpenAPI/AsyncAPI/GraphQL SDL
    parsed_at       TIMESTAMPTZ,
    endpoint_count  INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (api_id, version)
);

-- Component provides/consumes API relationships
CREATE TABLE component_api_relations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    component_id    UUID NOT NULL REFERENCES components(id) ON DELETE CASCADE,
    api_id          UUID NOT NULL REFERENCES apis(id) ON DELETE CASCADE,
    relation_type   VARCHAR(20) NOT NULL
                    CHECK (relation_type IN ('provides', 'consumes')),
    UNIQUE (component_id, api_id, relation_type)
);

CREATE INDEX idx_component_api_component ON component_api_relations(component_id);
CREATE INDEX idx_component_api_api ON component_api_relations(api_id);
```

### Resources

```sql
CREATE TABLE resources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    type            VARCHAR(50) NOT NULL
                    CHECK (type IN ('database', 'cache', 'queue', 'storage', 'cdn', 'vault', 'load-balancer')),
    owner_team_id   UUID NOT NULL REFERENCES teams(id),
    system_id       UUID REFERENCES systems(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);

CREATE TABLE component_resource_relations (
    component_id    UUID NOT NULL REFERENCES components(id) ON DELETE CASCADE,
    resource_id     UUID NOT NULL REFERENCES resources(id) ON DELETE CASCADE,
    relation_type   VARCHAR(20) NOT NULL DEFAULT 'depends_on'
                    CHECK (relation_type IN ('depends_on', 'owns')),
    PRIMARY KEY (component_id, resource_id)
);
```

---

## Documentation Tables

```sql
CREATE TABLE tech_docs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type     VARCHAR(30) NOT NULL
                    CHECK (entity_type IN ('component', 'api', 'system', 'resource')),
    entity_id       UUID NOT NULL,
    title           VARCHAR(500) NOT NULL,
    content_markdown TEXT NOT NULL,
    source          VARCHAR(30) NOT NULL DEFAULT 'manual'
                    CHECK (source IN ('manual', 'git-sync', 'ai-generated')),
    ai_model_version VARCHAR(100),       -- set when source = 'ai-generated'
    published       BOOLEAN NOT NULL DEFAULT false,
    version         INTEGER NOT NULL DEFAULT 1,
    author_user_id  UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tech_docs_entity ON tech_docs(entity_type, entity_id);
CREATE INDEX idx_tech_docs_source ON tech_docs(source);
```

---

## Scorecards and Engineering Standards

```sql
CREATE TABLE scorecards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    applies_to      VARCHAR(30) NOT NULL DEFAULT 'component'
                    CHECK (applies_to IN ('component', 'api', 'system', 'resource')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);

CREATE TABLE scorecard_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scorecard_id    UUID NOT NULL REFERENCES scorecards(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    category        VARCHAR(100),        -- e.g. 'reliability', 'security', 'documentation'
    weight          NUMERIC(5,2) NOT NULL DEFAULT 1.0,
    evaluation_type VARCHAR(30) NOT NULL
                    CHECK (evaluation_type IN ('boolean', 'threshold', 'regex', 'integration')),
    evaluation_config TEXT NOT NULL,      -- rule definition (JSON stored as text)
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE scorecard_evaluations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scorecard_id    UUID NOT NULL REFERENCES scorecards(id),
    entity_type     VARCHAR(30) NOT NULL,
    entity_id       UUID NOT NULL,
    rule_id         UUID NOT NULL REFERENCES scorecard_rules(id),
    passed          BOOLEAN NOT NULL,
    score           NUMERIC(5,2),
    details         TEXT,
    evaluated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_scorecard_evals_entity ON scorecard_evaluations(entity_type, entity_id);
CREATE INDEX idx_scorecard_evals_time ON scorecard_evaluations(evaluated_at DESC);
```

---

## DORA Metrics

```sql
CREATE TABLE deployment_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    component_id    UUID NOT NULL REFERENCES components(id),
    environment     VARCHAR(50) NOT NULL DEFAULT 'production',
    deployed_at     TIMESTAMPTZ NOT NULL,
    commit_sha      VARCHAR(64),
    deployer_id     UUID REFERENCES users(id),
    success         BOOLEAN NOT NULL DEFAULT true,
    lead_time_seconds BIGINT,            -- time from commit to deploy
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deployments_component ON deployment_events(component_id, deployed_at DESC);
CREATE INDEX idx_deployments_env ON deployment_events(environment, deployed_at DESC);

CREATE TABLE incidents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    title           VARCHAR(500) NOT NULL,
    severity        VARCHAR(20) NOT NULL
                    CHECK (severity IN ('sev-1', 'sev-2', 'sev-3', 'sev-4')),
    status          VARCHAR(30) NOT NULL DEFAULT 'open'
                    CHECK (status IN ('open', 'investigating', 'mitigating', 'resolved', 'closed')),
    started_at      TIMESTAMPTZ NOT NULL,
    resolved_at     TIMESTAMPTZ,
    time_to_restore_seconds BIGINT,      -- MTTR in seconds
    root_cause      TEXT,
    triggered_by_deployment_id UUID REFERENCES deployment_events(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE incident_affected_components (
    incident_id     UUID NOT NULL REFERENCES incidents(id) ON DELETE CASCADE,
    component_id    UUID NOT NULL REFERENCES components(id),
    PRIMARY KEY (incident_id, component_id)
);

CREATE TABLE dora_metric_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    component_id    UUID REFERENCES components(id),
    team_id         UUID REFERENCES teams(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    deployment_frequency  NUMERIC(10,4), -- deploys per day
    lead_time_seconds     BIGINT,        -- median lead time
    mttr_seconds          BIGINT,        -- median time to restore
    change_failure_rate   NUMERIC(5,4),  -- ratio 0.0 - 1.0
    dora_level            VARCHAR(20)
                    CHECK (dora_level IN ('elite', 'high', 'medium', 'low')),
    calculated_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dora_snapshots_component ON dora_metric_snapshots(component_id, period_start);
CREATE INDEX idx_dora_snapshots_team ON dora_metric_snapshots(team_id, period_start);
```

---

## Self-Service Actions

```sql
CREATE TABLE action_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    category        VARCHAR(50),
    input_schema    TEXT NOT NULL,        -- JSON Schema for action inputs
    execution_type  VARCHAR(30) NOT NULL
                    CHECK (execution_type IN ('webhook', 'github-action', 'terraform', 'script')),
    execution_config TEXT NOT NULL,
    requires_approval BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);

CREATE TABLE action_runs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id     UUID NOT NULL REFERENCES action_templates(id),
    triggered_by    UUID NOT NULL REFERENCES users(id),
    input_values    TEXT,                -- JSON of provided inputs
    status          VARCHAR(30) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'approved', 'rejected', 'running',
                                      'succeeded', 'failed', 'cancelled')),
    approved_by     UUID REFERENCES users(id),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    output_log      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_action_runs_template ON action_runs(template_id, created_at DESC);
CREATE INDEX idx_action_runs_status ON action_runs(status);
```

---

## Integrations

```sql
CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    type            VARCHAR(50) NOT NULL
                    CHECK (type IN ('github', 'gitlab', 'bitbucket', 'jira', 'slack',
                                    'pagerduty', 'prometheus', 'grafana', 'datadog',
                                    'opsgenie', 'custom-webhook')),
    config          TEXT NOT NULL,        -- encrypted JSON configuration
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'inactive', 'error')),
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE identity_providers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    protocol        VARCHAR(20) NOT NULL
                    CHECK (protocol IN ('oidc', 'saml', 'scim')),
    issuer_url      TEXT,
    client_id       VARCHAR(255),
    config          TEXT NOT NULL,        -- encrypted JSON
    enabled         BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Tags and Annotations

```sql
CREATE TABLE tags (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    key             VARCHAR(100) NOT NULL,
    value           VARCHAR(255) NOT NULL,
    UNIQUE (organization_id, key, value)
);

CREATE TABLE entity_tags (
    entity_type     VARCHAR(30) NOT NULL,
    entity_id       UUID NOT NULL,
    tag_id          UUID NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (entity_type, entity_id, tag_id)
);

CREATE INDEX idx_entity_tags_tag ON entity_tags(tag_id);

CREATE TABLE annotations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type     VARCHAR(30) NOT NULL,
    entity_id       UUID NOT NULL,
    key             VARCHAR(255) NOT NULL,
    value           TEXT NOT NULL,
    UNIQUE (entity_type, entity_id, key)
);

CREATE INDEX idx_annotations_entity ON annotations(entity_type, entity_id);
```

---

## Service Health (OpenTelemetry-derived)

```sql
CREATE TABLE service_health_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    component_id    UUID NOT NULL REFERENCES components(id),
    captured_at     TIMESTAMPTZ NOT NULL,
    error_rate      NUMERIC(7,4),        -- percentage
    p50_latency_ms  NUMERIC(10,2),
    p95_latency_ms  NUMERIC(10,2),
    p99_latency_ms  NUMERIC(10,2),
    request_rate    NUMERIC(12,2),       -- requests per second
    cpu_usage_pct   NUMERIC(5,2),
    memory_usage_mb NUMERIC(10,2),
    health_status   VARCHAR(20) NOT NULL DEFAULT 'healthy'
                    CHECK (health_status IN ('healthy', 'degraded', 'unhealthy', 'unknown'))
);

CREATE INDEX idx_health_snapshots_component ON service_health_snapshots(component_id, captured_at DESC);
```

---

## AI Features

```sql
CREATE TABLE ai_doc_generations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type     VARCHAR(30) NOT NULL,
    entity_id       UUID NOT NULL,
    model_name      VARCHAR(100) NOT NULL,
    prompt_hash     VARCHAR(64),
    input_sources   TEXT,                -- JSON: list of repos, files, commits analysed
    generated_doc   TEXT NOT NULL,
    confidence      NUMERIC(3,2),        -- 0.00 - 1.00
    accepted        BOOLEAN,
    reviewed_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE dependency_analysis_runs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    component_id    UUID NOT NULL REFERENCES components(id),
    analysis_type   VARCHAR(30) NOT NULL
                    CHECK (analysis_type IN ('blast-radius', 'dependency-chain', 'conflict-detection')),
    model_name      VARCHAR(100) NOT NULL,
    findings        TEXT NOT NULL,        -- JSON array of findings
    risk_score      NUMERIC(5,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE nl_query_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    query_text      TEXT NOT NULL,
    generated_sql   TEXT,
    result_count    INTEGER,
    response_time_ms INTEGER,
    feedback        VARCHAR(20)
                    CHECK (feedback IN ('helpful', 'not-helpful', 'incorrect')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## RBAC

```sql
CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    is_system       BOOLEAN NOT NULL DEFAULT false,
    UNIQUE (organization_id, name)
);

CREATE TABLE permissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_type   VARCHAR(50) NOT NULL,
    action          VARCHAR(30) NOT NULL,
    UNIQUE (resource_type, action)
);

CREATE TABLE role_permissions (
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id   UUID NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    scope_type      VARCHAR(30),         -- 'organization', 'domain', 'team'
    scope_id        UUID,                -- id of the scoped entity
    PRIMARY KEY (user_id, role_id, COALESCE(scope_type, ''), COALESCE(scope_id, '00000000-0000-0000-0000-000000000000'))
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organizations & Tenancy | 2 | organizations, domains |
| Teams & Users | 3 | users, teams, team_members |
| Catalog Entities | 8 | components, systems, apis, api_specs, resources, component_api_relations, component_resource_relations |
| Documentation | 1 | tech_docs |
| Scorecards | 3 | scorecards, scorecard_rules, scorecard_evaluations |
| DORA & Incidents | 4 | deployment_events, incidents, incident_affected_components, dora_metric_snapshots |
| Self-Service | 2 | action_templates, action_runs |
| Integrations | 2 | integrations, identity_providers |
| Tags & Annotations | 3 | tags, entity_tags, annotations |
| Observability | 1 | service_health_snapshots |
| AI Features | 3 | ai_doc_generations, dependency_analysis_runs, nl_query_log |
| RBAC | 4 | roles, permissions, role_permissions, user_roles |
| **Total** | **36** | |

---

## Key Design Decisions

1. **Separate tables per Backstage entity kind** — Components, APIs, Resources, and Systems each get their own table rather than a single polymorphic `entities` table. This enables typed foreign keys, kind-specific columns (e.g. `language` on components, `type` on APIs), and clearer query semantics at the cost of more tables.

2. **Polymorphic entity references via (entity_type, entity_id) pairs** — For cross-cutting concerns like tags, annotations, tech_docs, and scorecards that apply to multiple entity types, a type discriminator column is used. This avoids duplicating tag/annotation tables per entity kind.

3. **DORA metrics stored as both raw events and pre-computed snapshots** — `deployment_events` and `incidents` capture raw data; `dora_metric_snapshots` stores periodically calculated aggregates. This avoids expensive real-time aggregation while preserving source data.

4. **Text columns for JSON configuration** — Integration configs, scorecard rule definitions, and action input schemas are stored as TEXT rather than JSONB. This keeps the model purely relational — the JSON is treated as opaque configuration blobs, not queryable data.

5. **Hierarchical teams via self-referential FK** — `teams.parent_team_id` enables org chart hierarchies using recursive CTEs:
   ```sql
   WITH RECURSIVE team_tree AS (
       SELECT id, name, parent_team_id, 0 AS depth
       FROM teams WHERE id = :root_team_id
       UNION ALL
       SELECT t.id, t.name, t.parent_team_id, tt.depth + 1
       FROM teams t JOIN team_tree tt ON t.parent_team_id = tt.id
   )
   SELECT * FROM team_tree;
   ```

6. **UUID primary keys throughout** — Standard for modern SaaS applications, eliminates ID collision in multi-tenant and distributed systems.

7. **AI features as first-class tables** — Rather than treating AI as an add-on, doc generation, dependency analysis, and NL query logs have dedicated tables with model versioning and confidence scores, enabling quality tracking over time.
