# Internal Developer Portal — Phased Development Plan

> Project: 176-internal-developer-portal · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | TypeScript | Full-stack unification (backend + frontend in one language), strong type system for a complex domain model, aligns with Backstage ecosystem conventions (React + Node), and excellent LLM SDK support via the Anthropic and OpenAI TypeScript SDKs |
| API framework | Fastify (Node.js) | High-performance HTTP server with native OpenAPI 3.1 schema generation via `@fastify/swagger`, JSON Schema-based request/response validation, first-class TypeScript support, and plugin architecture that mirrors the IDP's own extensibility goals |
| Frontend | Next.js 15 (App Router) + React 19 | Server components reduce bundle size for a data-heavy dashboard, App Router enables streaming for large catalog views, and React 19's use-based data fetching simplifies concurrent scorecard/metrics loading. The Backstage ecosystem is React-based, so sharing UI component patterns is natural |
| Database | PostgreSQL 16 | Required by the hybrid relational + JSONB blueprint data model (Suggestion 3), supports GIN indexes for JSONB property queries, `ltree` extension for hierarchy paths, full-text search via `tsvector`, and mature operational tooling for self-hosted deployments |
| ORM / query builder | Drizzle ORM | Type-safe SQL with zero runtime overhead, native PostgreSQL JSONB support, migration generation, and a thinner abstraction layer than Prisma — important when writing complex JSONB and recursive CTE queries for dependency graphs |
| Task queue | BullMQ (Redis-backed) | Handles async workloads: catalog sync from Git, scorecard evaluation batches, AI doc generation, webhook delivery. Redis also serves as the caching layer for catalog queries and session storage |
| Cache | Redis 7 | Shared infrastructure with BullMQ; caches catalog projections, scorecard results, and DORA metric aggregates |
| Search | PostgreSQL full-text search (tsvector) + pg_trgm | Avoids introducing Elasticsearch for MVP; `tsvector` indexes on catalog entities provide full-text search; `pg_trgm` provides fuzzy matching for the NL query interface. Can migrate to a dedicated search engine post-MVP if scale demands it |
| LLM provider | Anthropic Claude API (primary), OpenAI-compatible fallback | Claude for documentation generation, dependency analysis, and NL query translation. Abstracted behind a provider interface to support self-hosted LLM deployments (Ollama, vLLM) |
| Authentication | Passport.js + `openid-client` | Implements OAuth 2.0 / OIDC (RFC 6749, RFC 7636 PKCE) for SSO with enterprise IdPs (Okta, Azure AD, Google). Passport's strategy pattern supports multiple IdPs per deployment. SCIM 2.0 (RFC 7644) provisioning via a dedicated endpoint |
| Containerisation | Docker + Docker Compose | Self-hosted deployment model; multi-stage build for production image. Docker Compose for local development with PostgreSQL + Redis |
| Testing | Vitest (unit/integration) + Playwright (E2E) | Vitest is the fastest TypeScript test runner with native ESM support. Playwright for browser E2E tests of the portal UI. `testcontainers` for integration tests against real PostgreSQL/Redis |
| Code quality | ESLint + Prettier + `typescript-eslint` | Standard TypeScript toolchain. Strict TypeScript (`strict: true`, `noUncheckedIndexedAccess: true`) |
| Package manager | pnpm | Workspace support for monorepo (API + frontend + shared types), faster installs than npm, strict dependency resolution |
| API documentation | OpenAPI 3.1 auto-generated from Fastify schemas | Backstage and all major IDPs expose OpenAPI specs; auto-generation ensures the spec stays in sync with the implementation |
| Monorepo structure | pnpm workspaces + Turborepo | Shared TypeScript types between API and frontend, parallel builds, incremental compilation |

### Data Model Decision

**Selected: Suggestion 3 — Hybrid Relational + JSONB (Blueprint Model)** with elements from Suggestion 4 (Graph-Relational).

Rationale: The blueprint model mirrors Port's proven architecture ($800M valuation), enabling platform engineers to define custom entity types without DDL changes. This delivers the "fast time to value" positioning from the README while supporting multi-tenant flexibility. Graph traversal for dependency analysis and blast-radius queries will use PostgreSQL recursive CTEs over the `entity_relations` table, with `ltree` paths (from Suggestion 4) added to team and domain hierarchies for efficient subtree queries. The audit log from Suggestion 3 provides change tracking without the operational complexity of full event sourcing (Suggestion 2).

### Project Structure

```
internal-developer-portal/
├── package.json
├── pnpm-workspace.yaml
├── turbo.json
├── docker-compose.yml
├── Dockerfile
├── .env.example
├── packages/
│   ├── shared/                          # Shared TypeScript types and utilities
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── types/
│   │       │   ├── catalog.ts           # Blueprint, Entity, Relation types
│   │       │   ├── scorecard.ts         # Scorecard, Rule, Evaluation types
│   │       │   ├── dora.ts              # DORA metric types
│   │       │   ├── actions.ts           # Self-service action types
│   │       │   ├── auth.ts              # User, Team, Role types
│   │       │   └── ai.ts               # AI feature types
│   │       ├── constants/
│   │       │   ├── entity-kinds.ts      # Default blueprint identifiers
│   │       │   └── edge-types.ts        # Relationship type constants
│   │       └── schemas/
│   │           ├── catalog-info.ts      # Backstage YAML descriptor schema
│   │           └── blueprint-schema.ts  # Blueprint JSON Schema validator
│   ├── api/                             # Fastify API server
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── server.ts                # Fastify app bootstrap
│   │       ├── config.ts                # Environment-based configuration
│   │       ├── db/
│   │       │   ├── schema.ts            # Drizzle schema definitions
│   │       │   ├── migrate.ts           # Migration runner
│   │       │   └── migrations/          # SQL migration files
│   │       ├── routes/
│   │       │   ├── blueprints.ts        # Blueprint CRUD
│   │       │   ├── entities.ts          # Entity CRUD + search
│   │       │   ├── relations.ts         # Entity relationship management
│   │       │   ├── scorecards.ts        # Scorecard CRUD + evaluation
│   │       │   ├── actions.ts           # Self-service action management
│   │       │   ├── deployments.ts       # Deployment event ingestion
│   │       │   ├── incidents.ts         # Incident management
│   │       │   ├── dora.ts              # DORA metrics API
│   │       │   ├── integrations.ts      # Integration management
│   │       │   ├── teams.ts             # Team/user management
│   │       │   ├── auth.ts              # Authentication endpoints
│   │       │   ├── ai.ts                # AI feature endpoints
│   │       │   └── health.ts            # Health check
│   │       ├── services/
│   │       │   ├── catalog.service.ts   # Catalog business logic
│   │       │   ├── blueprint.service.ts # Blueprint validation and management
│   │       │   ├── scorecard.service.ts # Scorecard evaluation engine
│   │       │   ├── dora.service.ts      # DORA metric calculation
│   │       │   ├── graph.service.ts     # Dependency graph traversal
│   │       │   ├── sync.service.ts      # Git/integration sync engine
│   │       │   ├── action.service.ts    # Action execution engine
│   │       │   └── ai/
│   │       │       ├── doc-generator.ts # AI documentation generation
│   │       │       ├── dependency-analyzer.ts  # Blast-radius analysis
│   │       │       ├── nl-query.ts      # Natural language query engine
│   │       │       └── llm-provider.ts  # LLM abstraction layer
│   │       ├── plugins/
│   │       │   ├── auth.plugin.ts       # Passport.js authentication plugin
│   │       │   ├── rbac.plugin.ts       # Role-based access control
│   │       │   └── audit.plugin.ts      # Audit log middleware
│   │       ├── workers/
│   │       │   ├── sync.worker.ts       # Git/integration sync worker
│   │       │   ├── scorecard.worker.ts  # Batch scorecard evaluation
│   │       │   ├── dora.worker.ts       # DORA metric aggregation
│   │       │   └── ai.worker.ts         # AI task processor
│   │       └── integrations/
│   │           ├── github.ts            # GitHub integration
│   │           ├── gitlab.ts            # GitLab integration
│   │           ├── kubernetes.ts        # Kubernetes integration
│   │           └── pagerduty.ts         # PagerDuty integration
│   └── web/                             # Next.js frontend
│       ├── package.json
│       ├── tsconfig.json
│       ├── next.config.ts
│       └── src/
│           ├── app/
│           │   ├── layout.tsx
│           │   ├── page.tsx             # Dashboard
│           │   ├── catalog/
│           │   │   ├── page.tsx          # Catalog list/search
│           │   │   └── [entityId]/
│           │   │       └── page.tsx      # Entity detail
│           │   ├── scorecards/
│           │   ├── actions/
│           │   ├── metrics/
│           │   ├── settings/
│           │   └── api/                 # Next.js API routes (BFF)
│           ├── components/
│           │   ├── catalog/
│           │   ├── scorecards/
│           │   ├── graphs/
│           │   ├── metrics/
│           │   └── common/
│           └── lib/
│               ├── api-client.ts
│               └── hooks/
└── tests/
    ├── fixtures/
    │   ├── catalog-info.yaml            # Sample Backstage descriptors
    │   ├── openapi-spec.yaml            # Sample OpenAPI spec
    │   └── blueprints/                  # Sample blueprint definitions
    ├── unit/
    ├── integration/
    └── e2e/
```

---

## Phase 1: Foundation and Core Schema

### Purpose
Establish the project skeleton, database schema, configuration system, and core type definitions. After this phase, the application boots, connects to PostgreSQL and Redis, runs migrations, and exposes a health check endpoint. All subsequent phases build on this foundation without restructuring.

### Tasks

#### 1.1 — Monorepo Setup and Configuration

**What**: Initialize the pnpm workspace monorepo with shared types, API, and web packages, plus Docker Compose for local development.

**Design**:

```typescript
// packages/api/src/config.ts
import { z } from 'zod';

export const configSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  PORT: z.coerce.number().default(3001),
  HOST: z.string().default('0.0.0.0'),

  // Database
  DATABASE_URL: z.string().url(),
  DATABASE_POOL_MIN: z.coerce.number().default(2),
  DATABASE_POOL_MAX: z.coerce.number().default(10),

  // Redis
  REDIS_URL: z.string().url().default('redis://localhost:6379'),

  // Auth
  OIDC_ISSUER_URL: z.string().url().optional(),
  OIDC_CLIENT_ID: z.string().optional(),
  OIDC_CLIENT_SECRET: z.string().optional(),
  SESSION_SECRET: z.string().min(32),

  // AI
  ANTHROPIC_API_KEY: z.string().optional(),
  OPENAI_API_KEY: z.string().optional(),
  AI_PROVIDER: z.enum(['anthropic', 'openai', 'ollama']).default('anthropic'),
  AI_MODEL: z.string().default('claude-sonnet-4-20250514'),

  // Feature flags
  ENABLE_AI_FEATURES: z.coerce.boolean().default(false),
  ENABLE_SCIM_PROVISIONING: z.coerce.boolean().default(false),
});

export type Config = z.infer<typeof configSchema>;
export const config = configSchema.parse(process.env);
```

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: idp
      POSTGRES_USER: idp
      POSTGRES_PASSWORD: idp_dev_password
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  pgdata:
```

```json
// pnpm-workspace.yaml
{
  "packages": ["packages/*"]
}
```

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "dev": { "cache": false, "persistent": true },
    "test": { "dependsOn": ["build"] },
    "lint": {},
    "typecheck": { "dependsOn": ["^build"] }
  }
}
```

**Testing**:
- `Unit: configSchema.parse with valid env → Config object with correct types`
- `Unit: configSchema.parse with missing DATABASE_URL → ZodError with path ["DATABASE_URL"]`
- `Unit: configSchema.parse with defaults only → PORT=3001, NODE_ENV=development`
- `Integration: docker-compose up → PostgreSQL accepts connections on port 5432`
- `Integration: docker-compose up → Redis accepts connections on port 6379`
- `Unit: pnpm build → all three packages compile without errors`

#### 1.2 — Database Schema and Migrations

**What**: Implement the hybrid relational + JSONB blueprint data model (Suggestion 3) with `ltree` support, using Drizzle ORM schema definitions and SQL migrations.

**Design**:

