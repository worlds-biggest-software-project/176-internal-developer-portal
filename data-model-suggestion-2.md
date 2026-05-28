# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Internal Developer Portal · Created: 2026-05-20

## Philosophy

This model treats every change to the service catalog as an immutable event recorded in an append-only event store. The current state of any entity is derived by replaying its event stream. Read-optimised materialised views (projections) serve the portal UI and API queries, following the CQRS (Command Query Responsibility Segregation) pattern where the write path (event store) and read path (projections) are architecturally separate.

Event sourcing is the natural fit for an IDP that promises full audit trails, temporal queries ("what did service X look like on March 15th?"), and AI-powered change pattern analysis. Every ownership transfer, scorecard evaluation, deployment, and configuration change is captured as a first-class event with actor, timestamp, and payload. This creates the richest possible dataset for training AI models on engineering patterns — the adaptive production-readiness scoring and incident correlation features identified in the project research become straightforward to implement when the complete history of every entity is available.

The CQRS separation also enables independent scaling: the event store handles high-volume writes from CI/CD pipelines, Git webhooks, and observability collectors, while the read projections are optimised for the specific query patterns of dashboards, scorecards, and natural-language search.

**Best for:** Organisations that require complete audit trails, need temporal queries across catalog history, and want to leverage AI/ML on the full change history of their service landscape.

**Trade-offs:**
- (+) Complete, immutable audit trail — every change is preserved forever
- (+) Temporal queries are trivial — replay events to any point in time
- (+) Rich dataset for AI/ML — change patterns, ownership history, incident correlations
- (+) Independent scaling of reads and writes via CQRS
- (+) New read models can be added without changing the write path
- (-) Higher complexity — eventual consistency between event store and projections
- (-) Projection rebuild can be slow for large event streams (requires snapshotting)
- (-) Debugging requires understanding event replay, not just current state
- (-) Storage grows indefinitely (event compaction strategies needed)
- (-) Developers unfamiliar with event sourcing face a steeper learning curve

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Backstage Catalog Descriptor Format | Entity kinds map to event stream categories (component.*, api.*, system.*) |
| OpenAPI 3.1 / AsyncAPI 3.1 | API spec changes recorded as `api.spec_updated` events |
| DORA Metrics Framework | Deployment and incident events are first-class event types; DORA metrics computed as projections |
| SLSA v1.0 | Supply chain assessments recorded as `component.slsa_assessed` events |
| OpenTelemetry | Health metric events ingested from OTel-compatible sources |
| CloudEvents v1.0 | Event envelope format follows CloudEvents specification for interoperability |
| OAuth 2.0 / OIDC | Actor identity in events uses OIDC subject identifiers |
| SCIM 2.0 | User/group sync events follow SCIM resource change patterns |

---

## Event Store

```sql
-- The single source of truth: an append-only event log
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,           -- aggregate/entity identifier
    stream_type     VARCHAR(50) NOT NULL,     -- 'component', 'api', 'team', 'scorecard', etc.
    event_type      VARCHAR(100) NOT NULL,    -- 'component.created', 'component.ownership_transferred', etc.
    event_version   INTEGER NOT NULL,         -- version within the stream (optimistic concurrency)
    organization_id UUID NOT NULL,
    actor_id        UUID,                    -- user who triggered the event (null for system events)
    actor_type      VARCHAR(20) NOT NULL DEFAULT 'user'
                    CHECK (actor_type IN ('user', 'system', 'integration', 'ai-agent')),
    payload         JSONB NOT NULL,          -- event-specific data
    metadata        JSONB,                   -- correlation IDs, source integration, request context
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, event_version)        -- optimistic concurrency control
);

-- Primary query: load all events for a stream in order
CREATE INDEX idx_events_stream ON events(stream_id, event_version);

-- Query by type across all streams
CREATE INDEX idx_events_type ON events(event_type, occurred_at DESC);

-- Query by organisation and time
CREATE INDEX idx_events_org_time ON events(organization_id, occurred_at DESC);

-- Query by actor
CREATE INDEX idx_events_actor ON events(actor_id, occurred_at DESC);

-- Partition by month for storage management
-- In production, this table would use declarative partitioning:
-- CREATE TABLE events (...) PARTITION BY RANGE (occurred_at);
```

