# Data Model Suggestion 4: Graph-Relational (Dependency-First)

> Project: Internal Developer Portal · Created: 2026-05-20

## Philosophy

This model places the dependency graph at the centre of the architecture. Every entity in the catalog is a node, every relationship is an edge, and the graph is stored in PostgreSQL using a dedicated `graph_nodes` and `graph_edges` pattern with `ltree` paths for efficient hierarchy traversal. Relational tables handle operational concerns (users, teams, RBAC, integrations), but the service catalog itself is a property graph — purpose-built for the dependency mapping, blast-radius analysis, and incident correlation that the project research identifies as key AI-native differentiators.

The graph-relational approach is motivated by the observation that the most valuable IDP features are fundamentally graph problems: "What is the blast radius if this database goes down?" requires traversing dependency edges. "Which services are affected by this deprecated API?" requires finding all consumers. "What is the ownership chain from this Lambda function to the VP responsible?" requires traversing team hierarchies. These queries are awkward and slow in a normalised relational model (requiring recursive CTEs or multiple joins) but natural and fast in a graph.

This model uses PostgreSQL's `ltree` extension for hierarchical paths (team and domain trees) and a property graph pattern (`graph_nodes` + `graph_edges` + JSONB properties) for the service catalog. For organisations that outgrow PostgreSQL's graph capabilities, the same node/edge schema maps directly to Neo4j or Apache AGE (PostgreSQL graph extension) without architectural changes.

**Best for:** Organisations where dependency analysis, blast-radius calculation, and relationship traversal are the primary use cases — particularly those with large microservices architectures where understanding "what connects to what" is the core problem.

**Trade-offs:**
- (+) Graph queries (shortest path, blast radius, transitive dependencies) are first-class operations
- (+) Dependency visualisation is trivial — the data is already in graph form
- (+) AI features (blast-radius analysis, incident correlation) operate directly on the graph
- (+) `ltree` enables fast subtree queries for team/domain hierarchies without recursive CTEs
- (+) Maps cleanly to dedicated graph databases (Neo4j, AGE) if PostgreSQL graph performance becomes a bottleneck
- (-) Graph queries in SQL require more complex query patterns than simple SELECT/JOIN
- (-) `ltree` extension must be installed (not available in all managed PostgreSQL offerings)
- (-) Node/edge pattern adds a level of indirection vs. directly typed tables
- (-) Graph integrity is application-enforced — no FK constraints between nodes and edges on `node_type`
- (-) Less intuitive for developers unfamiliar with graph data models

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Backstage Catalog Descriptor Format | Entity kinds become node types; Backstage relations (providesApi, consumesApi, dependsOn) become edge types |
| OpenAPI 3.1 / AsyncAPI 3.1 | API nodes carry spec metadata; consumer/provider edges link components to APIs |
| DORA Metrics Framework | Deployment events attached to component nodes; DORA aggregates computed via graph traversal |
| OpenTelemetry | Service health data stored as node properties; trace-derived dependencies create edges |
| SLSA v1.0 | Supply chain level stored as node property; build dependency edges model supply chain graph |
| ISO 3166 | Jurisdiction/region codes used in `ltree` paths for geographic hierarchy |
| CNCF Platform Engineering Maturity Model | Maturity scores stored as node properties on team nodes |
| MCP (Model Context Protocol) | Graph structure exposed as MCP resources; traversal queries exposed as MCP tools |

---

## Graph Core Tables