```typescript
// packages/api/src/db/schema.ts
import { pgTable, uuid, varchar, text, timestamp, jsonb, boolean,
         numeric, integer, index, uniqueIndex, primaryKey } from 'drizzle-orm/pg-core';

// --- Organizations ---
export const organizations = pgTable('organizations', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 100 }).notNull().unique(),
  settings: jsonb('settings').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

// --- Blueprints ---
export const blueprints = pgTable('blueprints', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id),
  identifier: varchar('identifier', { length: 100 }).notNull(),
  title: varchar('title', { length: 255 }).notNull(),
  description: text('description'),
  icon: varchar('icon', { length: 50 }),
  color: varchar('color', { length: 7 }),
  schema: jsonb('schema').notNull(),           // JSON Schema for properties
  calculationProperties: jsonb('calculation_properties'),
  mirrorProperties: jsonb('mirror_properties'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('uq_blueprints_org_identifier').on(table.organizationId, table.identifier),
]);

// --- Blueprint Relations ---
export const blueprintRelations = pgTable('blueprint_relations', {
  id: uuid('id').primaryKey().defaultRandom(),
  sourceBlueprintId: uuid('source_blueprint_id').notNull().references(() => blueprints.id, { onDelete: 'cascade' }),
  targetBlueprintId: uuid('target_blueprint_id').notNull().references(() => blueprints.id, { onDelete: 'cascade' }),
  identifier: varchar('identifier', { length: 100 }).notNull(),
  title: varchar('title', { length: 255 }).notNull(),
  description: text('description'),
  cardinality: varchar('cardinality', { length: 20 }).notNull().default('many'),
  required: boolean('required').notNull().default(false),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('uq_bp_relations_source_identifier').on(table.sourceBlueprintId, table.identifier),
]);

// --- Entities ---
export const entities = pgTable('entities', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id),
  blueprintId: uuid('blueprint_id').notNull().references(() => blueprints.id),
  identifier: varchar('identifier', { length: 255 }).notNull(),
  title: varchar('title', { length: 500 }),
  description: text('description'),
  properties: jsonb('properties').notNull().default({}),
  ownerTeamId: uuid('owner_team_id'),
  createdBy: uuid('created_by'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('uq_entities_org_bp_identifier').on(table.organizationId, table.blueprintId, table.identifier),
  index('idx_entities_blueprint').on(table.blueprintId),
  index('idx_entities_owner').on(table.ownerTeamId),
]);

// --- Entity Relations ---
export const entityRelations = pgTable('entity_relations', {
  id: uuid('id').primaryKey().defaultRandom(),
  sourceEntityId: uuid('source_entity_id').notNull().references(() => entities.id, { onDelete: 'cascade' }),
  targetEntityId: uuid('target_entity_id').notNull().references(() => entities.id, { onDelete: 'cascade' }),
  relationId: uuid('relation_id').notNull().references(() => blueprintRelations.id),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('uq_entity_relations').on(table.sourceEntityId, table.targetEntityId, table.relationId),
  index('idx_entity_relations_source').on(table.sourceEntityId),
  index('idx_entity_relations_target').on(table.targetEntityId),
]);

// --- Users ---
export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id),
  email: varchar('email', { length: 320 }).notNull(),
  displayName: varchar('display_name', { length: 255 }).notNull(),
  avatarUrl: text('avatar_url'),
  status: varchar('status', { length: 20 }).notNull().default('active'),
  profile: jsonb('profile').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('uq_users_org_email').on(table.organizationId, table.email),
]);

// --- Teams ---
export const teams = pgTable('teams', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 100 }).notNull(),
  description: text('description'),
  parentTeamId: uuid('parent_team_id'),
  hierarchyPath: text('hierarchy_path'),       // ltree stored as text, cast in queries
  properties: jsonb('properties').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('uq_teams_org_slug').on(table.organizationId, table.slug),
]);

// --- Team Members ---
export const teamMembers = pgTable('team_members', {
  teamId: uuid('team_id').notNull().references(() => teams.id, { onDelete: 'cascade' }),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  role: varchar('role', { length: 50 }).notNull().default('member'),
  joinedAt: timestamp('joined_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  primaryKey({ columns: [table.teamId, table.userId] }),
]);

// --- Audit Log ---
export const auditLog = pgTable('audit_log', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull(),
  actorId: uuid('actor_id'),
  actorType: varchar('actor_type', { length: 20 }).notNull().default('user'),
  action: varchar('action', { length: 100 }).notNull(),
  resourceType: varchar('resource_type', { length: 50 }).notNull(),
  resourceId: uuid('resource_id'),
  beforeState: jsonb('before_state'),
  afterState: jsonb('after_state'),
  metadata: jsonb('metadata'),
  occurredAt: timestamp('occurred_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_audit_org_time').on(table.organizationId, table.occurredAt),
  index('idx_audit_resource').on(table.resourceType, table.resourceId),
]);
```

SQL migration for extensions and GIN indexes (Drizzle custom SQL):

```sql
-- 0001_init.sql
CREATE EXTENSION IF NOT EXISTS ltree;
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- GIN index for JSONB property queries on entities
CREATE INDEX idx_entities_properties ON entities USING gin(properties);

-- Full-text search index on entities
CREATE INDEX idx_entities_search ON entities
    USING gin(to_tsvector('english', identifier || ' ' || COALESCE(title, '') || ' ' || COALESCE(description, '')));

-- ltree GiST index on teams
CREATE INDEX idx_teams_hierarchy ON teams USING gist(hierarchy_path::ltree);
```

**Testing**:
- `Unit: Drizzle schema compiles without type errors`
- `Integration (testcontainers): migrate.ts → all tables created in PostgreSQL 16`
- `Integration: INSERT into blueprints with valid schema → row created with UUID`
- `Integration: INSERT into entities with FK to blueprint → row created`
- `Integration: INSERT into entities violating unique constraint → error`
- `Integration: JSONB GIN index → containment query (@>) uses index (EXPLAIN ANALYZE)`
- `Integration: ltree extension → teams.hierarchy_path supports <@ operator`

#### 1.3 — Fastify Server Bootstrap and Health Check

**What**: Create the Fastify application with plugin registration, request logging, error handling (RFC 7807), OpenAPI spec generation, and a health check endpoint.

**Design**:

```typescript
// packages/api/src/server.ts
import Fastify from 'fastify';
import swagger from '@fastify/swagger';
import swaggerUi from '@fastify/swagger-ui';
import cors from '@fastify/cors';
import { config } from './config.js';
import { db } from './db/connection.js';

export async function buildApp() {
  const app = Fastify({
    logger: {
      level: config.NODE_ENV === 'production' ? 'info' : 'debug',
    },
    genReqId: () => crypto.randomUUID(),
  });

  // OpenAPI 3.1 spec generation
  await app.register(swagger, {
    openapi: {
      openapi: '3.1.0',
      info: {
        title: 'Internal Developer Portal API',
        version: '1.0.0',
        description: 'Service catalog, scorecards, self-service actions, and AI-powered developer tools.',
      },
      servers: [{ url: `http://${config.HOST}:${config.PORT}` }],
    },
  });

  await app.register(swaggerUi, { routePrefix: '/docs' });
  await app.register(cors, { origin: true });

  // RFC 7807 error handler
  app.setErrorHandler((error, request, reply) => {
    const statusCode = error.statusCode ?? 500;
    reply.status(statusCode).send({
      type: `https://developer-portal.dev/errors/${error.code ?? 'internal-error'}`,
      title: error.message,
      status: statusCode,
      detail: config.NODE_ENV !== 'production' ? error.stack : undefined,
      instance: request.url,
    });
  });

  // Health check
  app.get('/health', {
    schema: {
      response: {
        200: {
          type: 'object',
          properties: {
            status: { type: 'string', enum: ['healthy', 'degraded', 'unhealthy'] },
            version: { type: 'string' },
            checks: {
              type: 'object',
              properties: {
                database: { type: 'string' },
                redis: { type: 'string' },
              },
            },
          },
        },
      },
    },
  }, async () => {
    const dbOk = await db.execute('SELECT 1').then(() => 'ok').catch(() => 'fail');
    // Redis check added in Phase 1
    return {
      status: dbOk === 'ok' ? 'healthy' : 'unhealthy',
      version: '1.0.0',
      checks: { database: dbOk, redis: 'ok' },
    };
  });

  return app;
}
```

```typescript
// packages/shared/src/types/errors.ts
// RFC 7807 Problem Details
export interface ProblemDetails {
  type: string;          // URI reference identifying the problem type
  title: string;         // Short human-readable summary
  status: number;        // HTTP status code
  detail?: string;       // Detailed explanation
  instance?: string;     // URI reference for the specific occurrence
  [key: string]: unknown; // Extension members
}
```

**Testing**:
- `Unit: buildApp() → Fastify instance with registered routes`
- `Integration: GET /health with database up → { status: "healthy", checks: { database: "ok" } }`
- `Integration: GET /health with database down → { status: "unhealthy", checks: { database: "fail" } }`
- `Integration: GET /docs → OpenAPI 3.1 spec JSON returned`
- `Integration: GET /nonexistent → RFC 7807 error response with type, title, status fields`
- `Unit: error handler formats ZodError → ProblemDetails with validation details`

---

## Phase 2: Blueprint and Entity Catalog

### Purpose
Implement the core service catalog — the ability to define blueprints (entity type schemas) and create, read, update, delete, and search entities. This is the heart of the portal; everything else (scorecards, actions, AI features) operates on entities. After this phase, platform engineers can define custom entity types and developers can register services, APIs, and resources in the catalog.

### Tasks

#### 2.1 — Blueprint CRUD API

**What**: RESTful API for creating and managing blueprints (entity type definitions) with JSON Schema validation.

**Design**:

```typescript
// packages/shared/src/types/catalog.ts
export interface Blueprint {
  id: string;
  organizationId: string;
  identifier: string;         // e.g. 'service', 'api', 'data-pipeline'
  title: string;
  description?: string;
  icon?: string;
  color?: string;              // hex e.g. '#3B82F6'
  schema: BlueprintSchema;     // JSON Schema for entity properties
  calculationProperties?: Record<string, CalculationProperty>;
  mirrorProperties?: Record<string, MirrorProperty>;
  createdAt: string;
  updatedAt: string;
}

export interface BlueprintSchema {
  properties: Record<string, JSONSchemaProperty>;
  required?: string[];
}

export interface JSONSchemaProperty {
  type: 'string' | 'number' | 'boolean' | 'array' | 'object';
  title?: string;
  description?: string;
  enum?: string[];
  default?: unknown;
  format?: string;            // 'uri', 'email', 'date-time', etc.
  items?: JSONSchemaProperty; // for arrays
}

export interface CalculationProperty {
  title: string;
  type: 'count' | 'average' | 'aggregate';
  targetBlueprint: string;
  relation: string;
  property?: string;
}

export interface MirrorProperty {
  title: string;
  relation: string;
  property: string;
}

export interface CreateBlueprintRequest {
  identifier: string;
  title: string;
  description?: string;
  icon?: string;
  color?: string;
  schema: BlueprintSchema;
}

export interface UpdateBlueprintRequest {
  title?: string;
  description?: string;
  icon?: string;
  color?: string;
  schema?: BlueprintSchema;
}
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/blueprints` | Create a new blueprint |
| `GET` | `/api/v1/blueprints` | List all blueprints for the organization |
| `GET` | `/api/v1/blueprints/:identifier` | Get a blueprint by identifier |
| `PATCH` | `/api/v1/blueprints/:identifier` | Update a blueprint |
| `DELETE` | `/api/v1/blueprints/:identifier` | Delete a blueprint (fails if entities exist) |

```typescript
// packages/api/src/services/blueprint.service.ts
import Ajv from 'ajv';

export class BlueprintService {
  private ajv = new Ajv({ allErrors: true });

  async create(orgId: string, input: CreateBlueprintRequest): Promise<Blueprint> {
    // 1. Validate that the schema itself is valid JSON Schema
    this.ajv.validateSchema(input.schema);
    // 2. Check identifier uniqueness within org
    // 3. Insert into blueprints table
    // 4. Record audit log entry
    // 5. Return created blueprint
  }

  async update(orgId: string, identifier: string, input: UpdateBlueprintRequest): Promise<Blueprint> {
    // 1. Load existing blueprint
    // 2. If schema is changing, validate existing entities against new schema
    // 3. Update blueprint row
    // 4. Record audit log with before/after state
    // 5. Return updated blueprint
  }

  async delete(orgId: string, identifier: string): Promise<void> {
    // 1. Check no entities reference this blueprint
    // 2. Delete blueprint and cascade to blueprint_relations
    // 3. Record audit log
  }
}
```

**Testing**:
- `Unit: BlueprintService.create with valid schema → Blueprint object with generated UUID`
- `Unit: BlueprintService.create with invalid JSON Schema → ValidationError`
- `Unit: BlueprintService.create with duplicate identifier → ConflictError`
- `Unit: BlueprintService.update changing schema → validates existing entities`
- `Unit: BlueprintService.delete with existing entities → ForeignKeyError`
- `Integration: POST /api/v1/blueprints → 201, blueprint persisted in DB`
- `Integration: GET /api/v1/blueprints → 200, array of blueprints`
- `Integration: GET /api/v1/blueprints/nonexistent → 404, RFC 7807 error`
- `Integration: PATCH /api/v1/blueprints/service → 200, updated blueprint`
- `Integration: DELETE /api/v1/blueprints/service → 204 when no entities`
- `Fixture: default blueprints (service, api, resource, system, domain) seeded on first boot`

#### 2.2 — Entity CRUD and Search API

**What**: RESTful API for creating, updating, searching, and deleting entities (catalog entries) with JSONB property validation against their blueprint schema.

**Design**:

```typescript
// packages/shared/src/types/catalog.ts (continued)
export interface Entity {
  id: string;
  organizationId: string;
  blueprintId: string;
  blueprintIdentifier: string;  // denormalized for convenience
  identifier: string;
  title?: string;
  description?: string;
  properties: Record<string, unknown>;
  ownerTeamId?: string;
  ownerTeamName?: string;       // denormalized for list views
  relations?: EntityRelation[];
  createdBy?: string;
  createdAt: string;
  updatedAt: string;
}

export interface EntityRelation {
  relationIdentifier: string;
  targetEntityId: string;
  targetIdentifier: string;
  targetBlueprintIdentifier: string;
}

export interface CreateEntityRequest {
  blueprint: string;            // blueprint identifier
  identifier: string;
  title?: string;
  description?: string;
  properties: Record<string, unknown>;
  ownerTeam?: string;           // team slug
  relations?: Record<string, string | string[]>; // relation identifier -> target entity identifier(s)
}

export interface SearchEntitiesRequest {
  blueprint?: string;
  query?: string;              // full-text search
  properties?: Record<string, unknown>;  // JSONB containment filter
  ownerTeam?: string;
  page?: number;
  pageSize?: number;
}