### Event Type Taxonomy

```
-- Catalog lifecycle events
component.created
component.updated
component.ownership_transferred
component.lifecycle_changed       -- experimental -> production -> deprecated
component.deleted

api.created
api.updated
api.spec_uploaded
api.consumer_added
api.consumer_removed

system.created
system.updated
system.component_added
system.component_removed

resource.created
resource.updated
resource.dependency_added

-- Team and user events
team.created
team.member_added
team.member_removed
team.member_role_changed

user.created
user.deactivated
user.role_assigned

-- Scorecard events
scorecard.created
scorecard.rule_added
scorecard.rule_updated
scorecard.evaluated                -- payload contains pass/fail per rule

-- Deployment and incident events (DORA source data)
deployment.completed
deployment.failed
deployment.rolled_back

incident.opened
incident.escalated
incident.resolved
incident.postmortem_attached

-- Integration events
integration.connected
integration.sync_completed
integration.sync_failed
integration.disconnected

-- AI events
ai.doc_generated
ai.doc_accepted
ai.doc_rejected
ai.dependency_analysis_completed
ai.nl_query_executed

-- Self-service action events
action.triggered
action.approved
action.rejected
action.completed
action.failed
```

### Example Event Payloads

```sql
-- component.created event
-- payload: {
--   "name": "payment-service",
--   "slug": "payment-service",
--   "type": "service",
--   "lifecycle": "production",
--   "owner_team_id": "550e8400-e29b-41d4-a716-446655440000",
--   "repo_url": "https://github.com/acme/payment-service",
--   "language": "go",
--   "tier": "tier-1"
-- }

-- component.ownership_transferred event
-- payload: {
--   "previous_owner_team_id": "550e8400-e29b-41d4-a716-446655440000",
--   "new_owner_team_id": "660e8400-e29b-41d4-a716-446655440001",
--   "reason": "Team restructuring Q2 2026"
-- }

-- scorecard.evaluated event
-- payload: {
--   "scorecard_id": "770e8400-e29b-41d4-a716-446655440002",
--   "overall_score": 0.85,
--   "rules": [
--     {"rule_id": "...", "name": "Has runbook", "passed": true},
--     {"rule_id": "...", "name": "P95 latency < 200ms", "passed": false, "actual": 342}
--   ]
-- }
```

---

## Snapshot Store (Performance Optimisation)

```sql
-- Periodic snapshots to avoid replaying long event streams
CREATE TABLE snapshots (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,
    snapshot_version INTEGER NOT NULL,       -- event_version this snapshot is current to
    state           JSONB NOT NULL,          -- full current state of the aggregate
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);

-- To load current state:
-- 1. Load latest snapshot for stream_id
-- 2. Replay events with event_version > snapshot_version
-- 3. Apply each event to rebuild current state
```

---

## Read Projections (Materialised Views)

These tables are derived from the event store. They can be rebuilt from scratch by replaying all events.

### Catalog Projection