```sql
-- Enable ltree extension for hierarchical paths
CREATE EXTENSION IF NOT EXISTS ltree;

-- Every catalog entity is a node in the graph
CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    node_type       VARCHAR(50) NOT NULL,    -- 'component', 'api', 'resource', 'system', 'domain', 'team', 'user'
    identifier      VARCHAR(255) NOT NULL,
    title           VARCHAR(500),
    description     TEXT,
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties vary by node_type:
    --
    -- component: {
    --   "type": "service", "lifecycle": "production", "language": "go",
    --   "tier": "tier-1", "repo_url": "https://...", "framework": "gin",
    --   "k8s_namespace": "payments"
    -- }
    --
    -- api: {
    --   "spec_type": "openapi", "spec_version": "3.1.0",
    --   "lifecycle": "production", "endpoint_count": 24
    -- }
    --
    -- resource: {
    --   "type": "database", "engine": "postgresql", "version": "16",
    --   "environment": "production"
    -- }
    hierarchy_path  ltree,                   -- e.g. 'acme.platform.payments.payment_service'
    health_status   VARCHAR(20) DEFAULT 'unknown',
    scorecard_score NUMERIC(5,2),            -- denormalized latest score
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, node_type, identifier)
);

CREATE INDEX idx_nodes_org ON graph_nodes(organization_id);
CREATE INDEX idx_nodes_type ON graph_nodes(node_type);
CREATE INDEX idx_nodes_hierarchy ON graph_nodes USING gist(hierarchy_path);
CREATE INDEX idx_nodes_properties ON graph_nodes USING gin(properties);
CREATE INDEX idx_nodes_health ON graph_nodes(health_status) WHERE health_status != 'unknown';

-- Full-text search across the catalog
CREATE INDEX idx_nodes_search ON graph_nodes
    USING gin(to_tsvector('english', identifier || ' ' || COALESCE(title, '') || ' ' || COALESCE(description, '')));

-- Every relationship is an edge in the graph
CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type       VARCHAR(50) NOT NULL,    -- relationship type
    properties      JSONB NOT NULL DEFAULT '{}',
    weight          NUMERIC(5,2) DEFAULT 1.0, -- for weighted graph algorithms
    discovered_via  VARCHAR(30) DEFAULT 'manual',
                    -- 'manual', 'git-sync', 'otel-trace', 'api-spec', 'ai-inferred'
    confidence      NUMERIC(3,2) DEFAULT 1.0, -- 1.0 for explicit, lower for AI-inferred
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_node_id, target_node_id, edge_type)
);

CREATE INDEX idx_edges_source ON graph_edges(source_node_id);
CREATE INDEX idx_edges_target ON graph_edges(target_node_id);
CREATE INDEX idx_edges_type ON graph_edges(edge_type);
CREATE INDEX idx_edges_discovered ON graph_edges(discovered_via);
```

### Edge Type Taxonomy

```
-- Component relationships
provides_api        -- component -> api
consumes_api        -- component -> api
depends_on          -- component -> component | resource
part_of_system      -- component -> system

-- System relationships
belongs_to_domain   -- system -> domain

-- Ownership relationships
owned_by            -- component | api | resource | system -> team
member_of           -- user -> team
manages             -- team -> team (parent-child)

-- Infrastructure relationships
runs_on             -- component -> resource (e.g. runs on K8s cluster)
backed_by           -- component -> resource (e.g. backed by PostgreSQL)
routes_to           -- resource -> component (e.g. load balancer routes to service)

-- AI-discovered relationships
calls               -- component -> component (discovered from traces/code)
shares_data_with    -- component -> component (discovered from data flow analysis)
blocks              -- component -> component (discovered from incident correlation)
```

---

## Organizations, Teams, and Users (Relational)

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    settings        JSONB NOT NULL DEFAULT '{}',
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
    node_id         UUID REFERENCES graph_nodes(id),  -- link to graph for relationship queries
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
    hierarchy_path  ltree,                   -- e.g. 'acme.engineering.platform'
    node_id         UUID REFERENCES graph_nodes(id),  -- link to graph
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);

CREATE INDEX idx_teams_hierarchy ON teams USING gist(hierarchy_path);

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
    target_node_type VARCHAR(50) NOT NULL,   -- which node types this scorecard applies to
    rules           JSONB NOT NULL,
    levels          JSONB NOT NULL DEFAULT '["bronze", "silver", "gold"]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, identifier)
);

CREATE TABLE scorecard_evaluations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scorecard_id    UUID NOT NULL REFERENCES scorecards(id) ON DELETE CASCADE,
    node_id         UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    results         JSONB NOT NULL,
    overall_score   NUMERIC(5,2),
    level_achieved  VARCHAR(20),
    evaluated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_scorecard_evals_node ON scorecard_evaluations(node_id, evaluated_at DESC);
```

---

## DORA Metrics and Incidents

```sql
CREATE TABLE deployment_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_id         UUID NOT NULL REFERENCES graph_nodes(id),
    environment     VARCHAR(50) NOT NULL DEFAULT 'production',
    deployed_at     TIMESTAMPTZ NOT NULL,
    commit_sha      VARCHAR(64),
    deployer_id     UUID REFERENCES users(id),
    success         BOOLEAN NOT NULL DEFAULT true,
    lead_time_seconds BIGINT,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deployments_node ON deployment_events(node_id, deployed_at DESC);