export interface PaginatedResponse<T> {
  items: T[];
  total: number;
  page: number;
  pageSize: number;
  hasMore: boolean;
}
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/entities` | Create a new entity |
| `GET` | `/api/v1/entities` | Search/list entities (query params for filters) |
| `GET` | `/api/v1/entities/:id` | Get entity by ID with relations |
| `GET` | `/api/v1/blueprints/:blueprint/entities/:identifier` | Get entity by blueprint + identifier |
| `PATCH` | `/api/v1/entities/:id` | Update entity properties |
| `DELETE` | `/api/v1/entities/:id` | Delete entity and cascade relations |

```typescript
// packages/api/src/services/catalog.service.ts
export class CatalogService {
  async createEntity(orgId: string, input: CreateEntityRequest): Promise<Entity> {
    // 1. Load blueprint by identifier
    // 2. Validate input.properties against blueprint.schema using Ajv
    // 3. Resolve ownerTeam slug to team ID
    // 4. Insert entity row
    // 5. Create entity_relations for each relation in input.relations
    //    - Validate target entity exists and matches the expected blueprint per blueprint_relations
    // 6. Record audit log
    // 7. Return entity with resolved relations
  }

  async search(orgId: string, params: SearchEntitiesRequest): Promise<PaginatedResponse<Entity>> {
    // Build dynamic query:
    // - blueprint filter: JOIN blueprints WHERE identifier = params.blueprint
    // - text search: to_tsvector('english', ...) @@ plainto_tsquery(params.query)
    // - property filter: properties @> params.properties::jsonb
    // - ownerTeam: JOIN teams WHERE slug = params.ownerTeam
    // - pagination: LIMIT/OFFSET
  }

  async getEntityGraph(entityId: string, depth: number = 2): Promise<GraphData> {
    // Recursive CTE traversal of entity_relations up to `depth` hops
    // Returns nodes and edges for visualization
  }
}
```

**Testing**:
- `Unit: CatalogService.createEntity with valid properties → Entity with validated properties`
- `Unit: CatalogService.createEntity with invalid property type → ValidationError listing failing properties`
- `Unit: CatalogService.createEntity with missing required property → ValidationError`
- `Unit: CatalogService.createEntity with invalid relation target → NotFoundError`
- `Integration: POST /api/v1/entities → 201, entity persisted with JSONB properties`
- `Integration: GET /api/v1/entities?blueprint=service → paginated list of services`
- `Integration: GET /api/v1/entities?query=payment → full-text search returns matching entities`
- `Integration: GET /api/v1/entities?properties={"lifecycle":"production"} → JSONB filter works`
- `Integration: GET /api/v1/entities/:id → entity with resolved relations`
- `Integration: PATCH /api/v1/entities/:id → properties updated, audit log created`
- `E2E: create blueprint, create entity, search by property, retrieve by ID → full CRUD cycle`

#### 2.3 — Blueprint Relation Management

**What**: API for defining how blueprints relate to each other (e.g., "service provides_api api") and managing entity-level relationship edges.

**Design**:

```typescript
// packages/shared/src/types/catalog.ts (continued)
export interface BlueprintRelation {
  id: string;
  sourceBlueprintId: string;
  sourceBlueprintIdentifier: string;
  targetBlueprintId: string;
  targetBlueprintIdentifier: string;
  identifier: string;          // e.g. 'provides_api', 'depends_on', 'owned_by'
  title: string;
  description?: string;
  cardinality: 'one' | 'many';
  required: boolean;
}

export interface CreateBlueprintRelationRequest {
  sourceBlueprint: string;
  targetBlueprint: string;
  identifier: string;
  title: string;
  description?: string;
  cardinality?: 'one' | 'many';
  required?: boolean;
}
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/blueprint-relations` | Create a blueprint relation |
| `GET` | `/api/v1/blueprints/:identifier/relations` | List relations for a blueprint |
| `DELETE` | `/api/v1/blueprint-relations/:id` | Delete a relation definition |
| `POST` | `/api/v1/entities/:id/relations` | Add a relation edge between entities |
| `DELETE` | `/api/v1/entities/:id/relations/:relationId` | Remove a relation edge |

**Testing**:
- `Unit: create relation with valid source/target blueprints → BlueprintRelation`
- `Unit: create relation with nonexistent blueprint → NotFoundError`
- `Unit: create entity relation matching blueprint relation → EntityRelation`
- `Unit: create entity relation violating cardinality=one when one already exists → ConflictError`
- `Integration: create service->api provides_api relation → 201`
- `Integration: create entity edge from payment-service to payment-api → 201`
- `Integration: query entity with relations included → relations array populated`

#### 2.4 — Default Blueprints and Seed Data

**What**: Seed the database with default blueprints matching the Backstage catalog descriptor format (Component, API, Resource, System, Domain) and default blueprint relations, so the portal is immediately usable.

**Design**:

```typescript
// packages/shared/src/constants/entity-kinds.ts
export const DEFAULT_BLUEPRINTS = [
  {
    identifier: 'component',
    title: 'Component',
    icon: 'box',
    color: '#3B82F6',
    schema: {
      properties: {
        type: { type: 'string', enum: ['service', 'website', 'library', 'data-pipeline', 'mobile-app', 'cli'], title: 'Type' },
        lifecycle: { type: 'string', enum: ['experimental', 'development', 'production', 'deprecated'], title: 'Lifecycle' },
        language: { type: 'string', title: 'Language' },
        framework: { type: 'string', title: 'Framework' },
        tier: { type: 'string', enum: ['tier-1', 'tier-2', 'tier-3', 'tier-4'], title: 'Service Tier' },
        repoUrl: { type: 'string', format: 'uri', title: 'Repository URL' },
        k8sNamespace: { type: 'string', title: 'Kubernetes Namespace' },
      },
      required: ['type', 'lifecycle'],
    },
  },
  {
    identifier: 'api',
    title: 'API',
    icon: 'globe',
    color: '#10B981',
    schema: {
      properties: {
        specType: { type: 'string', enum: ['openapi', 'asyncapi', 'graphql', 'grpc'], title: 'Spec Type' },
        specUrl: { type: 'string', format: 'uri', title: 'Specification URL' },
        specVersion: { type: 'string', title: 'Spec Version' },
        lifecycle: { type: 'string', enum: ['experimental', 'development', 'production', 'deprecated'], title: 'Lifecycle' },
      },
      required: ['specType', 'lifecycle'],
    },
  },
  {
    identifier: 'resource',
    title: 'Resource',
    icon: 'database',
    color: '#F59E0B',
    schema: {
      properties: {
        type: { type: 'string', enum: ['database', 'cache', 'queue', 'storage', 'cdn', 'vault', 'load-balancer'], title: 'Type' },
        engine: { type: 'string', title: 'Engine' },
        version: { type: 'string', title: 'Version' },
        environment: { type: 'string', enum: ['development', 'staging', 'production'], title: 'Environment' },
      },
      required: ['type'],
    },
  },
  {
    identifier: 'system',
    title: 'System',
    icon: 'layers',
    color: '#8B5CF6',
    schema: {
      properties: {
        lifecycle: { type: 'string', enum: ['experimental', 'development', 'production', 'deprecated'], title: 'Lifecycle' },
      },
    },
  },
  {
    identifier: 'domain',
    title: 'Domain',
    icon: 'globe-2',
    color: '#EC4899',
    schema: { properties: {} },
  },
] as const;

export const DEFAULT_RELATIONS = [
  { sourceBlueprint: 'component', targetBlueprint: 'api', identifier: 'provides_api', title: 'Provides API', cardinality: 'many' },
  { sourceBlueprint: 'component', targetBlueprint: 'api', identifier: 'consumes_api', title: 'Consumes API', cardinality: 'many' },
  { sourceBlueprint: 'component', targetBlueprint: 'resource', identifier: 'depends_on', title: 'Depends On', cardinality: 'many' },
  { sourceBlueprint: 'component', targetBlueprint: 'system', identifier: 'part_of', title: 'Part Of', cardinality: 'one' },
  { sourceBlueprint: 'system', targetBlueprint: 'domain', identifier: 'belongs_to', title: 'Belongs To', cardinality: 'one' },
] as const;
```

**Testing**:
- `Integration: seed script → 5 default blueprints created`
- `Integration: seed script → 5 default relations created`
- `Integration: seed script is idempotent → running twice creates no duplicates`
- `Unit: DEFAULT_BLUEPRINTS schema validates as valid JSON Schema`

---

## Phase 3: Teams, Users, and Authentication

### Purpose
Implement team and user management, OAuth 2.0 / OIDC authentication, and role-based access control. After this phase, users can log in via SSO, belong to teams, and the API enforces permissions on all catalog operations.

### Tasks

#### 3.1 — Team and User Management API

**What**: CRUD API for teams (with hierarchy) and users, plus team membership management.

**Design**:

```typescript
// packages/shared/src/types/auth.ts
export interface User {
  id: string;
  organizationId: string;
  email: string;
  displayName: string;
  avatarUrl?: string;
  status: 'active' | 'inactive' | 'suspended';
  profile: Record<string, unknown>;
  teams: TeamMembership[];
  createdAt: string;
  updatedAt: string;
}

export interface Team {
  id: string;
  organizationId: string;
  name: string;
  slug: string;
  description?: string;
  parentTeamId?: string;
  hierarchyPath?: string;      // ltree path
  properties: Record<string, unknown>;
  memberCount: number;
  entityCount: number;         // components owned
  createdAt: string;
  updatedAt: string;
}

export interface TeamMembership {
  teamId: string;
  teamSlug: string;
  teamName: string;
  role: 'owner' | 'maintainer' | 'member';
  joinedAt: string;
}

export interface CreateTeamRequest {
  name: string;
  slug: string;
  description?: string;
  parentTeam?: string;        // slug of parent team
}
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/teams` | Create team |
| `GET` | `/api/v1/teams` | List teams (with hierarchy) |
| `GET` | `/api/v1/teams/:slug` | Get team with members and owned entities |
| `PATCH` | `/api/v1/teams/:slug` | Update team |
| `POST` | `/api/v1/teams/:slug/members` | Add member |
| `DELETE` | `/api/v1/teams/:slug/members/:userId` | Remove member |
| `GET` | `/api/v1/users/me` | Current user profile |
| `GET` | `/api/v1/users` | List users |

**Testing**:
- `Unit: create team with parent → hierarchyPath computed as parent.slug.child_slug`
- `Unit: create team without parent → hierarchyPath is org.slug`
- `Unit: add member → team_members row created`
- `Unit: remove last owner → error preventing orphaned team`
- `Integration: GET /api/v1/teams → hierarchical team list`
- `Integration: GET /api/v1/teams/platform?include=members,entities → team with nested data`
- `Integration: ltree query → "all teams under engineering" returns subtree`

#### 3.2 — OAuth 2.0 / OIDC Authentication

**What**: Implement SSO login via OpenID Connect with Passport.js, supporting multiple identity providers per organization.

**Design**:

```typescript
// packages/api/src/plugins/auth.plugin.ts
import { FastifyPluginAsync } from 'fastify';
import { Issuer, Strategy as OidcStrategy } from 'openid-client';

export const authPlugin: FastifyPluginAsync = async (app) => {
  // Session management via Redis
  await app.register(import('@fastify/session'), {
    secret: config.SESSION_SECRET,
    store: new RedisStore({ client: redisClient }),
    cookie: {
      secure: config.NODE_ENV === 'production',
      httpOnly: true,
      sameSite: 'lax',
      maxAge: 86400000, // 24 hours
    },
  });

  // Routes
  app.get('/auth/login', async (req, reply) => {
    // Redirect to IdP authorization endpoint with PKCE (RFC 7636)
  });

  app.get('/auth/callback', async (req, reply) => {
    // Exchange authorization code for tokens
    // Fetch user info from IdP
    // Upsert user in database
    // Create session
    // Redirect to frontend
  });

  app.post('/auth/logout', async (req, reply) => {
    // Destroy session, redirect to IdP logout if supported
  });

  // Authentication guard decorator
  app.decorate('authenticate', async (req: FastifyRequest, reply: FastifyReply) => {
    if (!req.session?.userId) {
      reply.status(401).send({
        type: 'https://developer-portal.dev/errors/unauthenticated',
        title: 'Authentication required',
        status: 401,
      });
    }
  });
};
```

Identity provider configuration stored in `identity_providers` table:

```sql
-- Schema from data model suggestion 3
CREATE TABLE identity_providers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    protocol        VARCHAR(20) NOT NULL,  -- 'oidc', 'saml'
    config          JSONB NOT NULL,        -- { issuer_url, client_id, client_secret (encrypted) }
    enabled         BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Testing**:
- `Unit: auth guard on unauthenticated request → 401 with RFC 7807 body`
- `Unit: auth guard with valid session → request proceeds`
- `Integration (mocked IdP): login flow → redirect to IdP with PKCE challenge`
- `Integration (mocked IdP): callback with valid code → user upserted, session created, redirect to /`
- `Integration (mocked IdP): callback with invalid code → 401 error`
- `Unit: session expired → 401 on next request`
- `Integration: logout → session destroyed`

#### 3.3 — Role-Based Access Control

**What**: Implement RBAC with organization-scoped and team-scoped roles, enforced as a Fastify preHandler hook.

**Design**:

```typescript
// packages/shared/src/types/auth.ts (continued)
export interface Role {
  id: string;
  organizationId: string;
  name: string;
  description?: string;
  permissions: Permission[];
  isSystem: boolean;
}

export interface Permission {
  resource: string;            // 'entity', 'blueprint', 'scorecard', 'action', 'team', 'integration'
  actions: ('read' | 'create' | 'update' | 'delete' | 'trigger')[];
  blueprint?: string;          // optional: restrict to a specific blueprint
}

// Default system roles
export const SYSTEM_ROLES = {
  admin: {
    name: 'admin',
    permissions: [{ resource: '*', actions: ['read', 'create', 'update', 'delete', 'trigger'] }],
  },
  editor: {
    name: 'editor',
    permissions: [
      { resource: 'entity', actions: ['read', 'create', 'update'] },
      { resource: 'scorecard', actions: ['read'] },
      { resource: 'action', actions: ['read', 'trigger'] },
    ],
  },
  viewer: {
    name: 'viewer',
    permissions: [
      { resource: 'entity', actions: ['read'] },
      { resource: 'scorecard', actions: ['read'] },
      { resource: 'action', actions: ['read'] },
    ],
  },
} as const;
```

```typescript
// packages/api/src/plugins/rbac.plugin.ts
export function requirePermission(resource: string, action: string) {
  return async (req: FastifyRequest, reply: FastifyReply) => {
    const userRoles = await getUserRoles(req.session.userId, req.orgId);
    const hasPermission = userRoles.some(role =>
      role.permissions.some(p =>
        (p.resource === '*' || p.resource === resource) &&
        p.actions.includes(action)
      )
    );
    if (!hasPermission) {
      reply.status(403).send({
        type: 'https://developer-portal.dev/errors/forbidden',
        title: 'Insufficient permissions',
        status: 403,
        detail: `Requires ${action} permission on ${resource}`,
      });
    }
  };
}
```

**Testing**:
- `Unit: admin role → all permissions granted`
- `Unit: viewer role → create entity denied`
- `Unit: editor role with blueprint scope → can edit entities of that blueprint only`
- `Unit: team-scoped role → permission applies only to entities owned by team`
- `Integration: unauthenticated POST /api/v1/entities → 401`
- `Integration: viewer POST /api/v1/entities → 403`
- `Integration: editor POST /api/v1/entities → 201`

---

## Phase 4: Scorecards and Engineering Standards

### Purpose
Implement scorecards that codify engineering standards as evaluatable rules against catalog entities. After this phase, platform engineers can define production-readiness checks (e.g., "tier-1 services must have a runbook") and evaluate entities against them, surfacing compliance scores per entity and team.

### Tasks

#### 4.1 — Scorecard Definition API

**What**: CRUD API for scorecards with a JSONB rule DSL that references entity JSONB properties.

**Design**:

```typescript
// packages/shared/src/types/scorecard.ts
export interface Scorecard {
  id: string;
  organizationId: string;
  name: string;
  identifier: string;
  description?: string;
  targetBlueprintId: string;
  targetBlueprintIdentifier: string;
  rules: ScorecardRule[];
  levels: string[];            // e.g. ['bronze', 'silver', 'gold']
  createdAt: string;
  updatedAt: string;
}

export interface ScorecardRule {
  identifier: string;
  title: string;
  description?: string;
  category: string;            // 'reliability', 'security', 'documentation', 'observability'
  level: string;               // which level this rule contributes to
  weight: number;              // 0.0 - 1.0
  query: RuleQuery;
}

export interface RuleQuery {
  combinator: 'and' | 'or';
  conditions: RuleCondition[];
}

export interface RuleCondition {
  property: string;            // dot-path, e.g. 'properties.runbook_url'
  operator: 'eq' | 'neq' | 'gt' | 'lt' | 'gte' | 'lte' |
            'contains' | 'isNotEmpty' | 'isEmpty' | 'in' | 'notIn';
  value?: unknown;
}

export interface CreateScorecardRequest {
  name: string;
  identifier: string;
  description?: string;
  targetBlueprint: string;
  rules: Omit<ScorecardRule, 'identifier'>[];
  levels?: string[];
}
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/scorecards` | Create scorecard |
| `GET` | `/api/v1/scorecards` | List scorecards |
| `GET` | `/api/v1/scorecards/:identifier` | Get scorecard with latest results |
| `PATCH` | `/api/v1/scorecards/:identifier` | Update scorecard rules |
| `DELETE` | `/api/v1/scorecards/:identifier` | Delete scorecard |

**Testing**:
- `Unit: create scorecard with valid rules → Scorecard object`
- `Unit: create scorecard referencing nonexistent blueprint → NotFoundError`
- `Unit: rule with invalid operator → ValidationError`
- `Integration: POST /api/v1/scorecards → 201, scorecard persisted`
- `Integration: GET /api/v1/scorecards → list with target blueprint info`

#### 4.2 — Scorecard Evaluation Engine

**What**: Engine that evaluates all entities of a blueprint against a scorecard's rules, producing per-entity scores and level achievements.

**Design**:

```typescript
// packages/api/src/services/scorecard.service.ts
export class ScorecardEvaluationEngine {
  /**
   * Evaluate a single entity against a scorecard.
   * Each rule.query is tested against entity.properties using the JSONB operator mapping.
   */
  evaluateEntity(entity: Entity, scorecard: Scorecard): ScorecardEvaluation {
    const ruleResults = scorecard.rules.map(rule => ({
      ruleIdentifier: rule.identifier,
      passed: this.evaluateQuery(entity, rule.query),
      weight: rule.weight,
      level: rule.level,
    }));

    const overallScore = this.calculateScore(ruleResults);
    const levelAchieved = this.determineLevelAchieved(ruleResults, scorecard.levels);

    return {
      scorecardId: scorecard.id,
      entityId: entity.id,
      results: ruleResults,
      overallScore,
      levelAchieved,
      evaluatedAt: new Date().toISOString(),
    };
  }

  private evaluateQuery(entity: Entity, query: RuleQuery): boolean {
    const evaluator = query.combinator === 'and' ? 'every' : 'some';
    return query.conditions[evaluator](c => this.evaluateCondition(entity, c));
  }

  private evaluateCondition(entity: Entity, condition: RuleCondition): boolean {
    const value = getNestedProperty(entity, condition.property);
    switch (condition.operator) {
      case 'isNotEmpty': return value != null && value !== '';
      case 'isEmpty': return value == null || value === '';
      case 'eq': return value === condition.value;
      case 'neq': return value !== condition.value;
      case 'gt': return Number(value) > Number(condition.value);
      case 'lt': return Number(value) < Number(condition.value);
      case 'contains': return String(value).includes(String(condition.value));
      case 'in': return Array.isArray(condition.value) && condition.value.includes(value);
      default: return false;
    }
  }

  private calculateScore(results: RuleResult[]): number {
    const totalWeight = results.reduce((sum, r) => sum + r.weight, 0);
    const passedWeight = results.filter(r => r.passed).reduce((sum, r) => sum + r.weight, 0);
    return totalWeight > 0 ? (passedWeight / totalWeight) * 100 : 0;
  }

  private determineLevelAchieved(results: RuleResult[], levels: string[]): string | null {
    // A level is achieved if all rules at that level and below pass
    for (let i = levels.length - 1; i >= 0; i--) {
      const level = levels[i];
      const levelRules = results.filter(r => levels.indexOf(r.level) <= i);
      if (levelRules.every(r => r.passed)) return level;
    }
    return null;
  }
}
```

```typescript
// packages/shared/src/types/scorecard.ts (continued)
export interface ScorecardEvaluation {
  id?: string;
  scorecardId: string;
  entityId: string;
  results: RuleResult[];
  overallScore: number;         // 0-100
  levelAchieved: string | null;
  evaluatedAt: string;
}

export interface RuleResult {
  ruleIdentifier: string;
  passed: boolean;
  weight: number;
  level: string;
}
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/scorecards/:identifier/evaluate` | Evaluate all matching entities |
| `GET` | `/api/v1/entities/:id/scorecards` | Get all scorecard results for an entity |
| `GET` | `/api/v1/scorecards/:identifier/results` | Get all results for a scorecard |
| `GET` | `/api/v1/teams/:slug/scorecard-summary` | Aggregate scores by team |

**Testing**:
- `Unit: evaluateEntity with all rules passing → score=100, level=gold`
- `Unit: evaluateEntity with bronze rules passing only → level=bronze`
- `Unit: evaluateEntity with no rules passing → score=0, level=null`
- `Unit: evaluateCondition isNotEmpty with truthy value → true`
- `Unit: evaluateCondition isNotEmpty with null → false`
- `Unit: evaluateCondition gt with 10 > 5 → true`
- `Unit: evaluateQuery AND combinator with one failing → false`
- `Unit: evaluateQuery OR combinator with one passing → true`
- `Integration: POST /api/v1/scorecards/production-readiness/evaluate → evaluations persisted for all components`
- `Integration: GET /api/v1/teams/platform/scorecard-summary → aggregated scores by team`
- `Fixture: sample scorecard with 5 rules, sample entities with varying compliance`

#### 4.3 — Scorecard Worker (Background Evaluation)

**What**: BullMQ worker that periodically re-evaluates all scorecards, enabling scheduled compliance checks.

**Design**:

```typescript
// packages/api/src/workers/scorecard.worker.ts
import { Worker, Queue } from 'bullmq';

export const scorecardQueue = new Queue('scorecard-evaluation', { connection: redisConnection });

// Schedule evaluation every 6 hours
await scorecardQueue.add('evaluate-all', {}, {
  repeat: { every: 6 * 60 * 60 * 1000 },
});

const worker = new Worker('scorecard-evaluation', async (job) => {
  const scorecards = await scorecardService.listAll(job.data.orgId);
  for (const scorecard of scorecards) {
    const entities = await catalogService.findByBlueprint(scorecard.targetBlueprintIdentifier);
    const evaluations = entities.map(e => engine.evaluateEntity(e, scorecard));
    await scorecardService.bulkSaveEvaluations(evaluations);
  }
}, { connection: redisConnection, concurrency: 2 });
```

**Testing**:
- `Integration: enqueue scorecard-evaluation job → worker processes and evaluations saved`
- `Integration: worker with 100 entities → all evaluations persisted within timeout`
- `Unit: worker handles empty entity list gracefully`

---

## Phase 5: DORA Metrics and Deployment Tracking

### Purpose
Ingest deployment events and incidents, calculate the four DORA metrics (deployment frequency, lead time, MTTR, change failure rate), and expose them via API and dashboard views. After this phase, teams can track their engineering performance against the DORA framework.

### Tasks

#### 5.1 — Deployment Event Ingestion API

**What**: API and webhook endpoint for recording deployment events from CI/CD pipelines.

**Design**:

```typescript
// packages/shared/src/types/dora.ts
export interface DeploymentEvent {
  id: string;
  entityId: string;
  entityIdentifier: string;
  environment: string;
  deployedAt: string;
  commitSha?: string;
  deployerId?: string;
  success: boolean;
  leadTimeSeconds?: number;
  metadata: Record<string, unknown>;
  createdAt: string;
}

export interface CreateDeploymentEventRequest {
  entityIdentifier: string;
  blueprint?: string;           // defaults to 'component'
  environment?: string;         // defaults to 'production'
  deployedAt?: string;          // defaults to now
  commitSha?: string;
  success?: boolean;            // defaults to true
  metadata?: Record<string, unknown>;
}

export interface Incident {
  id: string;
  organizationId: string;
  title: string;
  severity: 'sev-1' | 'sev-2' | 'sev-3' | 'sev-4';
  status: 'open' | 'investigating' | 'mitigating' | 'resolved' | 'closed';
  startedAt: string;
  resolvedAt?: string;
  timeToRestoreSeconds?: number;
  affectedEntities: string[];
  metadata: Record<string, unknown>;
  createdAt: string;
  updatedAt: string;
}
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/deployments` | Record a deployment event |
| `POST` | `/api/v1/webhooks/deployment` | Webhook for CI/CD pipeline events |
| `GET` | `/api/v1/entities/:id/deployments` | List deployments for an entity |
| `POST` | `/api/v1/incidents` | Create an incident |
| `PATCH` | `/api/v1/incidents/:id` | Update incident status/resolution |
| `GET` | `/api/v1/incidents` | List incidents |

**Testing**:
- `Integration: POST /api/v1/deployments → 201, deployment event persisted`
- `Integration: POST /api/v1/deployments with unknown entity → 404`
- `Integration: POST /api/v1/webhooks/deployment (GitHub Actions format) → deployment event created`
- `Integration: POST /api/v1/incidents → 201, incident created`
- `Integration: PATCH /api/v1/incidents/:id with resolvedAt → timeToRestoreSeconds calculated`

#### 5.2 — DORA Metric Calculation Engine

**What**: Service that computes the four DORA metrics from deployment events and incidents over configurable time periods.

**Design**:

```typescript
// packages/shared/src/types/dora.ts (continued)
export interface DORAMetrics {
  entityId?: string;
  teamId?: string;
  periodStart: string;
  periodEnd: string;
  deploymentFrequency: number;       // deploys per day
  leadTimeSeconds: number;           // median lead time to production
  mttrSeconds: number;               // median time to restore service
  changeFailureRate: number;         // ratio 0.0-1.0
  doraLevel: 'elite' | 'high' | 'medium' | 'low';
}

// DORA level thresholds (from DORA research)
export const DORA_THRESHOLDS = {
  elite: {
    deploymentFrequency: 1,          // >= 1 per day
    leadTimeSeconds: 86400,          // < 1 day
    mttrSeconds: 3600,               // < 1 hour
    changeFailureRate: 0.05,         // < 5%
  },
  high: {
    deploymentFrequency: 0.14,       // >= 1 per week
    leadTimeSeconds: 604800,         // < 1 week
    mttrSeconds: 86400,              // < 1 day
    changeFailureRate: 0.10,         // < 10%
  },
  medium: {
    deploymentFrequency: 0.033,      // >= 1 per month
    leadTimeSeconds: 2592000,        // < 1 month
    mttrSeconds: 604800,             // < 1 week
    changeFailureRate: 0.15,         // < 15%
  },
  // Everything below medium = low
} as const;
```

