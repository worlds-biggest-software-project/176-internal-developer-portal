# Data Model Suggestion 3: Hybrid Relational + JSONB (Blueprint Model)

> Project: Internal Developer Portal · Created: 2026-05-20

## Philosophy

This model combines fixed relational tables for structural integrity with JSONB columns for flexible, user-defined properties. It directly mirrors Port's "blueprint" architecture — where entity types are themselves configurable — enabling platform engineers to define custom entity kinds, properties, and relationships without DDL changes or application redeployment.

The core insight is that an IDP must model entities whose shape varies by organisation: one company tracks "microservices" with fields for Kubernetes namespace and Helm chart version; another tracks "data pipelines" with fields for schedule, upstream datasets, and SLA thresholds. A purely normalised model forces every possible field into the schema upfront, while a purely document-oriented model sacrifices relational integrity. The hybrid approach gives both: strong relational scaffolding for identity, ownership, and relationships, with JSONB columns that store organisation-specific properties validated against user-defined JSON Schemas.

This is the architecture that Port, the fastest-growing commercial IDP ($800M valuation, Series C), uses at its core. Blueprints define schemas; entities are instances of blueprints with properties stored as structured JSON. Relations between blueprints create a navigable graph. This approach delivers the fastest time-to-value because new entity types require zero code changes.

**Best for:** Teams that need maximum flexibility in their data model, want to support diverse entity types without code changes, and are building a multi-tenant SaaS where each customer defines their own catalog structure.