CREATE TABLE incidents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    title           VARCHAR(500) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
    started_at      TIMESTAMPTZ NOT NULL,
    resolved_at     TIMESTAMPTZ,
    time_to_restore_seconds BIGINT,
    root_cause_node_id UUID REFERENCES graph_nodes(id), -- the node that caused the incident
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Affected nodes: which nodes were impacted by this incident
CREATE TABLE incident_affected_nodes (
    incident_id     UUID NOT NULL REFERENCES incidents(id) ON DELETE CASCADE,
    node_id         UUID NOT NULL REFERENCES graph_nodes(id),
    impact_type     VARCHAR(30) NOT NULL DEFAULT 'affected'
                    CHECK (impact_type IN ('root_cause', 'affected', 'degraded')),
    PRIMARY KEY (incident_id, node_id)
);
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
    target_node_type VARCHAR(50),
    input_schema    JSONB NOT NULL,
    execution       JSONB NOT NULL,
    requires_approval BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, identifier)
);

CREATE TABLE action_runs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    action_id       UUID NOT NULL REFERENCES actions(id),
    node_id         UUID REFERENCES graph_nodes(id),
    triggered_by    UUID NOT NULL REFERENCES users(id),
    inputs          JSONB NOT NULL DEFAULT '{}',
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    output          JSONB,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_action_runs_status ON action_runs(status);
```

---

## Integrations

```sql
CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    type            VARCHAR(50) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    config          JSONB NOT NULL,
    mapping_config  JSONB NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    last_sync_at    TIMESTAMPTZ,
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
-- Blast-radius analysis results stored as graph snapshots
CREATE TABLE blast_radius_analyses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    root_node_id    UUID NOT NULL REFERENCES graph_nodes(id),
    analysis_type   VARCHAR(30) NOT NULL
                    CHECK (analysis_type IN ('failure', 'deprecation', 'security', 'change')),
    model_name      VARCHAR(100) NOT NULL,
    depth           INTEGER NOT NULL,        -- how many hops were traversed
    affected_nodes  JSONB NOT NULL,          -- array of {node_id, node_type, identifier, impact_score}
    affected_edges  JSONB NOT NULL,          -- array of {edge_id, edge_type, path}
    total_blast_radius INTEGER NOT NULL,     -- count of affected nodes
    risk_score      NUMERIC(5,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_blast_radius_node ON blast_radius_analyses(root_node_id, created_at DESC);

CREATE TABLE ai_doc_generations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_id         UUID NOT NULL REFERENCES graph_nodes(id),
    model_name      VARCHAR(100) NOT NULL,
    input_context   JSONB,
    generated_doc   TEXT NOT NULL,
    confidence      NUMERIC(3,2),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    reviewed_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE nl_query_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    query_text      TEXT NOT NULL,
    generated_query TEXT,
    traversal_path  JSONB,                   -- graph traversal path taken
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
    action          VARCHAR(100) NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID,
    changes         JSONB,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org_time ON audit_log(organization_id, occurred_at DESC);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
```

---

## RBAC

```sql
CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(100) NOT NULL,
    permissions     JSONB NOT NULL,
    is_system       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, name)
);

CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    team_id         UUID REFERENCES teams(id),
    PRIMARY KEY (user_id, role_id, COALESCE(team_id, '00000000-0000-0000-0000-000000000000'))
);
```

---

## Graph Query Examples

```sql
-- "What is the blast radius if the payments-db goes down?"
-- Find all nodes transitively dependent on payments-db (BFS up to 5 hops)
WITH RECURSIVE blast_radius AS (
    -- Start from the failing node
    SELECT
        gn.id AS node_id,
        gn.node_type,
        gn.identifier,
        0 AS depth,
        ARRAY[gn.id] AS path
    FROM graph_nodes gn
    WHERE gn.identifier = 'payments-db' AND gn.node_type = 'resource'

    UNION ALL

    -- Traverse edges to find dependents
    SELECT
        gn.id,
        gn.node_type,
        gn.identifier,
        br.depth + 1,
        br.path || gn.id
    FROM blast_radius br
    JOIN graph_edges ge ON ge.target_node_id = br.node_id
        AND ge.edge_type IN ('depends_on', 'backed_by', 'consumes_api')
    JOIN graph_nodes gn ON gn.id = ge.source_node_id
    WHERE br.depth < 5
      AND NOT gn.id = ANY(br.path)  -- prevent cycles
)
SELECT node_type, identifier, depth
FROM blast_radius
WHERE depth > 0
ORDER BY depth, node_type, identifier;

-- "Which teams are affected if this API is deprecated?"
SELECT DISTINCT
    t.name AS team_name,
    gn.identifier AS service_name,
    ge.edge_type
FROM graph_nodes api_node
JOIN graph_edges ge ON ge.target_node_id = api_node.id
    AND ge.edge_type = 'consumes_api'
JOIN graph_nodes gn ON gn.id = ge.source_node_id
JOIN graph_edges owner_edge ON owner_edge.source_node_id = gn.id
    AND owner_edge.edge_type = 'owned_by'