```typescript
// packages/api/src/services/dora.service.ts
export class DORAService {
  async calculateMetrics(
    orgId: string,
    entityId: string | null,
    teamId: string | null,
    periodStart: Date,
    periodEnd: Date,
  ): Promise<DORAMetrics> {
    // 1. Query deployment_events for the entity/team in the period
    // 2. deployment_frequency = count(deployments) / days_in_period
    // 3. lead_time = median(lead_time_seconds) where lead_time_seconds IS NOT NULL
    // 4. Query incidents where started_at in period and affected entities match
    // 5. mttr = median(time_to_restore_seconds) where resolved
    // 6. change_failure_rate = count(failed_deployments) / count(all_deployments)
    // 7. Determine DORA level by comparing all four metrics against thresholds
    // 8. Persist snapshot in dora_metric_snapshots (not in Drizzle schema yet — added this phase)
  }

  classifyLevel(metrics: Omit<DORAMetrics, 'doraLevel'>): DORAMetrics['doraLevel'] {
    // All four metrics must meet the threshold for a given level.
    // Level is the lowest classification across all four dimensions.
    for (const level of ['elite', 'high', 'medium'] as const) {
      const t = DORA_THRESHOLDS[level];
      if (
        metrics.deploymentFrequency >= t.deploymentFrequency &&
        metrics.leadTimeSeconds <= t.leadTimeSeconds &&
        metrics.mttrSeconds <= t.mttrSeconds &&
        metrics.changeFailureRate <= t.changeFailureRate
      ) {
        return level;
      }
    }
    return 'low';
  }
}
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/dora/entities/:id` | DORA metrics for an entity |
| `GET` | `/api/v1/dora/teams/:slug` | DORA metrics for a team |
| `GET` | `/api/v1/dora/organization` | Organization-wide DORA metrics |

**Testing**:
- `Unit: classifyLevel with elite metrics → 'elite'`
- `Unit: classifyLevel with mixed metrics → lowest dimension wins (e.g. high DF + low MTTR = low)`
- `Unit: calculateMetrics with 30 deploys in 30 days → deploymentFrequency=1.0`
- `Unit: calculateMetrics with 0 deployments → deploymentFrequency=0, level=low`
- `Unit: calculateMetrics with 2 of 20 failed → changeFailureRate=0.10`
- `Integration: POST deployments, POST incidents, GET /api/v1/dora/entities/:id → correct DORA metrics`
- `Integration: DORA metrics aggregated by team → averages across team entities`
- `Fixture: 90 days of deployment events and incidents for testing`

#### 5.3 — DORA Metrics Aggregation Worker

**What**: BullMQ worker that pre-computes DORA metric snapshots daily for all entities and teams.

**Design**:

```typescript
// packages/api/src/workers/dora.worker.ts
const doraQueue = new Queue('dora-aggregation', { connection: redisConnection });

await doraQueue.add('daily-aggregation', {}, {
  repeat: { cron: '0 2 * * *' }, // 2 AM daily
});

const worker = new Worker('dora-aggregation', async (job) => {
  const periods = [
    { start: subDays(new Date(), 7), end: new Date(), label: '7d' },
    { start: subDays(new Date(), 30), end: new Date(), label: '30d' },
    { start: subDays(new Date(), 90), end: new Date(), label: '90d' },
  ];
  // For each entity and team, compute and persist snapshots for each period
}, { connection: redisConnection });
```

**Testing**:
- `Integration: worker runs → dora_metric_snapshots populated for all entities`
- `Integration: multiple periods → 7d, 30d, 90d snapshots all saved`

---

## Phase 6: Self-Service Actions

### Purpose
Implement self-service actions that let developers trigger templated workflows (service creation, database provisioning, secret rotation) through the portal. After this phase, platform engineers can define action templates and developers can trigger them with approval workflows.

### Tasks

#### 6.1 — Action Template API

**What**: CRUD API for self-service action templates with JSON Schema input definitions and configurable execution backends (webhook, GitHub Action, Terraform).

**Design**:

```typescript
// packages/shared/src/types/actions.ts
export interface Action {
  id: string;
  organizationId: string;
  identifier: string;
  title: string;
  description?: string;
  targetBlueprintId?: string;
  triggerType: 'manual' | 'on_create' | 'on_update' | 'on_delete' | 'scheduled';
  inputSchema: Record<string, unknown>;   // JSON Schema for action inputs
  execution: ActionExecution;
  requiresApproval: boolean;
  approvalConfig?: ApprovalConfig;
  createdAt: string;
  updatedAt: string;
}

export interface ActionExecution {
  type: 'webhook' | 'github-action' | 'terraform' | 'script';
  config: WebhookConfig | GitHubActionConfig | TerraformConfig | ScriptConfig;
}

export interface WebhookConfig {
  url: string;
  method: 'POST' | 'PUT' | 'PATCH';
  headers?: Record<string, string>;
  payloadTemplate?: string;     // Handlebars template for payload
}

export interface GitHubActionConfig {
  owner: string;
  repo: string;
  workflowId: string;
  ref: string;
  inputs: Record<string, string>; // mapping from action input to workflow input
}

export interface ApprovalConfig {
  approvers: string[];          // team slugs or user IDs
  minApprovals: number;
  expiresAfterHours: number;
}

export interface ActionRun {
  id: string;
  actionId: string;
  entityId?: string;
  triggeredBy: string;
  inputs: Record<string, unknown>;
  status: 'pending' | 'approved' | 'rejected' | 'running' | 'succeeded' | 'failed' | 'cancelled';
  approvedBy?: string;
  output?: Record<string, unknown>;
  startedAt?: string;
  completedAt?: string;
  createdAt: string;
  updatedAt: string;
}
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/actions` | Create action template |
| `GET` | `/api/v1/actions` | List actions |
| `GET` | `/api/v1/actions/:identifier` | Get action |
| `PATCH` | `/api/v1/actions/:identifier` | Update action |
| `POST` | `/api/v1/actions/:identifier/trigger` | Trigger an action |
| `GET` | `/api/v1/action-runs` | List action runs |
| `GET` | `/api/v1/action-runs/:id` | Get action run status |
| `POST` | `/api/v1/action-runs/:id/approve` | Approve a pending action |
| `POST` | `/api/v1/action-runs/:id/reject` | Reject a pending action |

**Testing**:
- `Unit: create action with valid inputSchema → Action object`
- `Unit: trigger action with valid inputs → ActionRun with status=pending (if approval required)`
- `Unit: trigger action with invalid inputs → ValidationError`
- `Unit: approve action → status transitions pending→approved→running`
- `Unit: reject action → status transitions pending→rejected`
- `Integration: POST /api/v1/actions/create-service/trigger → 201, run created`
- `Integration: action run state machine → transitions enforce valid flows`

#### 6.2 — Action Execution Engine

**What**: BullMQ worker that executes approved action runs by calling the configured backend (webhook, GitHub Actions API).

**Design**:

```typescript
// packages/api/src/services/action.service.ts
export class ActionExecutionEngine {
  async execute(run: ActionRun, action: Action): Promise<void> {
    await this.updateRunStatus(run.id, 'running');
    try {
      switch (action.execution.type) {
        case 'webhook':
          await this.executeWebhook(run, action.execution.config as WebhookConfig);
          break;
        case 'github-action':
          await this.executeGitHubAction(run, action.execution.config as GitHubActionConfig);
          break;
        // ... other types
      }
      await this.updateRunStatus(run.id, 'succeeded');
    } catch (error) {
      await this.updateRunStatus(run.id, 'failed', { error: error.message });
    }
  }

  private async executeWebhook(run: ActionRun, config: WebhookConfig): Promise<void> {
    const payload = config.payloadTemplate
      ? renderTemplate(config.payloadTemplate, { run, inputs: run.inputs })
      : run.inputs;

    const response = await fetch(config.url, {
      method: config.method,
      headers: { 'Content-Type': 'application/json', ...config.headers },
      body: JSON.stringify(payload),
    });

    if (!response.ok) throw new Error(`Webhook returned ${response.status}`);
  }
}
```

**Testing**:
- `Integration (mocked webhook): execute webhook action → POST sent to webhook URL with correct payload`
- `Integration (mocked webhook): webhook returns 500 → run status set to failed`
- `Integration (mocked GitHub API): execute github-action → workflow_dispatch triggered`
- `Unit: Handlebars payload template renders with run inputs`
- `Unit: execution timeout → run marked as failed`

---

## Phase 7: Integration Engine

### Purpose
Build the integration framework that syncs catalog data from external sources (GitHub, GitLab, Kubernetes, PagerDuty) and imports Backstage `catalog-info.yaml` descriptors. After this phase, the catalog auto-populates from existing infrastructure rather than requiring manual entry.

### Tasks

#### 7.1 — Integration Framework and Mapping Engine

**What**: A pluggable integration framework where each integration type defines how to fetch data from an external source and map it to blueprint entities using a configurable mapping configuration.

**Design**:

```typescript
// packages/api/src/integrations/types.ts
export interface IntegrationProvider {
  type: string;                 // 'github', 'gitlab', 'kubernetes', etc.
  displayName: string;

  /** Validate integration config (API key, URL, etc.) */
  validateConfig(config: Record<string, unknown>): Promise<boolean>;

  /** Fetch raw data from the external source */
  fetchData(config: Record<string, unknown>): AsyncGenerator<ExternalRecord>;

  /** Default mapping config for this provider */
  defaultMappingConfig(): MappingConfig;
}

export interface MappingConfig {
  targetBlueprint: string;
  entityIdentifier: string;      // JSONPath or dot-path in external record
  properties: Record<string, string>;  // blueprint property -> source path
  relations?: Record<string, string>;  // relation identifier -> source path
}

export interface ExternalRecord {
  sourceId: string;
  data: Record<string, unknown>;
  fetchedAt: string;
}

// Integration stored in DB
export interface Integration {
  id: string;
  organizationId: string;
  type: string;
  title: string;
  config: Record<string, unknown>;  // encrypted connection details
  mappingConfig: MappingConfig;
  status: 'active' | 'inactive' | 'error';
  lastSyncAt?: string;
  syncIntervalSeconds: number;
}
```

```typescript
// packages/api/src/services/sync.service.ts
export class SyncService {
  async syncIntegration(integration: Integration): Promise<SyncResult> {
    const provider = this.providers.get(integration.type);
    let created = 0, updated = 0, errors = 0;

    for await (const record of provider.fetchData(integration.config)) {
      try {
        const entityData = this.mapRecord(record, integration.mappingConfig);
        await this.catalogService.upsertEntity(integration.organizationId, entityData);
        // Determine if it was created or updated
      } catch (e) {
        errors++;
      }
    }

    return { created, updated, errors, syncedAt: new Date().toISOString() };
  }

  private mapRecord(record: ExternalRecord, mapping: MappingConfig): CreateEntityRequest {
    return {
      blueprint: mapping.targetBlueprint,
      identifier: getPath(record.data, mapping.entityIdentifier),
      properties: Object.fromEntries(
        Object.entries(mapping.properties).map(([key, path]) => [key, getPath(record.data, path)])
      ),
    };
  }
}
```

**Testing**:
- `Unit: mapRecord with valid paths → CreateEntityRequest with properties populated`
- `Unit: mapRecord with missing path → property set to null`
- `Unit: syncIntegration with 10 records → 10 entities upserted`
- `Integration: sync worker processes integration queue → entities created in catalog`

#### 7.2 — GitHub Integration

**What**: Integration provider that syncs repositories as component entities, reads `catalog-info.yaml` descriptors, and ingests deployment events from GitHub Actions.

**Design**:

```typescript
// packages/api/src/integrations/github.ts
import { Octokit } from '@octokit/rest';

export class GitHubProvider implements IntegrationProvider {
  type = 'github';
  displayName = 'GitHub';

  async *fetchData(config: GitHubConfig): AsyncGenerator<ExternalRecord> {
    const octokit = new Octokit({ auth: config.token });

    // 1. List all repos in the org
    for await (const response of octokit.paginate.iterator(
      octokit.repos.listForOrg, { org: config.organization }
    )) {
      for (const repo of response.data) {
        // 2. Check for catalog-info.yaml
        const catalogInfo = await this.readCatalogInfo(octokit, repo);
        if (catalogInfo) {
          yield { sourceId: repo.full_name, data: { ...repo, catalogInfo }, fetchedAt: new Date().toISOString() };
        } else {
          yield { sourceId: repo.full_name, data: repo, fetchedAt: new Date().toISOString() };
        }
      }
    }
  }

  private async readCatalogInfo(octokit: Octokit, repo: any): Promise<any | null> {
    try {
      const { data } = await octokit.repos.getContent({
        owner: repo.owner.login,
        repo: repo.name,
        path: 'catalog-info.yaml',
      });
      // Parse YAML, return structured data
      return parseYaml(Buffer.from(data.content, 'base64').toString());
    } catch {
      return null;
    }
  }
}
```

**Testing**:
- `Integration (mocked Octokit): fetchData → yields ExternalRecord per repo`
- `Integration (mocked Octokit): repo with catalog-info.yaml → entity has catalogInfo data`
- `Integration (mocked Octokit): repo without catalog-info.yaml → entity created from repo metadata`
- `Unit: Backstage catalog-info.yaml parsed → correct entity properties`
- `Fixture: sample catalog-info.yaml files with Component, API, and System kinds`

#### 7.3 — Backstage YAML Descriptor Import