**Trade-offs:**
- (+) New entity types and properties added without DDL or deployment — platform engineers configure via UI/API
- (+) Multi-tenant flexibility — each organisation defines its own blueprints independently
- (+) Fast MVP development — core schema is small; complexity is in the JSONB layer
- (+) JSON Schema validation ensures JSONB data isn't a free-for-all
- (+) Mirrors proven architecture (Port's blueprint model)
- (-) JSONB queries are slower than indexed relational columns for complex filtering
- (-) Reporting across heterogeneous JSONB structures requires careful query design
- (-) JSON Schema validation is application-level, not database-enforced
- (-) Migrations of JSONB structure across existing entities require batch update scripts
- (-) Developers must understand both SQL and JSONB query syntax

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| JSON Schema (draft 2020-12) | Blueprint property definitions are JSON Schemas; entity properties validated against them |
| Backstage Catalog Descriptor Format | Default blueprints model Backstage entity kinds; YAML import maps to blueprint instances |
| OpenAPI 3.1 | API blueprint includes spec_type and spec_url properties; OpenAPI schema objects inform property validation |
| DORA Metrics Framework | DORA metrics stored as properties on a `deployment_metric` blueprint or dedicated projection |
| ISO/IEC 25010 (SQuaRE) | Scorecard rule categories align with SQuaRE quality attributes |
| OAuth 2.0 / OIDC | Identity providers modelled as a blueprint; tokens in relational auth tables |
| SCIM 2.0 | User sync populates user entities; SCIM attributes map to user blueprint properties |
| OpenTelemetry | Health data properties on component blueprint populated from OTel sources |

---

## Blueprint Definition Tables

```sql
-- Blueprints define the schema for entity types
CREATE TABLE blueprints (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    identifier      VARCHAR(100) NOT NULL,  -- e.g. 'service', 'api', 'data-pipeline', 'cloud-resource'
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    icon            VARCHAR(50),            -- icon identifier for UI
    color           VARCHAR(7),             -- hex color for UI
    schema          JSONB NOT NULL,         -- JSON Schema defining allowed properties
    -- schema example:
    -- {
    --   "properties": {
    --     "language": {"type": "string", "enum": ["go", "python", "java", "typescript", "rust"]},
    --     "framework": {"type": "string"},
    --     "tier": {"type": "string", "enum": ["tier-1", "tier-2", "tier-3", "tier-4"]},
    --     "repo_url": {"type": "string", "format": "uri"},
    --     "lifecycle": {"type": "string", "enum": ["experimental", "development", "production", "deprecated"]},
    --     "k8s_namespace": {"type": "string"},
    --     "helm_chart": {"type": "string"},
    --     "on_call_schedule": {"type": "string"}
    --   },
    --   "required": ["language", "lifecycle"]
    -- }
    calculation_properties JSONB,          -- derived properties computed from relations/data
    mirror_properties JSONB,               -- properties inherited from related entities
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, identifier)
);

CREATE INDEX idx_blueprints_org ON blueprints(organization_id);

-- Blueprint relations define how entity types connect
CREATE TABLE blueprint_relations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_blueprint_id UUID NOT NULL REFERENCES blueprints(id) ON DELETE CASCADE,
    target_blueprint_id UUID NOT NULL REFERENCES blueprints(id) ON DELETE CASCADE,
    identifier      VARCHAR(100) NOT NULL,  -- e.g. 'owned_by', 'depends_on', 'provides_api'
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    cardinality     VARCHAR(20) NOT NULL DEFAULT 'many'
                    CHECK (cardinality IN ('one', 'many')),
    required        BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_blueprint_id, identifier)
);
```

---

## Entity Instance Tables

```sql
-- Entities are instances of blueprints — the actual catalog entries
CREATE TABLE entities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    blueprint_id    UUID NOT NULL REFERENCES blueprints(id),
    identifier      VARCHAR(255) NOT NULL,  -- unique name within blueprint
    title           VARCHAR(500),
    description     TEXT,
    properties      JSONB NOT NULL DEFAULT '{}',  -- validated against blueprint.schema
    -- properties example for a 'service' blueprint:
    -- {
    --   "language": "go",
    --   "framework": "gin",
    --   "tier": "tier-1",
    --   "repo_url": "https://github.com/acme/payment-service",
    --   "lifecycle": "production",
    --   "k8s_namespace": "payments",
    --   "helm_chart": "payment-service-v2.3.1",
    --   "on_call_schedule": "payments-oncall-rotation"
    -- }
    owner_team_id   UUID,                   -- common enough to be a first-class column
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, blueprint_id, identifier)
);

CREATE INDEX idx_entities_blueprint ON entities(blueprint_id);
CREATE INDEX idx_entities_owner ON entities(owner_team_id);
CREATE INDEX idx_entities_org ON entities(organization_id);

-- GIN index for JSONB property queries
CREATE INDEX idx_entities_properties ON entities USING gin(properties);

-- Partial indexes for common filtered queries
CREATE INDEX idx_entities_production ON entities((properties->>'lifecycle'))
    WHERE properties->>'lifecycle' = 'production';

-- Entity relations are instances of blueprint_relations
CREATE TABLE entity_relations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_entity_id UUID NOT NULL REFERENCES entities(id) ON DELETE CASCADE,
    target_entity_id UUID NOT NULL REFERENCES entities(id) ON DELETE CASCADE,
    relation_id     UUID NOT NULL REFERENCES blueprint_relations(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_entity_id, target_entity_id, relation_id)
);

CREATE INDEX idx_entity_relations_source ON entity_relations(source_entity_id);
CREATE INDEX idx_entity_relations_target ON entity_relations(target_entity_id);
CREATE INDEX idx_entity_relations_relation ON entity_relations(relation_id);
```

---

## Organizations, Teams, and Users

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    settings        JSONB NOT NULL DEFAULT '{}',  -- org-level configuration
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    avatar_url      TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    profile         JSONB NOT NULL DEFAULT '{}',  -- extensible user profile properties
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
    properties      JSONB NOT NULL DEFAULT '{}',  -- team-specific metadata
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);