JOIN graph_nodes team_node ON team_node.id = owner_edge.target_node_id
JOIN teams t ON t.node_id = team_node.id
WHERE api_node.identifier = 'user-auth-api';

-- "Show me all services in the platform engineering subtree"
SELECT gn.identifier, gn.properties->>'type' AS service_type
FROM graph_nodes gn
WHERE gn.hierarchy_path <@ 'acme.engineering.platform'
  AND gn.node_type = 'component';

-- "Find the shortest path between two services"
WITH RECURSIVE path_search AS (
    SELECT
        ge.target_node_id AS current_node,
        ARRAY[ge.source_node_id, ge.target_node_id] AS path,
        1 AS hops
    FROM graph_edges ge
    WHERE ge.source_node_id = :source_service_id

    UNION ALL

    SELECT
        ge.target_node_id,
        ps.path || ge.target_node_id,
        ps.hops + 1
    FROM path_search ps
    JOIN graph_edges ge ON ge.source_node_id = ps.current_node
    WHERE NOT ge.target_node_id = ANY(ps.path)
      AND ps.hops < 10
)
SELECT path, hops
FROM path_search
WHERE current_node = :target_service_id
ORDER BY hops
LIMIT 1;

-- "Incident correlation: which recent deployments preceded this outage?"
SELECT
    gn.identifier AS service,
    de.deployed_at,
    de.commit_sha,
    de.success
FROM incident_affected_nodes ian
JOIN graph_nodes gn ON gn.id = ian.node_id
JOIN deployment_events de ON de.node_id = gn.id
    AND de.deployed_at BETWEEN :incident_started_at - INTERVAL '4 hours'
                          AND :incident_started_at
ORDER BY de.deployed_at DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organizations | 1 | organizations |
| Graph Core | 2 | graph_nodes, graph_edges |
| Teams & Users | 3 | users, teams, team_members |
| Scorecards | 2 | scorecards, scorecard_evaluations |
| DORA & Incidents | 3 | deployment_events, incidents, incident_affected_nodes |
| Self-Service Actions | 2 | actions, action_runs |
| Integrations | 2 | integrations, identity_providers |
| AI Features | 3 | blast_radius_analyses, ai_doc_generations, nl_query_log |
| Audit | 1 | audit_log |
| RBAC | 2 | roles, user_roles |
| **Total** | **21** | |

---

## Key Design Decisions

1. **Property graph in PostgreSQL** — `graph_nodes` and `graph_edges` with JSONB properties implement a property graph model within PostgreSQL. This avoids introducing a separate graph database while enabling graph traversal via recursive CTEs. If graph performance becomes a bottleneck, the schema maps directly to Neo4j (nodes become Neo4j nodes, edges become relationships, JSONB properties become Neo4j properties).

2. **`ltree` for hierarchical traversal** — Teams and catalog entities carry `hierarchy_path` columns using PostgreSQL's `ltree` extension. Subtree queries like "all services under platform engineering" use the `<@` operator with GiST indexes, which is dramatically faster than recursive CTEs for hierarchy-only queries.

3. **Edge provenance tracking** — Each edge records `discovered_via` (manual, git-sync, otel-trace, api-spec, ai-inferred) and `confidence` (0.0-1.0). AI-inferred dependencies from trace analysis or code scanning carry lower confidence, enabling the UI to distinguish confirmed from inferred relationships. This is critical for the "intelligent dependency analysis" feature.

4. **Blast-radius analysis as a first-class entity** — The `blast_radius_analyses` table caches the results of expensive graph traversals, storing the affected node set and risk score. This enables dashboards to show blast radius without real-time graph computation and provides training data for AI models.

5. **Dual identity: relational + graph** — Teams and users exist both as relational rows (for auth, RBAC, membership) and as graph nodes (for ownership traversal and relationship queries). The `node_id` FK links the two representations. This avoids forcing all operational queries through the graph while keeping relationship queries fast.

6. **Weighted edges for risk scoring** — Edges carry a `weight` column that influences graph algorithms. A tier-1 service depending on a resource has a higher weight than a tier-4 service. AI models can learn optimal weights from incident history.

7. **Edge types model Backstage relations** — The edge type taxonomy (`provides_api`, `consumes_api`, `depends_on`, `part_of_system`, `owned_by`) directly maps to Backstage's relation model, ensuring compatibility with `catalog-info.yaml` imports. Additional edge types (`calls`, `shares_data_with`, `blocks`) extend the model for AI-discovered relationships.

8. **21 tables — the most compact model** — By consolidating all catalog entities into `graph_nodes` and all relationships into `graph_edges`, the graph model has the fewest tables of all four suggestions while supporting the richest relationship queries.