**What**: Parser and importer for Backstage `catalog-info.yaml` descriptor files, enabling migration from existing Backstage installations.

**Design**:

```typescript
// packages/api/src/services/catalog-import.ts
import { parse as parseYaml } from 'yaml';

// Backstage Catalog Descriptor Format
export interface BackstageCatalogDescriptor {
  apiVersion: 'backstage.io/v1alpha1' | 'backstage.io/v1beta1';
  kind: 'Component' | 'API' | 'Resource' | 'System' | 'Domain' | 'Group' | 'User' | 'Template' | 'Location';
  metadata: {
    name: string;
    description?: string;
    annotations?: Record<string, string>;
    labels?: Record<string, string>;
    tags?: string[];
  };
  spec: Record<string, unknown>;
}

export class CatalogImporter {
  // Map Backstage kind to IDP blueprint identifier
  private kindMapping: Record<string, string> = {
    Component: 'component',
    API: 'api',
    Resource: 'resource',
    System: 'system',
    Domain: 'domain',
  };

  async importDescriptor(orgId: string, yamlContent: string): Promise<Entity[]> {
    const docs = parseYaml(yamlContent, { maxAliasCount: -1 });
    // Handle multi-document YAML
    const descriptors = Array.isArray(docs) ? docs : [docs];
    const results: Entity[] = [];

    for (const desc of descriptors) {
      const blueprint = this.kindMapping[desc.kind];
      if (!blueprint) continue; // Skip unsupported kinds

      const entityInput: CreateEntityRequest = {
        blueprint,
        identifier: desc.metadata.name,
        title: desc.metadata.name,
        description: desc.metadata.description,
        properties: this.mapSpecToProperties(desc.kind, desc.spec),
        ownerTeam: desc.spec.owner as string,
      };

      const entity = await this.catalogService.upsertEntity(orgId, entityInput);
      results.push(entity);
    }

    return results;
  }
}
```

API endpoint:

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/import/backstage` | Import Backstage YAML descriptors |

**Testing**:
- `Unit: parse valid Backstage Component descriptor → correct CreateEntityRequest`
- `Unit: parse Backstage API descriptor with openapi type → api entity with specType=openapi`
- `Unit: multi-document YAML with 3 descriptors → 3 entities created`
- `Unit: unsupported kind (Template) → skipped without error`
- `Integration: POST /api/v1/import/backstage with YAML body → entities created`
- `Fixture: complete Backstage catalog-info.yaml with Component + API + System`

#### 7.4 — Sync Worker

**What**: BullMQ worker that runs integration syncs on a configurable schedule.

**Design**:

```typescript
// packages/api/src/workers/sync.worker.ts
const syncQueue = new Queue('integration-sync', { connection: redisConnection });

// For each active integration, schedule based on its sync_interval_seconds
async function scheduleIntegrations() {
  const integrations = await db.select().from(integrationsTable).where(eq(status, 'active'));
  for (const integration of integrations) {
    await syncQueue.add(`sync-${integration.id}`, { integrationId: integration.id }, {
      repeat: { every: integration.syncIntervalSeconds * 1000 },
      jobId: `sync-${integration.id}`,
    });
  }
}

const worker = new Worker('integration-sync', async (job) => {
  const integration = await db.select().from(integrationsTable).where(eq(id, job.data.integrationId));
  const result = await syncService.syncIntegration(integration);
  // Update last_sync_at
  // Record audit log
}, { connection: redisConnection });
```

**Testing**:
- `Integration: sync worker processes GitHub integration → entities created in catalog`
- `Integration: sync worker handles API errors gracefully → integration status set to 'error'`
- `Unit: schedule function creates repeatable jobs for each active integration`

---

## Phase 8: Frontend — Portal UI

### Purpose
Build the Next.js web application that provides the developer-facing portal UI: catalog browser, entity detail pages, scorecard dashboards, DORA metrics visualizations, and self-service action triggers. After this phase, developers interact with the full portal through a browser.

### Tasks

#### 8.1 — Layout, Navigation, and API Client

**What**: Next.js app shell with sidebar navigation, authentication flow, and typed API client.

**Design**:

```typescript
// packages/web/src/lib/api-client.ts
export class PortalAPIClient {
  constructor(private baseUrl: string) {}

  async get<T>(path: string, params?: Record<string, string>): Promise<T> {
    const url = new URL(`${this.baseUrl}${path}`);
    if (params) Object.entries(params).forEach(([k, v]) => url.searchParams.set(k, v));
    const res = await fetch(url.toString(), { credentials: 'include' });
    if (!res.ok) throw await this.handleError(res);
    return res.json();
  }