CREATE TABLE team_members (
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (team_id, user_id)
);
```

---

## Scorecards

```sql
CREATE TABLE scorecards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    identifier      VARCHAR(100) NOT NULL,
    description     TEXT,
    target_blueprint_id UUID NOT NULL REFERENCES blueprints(id),
    rules           JSONB NOT NULL,       -- array of rule definitions
    -- rules example:
    -- [
    --   {
    --     "identifier": "has-runbook",
    --     "title": "Has Runbook",
    --     "category": "documentation",
    --     "level": "bronze",
    --     "query": {"combinator": "and", "conditions": [
    --       {"property": "properties.runbook_url", "operator": "isNotEmpty"}
    --     ]}
    --   },
    --   {
    --     "identifier": "tier1-has-oncall",
    --     "title": "Tier-1 Has On-Call",
    --     "category": "reliability",
    --     "level": "silver",
    --     "query": {"combinator": "or", "conditions": [
    --       {"property": "properties.tier", "operator": "!=", "value": "tier-1"},
    --       {"property": "properties.on_call_schedule", "operator": "isNotEmpty"}
    --     ]}
    --   }
    -- ]
    levels          JSONB NOT NULL DEFAULT '["bronze", "silver", "gold"]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, identifier)
);

CREATE TABLE scorecard_evaluations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scorecard_id    UUID NOT NULL REFERENCES scorecards(id) ON DELETE CASCADE,
    entity_id       UUID NOT NULL REFERENCES entities(id) ON DELETE CASCADE,
    results         JSONB NOT NULL,       -- per-rule pass/fail results
    overall_score   NUMERIC(5,2),
    level_achieved  VARCHAR(20),
    evaluated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_scorecard_evals_entity ON scorecard_evaluations(entity_id, evaluated_at DESC);
CREATE INDEX idx_scorecard_evals_scorecard ON scorecard_evaluations(scorecard_id, evaluated_at DESC);
```

---

## Self-Service Actions

```sql
CREATE TABLE actions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    identifier      VARCHAR(100) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    target_blueprint_id UUID REFERENCES blueprints(id),
    trigger_type    VARCHAR(30) NOT NULL
                    CHECK (trigger_type IN ('manual', 'on_create', 'on_update', 'on_delete', 'scheduled')),
    input_schema    JSONB NOT NULL,       -- JSON Schema for action inputs
    execution       JSONB NOT NULL,       -- execution configuration
    -- execution example:
    -- {
    --   "type": "webhook",
    --   "url": "https://api.acme.com/actions/provision-db",
    --   "method": "POST",
    --   "headers": {"Authorization": "Bearer {{secrets.WEBHOOK_TOKEN}}"}
    -- }
    requires_approval BOOLEAN NOT NULL DEFAULT false,
    approval_config JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, identifier)
);

CREATE TABLE action_runs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    action_id       UUID NOT NULL REFERENCES actions(id),
    entity_id       UUID REFERENCES entities(id),
    triggered_by    UUID NOT NULL REFERENCES users(id),
    inputs          JSONB NOT NULL DEFAULT '{}',
    status          VARCHAR(30) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'approved', 'rejected', 'running',
                                      'succeeded', 'failed', 'cancelled')),
    approved_by     UUID REFERENCES users(id),
    output          JSONB,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_action_runs_action ON action_runs(action_id, created_at DESC);
CREATE INDEX idx_action_runs_entity ON action_runs(entity_id, created_at DESC);
CREATE INDEX idx_action_runs_status ON action_runs(status);
```

---

## DORA Metrics and Deployments

```sql
CREATE TABLE deployment_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_id       UUID NOT NULL REFERENCES entities(id),
    environment     VARCHAR(50) NOT NULL DEFAULT 'production',
    deployed_at     TIMESTAMPTZ NOT NULL,
    commit_sha      VARCHAR(64),
    deployer_id     UUID REFERENCES users(id),
    success         BOOLEAN NOT NULL DEFAULT true,
    metadata        JSONB NOT NULL DEFAULT '{}',  -- CI/CD pipeline info, image tags, etc.
    lead_time_seconds BIGINT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deployments_entity ON deployment_events(entity_id, deployed_at DESC);