```sql
CREATE TABLE catalog_components_view (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    type            VARCHAR(50) NOT NULL,
    lifecycle       VARCHAR(30) NOT NULL,
    owner_team_id   UUID NOT NULL,
    owner_team_name VARCHAR(255),
    system_id       UUID,
    system_name     VARCHAR(255),
    domain_id       UUID,
    repo_url        TEXT,
    language        VARCHAR(50),
    framework       VARCHAR(100),
    tier            VARCHAR(20),
    tag_list        TEXT[],
    last_deployment_at TIMESTAMPTZ,
    latest_scorecard_score NUMERIC(5,2),
    health_status   VARCHAR(20),
    event_version   INTEGER NOT NULL,       -- tracks which event this projection is current to
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);

CREATE INDEX idx_catalog_components_owner ON catalog_components_view(owner_team_id);
CREATE INDEX idx_catalog_components_type ON catalog_components_view(type);
CREATE INDEX idx_catalog_components_lifecycle ON catalog_components_view(lifecycle);
-- Full-text search across catalog
CREATE INDEX idx_catalog_components_search ON catalog_components_view
    USING gin(to_tsvector('english', name || ' ' || COALESCE(description, '')));

CREATE TABLE catalog_apis_view (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    type            VARCHAR(30) NOT NULL,
    lifecycle       VARCHAR(30) NOT NULL,
    owner_team_id   UUID NOT NULL,
    owner_team_name VARCHAR(255),
    system_id       UUID,
    spec_version    VARCHAR(50),
    endpoint_count  INTEGER,
    consumer_count  INTEGER,
    event_version   INTEGER NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);

CREATE TABLE catalog_teams_view (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    member_count    INTEGER NOT NULL DEFAULT 0,
    component_count INTEGER NOT NULL DEFAULT 0,
    parent_team_id  UUID,
    event_version   INTEGER NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### DORA Metrics Projection

```sql
CREATE TABLE dora_metrics_view (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    component_id    UUID NOT NULL,
    team_id         UUID,
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    deployment_frequency  NUMERIC(10,4),
    lead_time_seconds     BIGINT,
    mttr_seconds          BIGINT,
    change_failure_rate   NUMERIC(5,4),
    dora_level            VARCHAR(20),
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dora_view_component ON dora_metrics_view(component_id, period_start);
```

### Scorecard Projection

```sql
CREATE TABLE scorecard_results_view (
    component_id    UUID NOT NULL,
    scorecard_id    UUID NOT NULL,
    scorecard_name  VARCHAR(255),
    overall_score   NUMERIC(5,2),
    rules_passed    INTEGER,
    rules_total     INTEGER,
    last_evaluated  TIMESTAMPTZ,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (component_id, scorecard_id)
);

CREATE INDEX idx_scorecard_results_score ON scorecard_results_view(overall_score);
```

### Dependency Graph Projection

```sql
CREATE TABLE dependency_edges_view (
    source_type     VARCHAR(30) NOT NULL,
    source_id       UUID NOT NULL,
    target_type     VARCHAR(30) NOT NULL,
    target_id       UUID NOT NULL,
    relation        VARCHAR(50) NOT NULL,    -- 'provides', 'consumes', 'depends_on', 'owns'
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (source_type, source_id, target_type, target_id, relation)
);

CREATE INDEX idx_dependency_target ON dependency_edges_view(target_type, target_id);
```

### Ownership History Projection

```sql
-- Unique to event-sourced model: complete ownership history derived from events
CREATE TABLE ownership_history_view (
    entity_type     VARCHAR(30) NOT NULL,
    entity_id       UUID NOT NULL,
    owner_team_id   UUID NOT NULL,
    owned_from      TIMESTAMPTZ NOT NULL,
    owned_until     TIMESTAMPTZ,             -- null = current owner
    transferred_by  UUID,
    reason          TEXT,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ownership_history_entity ON ownership_history_view(entity_type, entity_id, owned_from);
CREATE INDEX idx_ownership_history_team ON ownership_history_view(owner_team_id);
```

---

## Command Processing

```sql
-- Idempotency: prevent duplicate command processing
CREATE TABLE processed_commands (
    command_id      UUID PRIMARY KEY,
    command_type    VARCHAR(100) NOT NULL,
    processed_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Projection checkpoints: track which event each projection has processed
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Temporal Query Examples

```sql
-- "What did payment-service look like on March 15th, 2026?"
-- Replay events up to that date:
SELECT * FROM events
WHERE stream_id = :component_id
  AND occurred_at <= '2026-03-15T23:59:59Z'
ORDER BY event_version ASC;

-- "Who owned this service over the past year?"
SELECT * FROM ownership_history_view
WHERE entity_type = 'component' AND entity_id = :component_id
  AND owned_from >= now() - INTERVAL '1 year'
ORDER BY owned_from;

-- "What changed in the last 24 hours across all components?"
SELECT e.event_type, e.stream_id, e.payload, e.occurred_at,
       c.name AS component_name
FROM events e
JOIN catalog_components_view c ON c.id = e.stream_id
WHERE e.stream_type = 'component'
  AND e.occurred_at >= now() - INTERVAL '24 hours'
ORDER BY e.occurred_at DESC;

-- "Show me the deployment-to-incident correlation for tier-1 services"
SELECT
    d.stream_id AS component_id,
    d.payload->>'commit_sha' AS deploy_commit,
    d.occurred_at AS deployed_at,
    i.occurred_at AS incident_at,
    i.payload->>'severity' AS severity,
    EXTRACT(EPOCH FROM i.occurred_at - d.occurred_at) AS seconds_to_incident
FROM events d
JOIN events i ON i.stream_type = 'incident'
    AND i.event_type = 'incident.opened'
    AND i.occurred_at BETWEEN d.occurred_at AND d.occurred_at + INTERVAL '4 hours'
WHERE d.event_type = 'deployment.completed'
  AND d.occurred_at >= now() - INTERVAL '90 days'
ORDER BY d.occurred_at DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | events (partitioned by month) |
| Snapshots | 1 | snapshots |
| Catalog Projections | 3 | components_view, apis_view, teams_view |
| DORA Projection | 1 | dora_metrics_view |
| Scorecard Projection | 1 | scorecard_results_view |
| Dependency Projection | 1 | dependency_edges_view |
| Ownership Projection | 1 | ownership_history_view |
| Command Infrastructure | 2 | processed_commands, projection_checkpoints |
| **Total** | **11** | Plus the event store which logically contains all data |

---

## Key Design Decisions

1. **Single event store table as source of truth** — All domain data flows through one append-only table. This is architecturally simpler than distributed event stores and leverages PostgreSQL's ACID guarantees. The `events` table replaces 20+ tables from the normalised model — all entity state is encoded in event payloads.

2. **CloudEvents-inspired envelope** — Each event carries `event_type`, `stream_id`, `actor_id`, `metadata`, and `occurred_at`, aligning with the CloudEvents v1.0 specification. This makes events portable to external event buses (Kafka, EventBridge) if needed.

3. **Optimistic concurrency via stream versioning** — The `UNIQUE(stream_id, event_version)` constraint prevents conflicting writes to the same entity. Commands must read the current version and specify the expected next version, preventing lost updates without pessimistic locking.

4. **Projections are disposable** — Every `*_view` table can be dropped and rebuilt by replaying the event store. This enables adding entirely new read models (e.g. a cost-per-team view) without any write-path changes.

5. **Snapshotting for performance** — For entities with thousands of events (e.g. a frequently-deployed service), periodic snapshots avoid replaying the full stream. The application loads the latest snapshot and applies only subsequent events.

6. **Event-driven DORA metrics** — Deployment and incident events are first-class event types, not separate tables. DORA metrics are computed as projections, enabling recalculation with different time windows or aggregation strategies without modifying the source data.

7. **Ownership history as a natural byproduct** — In a normalised model, tracking ownership changes requires a separate audit table. In this model, `ownership_history_view` is simply a projection of `component.ownership_transferred` events — the history exists automatically because events are never deleted.

8. **AI training dataset built in** — The complete event history provides a rich training corpus for adaptive production-readiness scoring. Models can correlate sequences of events (deployment followed by incident) to learn risk patterns that static scorecards miss.