  async post<T>(path: string, body: unknown): Promise<T> {
    const res = await fetch(`${this.baseUrl}${path}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      credentials: 'include',
      body: JSON.stringify(body),
    });
    if (!res.ok) throw await this.handleError(res);
    return res.json();
  }

  private async handleError(res: Response): Promise<ProblemDetails> {
    return res.json(); // RFC 7807 error response
  }
}
```

Navigation structure:

```
Sidebar:
  - Dashboard (home)
  - Catalog
    - Services
    - APIs
    - Resources
    - Systems
    - Custom blueprints...
  - Scorecards
  - Metrics (DORA)
  - Actions
  - Teams
  - Settings
    - Blueprints
    - Integrations
    - Roles
```

**Testing**:
- `Unit: APIClient.get with 200 → parsed JSON`
- `Unit: APIClient.get with 404 → ProblemDetails error thrown`
- `E2E (Playwright): load / → sidebar navigation rendered with all sections`
- `E2E: unauthenticated access → redirected to login page`

#### 8.2 — Catalog Browser and Entity Detail

**What**: Catalog list page with search, filters, and sorting; entity detail page showing properties, relations, scorecards, deployments, and documentation.

**Design**:

```typescript
// packages/web/src/app/catalog/page.tsx
// Server component that fetches catalog entities with search params
export default async function CatalogPage({ searchParams }: { searchParams: SearchParams }) {
  const entities = await api.get<PaginatedResponse<Entity>>('/api/v1/entities', {
    blueprint: searchParams.blueprint,
    query: searchParams.q,
    page: searchParams.page,
  });

  return (
    <div>
      <CatalogSearch defaultQuery={searchParams.q} />
      <BlueprintFilter blueprints={blueprints} selected={searchParams.blueprint} />
      <EntityTable entities={entities.items} />
      <Pagination {...entities} />
    </div>
  );
}
```

Entity detail page sections:
1. **Header**: title, blueprint badge, lifecycle badge, owner team, tier
2. **Properties panel**: renders all JSONB properties based on blueprint schema
3. **Relations panel**: shows provides/consumes/depends_on relationships
4. **Dependency graph**: interactive visualization of entity relationships (D3.js force graph)
5. **Scorecards tab**: scorecard results with pass/fail per rule
6. **Deployments tab**: deployment history timeline
7. **Documentation tab**: TechDocs content (Markdown rendered)
8. **DORA metrics tab**: 7d/30d/90d DORA metrics for this entity

**Testing**:
- `E2E: navigate to /catalog → entity list displayed`
- `E2E: search for "payment" → filtered results shown`
- `E2E: click entity row → navigated to /catalog/[entityId]`
- `E2E: entity detail page → all tabs rendered (properties, relations, scorecards, deployments)`
- `E2E: dependency graph → D3 visualization renders with nodes and edges`

#### 8.3 — Scorecard Dashboard

**What**: Dashboard showing scorecard compliance across the organization, by team, and by blueprint, with drill-down to individual entity results.

**Design**:

Scorecard dashboard views:
1. **Organization overview**: overall compliance percentage, entities by level (gold/silver/bronze/none)
2. **Team leaderboard**: teams ranked by average scorecard score
3. **Scorecard detail**: individual scorecard showing all evaluated entities, pass/fail per rule
4. **Trend chart**: scorecard compliance over time (30/60/90 day trend)

**Testing**:
- `E2E: navigate to /scorecards → organization compliance overview displayed`
- `E2E: click scorecard → drill-down to entity-level results`
- `E2E: team leaderboard → teams sorted by score`

#### 8.4 — DORA Metrics Dashboard

**What**: DORA metrics visualization with per-entity, per-team, and organization views, showing the four key metrics with DORA level classification.

**Design**:

Dashboard components:
1. **Four metric cards**: deployment frequency, lead time, MTTR, change failure rate — each with current value, trend arrow, and DORA level color
2. **Time period selector**: 7d, 30d, 90d
3. **Entity/team selector**: switch between entity and team views
4. **Deployment timeline**: visual timeline of deployments with success/failure markers
5. **Incident overlay**: incidents shown on the deployment timeline

**Testing**:
- `E2E: navigate to /metrics → four DORA metric cards displayed`
- `E2E: switch to 30d period → metrics recalculated`
- `E2E: select specific team → team-level metrics shown`

#### 8.5 — Self-Service Action UI

**What**: Action catalog and trigger UI with dynamic form generation from action input schemas.

**Design**:

1. **Action catalog**: list of available actions with description, target blueprint, approval requirement
2. **Action trigger form**: JSON Schema-driven form using `@rjsf/core` (React JSON Schema Form) to render input fields from the action's `inputSchema`
3. **Action run status**: real-time status updates via polling or SSE

**Testing**:
- `E2E: navigate to /actions → action list displayed`
- `E2E: click "Create Service" action → form rendered from input schema`
- `E2E: submit form with valid inputs → action run created, status shown`
- `E2E: action requiring approval → "Pending Approval" status displayed`

---

## Phase 9: AI-Powered Documentation Generation

### Purpose
Implement the first AI-native differentiator: automated service documentation generation from code repositories, commit history, and existing runbooks. After this phase, LLMs auto-generate and maintain service documentation, addressing the chronic documentation gap identified in all competitor analyses.

### Tasks

#### 9.1 — LLM Provider Abstraction Layer

**What**: Abstract interface over LLM providers (Anthropic Claude, OpenAI, Ollama) so AI features work with any configured provider.

**Design**:

```typescript
// packages/api/src/services/ai/llm-provider.ts
export interface LLMProvider {
  name: string;
  chat(params: ChatParams): Promise<ChatResponse>;
  streamChat(params: ChatParams): AsyncGenerator<string>;
}

export interface ChatParams {
  model: string;
  systemPrompt: string;
  messages: ChatMessage[];
  temperature?: number;
  maxTokens?: number;
}

export interface ChatMessage {
  role: 'user' | 'assistant';
  content: string;
}

export interface ChatResponse {
  content: string;
  usage: { inputTokens: number; outputTokens: number };
  model: string;
}

// Anthropic implementation
export class AnthropicProvider implements LLMProvider {
  name = 'anthropic';
  private client: Anthropic;

  constructor(apiKey: string) {
    this.client = new Anthropic({ apiKey });
  }

  async chat(params: ChatParams): Promise<ChatResponse> {
    const response = await this.client.messages.create({
      model: params.model,
      max_tokens: params.maxTokens ?? 4096,
      system: params.systemPrompt,
      messages: params.messages.map(m => ({ role: m.role, content: m.content })),
      temperature: params.temperature ?? 0.3,
    });
    return {
      content: response.content[0].type === 'text' ? response.content[0].text : '',
      usage: { inputTokens: response.usage.input_tokens, outputTokens: response.usage.output_tokens },
      model: response.model,
    };
  }
}

// Factory
export function createLLMProvider(config: Config): LLMProvider {
  switch (config.AI_PROVIDER) {
    case 'anthropic': return new AnthropicProvider(config.ANTHROPIC_API_KEY!);
    case 'openai': return new OpenAIProvider(config.OPENAI_API_KEY!);
    case 'ollama': return new OllamaProvider(config.OLLAMA_URL!);
    default: throw new Error(`Unknown AI provider: ${config.AI_PROVIDER}`);
  }
}
```

**Testing**:
- `Unit: AnthropicProvider.chat → calls Anthropic API with correct params`
- `Unit: OpenAIProvider.chat → calls OpenAI API with correct params`
- `Unit: createLLMProvider with 'anthropic' → AnthropicProvider instance`
- `Unit: createLLMProvider with unknown → error thrown`
- `Integration (mocked API): chat with valid prompt → ChatResponse with content`

#### 9.2 — Documentation Generator

**What**: Service that analyzes a code repository (via the GitHub integration) and generates comprehensive service documentation using an LLM.

**Design**:

```typescript
// packages/api/src/services/ai/doc-generator.ts
export class DocGenerator {
  constructor(
    private llm: LLMProvider,
    private githubClient: Octokit,
  ) {}

  async generateDocumentation(entity: Entity): Promise<AIDocGeneration> {
    // 1. Fetch repository context
    const repoUrl = entity.properties.repoUrl as string;
    const context = await this.gatherContext(repoUrl);

    // 2. Build LLM prompt
    const prompt = this.buildPrompt(entity, context);

    // 3. Generate documentation
    const response = await this.llm.chat({
      model: config.AI_MODEL,
      systemPrompt: DOC_GENERATION_SYSTEM_PROMPT,
      messages: [{ role: 'user', content: prompt }],
      temperature: 0.3,
      maxTokens: 8192,
    });

    // 4. Persist generation record
    return {
      entityId: entity.id,
      modelName: response.model,
      inputContext: context.summary,
      generatedDoc: response.content,
      confidence: this.assessConfidence(context),
      status: 'pending',
    };
  }

  private async gatherContext(repoUrl: string): Promise<RepoContext> {
    // Extract owner/repo from URL
    // Fetch: README.md, package.json/go.mod, key source files, recent commits, open issues
    // Summarize into a structured context object
    return {
      readme: '...',
      packageInfo: { name: '...', dependencies: {} },
      recentCommits: [],
      fileTree: [],
      summary: {},
    };
  }
}

const DOC_GENERATION_SYSTEM_PROMPT = `You are a technical documentation writer for an internal developer portal.
Given information about a software service (code structure, dependencies, recent changes, and existing docs),
generate comprehensive service documentation in Markdown format.

The documentation should include:
1. Service Overview: what the service does, its purpose, and key stakeholders
2. Architecture: high-level architecture, key components, and design decisions
3. API Reference: endpoints, request/response formats (if applicable)
4. Dependencies: upstream and downstream service dependencies
5. Deployment: how to deploy, environment variables, infrastructure requirements
6. Runbook: common operational procedures, troubleshooting steps
7. Development Guide: how to set up local development, run tests

Write clearly and concisely. Use headings, code blocks, and tables for readability.
Do not fabricate information — if context is insufficient, note what is missing.`;
```

```typescript
// packages/shared/src/types/ai.ts
export interface AIDocGeneration {
  id?: string;
  entityId: string;
  modelName: string;
  inputContext: Record<string, unknown>;
  generatedDoc: string;
  confidence: number;           // 0.0 - 1.0
  status: 'pending' | 'accepted' | 'rejected';
  reviewedBy?: string;
  createdAt?: string;
}
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/entities/:id/generate-docs` | Trigger doc generation for an entity |
| `GET` | `/api/v1/entities/:id/ai-docs` | List generated docs for an entity |
| `POST` | `/api/v1/ai-docs/:id/accept` | Accept a generated doc (publishes it) |
| `POST` | `/api/v1/ai-docs/:id/reject` | Reject a generated doc |

**Testing**:
- `Unit (mocked LLM): generateDocumentation → AIDocGeneration with Markdown content`
- `Unit (mocked GitHub): gatherContext → RepoContext with readme, commits, fileTree`
- `Unit: assessConfidence with rich context → high confidence (>0.8)`
- `Unit: assessConfidence with minimal context → low confidence (<0.5)`
- `Integration (mocked): POST /api/v1/entities/:id/generate-docs → 202 Accepted, job queued`
- `Integration: accept generated doc → doc published as TechDoc`

#### 9.3 — AI Documentation Worker

**What**: BullMQ worker that processes documentation generation requests asynchronously.

**Design**:

```typescript
// packages/api/src/workers/ai.worker.ts
const aiQueue = new Queue('ai-tasks', { connection: redisConnection });

const worker = new Worker('ai-tasks', async (job) => {
  switch (job.name) {
    case 'generate-docs': {
      const entity = await catalogService.getEntity(job.data.entityId);
      const result = await docGenerator.generateDocumentation(entity);
      await aiDocService.save(result);
      break;
    }
    // Future: dependency-analysis, nl-query, etc.
  }
}, {
  connection: redisConnection,
  concurrency: 3,
  limiter: { max: 10, duration: 60000 }, // Rate limit: 10 jobs per minute
});
```

**Testing**:
- `Integration: enqueue generate-docs job → worker processes and AIDocGeneration saved`
- `Integration: rate limiter → max 10 jobs per minute enforced`
- `Unit: worker handles LLM API error gracefully → job retried with backoff`

---

## Phase 10: AI-Powered Dependency Analysis and NL Queries

### Purpose
Implement the second and third AI differentiators: intelligent dependency graph analysis (blast-radius detection) and natural language catalog querying. After this phase, engineers can ask questions like "which services call this deprecated API?" and get blast-radius risk assessments before making changes.

### Tasks

#### 10.1 — Dependency Graph Traversal Service

**What**: Graph traversal service using recursive CTEs on the `entity_relations` table to compute blast radius, shortest paths, and transitive dependency chains.

**Design**:

```typescript
// packages/api/src/services/graph.service.ts
export class GraphService {
  /**
   * Compute blast radius: all entities transitively affected if the root entity fails.
   * Uses a BFS traversal via recursive CTE on entity_relations.
   */
  async computeBlastRadius(entityId: string, maxDepth: number = 5): Promise<BlastRadiusResult> {
    // Recursive CTE: traverse entity_relations where target = current node
    // (find all entities that depend on the root entity, transitively)
    const sql = `
      WITH RECURSIVE blast AS (
        SELECT
          er.source_entity_id AS node_id,
          e.identifier,
          bp.identifier AS blueprint,
          1 AS depth,
          ARRAY[er.source_entity_id] AS path
        FROM entity_relations er
        JOIN entities e ON e.id = er.source_entity_id
        JOIN blueprints bp ON bp.id = e.blueprint_id
        WHERE er.target_entity_id = $1

        UNION ALL

        SELECT
          er.source_entity_id,
          e.identifier,
          bp.identifier,
          b.depth + 1,
          b.path || er.source_entity_id
        FROM blast b
        JOIN entity_relations er ON er.target_entity_id = b.node_id
        JOIN entities e ON e.id = er.source_entity_id
        JOIN blueprints bp ON bp.id = e.blueprint_id
        WHERE b.depth < $2
          AND NOT er.source_entity_id = ANY(b.path)
      )
      SELECT DISTINCT node_id, identifier, blueprint, depth
      FROM blast
      ORDER BY depth, identifier
    `;

    const affected = await db.execute(sql, [entityId, maxDepth]);
    return {
      rootEntityId: entityId,
      affectedEntities: affected.rows,
      totalBlastRadius: affected.rows.length,
      maxDepth: Math.max(...affected.rows.map(r => r.depth), 0),
    };
  }

  async findShortestPath(sourceId: string, targetId: string): Promise<GraphPath | null> {
    // BFS via recursive CTE
  }

  async getDependencyTree(entityId: string, direction: 'upstream' | 'downstream'): Promise<TreeNode> {
    // Recursive CTE in the specified direction
  }
}
```

```typescript
// packages/shared/src/types/ai.ts (continued)
export interface BlastRadiusResult {
  rootEntityId: string;
  affectedEntities: AffectedEntity[];
  totalBlastRadius: number;
  maxDepth: number;
  riskScore?: number;
  analysisType?: string;
}

export interface AffectedEntity {
  nodeId: string;
  identifier: string;
  blueprint: string;
  depth: number;
}
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/entities/:id/blast-radius` | Compute blast radius |
| `GET` | `/api/v1/entities/:id/dependencies` | Dependency tree (upstream/downstream) |
| `GET` | `/api/v1/graph/path` | Shortest path between two entities |

**Testing**:
- `Unit: blast radius with linear chain A→B→C → returns [B, C] with depths [1, 2]`
- `Unit: blast radius with diamond A→B, A→C, B→D, C→D → returns [B, C, D] without duplicates`
- `Unit: blast radius with cycle A→B→A → terminates without infinite loop`
- `Unit: blast radius with maxDepth=1 → only direct dependents`
- `Integration: GET /api/v1/entities/:id/blast-radius → correct affected entities`
- `Fixture: 20-entity dependency graph with known blast-radius patterns`

#### 10.2 — AI-Powered Blast-Radius Analysis

**What**: LLM-enhanced blast-radius analysis that supplements graph traversal with risk scoring, impact narrative, and mitigation suggestions.

**Design**:

```typescript
// packages/api/src/services/ai/dependency-analyzer.ts
export class DependencyAnalyzer {
  constructor(
    private llm: LLMProvider,
    private graphService: GraphService,
  ) {}

  async analyzeBlastRadius(entityId: string): Promise<AIBlastRadiusAnalysis> {
    // 1. Compute graph-based blast radius
    const graphResult = await this.graphService.computeBlastRadius(entityId);

    // 2. Enrich with entity metadata (tier, health status, recent incidents)
    const enrichedEntities = await this.enrichWithMetadata(graphResult.affectedEntities);

    // 3. Use LLM to generate risk assessment
    const analysis = await this.llm.chat({
      model: config.AI_MODEL,
      systemPrompt: BLAST_RADIUS_SYSTEM_PROMPT,
      messages: [{
        role: 'user',
        content: JSON.stringify({
          rootEntity: await this.catalogService.getEntity(entityId),
          affectedEntities: enrichedEntities,
          totalBlastRadius: graphResult.totalBlastRadius,
        }),
      }],
      temperature: 0.2,
    });

    return {
      rootEntityId: entityId,
      graphResult,
      riskScore: this.extractRiskScore(analysis.content),
      narrative: analysis.content,
      mitigations: this.extractMitigations(analysis.content),
    };
  }
}

const BLAST_RADIUS_SYSTEM_PROMPT = `You are a platform engineering expert analyzing the blast radius of a service failure.
Given a root entity and the list of transitively affected entities (with their tiers, health status, and recent incidents),
provide:
1. A risk score from 0-100 (higher = more risk)
2. A concise narrative explaining the key risks
3. Specific mitigation recommendations

Focus on tier-1 services in the blast radius, recent incident patterns, and single points of failure.
Output as JSON: { "riskScore": number, "narrative": string, "mitigations": string[] }`;
```

**Testing**:
- `Unit (mocked LLM): analyzeBlastRadius → AIBlastRadiusAnalysis with riskScore and narrative`
- `Unit: risk score extraction from LLM response → number 0-100`
- `Unit: entity with tier-1 services in blast radius → higher risk score`
- `Integration (mocked): POST /api/v1/entities/:id/analyze-blast-radius → 202, analysis queued`

#### 10.3 — Natural Language Catalog Query

**What**: Natural language query interface that translates English questions about the catalog into SQL queries and returns results.

**Design**:

```typescript
// packages/api/src/services/ai/nl-query.ts
export class NLQueryService {
  constructor(private llm: LLMProvider) {}

  async query(orgId: string, queryText: string, userId: string): Promise<NLQueryResult> {
    // 1. Get schema context (blueprint definitions for this org)
    const blueprints = await this.blueprintService.list(orgId);
    const schemaContext = this.buildSchemaContext(blueprints);

    // 2. Translate NL to SQL using LLM
    const sqlResponse = await this.llm.chat({
      model: config.AI_MODEL,
      systemPrompt: this.buildNLQueryPrompt(schemaContext),
      messages: [{ role: 'user', content: queryText }],
      temperature: 0.1,
    });

    const generatedSql = this.extractSQL(sqlResponse.content);

    // 3. Validate SQL (prevent injection, ensure read-only)
    this.validateSQLSafety(generatedSql);

    // 4. Execute query with read-only transaction
    const startTime = Date.now();
    const results = await db.execute(generatedSql, { readOnly: true });
    const responseTime = Date.now() - startTime;

    // 5. Log query for feedback and improvement
    await this.logQuery(userId, queryText, generatedSql, results.length, responseTime);

    return {
      query: queryText,
      generatedSql,
      results: results.rows,
      resultCount: results.rows.length,
      responseTimeMs: responseTime,
    };
  }

  private validateSQLSafety(sql: string): void {
    const forbidden = ['INSERT', 'UPDATE', 'DELETE', 'DROP', 'ALTER', 'CREATE', 'TRUNCATE', 'GRANT'];
    const upperSql = sql.toUpperCase();
    for (const keyword of forbidden) {
      if (upperSql.includes(keyword)) {
        throw new Error(`Generated SQL contains forbidden keyword: ${keyword}`);
      }
    }
  }
}
```

```typescript
// packages/shared/src/types/ai.ts (continued)
export interface NLQueryResult {
  query: string;
  generatedSql: string;
  results: Record<string, unknown>[];
  resultCount: number;
  responseTimeMs: number;
}
```

API endpoint:

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/query` | Natural language catalog query |

**Testing**:
- `Unit (mocked LLM): "which services use PostgreSQL" → valid SELECT query`
- `Unit: validateSQLSafety with SELECT → passes`
- `Unit: validateSQLSafety with DROP TABLE → throws error`
- `Unit: validateSQLSafety with DELETE → throws error`
- `Integration (mocked LLM): POST /api/v1/query → results returned`
- `Integration: query logged in nl_query_log with response time`
- `Unit: buildSchemaContext → includes blueprint names, property definitions, relation types`

---

## Phase 11: Incident Correlation and Adaptive Scoring

### Purpose
Implement the remaining AI differentiators: automated incident correlation (linking failures to deployments, owners, and runbooks) and adaptive production-readiness scoring that learns from incident patterns. After this phase, the portal actively helps reduce MTTR and continuously improves its engineering standards.

### Tasks

#### 11.1 — Incident Correlation Engine

**What**: Service that automatically correlates incidents to recent deployments, affected services, on-call owners, and relevant runbooks.

**Design**:

```typescript
// packages/api/src/services/ai/incident-correlator.ts
export class IncidentCorrelator {
  async correlate(incidentId: string): Promise<IncidentCorrelation> {
    const incident = await this.incidentService.get(incidentId);
    const affectedEntities = await this.incidentService.getAffectedEntities(incidentId);

    // 1. Find recent deployments to affected entities (4-hour window before incident)
    const recentDeployments = await this.findRecentDeployments(
      affectedEntities.map(e => e.id),
      incident.startedAt,
      4 * 60 * 60, // 4 hours
    );

    // 2. Identify on-call owners for affected entities
    const owners = await this.resolveOwners(affectedEntities);

    // 3. Find relevant runbooks/documentation
    const runbooks = await this.findRelevantDocs(affectedEntities);

    // 4. Compute blast radius from the suspected root cause
    const blastRadius = recentDeployments.length > 0
      ? await this.graphService.computeBlastRadius(recentDeployments[0].entityId)
      : null;

    // 5. Use LLM to generate correlation summary
    const summary = await this.generateCorrelationSummary(
      incident, recentDeployments, owners, runbooks, blastRadius
    );

    return {
      incidentId,
      recentDeployments,
      owners,
      runbooks,
      blastRadius,
      summary: summary.content,
      suggestedActions: this.extractSuggestedActions(summary.content),
    };
  }
}

export interface IncidentCorrelation {
  incidentId: string;
  recentDeployments: DeploymentEvent[];
  owners: OwnerInfo[];
  runbooks: AIDocGeneration[];
  blastRadius: BlastRadiusResult | null;
  summary: string;
  suggestedActions: string[];
}

export interface OwnerInfo {
  entityId: string;
  entityIdentifier: string;
  teamName: string;
  teamSlug: string;
  onCallUser?: string;
}
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/incidents/:id/correlate` | Run correlation analysis |
| `GET` | `/api/v1/incidents/:id/correlation` | Get correlation results |

**Testing**:
- `Unit: correlate with recent deployment → deployment included in recentDeployments`
- `Unit: correlate with no recent deployments → empty recentDeployments, no blast radius`
- `Unit: owners resolved for all affected entities → OwnerInfo populated`
- `Unit (mocked LLM): correlation summary → narrative with suggested actions`
- `Integration: POST /api/v1/incidents/:id/correlate → correlation results saved`
- `Fixture: incident with 3 affected services, 2 recent deployments, known owners`

#### 11.2 — Adaptive Production-Readiness Scoring

**What**: System that analyzes historical incident data to suggest new scorecard rules or adjust existing rule weights, enabling scorecards that improve based on observed failure patterns.

**Design**:

```typescript
// packages/api/src/services/ai/adaptive-scoring.ts
export class AdaptiveScoringService {
  async analyzeAndSuggest(orgId: string, scorecardId: string): Promise<ScorecardSuggestion[]> {
    // 1. Load incident history (last 6 months)
    const incidents = await this.incidentService.listRecent(orgId, 180);

    // 2. Load current scorecard rules
    const scorecard = await this.scorecardService.get(scorecardId);

    // 3. Load entity evaluations and correlate with incidents
    const correlations = await this.buildIncidentCorrelations(incidents, scorecard);

    // 4. Use LLM to identify patterns and suggest rule changes
    const response = await this.llm.chat({
      model: config.AI_MODEL,
      systemPrompt: ADAPTIVE_SCORING_PROMPT,
      messages: [{
        role: 'user',
        content: JSON.stringify({
          currentRules: scorecard.rules,
          incidentPatterns: correlations,
          entityCount: correlations.length,
        }),
      }],
      temperature: 0.3,
    });

    return this.parseSuggestions(response.content);
  }
}

export interface ScorecardSuggestion {
  type: 'add_rule' | 'modify_weight' | 'modify_threshold';
  rationale: string;
  rule?: Partial<ScorecardRule>;
  currentWeight?: number;
  suggestedWeight?: number;
  incidentEvidence: string[];     // incident IDs that support this suggestion
}
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/scorecards/:identifier/analyze` | Generate adaptive suggestions |
| `GET` | `/api/v1/scorecards/:identifier/suggestions` | List pending suggestions |
| `POST` | `/api/v1/scorecards/:identifier/suggestions/:id/apply` | Apply a suggestion |

**Testing**:
- `Unit (mocked LLM): analyze with incidents → ScorecardSuggestion[] with rationale`
- `Unit: incidents correlated with missing runbooks → suggest "has-runbook" rule`
- `Unit: incidents correlated with high lead time → suggest increasing deployment frequency weight`
- `Integration: apply suggestion → scorecard rule updated`

---

## Phase 12: MCP Server, Observability Integration, and Production Hardening

### Purpose
Expose the IDP as an MCP (Model Context Protocol) server for AI coding assistants, integrate OpenTelemetry-based service health data, and harden the application for production deployment. After this phase, the portal is production-ready with full observability, security, and AI-agent interoperability.

### Tasks

#### 12.1 — MCP Server Implementation

**What**: Implement the Model Context Protocol server that exposes catalog entities as MCP resources and self-service actions as MCP tools, enabling AI coding assistants to query and interact with the portal.

**Design**:

```typescript
// packages/api/src/mcp/server.ts
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';

export function createMCPServer(catalogService: CatalogService, actionService: ActionService) {
  const server = new Server({
    name: 'internal-developer-portal',
    version: '1.0.0',
  }, {
    capabilities: {
      resources: {},
      tools: {},
    },
  });

  // Expose catalog entities as resources
  server.setRequestHandler('resources/list', async () => ({
    resources: [
      { uri: 'idp://catalog/services', name: 'Service Catalog', mimeType: 'application/json' },
      { uri: 'idp://catalog/apis', name: 'API Catalog', mimeType: 'application/json' },
      { uri: 'idp://scorecards', name: 'Scorecard Results', mimeType: 'application/json' },
      { uri: 'idp://dora-metrics', name: 'DORA Metrics', mimeType: 'application/json' },
    ],
  }));

  server.setRequestHandler('resources/read', async (request) => {
    // Parse URI and fetch data from catalog
  });

  // Expose actions as tools
  server.setRequestHandler('tools/list', async () => ({
    tools: [
      {
        name: 'search_catalog',
        description: 'Search the service catalog by query or properties',
        inputSchema: { type: 'object', properties: { query: { type: 'string' }, blueprint: { type: 'string' } } },
      },
      {
        name: 'get_blast_radius',
        description: 'Get the blast radius for a service',
        inputSchema: { type: 'object', properties: { entityIdentifier: { type: 'string' } }, required: ['entityIdentifier'] },
      },
      {
        name: 'trigger_action',
        description: 'Trigger a self-service action',
        inputSchema: { type: 'object', properties: { actionIdentifier: { type: 'string' }, inputs: { type: 'object' } } },
      },
    ],
  }));

  server.setRequestHandler('tools/call', async (request) => {
    // Dispatch to appropriate service based on tool name
  });

  return server;
}
```

**Testing**:
- `Unit: resources/list → list of catalog resources`
- `Unit: resources/read with idp://catalog/services → JSON array of services`
- `Unit: tools/list → list of available tools`
- `Unit: tools/call search_catalog → catalog search results`
- `Unit: tools/call get_blast_radius → blast radius data`
- `Integration: MCP client connects → handshake succeeds`

#### 12.2 — OpenTelemetry Service Health Integration

**What**: Ingest service health metrics from OpenTelemetry-compatible sources (Prometheus, Grafana, Datadog) and surface them on entity detail pages.

**Design**:

```typescript
// packages/api/src/services/health.service.ts
export interface ServiceHealthSnapshot {
  entityId: string;
  capturedAt: string;
  errorRate: number;             // percentage
  p50LatencyMs: number;
  p95LatencyMs: number;
  p99LatencyMs: number;
  requestRate: number;           // requests per second
  cpuUsagePct: number;
  memoryUsageMb: number;
  healthStatus: 'healthy' | 'degraded' | 'unhealthy' | 'unknown';
}

export class HealthService {
  async ingestMetrics(entityId: string, metrics: Partial<ServiceHealthSnapshot>): Promise<void> {
    // Determine health status based on thresholds
    const healthStatus = this.classifyHealth(metrics);
    // Insert snapshot
    // Update entity's denormalized health status
  }

  classifyHealth(metrics: Partial<ServiceHealthSnapshot>): string {
    if ((metrics.errorRate ?? 0) > 5) return 'unhealthy';
    if ((metrics.errorRate ?? 0) > 1 || (metrics.p95LatencyMs ?? 0) > 1000) return 'degraded';
    return 'healthy';
  }
}
```

Database table (added to schema):

```sql
CREATE TABLE service_health_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_id       UUID NOT NULL REFERENCES entities(id),
    captured_at     TIMESTAMPTZ NOT NULL,
    error_rate      NUMERIC(7,4),
    p50_latency_ms  NUMERIC(10,2),
    p95_latency_ms  NUMERIC(10,2),
    p99_latency_ms  NUMERIC(10,2),
    request_rate    NUMERIC(12,2),
    cpu_usage_pct   NUMERIC(5,2),
    memory_usage_mb NUMERIC(10,2),
    health_status   VARCHAR(20) NOT NULL DEFAULT 'unknown'
);

CREATE INDEX idx_health_entity ON service_health_snapshots(entity_id, captured_at DESC);
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/entities/:id/health` | Ingest health metrics |
| `GET` | `/api/v1/entities/:id/health` | Get health history |
| `POST` | `/api/v1/webhooks/otel` | OpenTelemetry webhook receiver |

**Testing**:
- `Unit: classifyHealth with errorRate=0.5, p95=200ms → 'healthy'`
- `Unit: classifyHealth with errorRate=3.0 → 'degraded'`
- `Unit: classifyHealth with errorRate=10.0 → 'unhealthy'`
- `Integration: POST /api/v1/entities/:id/health → snapshot persisted`
- `Integration: health ingestion updates entity's denormalized health_status`

#### 12.3 — SCIM 2.0 User Provisioning

**What**: Implement SCIM 2.0 (RFC 7643/7644) endpoints for automated user and group provisioning from enterprise identity providers.

**Design**:

SCIM endpoints per RFC 7644:

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/scim/v2/Users` | List/search users |
| `POST` | `/scim/v2/Users` | Create user |
| `GET` | `/scim/v2/Users/:id` | Get user |
| `PUT` | `/scim/v2/Users/:id` | Replace user |
| `PATCH` | `/scim/v2/Users/:id` | Modify user |
| `DELETE` | `/scim/v2/Users/:id` | Deactivate user |
| `GET` | `/scim/v2/Groups` | List/search groups (teams) |
| `POST` | `/scim/v2/Groups` | Create group |
| `PATCH` | `/scim/v2/Groups/:id` | Modify group membership |

**Testing**:
- `Integration: POST /scim/v2/Users → user created, SCIM response format`
- `Integration: PATCH /scim/v2/Users/:id with active=false → user deactivated`
- `Integration: POST /scim/v2/Groups → team created`
- `Integration: PATCH /scim/v2/Groups/:id with members → team membership updated`
- `Unit: SCIM filter parser → supports eq, co, sw operators`

#### 12.4 — Production Hardening

**What**: Add application-level observability (structured logging, metrics, tracing), rate limiting, request validation, and Docker production build.

**Design**:

```typescript
// Rate limiting
await app.register(import('@fastify/rate-limit'), {
  max: 100,
  timeWindow: '1 minute',
  keyGenerator: (req) => req.session?.userId ?? req.ip,
});

// Request ID propagation
app.addHook('onRequest', (req, reply, done) => {
  req.headers['x-request-id'] ??= crypto.randomUUID();
  reply.header('x-request-id', req.headers['x-request-id']);
  done();
});

// Structured logging with pino
// OpenTelemetry instrumentation
import { NodeSDK } from '@opentelemetry/sdk-node';
const sdk = new NodeSDK({
  serviceName: 'internal-developer-portal',
  // ... exporters
});
```

```dockerfile
# Dockerfile
FROM node:22-alpine AS base
RUN corepack enable

FROM base AS builder
WORKDIR /app
COPY . .
RUN pnpm install --frozen-lockfile
RUN pnpm build

FROM base AS runner
WORKDIR /app
COPY --from=builder /app/packages/api/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3001
CMD ["node", "dist/server.js"]
```

**Testing**:
- `Integration: rate limit exceeded → 429 Too Many Requests`
- `Integration: request without x-request-id → generated and returned in response`
- `Unit: Docker build → image builds successfully`
- `Unit: Docker container starts → health check passes`
- `Integration: structured log output → JSON format with request_id, user_id, duration`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                    ─── required by everything
    │
Phase 2: Blueprint & Entity Catalog   ─── requires Phase 1
    │
Phase 3: Teams, Users, Auth           ─── requires Phase 1 (can parallel Phase 2)
    │
    ├── Phase 4: Scorecards            ─── requires Phase 2 + 3
    │
    ├── Phase 5: DORA Metrics          ─── requires Phase 2 + 3
    │
    └── Phase 6: Self-Service Actions  ─── requires Phase 2 + 3
         │
Phase 7: Integration Engine            ─── requires Phase 2 + 3
    │
Phase 8: Frontend Portal UI            ─── requires Phases 2-6 (can start after Phase 2, iteratively adds as backend phases complete)
    │
Phase 9: AI Documentation              ─── requires Phase 7 (GitHub integration) + Phase 2
    │
Phase 10: AI Dependency & NL Query     ─── requires Phase 2 (entity relations) + Phase 9 (LLM provider)
    │
Phase 11: Incident Correlation         ─── requires Phase 5 (incidents) + Phase 10 (graph service)
    │
Phase 12: MCP, OTel, Hardening         ─── requires all previous phases
```

Parallelism opportunities:
- Phases 2 and 3 can be developed concurrently after Phase 1
- Phases 4, 5, and 6 can be developed concurrently after Phases 2+3
- Phase 8 (frontend) can start after Phase 2 and add UI incrementally as backend features land
- Phases 9, 10, and 11 are sequential (each builds on the previous AI capability)

---

## Definition of Done (per phase)

1. All tasks implemented with code matching the design specifications.
2. All unit tests pass (`pnpm test`).
3. All integration tests pass (using `testcontainers` for PostgreSQL/Redis).
4. ESLint passes with zero errors (`pnpm lint`).
5. TypeScript strict mode compiles with zero errors (`pnpm typecheck`).
6. Prettier formatting passes (`pnpm format:check`).
7. Docker build succeeds (`docker build .`).
8. Feature works end-to-end (manual or E2E test verification).
9. New API endpoints appear in auto-generated OpenAPI 3.1 spec at `/docs`.
10. Database migrations run cleanly on a fresh database.
11. Audit log entries created for all write operations.
12. New environment variables documented in `.env.example`.
13. No secrets or credentials committed to source control.