CREATE TABLE incidents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    title           VARCHAR(500) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
    started_at      TIMESTAMPTZ NOT NULL,
    resolved_at     TIMESTAMPTZ,
    time_to_restore_seconds BIGINT,
    metadata        JSONB NOT NULL DEFAULT '{}',  -- root cause, postmortem links, etc.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE incident_entities (
    incident_id     UUID NOT NULL REFERENCES incidents(id) ON DELETE CASCADE,
    entity_id       UUID NOT NULL REFERENCES entities(id),
    PRIMARY KEY (incident_id, entity_id)
);
```

---

## Integrations

```sql
CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    type            VARCHAR(50) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    config          JSONB NOT NULL,          -- encrypted connection details
    mapping_config  JSONB NOT NULL,          -- maps external data to blueprint properties
    -- mapping_config example:
    -- {
    --   "target_blueprint": "service",
    --   "entity_identifier": ".metadata.name",
    --   "properties": {
    --     "language": ".spec.language",
    --     "repo_url": ".metadata.annotations[\"backstage.io/source-location\"]"
    --   },
    --   "relations": {
    --     "owned_by": ".spec.owner"
    --   }
    -- }
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    last_sync_at    TIMESTAMPTZ,
    sync_interval_seconds INTEGER DEFAULT 3600,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE identity_providers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    protocol        VARCHAR(20) NOT NULL,
    config          JSONB NOT NULL,
    enabled         BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## AI Features

```sql
CREATE TABLE ai_doc_generations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_id       UUID NOT NULL REFERENCES entities(id),
    model_name      VARCHAR(100) NOT NULL,
    input_context   JSONB,               -- repos, files, commits analysed
    generated_doc   TEXT NOT NULL,
    confidence      NUMERIC(3,2),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'accepted', 'rejected')),
    reviewed_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE dependency_analyses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_id       UUID NOT NULL REFERENCES entities(id),
    analysis_type   VARCHAR(30) NOT NULL,
    model_name      VARCHAR(100) NOT NULL,
    findings        JSONB NOT NULL,
    risk_score      NUMERIC(5,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE nl_query_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    query_text      TEXT NOT NULL,
    generated_query TEXT,                -- generated search/SQL
    result_count    INTEGER,
    response_time_ms INTEGER,
    feedback        VARCHAR(20),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    actor_id        UUID,
    actor_type      VARCHAR(20) NOT NULL DEFAULT 'user',
    action          VARCHAR(100) NOT NULL,   -- 'entity.created', 'blueprint.updated', etc.
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID,
    before_state    JSONB,                   -- previous JSONB properties (for updates)
    after_state     JSONB,                   -- new JSONB properties
    metadata        JSONB,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_org_time ON audit_log(organization_id, occurred_at DESC);
CREATE INDEX idx_audit_log_resource ON audit_log(resource_type, resource_id, occurred_at DESC);
CREATE INDEX idx_audit_log_actor ON audit_log(actor_id, occurred_at DESC);
```

---

## RBAC

```sql
CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    permissions     JSONB NOT NULL,          -- array of permission objects
    -- permissions example:
    -- [
    --   {"resource": "entity", "actions": ["read", "create", "update"], "blueprint": "service"},
    --   {"resource": "scorecard", "actions": ["read"]},
    --   {"resource": "action", "actions": ["trigger"], "blueprint": "service"}
    -- ]
    is_system       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, name)
);

CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    team_id         UUID REFERENCES teams(id),   -- optional scope to team
    PRIMARY KEY (user_id, role_id, COALESCE(team_id, '00000000-0000-0000-0000-000000000000'))
);
```

---

## Example JSONB Queries

```sql
-- "Find all production services written in Go"
SELECT e.identifier, e.title, e.properties
FROM entities e
JOIN blueprints b ON b.id = e.blueprint_id
WHERE b.identifier = 'service'
  AND e.properties->>'lifecycle' = 'production'
  AND e.properties->>'language' = 'go';

-- "Find tier-1 services missing a runbook URL"
SELECT e.identifier, e.title
FROM entities e
JOIN blueprints b ON b.id = e.blueprint_id
WHERE b.identifier = 'service'
  AND e.properties->>'tier' = 'tier-1'
  AND (e.properties->>'runbook_url' IS NULL OR e.properties->>'runbook_url' = '');

-- "Navigate the dependency graph: what does payment-service depend on?"
SELECT
    target.identifier AS dependency,
    tb.identifier AS dependency_type,
    br.identifier AS relation_type
FROM entity_relations er
JOIN entities source ON source.id = er.source_entity_id
JOIN entities target ON target.id = er.target_entity_id
JOIN blueprints tb ON tb.id = target.blueprint_id
JOIN blueprint_relations br ON br.id = er.relation_id
WHERE source.identifier = 'payment-service';

-- "Aggregate scorecard scores by team"
SELECT
    t.name AS team_name,
    AVG(se.overall_score) AS avg_score,
    COUNT(*) FILTER (WHERE se.level_achieved = 'gold') AS gold_count
FROM scorecard_evaluations se
JOIN entities e ON e.id = se.entity_id
JOIN teams t ON t.id = e.owner_team_id
WHERE se.evaluated_at >= now() - INTERVAL '30 days'
GROUP BY t.name
ORDER BY avg_score DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organizations | 1 | organizations |
| Blueprints & Relations | 2 | blueprints, blueprint_relations |
| Entities & Relations | 2 | entities, entity_relations |
| Teams & Users | 3 | users, teams, team_members |
| Scorecards | 2 | scorecards, scorecard_evaluations |
| Self-Service Actions | 2 | actions, action_runs |
| DORA & Incidents | 3 | deployment_events, incidents, incident_entities |
| Integrations | 2 | integrations, identity_providers |
| AI Features | 3 | ai_doc_generations, dependency_analyses, nl_query_log |
| Audit | 1 | audit_log |
| RBAC | 2 | roles, user_roles |
| **Total** | **23** | |

---

## Key Design Decisions

1. **Blueprints as user-defined schemas** — Instead of a fixed table per entity kind, `blueprints` stores the JSON Schema that defines what properties an entity of that type can have. Adding a new entity kind (e.g. "data-pipeline" or "ML-model") requires only inserting a new blueprint row — zero DDL, zero deployment.

2. **JSONB properties with GIN indexing** — Entity properties live in a JSONB column indexed with a GIN index. This enables containment queries (`@>`) and key-existence checks. For high-cardinality properties queried frequently (like `lifecycle`), partial B-tree indexes on specific JSONB keys provide relational-speed lookups.

3. **Relational scaffolding for structural data** — `owner_team_id` is a first-class FK column on entities, not a JSONB property, because ownership queries (join to teams, filter by team) are universal and performance-critical. Similarly, `organization_id` is relational for tenant isolation.

4. **Blueprint relations define the graph schema** — `blueprint_relations` defines what kinds of entities can relate to what other kinds (e.g. "service depends_on cloud-resource"). `entity_relations` stores the actual edges. This two-level design enables validation (you can't relate a service to a scorecard unless a blueprint relation permits it) while keeping the graph navigable.

5. **Integration mapping config in JSONB** — Each integration stores a `mapping_config` that describes how to transform external data (e.g. a Git repo's metadata) into entity properties. This is the "no-code" magic — platform engineers configure mappings via UI rather than writing code.

6. **Scorecard rules in JSONB** — Rules are stored as a JSONB array within the scorecard, using a query DSL that references entity properties by path (e.g. `properties.runbook_url`). This makes rules portable across blueprints and avoids a proliferation of rule tables.

7. **Audit log captures JSONB diffs** — The `before_state` and `after_state` columns in the audit log capture the full JSONB property state, enabling point-in-time reconstruction without the complexity of full event sourcing.

8. **23 tables vs. 36 in the normalised model** — The blueprint pattern consolidates what would be separate tables (components, apis, resources, systems) into a single `entities` table with different `blueprint_id` values. This reduction in table count simplifies migrations, backups, and operational management.
