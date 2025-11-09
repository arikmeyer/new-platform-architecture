# Windmill + Drizzle + Zod + Neon DB Integration Specification

**Last Updated:** 2025-11-09
**Author:** AI-First Architecture Team
**Status:** Work-in-Progress - Zod 4 & Drizzle 2025 Pattern Exploration

**⚠️ Important**: This document represents evolving technical thinking and implementation patterns. It is **by no means complete or final**, but rather serves as a basis for further exploration, discussion, and implementation planning.

**📋 Related Documents:**
- **[Naming Convention Strategy](./Naming-Convention-Strategy.md)** - Proposed naming conventions for code in this specification

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Architecture Overview](#2-architecture-overview)
3. [Technology Stack Justification](#3-technology-stack-justification)
4. [Infrastructure Setup](#4-infrastructure-setup)
5. [Windmill Configuration](#5-windmill-configuration)
6. [Schema Design & Organization](#6-schema-design--organization)
7. [Shared Code Libraries](#7-shared-code-libraries)
8. [Business Intent Implementation Pattern](#8-business-intent-implementation-pattern)
9. [Query Pattern](#9-query-pattern)
10. [Event Emission & Transactional Outbox](#10-event-emission--transactional-outbox)
11. [Row-Level Security (RLS)](#11-row-level-security-rls)
12. [Migration Management](#12-migration-management)
13. [AI-First Development Workflow](#13-ai-first-development-workflow)
14. [Testing Strategy](#14-testing-strategy)
15. [Performance Optimization](#15-performance-optimization)
16. [CI/CD & Deployment](#16-cicd--deployment)
17. [Complete Examples](#17-complete-examples)
18. [Troubleshooting Guide](#18-troubleshooting-guide)
19. [Best Practices](#19-best-practices)
20. [Reference](#20-reference)
21. [AI Tool Integration](#21-ai-tool-integration)

---

## 1. Executive Summary

This specification defines how **Drizzle ORM**, **Zod validation**, and **Neon PostgreSQL** integrate within the **Windmill platform** to implement SwitchUp's **entire platform architecture**, with primary focus on the /lifecycle domain (System of Record).

### Scope

While this spec provides detailed implementation for the /lifecycle domain, **the same patterns apply across all 7 SwitchUp domains**:
- **Database**: PostgreSQL + Drizzle + Neon (no MongoDB)
- **Validation**: Zod schemas with drizzle-zod auto-generation
- **Type Safety**: TypeScript with cross-domain type sharing
- **Execution**: Mix of Windmill Scripts (atomic operations) and Flows (multi-step orchestration)

**Critical Understanding**: **Flows are the predominant structure** across 6 of 7 domains. Only /lifecycle is primarily scripts (95%). All other domains use 60-90% flows for RPA automation, LLM chains, data pipelines, and business process orchestration.

### Key Design Principles

1. **Serverless-First**: Optimized for Windmill's stateless, distributed execution model (scripts AND flows)
2. **HTTP-Based Connections**: Neon's `neon-http` driver (GA 1.0+) for optimal serverless performance
3. **AI-Friendly**: Zod 4 metadata system enables 85% AI code generation with rich context
4. **Type-Safe**: TypeScript + Drizzle + Zod 4 provide compile-time and runtime safety (14× faster validation)
5. **Single Source of Truth**: Drizzle schema auto-generates Zod validation and TypeScript types
6. **Transactionally Consistent**: All state changes are atomic with event emission
7. **Cross-Domain Type Sharing**: Lifecycle exports PUBLIC types, other domains import (never direct DB access)
8. **Performance-Optimized**: Native caching with Upstash Redis (opt-in, 3-5× improvement)

### Why This Stack?

| Requirement         | Solution                                      | Windmill Synergy                   |
| ------------------- | --------------------------------------------- | ---------------------------------- |
| Fast cold starts    | Drizzle + neon-http (lightweight, 57% smaller bundle) | Serverless script execution        |
| Stateless execution | HTTP driver (no persistent connections)       | Matches Windmill's execution model |
| Shared schema       | Relative imports                              | Windmill auto-tracks dependencies  |
| Type safety         | TypeScript + Drizzle + Zod 4                  | Compile + runtime validation (14× faster) |
| AI velocity         | Zod 4 metadata + consistent patterns          | 60% less code via AI generation    |
| Query performance   | Drizzle native caching                        | Read-heavy lifecycle queries       |
| Error resilience    | safeParse + Windmill error handlers           | Graceful degradation               |
| Serverless DB       | Neon PostgreSQL HTTP API                      | Perfect for short-lived scripts    |

### Expected Outcomes

- **60 Business Intents** implemented in **45 hours** (AI + human review)
- **Type safety**: 100% coverage (compile-time + runtime)
- **AI error rate**: 15-20% (vs 40-50% with other stacks)
- **Development velocity**: 3× faster than polyglot alternatives
- **Maintenance cost**: $25K vs $78K over 3 years (3.1× savings)

---

## 2. Architecture Overview

### Windmill Execution Model

Windmill is a **serverless workflow engine** where:
- **Scripts** are stateless functions with `main()` entry points (database-stored with paths like `f/lifecycle/command/report-price-increase`)
- **Flows** are single JSON definitions (FlowValue) orchestrating multiple steps across distributed workers
- **Resources** (DB credentials) are injected as parameters
- **Dependencies** are auto-resolved from imports (relative imports work via database resolution)
- **Shared code** works via relative imports between scripts (e.g., `import type { Contract } from '../../../lifecycle/lib/types'`)

### Flow vs Script Distribution Across SwitchUp Domains

**IMPORTANT**: Flows are the predominant structure across nearly all domains. This integration spec focuses on the /lifecycle domain (script-heavy), but the patterns here apply equally to scripts referenced by flows in other domains.

| Domain           | Scripts | Flows | Primary Use                                                  |
| ---------------- | ------- | ----- | ------------------------------------------------------------ |
| **Lifecycle**    | 95%     | 5%    | Transactional state changes (atomicity required)             |
| **Case**         | 10%     | 90%   | Pure orchestration (calls multiple tool domains)             |
| **Provider**     | 30%     | 70%   | RPA workflows (bot automation, multi-step portal operations) |
| **Offer**        | 40%     | 60%   | Data pipelines (ingestion → normalization → matching)        |
| **Optimisation** | 40%     | 60%   | Multi-step ML workflows, complex calculation chains          |
| **Growth**       | 30%     | 70%   | Content generation, A/B testing, onboarding funnels          |
| **Service**      | 30%     | 70%   | Email processing, LLM chains, multi-step integrations        |

**Examples of complex flows in other domains**:
- **Provider RPA**: reset password (bot) → fetch password reset email → extract link → login (bot) → scrape data → download PDFs → submit meter count
- **Service Email**: fetch emails → cleanse (LLM) → store memory → classify (subflow with multiple LLM calls) → route to case domain
- **Optimisation**: fetch contracts → calculate savings projections → run ML models → rank alternatives → generate recommendations

**Flow Structure**: Each flow is a single JSON definition containing an array of steps (FlowModule[]). Steps can reference scripts (which use the patterns in this spec), include inline code, or call other flows.

### Why neon-http is Perfect for Windmill

| Windmill Characteristic        | neon-http Advantage                             |
| ------------------------------ | ----------------------------------------------- |
| Short-lived scripts (1-30s)    | HTTP optimized for single queries               |
| No persistent state            | Stateless HTTP (no connection management)       |
| Fresh execution per invocation | No WebSocket handshake overhead                 |
| Distributed workers            | Each HTTP request is independent                |
| Single transaction per script  | HTTP faster than WebSocket for one-shot queries |

**From Neon docs**: *"Querying over HTTP is faster for single, non-interactive transactions"*

### Cross-Domain Database Strategy

**PostgreSQL + Drizzle + Neon for ALL domains** (no MongoDB):

| Domain           | Has Own DB?   | Stores                                                                              | Accesses Lifecycle DB?     |
| ---------------- | ------------- | ----------------------------------------------------------------------------------- | -------------------------- |
| **Lifecycle**    | Yes (PRIMARY) | Contract, User, Task, Events (authoritative source of truth)                        | N/A (owns the data)        |
| **Provider**     | Yes           | Provider configurations, portal credentials, RPA state, scraping results (JSONB)    | Via lifecycle scripts only |
| **Offer**        | Yes           | Normalized offers, ingestion metadata, matching rules, market data (JSONB)          | Via lifecycle scripts only |
| **Optimisation** | Yes           | ML model outputs, calculation results, A/B test configurations, projections (JSONB) | Via lifecycle scripts only |
| **Case**         | Maybe         | Process execution state, routing decisions (mostly calls other domains)             | Via lifecycle scripts only |
| **Growth**       | Yes           | Content generation results, funnel analytics, personalization state (JSONB)         | Via lifecycle scripts only |
| **Service**      | Yes           | Email/conversation memory, LLM chain state, classification results (JSONB)          | Via lifecycle scripts only |

**Why PostgreSQL Everywhere?**:
1. **JSONB for flexibility**: Handles dynamic/semi-structured data (provider configs, offer metadata, LLM outputs)
2. **Consistency**: Same tooling, same patterns, same deployment across all domains
3. **Windmill optimization**: neon-http driver perfect for both scripts and flow steps
4. **Type safety**: drizzle-zod auto-generates validation for all schemas
5. **Cost efficiency**: Single vendor (Neon), simpler ops than polyglot persistence

**Database Access Rules** (Hub and Spoke):
- ✅ **Other domains call /lifecycle scripts** to read/write core entities (Contract, User, Task)
- ✅ **Each domain manages its own database** for domain-specific data
- ❌ **Never direct database access** from one domain to another's database
- ❌ **No lateral communication** between tool domains (only via /case or /lifecycle)

**Example - Provider Domain Schema**:
```typescript
// f/provider/lib/schema.ts
export const providerPortalConfigs = pgTable('provider_portal_configs', {
  id: uuid('id').primaryKey().defaultRandom(),
  providerId: uuid('provider_id').notNull(), // Reference to external provider system
  portalUrl: varchar('portal_url', { length: 500 }).notNull(),
  credentials: jsonb('credentials').notNull(), // Encrypted credentials (JSONB)
  scraperConfig: jsonb('scraper_config'), // Bot automation config (JSONB)
  lastSuccessfulLogin: timestamp('last_successful_login'),
  createdAt: timestamp('created_at').notNull().defaultNow(),
})

export const rpaExecutionLogs = pgTable('rpa_execution_logs', {
  id: uuid('id').primaryKey().defaultRandom(),
  contractId: uuid('contract_id').notNull(), // Links to lifecycle.contracts (logical reference)
  providerId: uuid('provider_id').notNull(),
  executionType: varchar('execution_type', { length: 100 }).notNull(), // 'scrape', 'submit_meter', etc.
  executionState: jsonb('execution_state').notNull(), // Flow state snapshot (JSONB)
  result: jsonb('result'), // Scraped data or execution outcome (JSONB)
  status: varchar('status', { length: 20 }).notNull(),
  createdAt: timestamp('created_at').notNull().defaultNow(),
})
```

**Shared type definitions via `lib/types` (PUBLIC) and `lib/schema` (PRIVATE)**

---

## 3. Technology Stack Justification

### Why Zod 4?

**Performance Benchmarks vs Zod 3:**

| Metric | Zod 4 | Zod 3 | Improvement |
|--------|-------|-------|-------------|
| String parsing | Baseline | 14× slower | 1400% |
| Array parsing | Baseline | 7× slower | 700% |
| Object parsing | Baseline | 6.5× slower | 650% |
| Bundle size | 4.2KB | 7.4KB | 57% smaller |
| Cold start | <5ms | <10ms | 50% faster |

**Key Features for SwitchUp:**
- **z.interface()**: Precise optionality control (key optional vs value optional)
- **Metadata System**: Rich context for AI agents and LLM function calling
- **Performance**: 14× faster validation (critical for Windmill script performance)
- **Codecs**: Bi-directional transformations (encode/decode for API layers)
- **Enhanced Errors**: Better formatting (flatten, treeify, prettify) for debugging
- **JSON Schema**: Native conversion for OpenAPI and structured LLM outputs

**Choice: Zod 4** because:
1. 14× faster validation reduces script execution time
2. Metadata system enables AI agent context enrichment
3. z.interface() provides precise TypeScript alignment for business intents
4. Native JSON Schema generation for LLM integration
5. 57% smaller bundle → faster Windmill deployment

---

### Why Drizzle ORM 0.44.7?

**Comparison Matrix:**

| Feature             | Drizzle 0.44.7 | Prisma          | sqlc    | Diesel |
| ------------------- | -------------- | --------------- | ------- | ------ |
| Bundle size         | 50KB ✅         | 10MB ❌          | N/A     | N/A    |
| Cold start          | <10ms ✅        | 100ms+ ❌        | <10ms ✅ | 65ms ⚠️ |
| HTTP driver support | ✅              | ❌               | ❌       | ❌      |
| Pessimistic locking | ✅              | ❌               | ✅       | ✅      |
| Auto-generation     | drizzle-zod ✅  | Prisma Client ⚠️ | sqlc ✅  | N/A    |
| Native caching      | ✅ (0.44.0+)    | ❌               | ❌       | ❌      |
| Serverless-first    | ✅              | ⚠️               | ⚠️       | ❌      |

**2025 Features:**
- **Native Caching** (0.44.0): Opt-in Upstash Redis integration with automatic invalidation
- **DrizzleQueryError** (0.44.0): Enhanced error handling exposing SQL, params, and stack traces
- **Read Replicas** (0.29.0+): Automatic query routing with weighted selection
- **Identity Columns** (0.32.0+): PostgreSQL-recommended replacement for serial types
- **Generated Columns** (0.32.0+): Computed columns with automatic updates

**Winner: Drizzle 0.44.7** because:
1. Native `neon-http` adapter (perfect for Windmill)
2. Lightweight (critical for cold starts)
3. No build step (schema is TypeScript code)
4. SQL-like syntax (leverages AI's SQL knowledge)
5. Native pessimistic locking (`.for('update')`)
6. Native caching (3-5× performance improvement for read-heavy queries)

### Why Neon HTTP Driver?

**Neon offers TWO drivers:**

1. **`neon-http`** (HTTP adapter) ← **RECOMMENDED FOR WINDMILL**
   - Optimized for serverless/edge environments
   - Faster for single, non-interactive transactions
   - No WebSocket connection overhead
   - Stateless (matches Windmill's model)

2. **`neon-serverless`** (WebSocket adapter)
   - For long-running applications
   - Supports interactive transactions
   - Connection pooling
   - NOT needed for Windmill

**Choice: `neon-http`** because Windmill scripts are short-lived with single transactions.

---

## 4. Infrastructure Setup

### 4.1 Neon Database Architecture

#### Primary Database (Production)

```bash
# Database name
switchup_production

# Connection string (with security parameters)
postgres://app_user:password@ep-xyz.us-east-2.aws.neon.tech:5432/switchup_production?sslmode=require&channel_binding=require
```

**Key points:**
- **`sslmode=require`**: Enforces SSL/TLS encryption
- **`channel_binding=require`**: Additional security layer (prevents MITM attacks)
- **No `-pooler` suffix needed**: HTTP driver doesn't use connection pooling

**Configuration:**
- **Compute tier**: Autoscaling (0.25 - 2 CU)
- **Storage**: Auto-scaling
- **Backup retention**: 7 days (point-in-time recovery)

#### Branch Databases (Migration Testing)

```bash
# Create branch for migration testing
neon branch create --name migration-test-2024-01

# Get branch connection string
neon connection-string migration-test-2024-01

# Result:
postgres://app_user:password@ep-branch-xyz.us-east-2.aws.neon.tech/switchup_production?sslmode=require&channel_binding=require
```

**Benefits:**
- Instant copy-on-write snapshot of production
- Test migrations on production-scale data
- Zero impact on production database
- Delete after testing (no cost)

### 4.2 Database User Roles

```sql
-- Application user (limited permissions)
CREATE ROLE app_user WITH LOGIN PASSWORD 'secure_password';
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_user;

-- Migration user (elevated permissions)
CREATE ROLE migration_user WITH LOGIN PASSWORD 'secure_password';
GRANT ALL PRIVILEGES ON SCHEMA public TO migration_user;

-- Read-only user (for analytics)
CREATE ROLE readonly_user WITH LOGIN PASSWORD 'secure_password';
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_user;
```

**CRITICAL**: Never use `neondb_owner` role for application connections—it bypasses Row-Level Security!

---

## 5. Windmill Configuration

### 5.1 PostgreSQL Resources

Windmill PostgreSQL resource type:

```typescript
type Postgresql = {
  host: string
  port: number
  user: string
  password: string
  dbname: string
  sslmode: string
  root_certificate_pem?: string
}
```

#### Production Resource

**Name:** `postgresql_production`

**Configuration:**
```json
{
  "host": "ep-xyz.us-east-2.aws.neon.tech",
  "port": 5432,
  "user": "app_user",
  "password": "***",
  "dbname": "switchup_production",
  "sslmode": "require"
}
```

**Note**: No `-pooler` suffix needed with `neon-http` driver.

---

## 6. Schema Design & Organization

### 6.1 Complete Schema with RLS

```typescript
import { pgTable, uuid, varchar, integer, timestamp, boolean, jsonb, pgEnum, index } from 'drizzle-orm/pg-core'
import { sql } from 'drizzle-orm'
import { crudPolicy, authenticatedRole, authUid } from 'drizzle-orm/neon'

// ========== ENUMS ==========
export const contractStatus = pgEnum('contract_status', [
  'pending_verification',
  'active',
  'switch_pending',
  'price_increase_reported',
  'cancelled',
  'expired',
  'archived'
])

export const taskStatus = pgEnum('task_status', [
  'pending',
  'assigned',
  'in_progress',
  'completed',
  'failed',
  'cancelled'
])

// ========== CORE ENTITIES WITH RLS ==========

export const users = pgTable('users', {
  // Identity column (PostgreSQL recommended over serial)
  id: integer('id').primaryKey().generatedAlwaysAsIdentity({
    startWith: 1000
  }),

  // UUID for distributed system compatibility
  uuid: uuid('uuid').defaultRandom().unique().notNull(),

  email: varchar('email', { length: 255 }).notNull().unique(),
  passwordHash: varchar('password_hash', { length: 255 }).notNull(),
  firstName: varchar('first_name', { length: 100 }).notNull(),
  lastName: varchar('last_name', { length: 100 }).notNull(),
  createdAt: timestamp('created_at').notNull().defaultNow(),
  updatedAt: timestamp('updated_at').notNull().defaultNow(),
}, (table) => [
  index('users_email_idx').on(table.email),

  // RLS: Users can only access their own data
  crudPolicy({
    role: authenticatedRole,
    read: authUid(table.uuid),
    modify: authUid(table.uuid),
  })
])

export const contracts = pgTable('contracts', {
  // Identity column (PostgreSQL recommended)
  id: integer('id').primaryKey().generatedAlwaysAsIdentity(),

  // UUID for distributed system compatibility
  uuid: uuid('uuid').defaultRandom().unique().notNull(),

  // Foreign keys to integer IDs
  userId: integer('user_id').notNull().references(() => users.id),
  status: contractStatus('status').notNull(),
  providerId: uuid('provider_id').notNull(),

  // Contract details
  serviceType: varchar('service_type', { length: 50 }).notNull(),
  startDate: timestamp('start_date').notNull(),
  endDate: timestamp('end_date'),
  price: integer('price').notNull(), // Price in cents

  // Price increase tracking
  pendingPriceChange: integer('pending_price_change'),
  pendingPriceEffectiveDate: timestamp('pending_price_effective_date'),

  // Generated column - effective price (computed automatically)
  effectivePrice: integer('effective_price').generatedAlwaysAs(
    (): SQL => sql`COALESCE(${contracts.pendingPriceChange}, ${contracts.price})`
  ),

  // Metadata (JSONB for flexibility)
  externalContractId: varchar('external_contract_id', { length: 255 }),
  metadata: jsonb('metadata').$type<{
    externalId?: string
    documentHash?: string
    providerSpecific?: Record<string, unknown>
  }>(),

  // Audit
  createdAt: timestamp('created_at').notNull().defaultNow(),
  updatedAt: timestamp('updated_at').notNull().defaultNow(),
}, (table) => [
  index('contracts_user_id_idx').on(table.userId),
  index('contracts_status_idx').on(table.status),
  index('contracts_user_status_idx').on(table.userId, table.status),

  // RLS: Users can only access their own contracts
  crudPolicy({
    role: authenticatedRole,
    read: authUid(table.userId),
    modify: authUid(table.userId),
  })
])

export const tasks = pgTable('tasks', {
  id: uuid('id').primaryKey().defaultRandom(),
  type: varchar('type', { length: 100 }).notNull(),
  status: taskStatus('status').notNull().default('Pending'),
  priority: integer('priority').notNull().default(5),

  // Associations
  contractId: uuid('contract_id').references(() => contracts.id),
  assignedTo: uuid('assigned_to').references(() => users.id),

  // Task data
  inputData: jsonb('input_data').notNull(),
  resolutionData: jsonb('resolution_data'),

  // Originating process (for callback)
  originatingProcessName: varchar('originating_process_name', { length: 255 }),
  originatingFlowRunId: varchar('originating_flow_run_id', { length: 255 }),

  // Timestamps
  createdAt: timestamp('created_at').notNull().defaultNow(),
  completedAt: timestamp('completed_at'),
}, (table) => [
  index('tasks_contract_id_idx').on(table.contractId),
  index('tasks_status_idx').on(table.status),

  // RLS: Access via contract ownership
  crudPolicy({
    role: authenticatedRole,
    read: sql`EXISTS (
      SELECT 1 FROM ${contracts}
      WHERE ${contracts.id} = ${table.contractId}
      AND ${contracts.userId} = auth.user_id()
    )`,
    modify: sql`EXISTS (
      SELECT 1 FROM ${contracts}
      WHERE ${contracts.id} = ${table.contractId}
      AND ${contracts.userId} = auth.user_id()
    )`,
  })
])

// ========== AUDIT TABLES ==========

export const priceIncreaseEvents = pgTable('price_increase_events', {
  id: uuid('id').primaryKey().defaultRandom(),
  contractId: uuid('contract_id').notNull().references(() => contracts.id),
  oldPrice: integer('old_price').notNull(),
  newPrice: integer('new_price').notNull(),
  effectiveDate: timestamp('effective_date').notNull(),
  reportedBy: uuid('reported_by').notNull(),
  notificationMethod: varchar('notification_method', { length: 50 }),
  createdAt: timestamp('created_at').notNull().defaultNow(),
})

// ========== TRANSACTIONAL OUTBOX ==========

export const outboxEvents = pgTable('outbox_events', {
  id: uuid('id').primaryKey().defaultRandom(),
  eventType: varchar('event_type', { length: 100 }).notNull(),
  entityType: varchar('entity_type', { length: 50 }).notNull(),
  entityId: uuid('entity_id').notNull(),
  payload: jsonb('payload').notNull(),
  status: varchar('status', { length: 20 }).notNull().default('pending'),
  publishedAt: timestamp('published_at'),
  createdAt: timestamp('created_at').notNull().defaultNow(),
}, (table) => [
  index('outbox_events_status_idx').on(table.status),
  index('outbox_events_created_at_idx').on(table.createdAt),
])
```

---

## 7. Shared Code Libraries

### 7.0 Cross-Domain Type Sharing

**Context**: The /lifecycle domain owns core business entity definitions (Contract, User, Task). Other domains need to reference these types without directly accessing the database.

**Pattern**: Each domain exposes PUBLIC types and keeps implementation PRIVATE.

**Directory Structure** (per domain):
```
f/lifecycle/
├── lib/
│   ├── types.ts         ← PUBLIC: Type definitions exported to other domains
│   ├── schema.ts        ← PRIVATE: Drizzle schema (only lifecycle imports)
│   ├── db.ts           ← PRIVATE: Database connection helpers
│   ├── validation.ts   ← PRIVATE: Zod schemas for lifecycle commands
│   └── events.ts       ← PRIVATE: Event creation utilities
├── contract/
│   ├── report-price-increase.ts    ← Script (called by flows or other scripts)
│   └── get-details.ts              ← Script
└── query/
    └── get-contract-timeline.ts    ← Script
```

**Type Export Pattern** (`f/lifecycle/lib/types.ts`):

```typescript
// Export types derived from Drizzle schema without exposing schema itself
import { contracts, users, tasks } from './schema'

// Infer types from schema
export type Contract = typeof contracts.$inferSelect
export type ContractInsert = typeof contracts.$inferInsert
export type User = typeof users.$inferSelect
export type Task = typeof tasks.$inferSelect

// Export enums (safe to share)
export { contractStatus, taskStatus } from './schema'

// DO NOT export: schema tables, database connection, validation schemas
```

**Type Import Pattern** (from other domains):

```typescript
// In a Provider domain script: f/provider/portal/scrape-contract-details.ts
import type { Contract, ContractInsert } from '../../../lifecycle/lib/types'

// Flow step references this script, receives Contract data, uses types for validation
export async function main(contractId: string): Promise<Contract> {
  // Call /lifecycle script to get contract (not direct DB access)
  const contract = await wmill.call('f/lifecycle/query/get-contract-details', { contractId })

  // Use Contract type for type safety
  return contract as Contract
}
```

**In Flows**: Flow definitions (JSON) reference typed scripts. Type safety is maintained at the script level.

```json
{
  "summary": "Provider Portal Scraper Flow",
  "value": {
    "modules": [
      {
        "id": "a",
        "value": {
          "type": "script",
          "path": "f/lifecycle/query/get-contract-details",
          "input_transforms": {
            "contractId": { "type": "javascript", "expr": "flow_input.contractId" }
          }
        }
      },
      {
        "id": "b",
        "value": {
          "type": "script",
          "path": "f/provider/portal/scrape-contract-details",
          "input_transforms": {
            "contractId": { "type": "javascript", "expr": "results.a.id" }
          }
        }
      }
    ]
  }
}
```

**Shared Primitives** (for all domains):

```typescript
// f/shared/validation/primitives.ts
import { z } from 'zod'

// Top-level validators (Zod 4 - 14× faster)
export const emailSchema = z.email()  // Native top-level validator
export const uuidSchema = z.uuid()    // Native top-level validator
export const hashSchema = z.hash("sha256", "hex")  // Native top-level validator
export const positiveIntegerSchema = z.number().int().positive()

// Refined validators (Zod 4 unified error parameter)
export const futureDateSchema = z.coerce.date().refine(
  (date) => date > new Date(),
  { error: { message: 'Date must be in the future' } }  // Zod 4: unified 'error' parameter
)
```

**Golden Rule**:
- **Other domains import types from /lifecycle** to maintain type safety
- **Other domains call /lifecycle scripts** to access/mutate core business entities
- **Never direct database access** from other domains to lifecycle database
- **Each domain may have its own database** for domain-specific data (using same Drizzle + Neon pattern)

---

### 7.1 Database Connection Helper (`/lifecycle/lib/db.ts`)

```typescript
import { neon } from '@neondatabase/serverless'
import { drizzle } from 'drizzle-orm/neon-http'
import type { NeonHttpDatabase, DrizzleConfig } from 'drizzle-orm/neon-http'
import { upstashCache } from 'drizzle-orm/cache/upstash'
import * as schema from './schema'

/**
 * Windmill PostgreSQL Resource type
 */
export type PostgresqlResource = {
  host: string
  port: number
  user: string
  password: string
  dbname: string
  sslmode: string
  root_certificate_pem?: string
}

/**
 * Create a Drizzle database connection using Neon's HTTP driver
 *
 * IMPORTANT: This uses the neon-http driver which is optimized for
 * serverless environments like Windmill. Each query is an independent
 * HTTP request - no persistent connections or pooling needed.
 *
 * @param resource - Windmill-injected PostgreSQL resource
 * @param options - Optional configuration
 * @param options.cache - Enable native caching with Upstash Redis (default: false)
 * @returns Drizzle database instance with schema
 */
export function createDbConnection(
  resource: PostgresqlResource,
  options?: { cache?: boolean }
): NeonHttpDatabase<typeof schema> {
  // Build connection string with security parameters
  const connectionString = `postgres://${resource.user}:${resource.password}@${resource.host}:${resource.port}/${resource.dbname}?sslmode=${resource.sslmode}&channel_binding=require`

  // Create Neon HTTP client (stateless)
  const sql = neon(connectionString)

  // Configure Drizzle with optional caching
  const config: DrizzleConfig<typeof schema> = { schema }

  if (options?.cache) {
    // Enable native caching with Upstash Redis
    config.cache = upstashCache({
      global: false, // Opt-in per query
      config: { ex: 300 } // Default TTL: 5 minutes
    })
  }

  // Return Drizzle instance with schema and optional caching
  return drizzle(sql, config)
}

/**
 * Create authenticated connection with JWT token for RLS
 *
 * @param resource - Windmill PostgreSQL resource
 * @param authToken - JWT token from authentication
 * @returns Authenticated Drizzle instance
 */
export function createAuthenticatedDbConnection(
  resource: PostgresqlResource,
  authToken: string
): NeonHttpDatabase<typeof schema> {
  const db = createDbConnection(resource)
  return db.$withAuth(authToken)
}
```

### 7.2 Validation Layer (`/lifecycle/lib/validation.ts`)

```typescript
import { z } from 'zod'
import { createSelectSchema, createInsertSchema, createUpdateSchema } from 'drizzle-zod'
import { contracts, users, tasks } from './schema'

// ========== EXTEND GLOBAL REGISTRY (for AI integration) ==========

declare module 'zod' {
  namespace z {
    interface GlobalRegistryMetadata {
      // JSON Schema standard
      id?: string
      title?: string
      description?: string
      examples?: any[]
      deprecated?: boolean

      // AI Agent Context
      context?: string
      businessDomain?: string
      validationPriority?: 'critical' | 'important' | 'optional'

      // Lifecycle-specific
      relatedEntities?: string[]
      queryComplexity?: 'simple' | 'moderate' | 'expensive'
      cacheable?: boolean
    }
  }
}

// ========== ENTITY CRUD SCHEMAS (z.object from drizzle-zod) ==========

// Contract schemas with all validation upfront
export const contractSelectSchema = createSelectSchema(contracts)

export const contractInsertSchema = createInsertSchema(contracts, {
  // Callback refinement: Extend existing validation
  serviceType: (schema) => schema.serviceType.max(50),

  // Zod 4 validators with metadata
  price: z.number().int().positive({
    error: (issue) => {
      const value = issue.input as number
      if (value <= 0) return `Price must be positive. Received ${value}.`
      if (value > 1000000) return `Price unusually high (£${(value/100).toFixed(2)})`
      return "Invalid price"
    }
  }).meta({
    title: "Monthly Price (cents)",
    description: "Contract monthly price in cents",
    examples: [10000, 15000, 20000],
    context: "Typical: £50-£500/month (5000-50000 cents)",
    validationPriority: "critical"
  }),

  // Transform metadata to specific structure
  metadata: z.object({
    externalId: z.string().optional(),
    documentHash: z.hash("sha256", "hex").optional(),
    providerSpecific: z.record(z.unknown()).optional(),
  }).optional(),
})

export const contractUpdateSchema = createUpdateSchema(contracts, {
  serviceType: (schema) => schema.serviceType.max(50),
  price: (schema) => schema.price.positive('Price must be positive'),
})

// User schemas with all validation upfront
export const userSelectSchema = createSelectSchema(users)

export const userInsertSchema = createInsertSchema(users, {
  // Zod 4 top-level validators
  email: z.email().max(255).toLowerCase().meta({
    title: "Email Address",
    description: "User's primary email",
    context: "Used for notifications and account recovery",
    validationPriority: "critical"
  }),

  // Complete overwrite: Replace with custom validation
  passwordHash: z.string().min(60).max(60).meta({
    description: "bcrypt hash",
    context: "Never store plain passwords"
  }),
})

export const userUpdateSchema = createUpdateSchema(users, {
  email: (schema) => schema.email.max(255).toLowerCase(),
})

// Task schemas with all validation upfront
export const taskSelectSchema = createSelectSchema(tasks)
export const taskInsertSchema = createInsertSchema(tasks)
export const taskUpdateSchema = createUpdateSchema(tasks)

// ========== BUSINESS INTENT SCHEMAS (z.interface for precise optionality) ==========

export const reportPriceIncreaseSchema = z.interface({
  contractId: z.uuid().meta({
    id: "contract_id",
    title: "Contract ID",
    description: "UUID of contract receiving price increase",
    context: "Must be existing active contract. Query lifecycle to verify.",
    businessDomain: "lifecycle",
    validationPriority: "critical"
  }),

  newPrice: z.number().int().positive().meta({
    title: "New Price (cents)",
    description: "New monthly price in cents",
    examples: [12000, 15000, 20000],
    context: "Must be higher than current. Typical: 5-30% increase.",
    validationPriority: "critical"
  }),

  effectiveDate: z.coerce.date().refine(
    (date) => date > new Date(),
    { error: { message: 'Effective date must be in future' } }
  ).meta({
    title: "Effective Date",
    description: "When increase takes effect",
    context: "UK regulations: 30 days energy, 14 days telco",
    validationPriority: "critical"
  }),

  // KEY OPTIONAL (property can be omitted)
  "notificationMethod?": z.enum(['email', 'letter', 'portal']).meta({
    title: "Notification Method",
    description: "How user was notified",
    context: "Email 70%, letter 25%, portal 5%",
    validationPriority: "important"
  }),

  // VALUE OPTIONAL (property required, value can be undefined)
  documentHash: z.hash("sha256", "hex").optional().meta({
    title: "Document Hash",
    description: "SHA-256 of notification document",
    context: "Optional but recommended for audit trail",
    validationPriority: "optional"
  }),

  reportedBy: z.uuid().meta({
    title: "Reporter ID",
    description: "User or system that reported increase",
    validationPriority: "critical"
  })
})

export const createTaskSchema = z.interface({
  type: z.enum(['price_optimization', 'cancellation', 'activation_check', 'renewal']),
  priority: z.number().int().min(1).max(10).default(5),

  "contractId?": z.uuid(),
  inputData: z.record(z.unknown()),
  "originatingProcessName?": z.string(),
  "originatingFlowRunId?": z.string(),
})

export const completeTaskSchema = z.interface({
  taskId: z.uuid().meta({
    context: "Must reference existing pending or in_progress task"
  }),
  resolutionData: z.record(z.unknown()),
  completedBy: z.uuid(),
})

// ========== CODECS (for API layer bi-directional transformations) ==========

export const dateCodec = z.codec({
  decode: (input: string) => new Date(input),
  encode: (output: Date) => output.toISOString()
})

export const apiContractSchema = z.interface({
  id: z.uuid(),
  startDate: dateCodec,
  "endDate?": dateCodec,
  // ... other fields
})

// ========== TYPE INFERENCE ==========

export type ReportPriceIncreaseParams = z.infer<typeof reportPriceIncreaseSchema>
export type CreateTaskParams = z.infer<typeof createTaskSchema>
export type CompleteTaskParams = z.infer<typeof completeTaskSchema>

export type Contract = z.infer<typeof contractSelectSchema>
export type ContractInsert = z.infer<typeof contractInsertSchema>
export type ContractUpdate = z.infer<typeof contractUpdateSchema>

export type User = z.infer<typeof userSelectSchema>
export type Task = z.infer<typeof taskSelectSchema>
```

### 7.3 Event Utilities (`/lifecycle/lib/events.ts`)

```typescript
import { outboxEvents } from './schema'
import type { NeonHttpDatabase } from 'drizzle-orm/neon-http'
import * as schema from './schema'

/**
 * Canonical event envelope structure
 */
export interface LifecycleEvent {
  eventId: string
  eventType: string
  eventSource: string
  eventVersion: string
  timestamp: string
  traceId: string
  entity: {
    id: string
    type: string
  }
  payload: {
    transition?: {
      fromStatus?: string
      toStatus?: string
    }
    context: Record<string, unknown>
  }
}

/**
 * Emit event to transactional outbox
 *
 * IMPORTANT: This function must be called within a transaction
 * to ensure atomicity between state changes and event emission.
 */
export async function emitEvent(
  tx: Parameters<Parameters<NeonHttpDatabase<typeof schema>['transaction']>[0]>[0],
  event: LifecycleEvent
): Promise<void> {
  await tx.insert(outboxEvents).values({
    eventType: event.eventType,
    entityType: event.entity.type,
    entityId: event.entity.id,
    payload: event as unknown as Record<string, unknown>,
    status: 'pending',
  })
}

/**
 * Create price increase event
 */
export function createPriceIncreaseEvent(params: {
  contractId: string
  oldPrice: number
  newPrice: number
  userId: string
  providerId: string
  traceId: string
  eventSource: string
}): LifecycleEvent {
  return {
    eventId: crypto.randomUUID(),
    eventType: 'lifecycle.contract.price_increase_reported',
    eventSource: params.eventSource,
    eventVersion: '1.0',
    timestamp: new Date().toISOString(),
    traceId: params.traceId,
    entity: {
      id: params.contractId,
      type: 'contract',
    },
    payload: {
      transition: {
        fromStatus: 'active',
        toStatus: 'price_increase_reported',
      },
      context: {
        userId: params.userId,
        providerId: params.providerId,
        oldPrice: params.oldPrice,
        newPrice: params.newPrice,
        priceDelta: params.newPrice - params.oldPrice,
      },
    },
  }
}

/**
 * Create task completed event
 */
export function createTaskCompletedEvent(params: {
  taskId: string
  taskType: string
  contractId: string | null
  resolutionOutcome: string
  originatingProcessName: string | null
  originatingFlowRunId: string | null
  traceId: string
  eventSource: string
}): LifecycleEvent {
  return {
    eventId: crypto.randomUUID(),
    eventType: 'lifecycle.task.status_updated',
    eventSource: params.eventSource,
    eventVersion: '1.0',
    timestamp: new Date().toISOString(),
    traceId: params.traceId,
    entity: {
      id: params.taskId,
      type: 'task',
    },
    payload: {
      transition: {
        fromStatus: 'in_progress',
        toStatus: 'completed',
      },
      context: {
        taskType: params.taskType,
        contractId: params.contractId,
        resolutionOutcome: params.resolutionOutcome,
        originatingProcessName: params.originatingProcessName,
        originatingFlowRunId: params.originatingFlowRunId,
      },
    },
  }
}

/**
 * Create contract activation event
 */
export function createContractActivationEvent(params: {
  contractId: string
  userId: string
  providerId: string
  serviceType: string
  traceId: string
  eventSource: string
}): LifecycleEvent {
  return {
    eventId: crypto.randomUUID(),
    eventType: 'lifecycle.contract.status_updated',
    eventSource: params.eventSource,
    eventVersion: '1.0',
    timestamp: new Date().toISOString(),
    traceId: params.traceId,
    entity: {
      id: params.contractId,
      type: 'contract',
    },
    payload: {
      transition: {
        fromStatus: 'pending_verification',
        toStatus: 'active',
      },
      context: {
        userId: params.userId,
        providerId: params.providerId,
        serviceType: params.serviceType,
      },
    },
  }
}
```

### 7.4 Error Handling (`/lifecycle/lib/errors.ts`)

```typescript
import { z } from 'zod'
import { DrizzleQueryError } from 'drizzle-orm'

/**
 * Base error class for lifecycle domain
 */
export class LifecycleError extends Error {
  constructor(
    message: string,
    public readonly code: string,
    public readonly statusCode: number = 500,
    public readonly details?: Record<string, unknown>
  ) {
    super(message)
    this.name = 'LifecycleError'
  }

  toJSON() {
    return {
      error: this.name,
      code: this.code,
      message: this.message,
      statusCode: this.statusCode,
      details: this.details,
    }
  }
}

/**
 * Entity not found error
 */
export class NotFoundError extends LifecycleError {
  constructor(entityType: string, entityId: string) {
    super(
      `${entityType} with ID ${entityId} not found`,
      'NOT_FOUND',
      404,
      { entityType, entityId }
    )
    this.name = 'NotFoundError'
  }
}

/**
 * Invalid command for current entity state
 */
export class InvalidCommandError extends LifecycleError {
  constructor(
    message: string,
    currentState?: string,
    attemptedCommand?: string
  ) {
    super(
      message,
      'INVALID_COMMAND',
      422,
      { currentState, attemptedCommand }
    )
    this.name = 'InvalidCommandError'
  }
}

/**
 * Validation error (from Zod safeParse failures)
 */
export class ValidationError extends LifecycleError {
  constructor(message: string, fieldErrors: Record<string, string[]>) {
    super(
      message,
      'VALIDATION_ERROR',
      400,
      { fieldErrors }
    )
    this.name = 'ValidationError'
  }
}

/**
 * Database constraint violation
 */
export class ConstraintViolationError extends LifecycleError {
  constructor(constraintName: string, details: Record<string, unknown>) {
    super(
      `Database constraint violated: ${constraintName}`,
      'CONSTRAINT_VIOLATION',
      409,
      { constraintName, ...details }
    )
    this.name = 'ConstraintViolationError'
  }
}

/**
 * Convert Zod safeParse error to structured response (Windmill-friendly)
 *
 * NOTE: Use safeParse() pattern instead - this is for legacy compatibility
 */
export function formatZodError(error: z.ZodError) {
  return {
    success: false,
    errorType: 'VALIDATION_ERROR',
    message: 'Input validation failed',
    details: {
      summary: z.prettifyError(error), // Zod 4: human-readable formatting
      fields: z.flattenError(error).fieldErrors, // Zod 4: field-level errors
      tree: z.treeifyError(error) // Zod 4: nested structure
    },
    retryable: true,
    action: 'REQUIRE_INPUT_CORRECTION'
  }
}

/**
 * Handle Drizzle query errors (Drizzle 0.44.0+)
 */
export function handleDrizzleQueryError(error: unknown): never {
  if (error instanceof DrizzleQueryError) {
    // Extract SQL and params for debugging
    throw new LifecycleError(
      'Database query failed',
      'DATABASE_QUERY_ERROR',
      500,
      {
        sql: error.sql, // The failed SQL query
        params: error.params, // Query parameters
        cause: error.cause, // Underlying database error
        stack: error.stack
      }
    )
  }

  throw error
}

/**
 * Handle database errors (PostgreSQL-specific)
 */
export function handleDatabaseError(error: unknown): never {
  if (error instanceof Error) {
    // PostgreSQL unique constraint violation
    if ('code' in error && error.code === '23505') {
      const match = error.message.match(/Key \((.+)\)=\((.+)\) already exists/)
      throw new ConstraintViolationError('unique_violation', {
        field: match?.[1] || 'unknown',
        value: match?.[2] || 'unknown',
      })
    }

    // PostgreSQL foreign key violation
    if ('code' in error && error.code === '23503') {
      throw new ConstraintViolationError('foreign_key_violation', {
        message: error.message,
      })
    }

    // PostgreSQL check constraint violation
    if ('code' in error && error.code === '23514') {
      throw new ConstraintViolationError('check_violation', {
        message: error.message,
      })
    }
  }

  throw error
}

/**
 * Wrap business intent execution with error handling
 *
 * NOTE: Prefer safeParse() pattern in business intents directly
 * This is for wrapping database operations that may throw
 */
export async function withErrorHandling<T>(
  fn: () => Promise<T>
): Promise<T> {
  try {
    return await fn()
  } catch (error) {
    // Let lifecycle errors pass through
    if (error instanceof LifecycleError) {
      throw error
    }

    // Handle Drizzle query errors
    if (error instanceof DrizzleQueryError) {
      handleDrizzleQueryError(error)
    }

    // Handle database errors
    handleDatabaseError(error)

    // Unknown error - wrap it
    throw new LifecycleError(
      error instanceof Error ? error.message : 'Unknown error occurred',
      'INTERNAL_ERROR',
      500,
      { originalError: String(error) }
    )
  }
}
```

---

## 8. Business Intent Implementation Pattern

**Complete command handler implementation with Zod 4 and Drizzle 2025:**

```typescript
// /lifecycle/contract/report-price-increase.ts

import { eq } from 'drizzle-orm'
import { DrizzleQueryError } from 'drizzle-orm'
import { createDbConnection } from '../lib/db'
import type { PostgresqlResource } from '../lib/db'
import { contracts, priceIncreaseEvents } from '../lib/schema'
import { reportPriceIncreaseSchema } from '../lib/validation'
import type { ReportPriceIncreaseParams } from '../lib/validation'
import { emitEvent, createPriceIncreaseEvent } from '../lib/events'
import { z } from 'zod'
import {
  NotFoundError,
  InvalidCommandError,
} from '../lib/errors'

/**
 * Business Intent: Report Price Increase
 *
 * Records a provider-initiated price increase on a contract.
 * Updates the contract's pending price change and emits an event.
 *
 * Uses neon-http driver optimized for Windmill's serverless execution.
 * Uses safeParse for graceful error handling (Windmill-friendly pattern).
 */
export async function main(
  dbResource: PostgresqlResource,
  rawParams: unknown
) {
  // ========== STEP 1: VALIDATE INPUT (safeParse pattern) ==========
  const parseResult = reportPriceIncreaseSchema.safeParse(rawParams)

  if (!parseResult.success) {
    // Return structured error (don't throw - Windmill-friendly)
    return {
      success: false,
      errorType: 'VALIDATION_ERROR',
      message: 'Invalid input parameters',
      details: {
        summary: z.prettifyError(parseResult.error),
        fields: z.flattenError(parseResult.error).fieldErrors,
        received: rawParams
      },
      retryable: true,
      action: 'REQUIRE_INPUT_CORRECTION'
    }
  }

  const params = parseResult.data

  // ========== STEP 2: CREATE DATABASE CONNECTION ==========
  // NOTE: With neon-http, this creates a stateless HTTP client
  // No need to manage connection lifecycle or pooling
  // Enable caching for read-heavy operations
  const db = createDbConnection(dbResource, { cache: true })

  // ========== STEP 3: EXECUTE TRANSACTIONAL LOGIC ==========
  const result = await db.transaction(async (tx) => {
    // Step 3a: Lock and fetch contract (PESSIMISTIC LOCKING)
    const [contract] = await tx
      .select()
      .from(contracts)
      .where(eq(contracts.id, params.contractId))
      .for('update') // ← CRITICAL: Prevents concurrent modifications

    if (!contract) {
      throw new NotFoundError('Contract', params.contractId)
    }

    // Step 3b: Validate business rules
    if (contract.status === 'cancelled' || contract.status === 'archived') {
      throw new InvalidCommandError(
        `Cannot report price increase for ${contract.status} contract`,
        contract.status,
        'report-price-increase'
      )
    }

    if (params.newPrice <= contract.price) {
      throw new InvalidCommandError(
        `New price (${params.newPrice}) must be higher than current price (${contract.price})`
      )
    }

    // Step 3c: Record audit trail
    await tx.insert(priceIncreaseEvents).values({
      contractId: params.contractId,
      oldPrice: contract.price,
      newPrice: params.newPrice,
      effectiveDate: params.effectiveDate,
      reportedBy: params.reportedBy,
      notificationMethod: params.notificationMethod,
    })

    // Step 3d: Update contract state
    const [updatedContract] = await tx
      .update(contracts)
      .set({
        pendingPriceChange: params.newPrice,
        pendingPriceEffectiveDate: params.effectiveDate,
        status: 'price_increase_reported',
        updatedAt: new Date(),
      })
      .where(eq(contracts.id, params.contractId))
      .returning()

    // Step 3e: Emit event (transactional outbox)
    const event = createPriceIncreaseEvent({
      contractId: params.contractId,
      oldPrice: contract.price,
      newPrice: params.newPrice,
      userId: contract.userId,
      providerId: contract.providerId,
      traceId: crypto.randomUUID(),
      eventSource: 'lifecycle/contract/report-price-increase',
    })

    await emitEvent(tx, event)

    return updatedContract
  })

  // ========== STEP 4: RETURN RESULT ==========
  return {
    success: true,
    contract: result,
    message: `Price increase reported: ${params.newPrice / 100} (effective ${params.effectiveDate.toDateString()})`
  }
}
```

**Zod 4 & Drizzle 2025 Key Features:**
- ✅ **safeParse()**: Never throws ZodErrors - returns structured error responses
- ✅ **z.prettifyError()**: Human-readable error summaries (Zod 4 feature)
- ✅ **z.flattenError()**: Field-level error details for forms (Zod 4 feature)
- ✅ **DrizzleQueryError**: Enhanced error handling with SQL + params + stack trace
- ✅ **Caching opt-in**: createDbConnection(resource, { cache: true })
- ✅ **Graceful degradation**: Structured errors enable Windmill retry logic
- ✅ **Transaction syntax**: IDENTICAL (Drizzle abstracts HTTP vs WebSocket)
- ✅ **No connection cleanup**: HTTP is stateless
- ✅ **Pessimistic locking**: Works with HTTP driver

**Error Response Structure** (Windmill-friendly):
```typescript
{
  success: boolean,
  errorType: 'VALIDATION_ERROR' | 'NOT_FOUND' | 'BUSINESS_RULE_VIOLATION' | 'DATABASE_ERROR' | 'UNKNOWN_ERROR',
  message: string,
  details: Record<string, unknown>,
  retryable: boolean,
  action?: 'REQUIRE_INPUT_CORRECTION' | 'REQUIRE_MANUAL_INTERVENTION' | 'RETRY_LATER'
}
```

---

## 9. Query Pattern

### 9.1 Simple Query (Get by ID)

```typescript
// /lifecycle/contract/get-details.ts

import { eq } from 'drizzle-orm'
import { createDbConnection } from '../lib/db'
import type { PostgresqlResource } from '../lib/db'
import { contracts } from '../lib/schema'
import { z } from 'zod'

// Input validation schema (Zod 4)
const inputSchema = z.interface({
  contractId: z.uuid().meta({
    title: "Contract ID",
    description: "UUID of contract to retrieve",
    context: "Must be existing contract. Will be cached for 10 minutes.",
    validationPriority: "critical"
  })
})

export async function main(
  dbResource: PostgresqlResource,
  rawParams: unknown
) {
  // ========== STEP 1: VALIDATE INPUT (safeParse pattern) ==========
  const parseResult = inputSchema.safeParse(rawParams)

  if (!parseResult.success) {
    return {
      success: false,
      errorType: 'VALIDATION_ERROR',
      message: 'Invalid input parameters',
      details: {
        summary: z.prettifyError(parseResult.error),
        fields: z.flattenError(parseResult.error).fieldErrors
      },
      retryable: true
    }
  }

  const { contractId } = parseResult.data

  // ========== STEP 2: CREATE DATABASE CONNECTION WITH CACHING ==========
  const db = createDbConnection(dbResource, { cache: true })

  // ========== STEP 3: QUERY WITH CACHE (3× faster on cache hit) ==========
  const results = await db
    .select()
    .from(contracts)
    .where(eq(contracts.id, contractId))
    .$withCache({
      tag: `contract:${contractId}`,
      config: { ex: 600 } // TTL: 10 minutes
    })

  const contract = results[0]

  if (!contract) {
    return {
      success: false,
      errorType: 'NOT_FOUND',
      message: `Contract with ID ${contractId} not found`,
      details: { entityType: 'Contract', entityId: contractId },
      retryable: false
    }
  }

  // ========== STEP 4: ADD COMPUTED PROPERTIES ==========
  return {
    success: true,
    data: {
      ...contract,
      computed: {
        is_bonus_eligible: computeBonusEligibility(contract),
        is_within_cancellation_window: computeCancellationWindow(contract),
        days_until_renewal: computeDaysUntilRenewal(contract),
        current_business_phase: computeBusinessPhase(contract),
      },
    }
  }
}

// Pure, deterministic computed properties
function computeBonusEligibility(contract: typeof contracts.$inferSelect): boolean {
  const now = new Date()
  const startDate = new Date(contract.startDate)
  const daysSinceStart = Math.floor((now.getTime() - startDate.getTime()) / (1000 * 60 * 60 * 24))
  return contract.status === 'active' && daysSinceStart >= 60
}

function computeCancellationWindow(contract: typeof contracts.$inferSelect): boolean {
  if (!contract.endDate) return false
  const now = new Date()
  const endDate = new Date(contract.endDate)
  const daysUntilEnd = Math.floor((endDate.getTime() - now.getTime()) / (1000 * 60 * 60 * 24))
  return daysUntilEnd >= 30 && daysUntilEnd <= 90
}

function computeDaysUntilRenewal(contract: typeof contracts.$inferSelect): number | null {
  if (!contract.endDate) return null
  const now = new Date()
  const endDate = new Date(contract.endDate)
  return Math.floor((endDate.getTime() - now.getTime()) / (1000 * 60 * 60 * 24))
}

function computeBusinessPhase(contract: typeof contracts.$inferSelect): string {
  if (contract.status === 'pending_verification') return 'activation'
  if (contract.status === 'price_increase_reported') return 'optimization'
  if (contract.status === 'active' && computeCancellationWindow(contract)) return 'renewal'
  return 'monitoring'
}
```

### 9.2 Complex Query with Joins

```typescript
// /lifecycle/user/get-user-with-contracts.ts

import { eq } from 'drizzle-orm'
import { createDbConnection } from '../lib/db'
import type { PostgresqlResource } from '../lib/db'
import { users, contracts } from '../lib/schema'
import { z } from 'zod'

// Input validation schema (Zod 4)
const inputSchema = z.interface({
  userId: z.uuid().meta({
    title: "User ID",
    description: "UUID of user to retrieve with contracts",
    context: "User contract list cached for 5 minutes. High-frequency query.",
    validationPriority: "critical"
  })
})

export async function main(
  dbResource: PostgresqlResource,
  rawParams: unknown
) {
  // ========== STEP 1: VALIDATE INPUT (safeParse pattern) ==========
  const parseResult = inputSchema.safeParse(rawParams)

  if (!parseResult.success) {
    return {
      success: false,
      errorType: 'VALIDATION_ERROR',
      message: 'Invalid input parameters',
      details: {
        summary: z.prettifyError(parseResult.error),
        fields: z.flattenError(parseResult.error).fieldErrors
      },
      retryable: true
    }
  }

  const { userId } = parseResult.data

  // ========== STEP 2: CREATE DATABASE CONNECTION WITH CACHING ==========
  const db = createDbConnection(dbResource, { cache: true })

  // ========== STEP 3: QUERY WITH CACHE ==========
  // Fetch user with all contracts (one query with join)
  const userWithContracts = await db
    .select({
      user: users,
      contract: contracts,
    })
    .from(users)
    .leftJoin(contracts, eq(contracts.userId, users.id))
    .where(eq(users.id, userId))
    .$withCache({
      tag: `user-contracts:${userId}`,
      config: { ex: 300 } // TTL: 5 minutes (user contract lists change frequently)
    })

  if (userWithContracts.length === 0) {
    return {
      success: false,
      errorType: 'NOT_FOUND',
      message: `User with ID ${userId} not found`,
      details: { entityType: 'User', entityId: userId },
      retryable: false
    }
  }

  // ========== STEP 4: GROUP CONTRACTS BY USER ==========
  // (Drizzle doesn't auto-group joins)
  const user = userWithContracts[0].user
  const contractList = userWithContracts
    .map((row) => row.contract)
    .filter((c) => c !== null)

  return {
    success: true,
    data: {
      ...user,
      contracts: contractList,
      computed: {
        total_contracts: contractList.length,
        active_contracts: contractList.filter((c) => c.status === 'active').length,
        total_monthly_cost: contractList.reduce((sum, c) => sum + c.price, 0),
      },
    }
  }
}
```

### 9.3 Filtered List Query

```typescript
// /lifecycle/contract/list-contracts.ts

import { eq, and, gte, lte, inArray } from 'drizzle-orm'
import { createDbConnection } from '../lib/db'
import type { PostgresqlResource } from '../lib/db'
import { contracts } from '../lib/schema'
import { z } from 'zod'

// Input validation schema (Zod 4 with z.interface for optional keys)
const filterSchema = z.interface({
  "userId?": z.uuid().meta({
    title: "User ID Filter",
    description: "Filter contracts by user ID",
    context: "Optional filter. If provided, returns user's contracts only.",
  }),
  "statuses?": z.array(z.enum(['active', 'price_increase_reported', 'cancelled'])).meta({
    title: "Status Filter",
    description: "Filter contracts by status",
    examples: [['active'], ['active', 'price_increase_reported']],
  }),
  "minPrice?": z.number().int().meta({
    title: "Minimum Price (cents)",
    description: "Filter contracts with price >= this value",
  }),
  "maxPrice?": z.number().int().meta({
    title: "Maximum Price (cents)",
    description: "Filter contracts with price <= this value",
  }),
  limit: z.number().int().min(1).max(100).default(50).meta({
    title: "Page Limit",
    description: "Number of results per page (1-100)",
  }),
  offset: z.number().int().min(0).default(0).meta({
    title: "Page Offset",
    description: "Number of results to skip (for pagination)",
  }),
})

export async function main(
  dbResource: PostgresqlResource,
  rawFilters: unknown
) {
  // ========== STEP 1: VALIDATE INPUT (safeParse pattern) ==========
  const parseResult = filterSchema.safeParse(rawFilters)

  if (!parseResult.success) {
    return {
      success: false,
      errorType: 'VALIDATION_ERROR',
      message: 'Invalid filter parameters',
      details: {
        summary: z.prettifyError(parseResult.error),
        fields: z.flattenError(parseResult.error).fieldErrors
      },
      retryable: true
    }
  }

  const filters = parseResult.data

  // ========== STEP 2: CREATE DATABASE CONNECTION (no caching for filtered lists) ==========
  const db = createDbConnection(dbResource)

  // ========== STEP 3: BUILD DYNAMIC WHERE CLAUSE ==========
  const conditions = []
  if (filters.userId) conditions.push(eq(contracts.userId, filters.userId))
  if (filters.statuses) conditions.push(inArray(contracts.status, filters.statuses))
  if (filters.minPrice) conditions.push(gte(contracts.price, filters.minPrice))
  if (filters.maxPrice) conditions.push(lte(contracts.price, filters.maxPrice))

  const whereClause = conditions.length > 0 ? and(...conditions) : undefined

  // ========== STEP 4: EXECUTE QUERY WITH PAGINATION ==========
  const results = await db
    .select()
    .from(contracts)
    .where(whereClause)
    .limit(filters.limit)
    .offset(filters.offset)
    .orderBy(contracts.createdAt)

  return {
    success: true,
    data: {
      contracts: results,
      pagination: {
        limit: filters.limit,
        offset: filters.offset,
        count: results.length,
      },
    }
  }
}
```

### 9.4 Timeline Projection Query

```typescript
// /lifecycle/contract/get-timeline-projection.ts

import { eq } from 'drizzle-orm'
import { createDbConnection } from '../lib/db'
import type { PostgresqlResource } from '../lib/db'
import { contracts, priceIncreaseEvents } from '../lib/schema'
import { z } from 'zod'

interface StateSnapshot {
  timestamp: string
  event_type: string
  source: 'HISTORY' | 'PROJECTION'
  state_snapshot: {
    status: string
    price: number
    computed: Record<string, unknown>
  }
}

// Input validation schema (Zod 4)
const inputSchema = z.interface({
  contractId: z.uuid().meta({
    title: "Contract ID",
    description: "UUID of contract for timeline projection",
    context: "Timeline includes history + future projections. Not cached (dynamic).",
    validationPriority: "critical"
  })
})

export async function main(
  dbResource: PostgresqlResource,
  rawParams: unknown
) {
  // ========== STEP 1: VALIDATE INPUT (safeParse pattern) ==========
  const parseResult = inputSchema.safeParse(rawParams)

  if (!parseResult.success) {
    return {
      success: false,
      errorType: 'VALIDATION_ERROR',
      message: 'Invalid input parameters',
      details: {
        summary: z.prettifyError(parseResult.error),
        fields: z.flattenError(parseResult.error).fieldErrors
      },
      retryable: true
    }
  }

  const { contractId } = parseResult.data

  // ========== STEP 2: CREATE DATABASE CONNECTION (no caching - dynamic data) ==========
  const db = createDbConnection(dbResource)

  // ========== STEP 3: FETCH CONTRACT ==========
  const [contract] = await db
    .select()
    .from(contracts)
    .where(eq(contracts.id, contractId))

  if (!contract) {
    return {
      success: false,
      errorType: 'NOT_FOUND',
      message: `Contract with ID ${contractId} not found`,
      details: { entityType: 'Contract', entityId: contractId },
      retryable: false
    }
  }

  // ========== STEP 4: FETCH HISTORICAL PRICE INCREASE EVENTS ==========
  const priceHistory = await db
    .select()
    .from(priceIncreaseEvents)
    .where(eq(priceIncreaseEvents.contractId, contractId))
    .orderBy(priceIncreaseEvents.createdAt)

  const timeline: StateSnapshot[] = []

  // HISTORY: Contract activation
  timeline.push({
    timestamp: contract.createdAt.toISOString(),
    event_type: 'ACTIVATION_CONFIRMED',
    source: 'HISTORY',
    state_snapshot: {
      status: 'active',
      price: contract.price,
      computed: {},
    },
  })

  // HISTORY: Price increases
  for (const event of priceHistory) {
    timeline.push({
      timestamp: event.createdAt.toISOString(),
      event_type: 'PRICE_INCREASE_REPORTED',
      source: 'HISTORY',
      state_snapshot: {
        status: 'price_increase_reported',
        price: event.newPrice,
        computed: {
          price_delta: event.newPrice - event.oldPrice,
        },
      },
    })
  }

  // PROJECTION: Pending price increase (if exists)
  if (contract.pendingPriceChange && contract.pendingPriceEffectiveDate) {
    timeline.push({
      timestamp: contract.pendingPriceEffectiveDate.toISOString(),
      event_type: 'PROJECTED_PRICE_INCREASE',
      source: 'PROJECTION',
      state_snapshot: {
        status: 'active',
        price: contract.pendingPriceChange,
        computed: {
          price_delta: contract.pendingPriceChange - contract.price,
        },
      },
    })
  }

  // PROJECTION: Contract renewal (if end date exists)
  if (contract.endDate) {
    timeline.push({
      timestamp: contract.endDate.toISOString(),
      event_type: 'PROJECTED_RENEWAL',
      source: 'PROJECTION',
      state_snapshot: {
        status: 'active',
        price: contract.price,
        computed: {},
      },
    })
  }

  return {
    success: true,
    data: {
      contract_id: contractId,
      timeline: timeline.sort((a, b) => a.timestamp.localeCompare(b.timestamp)),
    }
  }
}
```

---

## 10. Event Emission & Transactional Outbox

### 10.1 The Transactional Outbox Pattern

**Problem**: How do we ensure events are emitted if and only if the state change succeeds?

**Solution**: Write events to an `outbox_events` table **within the same transaction** as the state change, then have a separate process publish them.

**Benefits**:
- **Atomicity**: Event emission is part of the transaction (all-or-nothing)
- **Reliability**: Events are persisted even if webhook fails
- **Decoupling**: Command handler doesn't wait for event delivery
- **Retry logic**: Separate process can retry failed events

### 10.2 Outbox Publisher (Windmill Flow)

```typescript
// /lifecycle/outbox/publish-pending-events.ts

import { eq, and, lt } from 'drizzle-orm'
import { createDbConnection } from '../lib/db'
import type { PostgresqlResource } from '../lib/db'
import { outboxEvents } from '../lib/schema'

/**
 * Publish pending events from the outbox to the lifecycle-event-handler webhook
 *
 * This script should be run periodically (e.g., every 10 seconds) via a Windmill schedule.
 */
export async function main(
  dbResource: PostgresqlResource,
  webhookUrl: string, // e.g., "https://windmill.example.com/webhooks/u/system/lifecycle-event-handler"
  batchSize: number = 50
) {
  const db = createDbConnection(dbResource)

  // Fetch pending events (oldest first, limit batch size)
  const pendingEvents = await db
    .select()
    .from(outboxEvents)
    .where(eq(outboxEvents.status, 'pending'))
    .orderBy(outboxEvents.createdAt)
    .limit(batchSize)

  const results = {
    processed: 0,
    succeeded: 0,
    failed: 0,
    errors: [] as Array<{ eventId: string; error: string }>,
  }

  for (const event of pendingEvents) {
    try {
      // Publish to webhook
      const response = await fetch(webhookUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(event.payload),
      })

      if (!response.ok) {
        throw new Error(`Webhook returned ${response.status}: ${await response.text()}`)
      }

      // Mark as published
      await db
        .update(outboxEvents)
        .set({
          status: 'published',
          publishedAt: new Date(),
        })
        .where(eq(outboxEvents.id, event.id))

      results.succeeded++
    } catch (error) {
      results.failed++
      results.errors.push({
        eventId: event.id,
        error: error instanceof Error ? error.message : String(error),
      })

      // Optional: Mark as failed after N retries (requires retry_count column)
      // For now, leave as 'pending' so it retries on next run
    }

    results.processed++
  }

  return results
}
```

**Windmill Schedule Configuration**:
```json
{
  "path": "lifecycle/outbox/publish-pending-events",
  "schedule": "*/10 * * * * *",
  "args": {
    "dbResource": "$res:postgresql_production",
    "webhookUrl": "https://windmill.example.com/webhooks/u/system/lifecycle-event-handler",
    "batchSize": 50
  }
}
```

### 10.3 Event Handler Webhook (Entry Point for /case Domain)

```typescript
// /case/ingestion/lifecycle-event-handler.ts

import type { LifecycleEvent } from '../../lifecycle/lib/events'

/**
 * Webhook endpoint that receives lifecycle events and routes them to appropriate case flows
 *
 * This is registered as a Windmill webhook at:
 * /webhooks/u/system/lifecycle-event-handler
 */
export async function main(event: LifecycleEvent) {
  // Route based on event type
  switch (event.eventType) {
    case 'lifecycle.contract.price_increase_reported':
      return await handlePriceIncrease(event)

    case 'lifecycle.task.status_updated':
      return await handleTaskStatusUpdate(event)

    case 'lifecycle.contract.status_updated':
      return await handleContractStatusUpdate(event)

    default:
      console.log(`Unhandled event type: ${event.eventType}`)
      return { status: 'ignored', reason: 'Unknown event type' }
  }
}

async function handlePriceIncrease(event: LifecycleEvent) {
  // Call /case/meta/process-dispatcher to route to appropriate flow
  // This is where the /case domain takes over
  return {
    status: 'routed',
    target: 'case/lifecycle_management/handle-price-increase',
    traceId: event.traceId,
  }
}

async function handleTaskStatusUpdate(event: LifecycleEvent) {
  const { originatingProcessName, originatingFlowRunId, resolutionOutcome } =
    event.payload.context

  if (!originatingFlowRunId) {
    return { status: 'ignored', reason: 'No originating flow to resume' }
  }

  // Resume the originating flow with the task result
  // (This would use Windmill's resume API)
  return {
    status: 'resumed',
    flowRunId: originatingFlowRunId,
    resolution: resolutionOutcome,
  }
}

async function handleContractStatusUpdate(event: LifecycleEvent) {
  const { toStatus } = event.payload.transition || {}

  if (toStatus === 'active') {
    // New contract activated - trigger welcome flow
    return {
      status: 'routed',
      target: 'case/onboarding/welcome-new-contract',
      traceId: event.traceId,
    }
  }

  return { status: 'ignored', reason: 'No action required' }
}
```

---

## 11. Row-Level Security (RLS)

### 11.1 Overview

Neon provides powerful RLS integration with Drizzle using:
- **`crudPolicy()`**: Generate all CRUD policies from simple config
- **`authUid()`**: Check if current user matches a column value
- **`authenticatedRole`** / **`anonymousRole`**: Predefined Neon roles
- **`auth.user_id()`**: PostgreSQL function to get user from JWT

### 11.2 Basic RLS Pattern

```typescript
import { crudPolicy, authenticatedRole, authUid } from 'drizzle-orm/neon'

export const todos = pgTable('todos', {
  id: bigint('id', { mode: 'number' }).primaryKey(),
  userId: text('user_id').notNull().default(sql`(auth.user_id())`),
  task: text('task').notNull(),
}, (table) => [
  // Users can only see their own todos
  crudPolicy({
    role: authenticatedRole,
    read: authUid(table.userId),
    modify: authUid(table.userId),
  })
])
```

**What this generates:**
- SELECT policy: `USING (user_id = auth.user_id())`
- INSERT policy: `WITH CHECK (user_id = auth.user_id())`
- UPDATE policy: `USING (user_id = auth.user_id()) WITH CHECK (user_id = auth.user_id())`
- DELETE policy: `USING (user_id = auth.user_id())`

### 11.3 Advanced RLS Patterns

**Different read/modify permissions:**

```typescript
crudPolicy({
  role: authenticatedRole,
  read: true, // Everyone can read
  modify: authUid(table.userId), // Only owner can modify
})
```

**Complex relationship checks:**

```typescript
// Access notes only if user owns parent document
crudPolicy({
  role: authenticatedRole,
  read: sql`EXISTS (
    SELECT 1 FROM ${documents}
    WHERE ${documents.id} = ${table.documentId}
    AND ${documents.ownerId} = auth.user_id()
  )`,
})
```

**Multiple policies for different roles:**

```typescript
(table) => [
  // Anonymous users: read only
  crudPolicy({
    role: anonymousRole,
    read: true,
    modify: false,
  }),

  // Authenticated users: read all, modify own
  crudPolicy({
    role: authenticatedRole,
    read: true,
    modify: authUid(table.userId),
  }),
]
```

### 11.4 Using RLS in Windmill Scripts

**Option 1: Multiple Windmill Resources (Recommended)**

```typescript
// Create separate resources for different contexts
// postgresql_production_admin (neondb_owner - bypasses RLS)
// postgresql_production_authenticated (authenticated role)
// postgresql_production_anonymous (anonymous role)

// In script:
export async function main(
  dbResource: PostgresqlResource, // Use authenticated resource
  params: unknown
) {
  const db = createDbConnection(dbResource)
  // RLS automatically enforced based on resource role
}
```

**Option 2: JWT Token Authentication**

```typescript
import { createAuthenticatedDbConnection } from '../lib/db'

export async function main(
  dbResource: PostgresqlResource,
  authToken: string, // Pass JWT from upstream
  params: unknown
) {
  // Connection with RLS enforced based on JWT claims
  const db = createAuthenticatedDbConnection(dbResource, authToken)

  const contracts = await db.select().from(contracts)
  // Only returns contracts owned by authenticated user
}
```

### 11.5 RLS Best Practices

1. ✅ **Never use `neondb_owner` for application queries** (bypasses RLS)
2. ✅ **Test policies at database level** before relying on them
3. ✅ **Use `crudPolicy()` for simple patterns** (cleaner than `pgPolicy`)
4. ✅ **Document complex SQL expressions** in policy definitions
5. ✅ **Combine with application-level checks** for better UX (fail fast)

---

## 12. Migration Management

### 12.1 Migration Workflow

**1. Generate migration from schema changes:**

```bash
npx drizzle-kit generate
```

This creates SQL migration files in `./lifecycle/migrations/meta/`.

**2. Test migration on Neon branch:**

```bash
# Create branch
neon branch create --name migration-test-$(date +%Y%m%d)

# Get branch connection string
export DATABASE_URL=$(neon connection-string migration-test-$(date +%Y%m%d))

# Apply migration to branch
npx drizzle-kit push
```

**3. Verify migration:**

```bash
# Run test suite against branch
npm test

# Manual verification
npx drizzle-kit studio
```

**4. Apply to production (via Windmill script):**

Create `/lifecycle/migrations/run-migration.ts`:

```typescript
import { neon } from '@neondatabase/serverless'
import { drizzle } from 'drizzle-orm/neon-http'
import { migrate } from 'drizzle-orm/neon-http/migrator'
import type { PostgresqlResource } from '../lib/db'

export async function main(
  dbResource: PostgresqlResource
) {
  const connectionString = `postgres://${dbResource.user}:${dbResource.password}@${dbResource.host}:${dbResource.port}/${dbResource.dbname}?sslmode=${dbResource.sslmode}&channel_binding=require`

  const sql = neon(connectionString)
  const db = drizzle(sql)

  // Apply all pending migrations
  await migrate(db, { migrationsFolder: './lifecycle/migrations/meta' })

  return { success: true, message: 'Migrations applied successfully' }
}
```

**5. Delete test branch:**

```bash
neon branch delete migration-test-$(date +%Y%m%d)
```

### 12.2 Migration Best Practices

1. ✅ **Always test on Neon branch first** (zero risk)
2. ✅ **Use drizzle-kit generate** (never hand-write SQL migrations)
3. ✅ **Include rollback plan** for destructive changes
4. ✅ **Run migrations during low-traffic windows**
5. ✅ **Monitor query performance** after schema changes
6. ⚠️ **Avoid renaming columns** (Drizzle sees as drop + add - data loss!)
   - Instead: Add new column → migrate data → drop old column (3 migrations)

### 12.3 Safe Schema Evolution Patterns

**Adding a column:**

```typescript
// Safe: Drizzle handles this automatically
export const contracts = pgTable('contracts', {
  // ... existing columns
  newColumn: varchar('new_column', { length: 100 }), // ← NEW
})
```

**Renaming a column (3-step process):**

```typescript
// Migration 1: Add new column
export const contracts = pgTable('contracts', {
  oldName: varchar('old_name', { length: 100 }),
  newName: varchar('new_name', { length: 100 }), // ← ADD
})

// Migration 2: Backfill data (custom SQL)
// UPDATE contracts SET new_name = old_name WHERE new_name IS NULL;

// Migration 3: Drop old column
export const contracts = pgTable('contracts', {
  newName: varchar('new_name', { length: 100 }),
  // oldName removed ← DROP
})
```

**Changing column type:**

```typescript
// Migration 1: Add new column with new type
price: integer('price').notNull(), // existing
priceDecimal: numeric('price_decimal', { precision: 10, scale: 2 }), // ← ADD

// Migration 2: Backfill
// UPDATE contracts SET price_decimal = price / 100.0;

// Migration 3: Drop old column
priceDecimal: numeric('price_decimal', { precision: 10, scale: 2 }),
// price removed ← DROP
```

---

## 13. AI-First Development Workflow

### 13.1 AI-Assisted Business Intent Generation

**Prompt template for generating new command handlers with Zod 4:**

```
Generate a Windmill TypeScript script for the following business intent:

**Entity**: Contract
**Intent**: ConfirmActivation
**Description**: Mark a pending contract as successfully activated after provider confirmation.

**Business Rules**:
1. Contract must be in "pending_verification" status
2. Update status to "active"
3. Record activation timestamp
4. Emit "lifecycle.contract.status_updated" event

**Input Schema** (use Zod 4 with z.interface and metadata):
- contractId: UUID (required, critical validation)
- activationConfirmationId: string (required from provider)
- confirmedBy?: UUID (optional - user or system)

**Use the following patterns**:
- **Validation**: Zod 4 z.interface() with .meta() containing:
  - title, description, examples
  - context field (for AI agent guidance)
  - validationPriority (critical/important/optional)
- **Input handling**: safeParse() pattern (never throw - return structured errors)
- **Database**: createDbConnection from /lifecycle/lib/db
- **Transaction**: Pessimistic locking with .for('update')
- **Event**: createContractActivationEvent from /lifecycle/lib/events
- **Error handling**: Structured error responses (not throwing errors)
- **Zod 4 error formatting**: z.prettifyError(), z.flattenError(), z.treeifyError()

**Key Zod 4 features to use**:
1. z.interface() for precise optionality (key optional vs value optional)
2. .meta() for rich metadata (especially 'context' field for AI agents)
3. Top-level validators: z.uuid(), z.email(), z.hash(), etc.
4. safeParse() instead of parse() (Windmill-friendly)
5. Enhanced error formatting for debugging

**Output location**: /lifecycle/contract/confirm-activation.ts
```

**Expected AI output quality with Zod 4 metadata**:
- ✅ **92% correct** on first attempt (up from 85% with Zod 3)
- ✅ **8% requiring minor fixes** (usually business rule edge cases)
- ⚠️ **0% requiring major refactoring** (if prompt follows this spec)
- ✅ **AI agents can read metadata** via z.toJSONSchema() for better context

### 13.2 AI-Assisted Schema Evolution

**Prompt for adding new entity:**

```
Add a new entity "Offer" to the Drizzle schema at /lifecycle/lib/schema.ts

**Requirements**:
- UUID primary key with defaultRandom()
- Fields: provider_id (UUID FK), offer_code (varchar 50), price (integer), description (text)
- Timestamps: created_at, updated_at (auto-managed)
- Index on provider_id
- RLS: Public read, admin-only modify

**Follow existing patterns** from contracts and users tables.
```

### 13.3 AI Review Checklist

Before accepting AI-generated code, verify:

1. ✅ **Imports are correct** (`drizzle-orm/neon-http` not `postgres-js`, `zod 4.1.12+`)
2. ✅ **Validation schema uses Zod 4 patterns**:
   - z.interface() for business intents (not z.object())
   - .meta() with rich metadata (title, description, context, examples)
   - Top-level validators (z.uuid(), z.email(), z.hash())
3. ✅ **safeParse() pattern used** (never parse() - Windmill-friendly)
4. ✅ **Structured error responses** (not throwing ZodError)
5. ✅ **Zod 4 error formatting** (z.prettifyError(), z.flattenError())
6. ✅ **Pessimistic locking used** for state mutations (.for('update'))
7. ✅ **Event emitted** within transaction
8. ✅ **Business rules enforced** before state change
9. ✅ **Type inference used** (`z.infer`, `typeof schema.$inferSelect`)
10. ✅ **No SQL injection risk** (using Drizzle's query builder only)
11. ✅ **Caching considered** (opt-in with .$withCache() for read-heavy queries)
12. ✅ **DrizzleQueryError handling** (for Drizzle 0.44.0+ error transparency)

---

## 14. Testing Strategy

### 14.1 Unit Tests (Business Logic)

```typescript
// /lifecycle/contract/__tests__/report-price-increase.test.ts

import { describe, it, expect, beforeEach } from 'vitest'
import { main } from '../report-price-increase'
import { createDbConnection } from '../../lib/db'
import type { PostgresqlResource } from '../../lib/db'
import { contracts } from '../../lib/schema'

describe('report-price-increase', () => {
  let dbResource: PostgresqlResource

  beforeEach(async () => {
    // Use Neon branch for testing
    dbResource = {
      host: process.env.TEST_DB_HOST!,
      port: 5432,
      user: 'test_user',
      password: process.env.TEST_DB_PASSWORD!,
      dbname: 'switchup_test',
      sslmode: 'require',
    }

    // Clean database
    const db = createDbConnection(dbResource)
    await db.delete(contracts)
  })

  it('should report price increase for active contract', async () => {
    // Arrange: Create test contract
    const db = createDbConnection(dbResource)
    const [contract] = await db.insert(contracts).values({
      userId: 'user-123',
      status: 'active',
      providerId: 'provider-abc',
      serviceType: 'electricity',
      price: 10000, // £100.00
      startDate: new Date('2024-01-01'),
    }).returning()

    // Act: Report price increase
    const result = await main(dbResource, {
      contractId: contract.id,
      newPrice: 12000, // £120.00
      effectiveDate: new Date('2024-12-01'),
      notificationMethod: 'email',
      reportedBy: 'system',
    })

    // Assert: Structured success response (Zod 4 safeParse pattern)
    expect(result.success).toBe(true)
    expect(result.data.contract.status).toBe('price_increase_reported')
    expect(result.data.contract.pendingPriceChange).toBe(12000)

    // Assert: Event created in outbox
    const events = await db.select().from(outboxEvents)
    expect(events).toHaveLength(1)
    expect(events[0].eventType).toBe('lifecycle.contract.price_increase_reported')
  })

  it('should return structured error for cancelled contract', async () => {
    // Arrange
    const db = createDbConnection(dbResource)
    const [contract] = await db.insert(contracts).values({
      userId: 'user-123',
      status: 'cancelled',
      providerId: 'provider-abc',
      serviceType: 'electricity',
      price: 10000,
      startDate: new Date('2024-01-01'),
    }).returning()

    // Act: Attempt price increase on cancelled contract
    const result = await main(dbResource, {
      contractId: contract.id,
      newPrice: 12000,
      effectiveDate: new Date('2024-12-01'),
      notificationMethod: 'email',
      reportedBy: 'system',
    })

    // Assert: Structured error response (no throw - Windmill-friendly)
    expect(result.success).toBe(false)
    expect(result.errorType).toBe('INVALID_STATE')
    expect(result.message).toContain('cancelled')
    expect(result.retryable).toBe(false)
  })

  it('should return validation error for invalid input', async () => {
    // Act: Invalid input (Zod 4 safeParse catches this)
    const result = await main(dbResource, {
      contractId: 'not-a-uuid',  // Invalid UUID format
      newPrice: -100,            // Negative price
      effectiveDate: 'invalid',
    })

    // Assert: Structured validation error (Zod 4 error formatting)
    expect(result.success).toBe(false)
    expect(result.errorType).toBe('VALIDATION_ERROR')
    expect(result.details.fields).toBeDefined()
    expect(result.details.summary).toBeDefined()  // z.prettifyError()
    expect(result.retryable).toBe(true)
  })
})
```

### 14.2 Integration Tests (End-to-End)

```typescript
// /lifecycle/__tests__/integration/price-increase-flow.test.ts

import { describe, it, expect } from 'vitest'
import { createDbConnection } from '../../lib/db'
import { main as reportPriceIncrease } from '../../contract/report-price-increase'
import { main as publishEvents } from '../../outbox/publish-pending-events'

describe('Price Increase Flow (E2E)', () => {
  it('should complete full price increase workflow', async () => {
    const dbResource = getTestDbResource()

    // Step 1: Report price increase (command handler)
    const reportResult = await reportPriceIncrease(dbResource, {
      contractId: 'test-contract-id',
      newPrice: 12000,
      effectiveDate: new Date('2024-12-01'),
      notificationMethod: 'email',
      reportedBy: 'system',
    })

    expect(reportResult.success).toBe(true)

    // Step 2: Publish outbox events
    const publishResult = await publishEvents(
      dbResource,
      'http://localhost:8000/test-webhook',
      10
    )

    expect(publishResult.succeeded).toBe(1)
    expect(publishResult.failed).toBe(0)

    // Step 3: Verify event was published
    const db = createDbConnection(dbResource)
    const events = await db
      .select()
      .from(outboxEvents)
      .where(eq(outboxEvents.status, 'published'))

    expect(events).toHaveLength(1)
  })
})
```

### 14.3 Test Database Setup (Neon Branch)

```bash
#!/bin/bash
# scripts/setup-test-db.sh

# Create ephemeral test branch
TEST_BRANCH="test-$(date +%Y%m%d-%H%M%S)"
neon branch create --name $TEST_BRANCH

# Get connection string
export TEST_DATABASE_URL=$(neon connection-string $TEST_BRANCH)

# Apply migrations
npx drizzle-kit push --connectionString=$TEST_DATABASE_URL

echo "Test database ready: $TEST_BRANCH"
echo "TEST_DATABASE_URL=$TEST_DATABASE_URL"
```

**Cleanup:**

```bash
#!/bin/bash
# scripts/cleanup-test-db.sh

neon branch delete $TEST_BRANCH
```

**In CI/CD:**

```yaml
# .github/workflows/test.yml
jobs:
  test:
    steps:
      - name: Setup test database
        run: ./scripts/setup-test-db.sh

      - name: Run tests
        run: npm test
        env:
          TEST_DATABASE_URL: ${{ env.TEST_DATABASE_URL }}

      - name: Cleanup
        if: always()
        run: ./scripts/cleanup-test-db.sh
```

---

## 15. Performance Optimization

### 15.1 HTTP Driver Advantages

**Why neon-http is faster for Windmill:**

| Aspect             | neon-http (HTTP)         | neon-serverless (WebSocket)      |
| ------------------ | ------------------------ | -------------------------------- |
| Connection setup   | None (stateless HTTP)    | WebSocket handshake required     |
| Per-query overhead | Single HTTP request      | Connection management            |
| Windmill fit       | Perfect (1 query/script) | Overkill (persistent connection) |
| Cold start         | Fastest                  | Slower (WebSocket setup)         |

**From Neon docs**: *"Querying over HTTP is faster for single, non-interactive transactions"*

---

### 15.2 Native Caching with Upstash Redis (Drizzle 0.44.0+)

**Strategy**: Opt-in caching for read-heavy lifecycle queries (3-5× performance improvement).

**Setup** (in `db.ts`):

```typescript
import { upstashCache } from 'drizzle-orm/cache/upstash'

export function createDbConnection(
  resource: PostgresqlResource,
  options?: { cache?: boolean }
) {
  const config: DrizzleConfig<typeof schema> = { schema }

  if (options?.cache) {
    config.cache = upstashCache({
      global: false, // Opt-in per query (recommended)
      config: { ex: 300 } // Default TTL: 5 minutes
    })
  }

  return drizzle(sql, config)
}
```

**Usage Patterns:**

```typescript
// Enable caching for read-heavy script
const db = createDbConnection(dbResource, { cache: true })

// Per-query caching with custom TTL
const contract = await db
  .select()
  .from(contracts)
  .where(eq(contracts.id, contractId))
  .$withCache({
    tag: `contract:${contractId}`, // Cache key
    config: { ex: 600 } // TTL: 10 minutes
  })

// Query user's contracts list (high read volume)
const userContracts = await db
  .select()
  .from(contracts)
  .where(eq(contracts.userId, userId))
  .$withCache({
    tag: `user-contracts:${userId}`,
    config: { ex: 300 } // TTL: 5 minutes
  })
```

**Cache Invalidation**:

```typescript
// After mutation, invalidate relevant caches
await db.transaction(async (tx) => {
  // Update contract
  const updated = await tx
    .update(contracts)
    .set({ status: 'active' })
    .where(eq(contracts.id, contractId))
    .returning()

  // Invalidate cache
  await db.$invalidateCache([
    `contract:${contractId}`,
    `user-contracts:${updated[0].userId}`
  ])
})
```

**When to Cache:**
- ✅ **Contract details**: Read-heavy, infrequent updates (10-30 reads per write)
- ✅ **User contract lists**: High read volume (dashboard queries)
- ✅ **Provider lookups**: Rarely change (static reference data)
- ✅ **Timeline projections**: Expensive computations
- ❌ **Don't cache**: Real-time data (pending tasks, active workflows)
- ❌ **Don't cache**: Frequently mutated entities (task status updates)

**Performance Impact**:
- **Contract lookups**: 50-100ms → 5-10ms (cache hit)
- **User contract lists**: 100-200ms → 10-20ms
- **Provider queries**: 20-50ms → <5ms
- **Read throughput**: 3-5× improvement

**Upstash Configuration** (Environment Variables):

```bash
# .env
UPSTASH_REDIS_REST_URL=https://your-endpoint.upstash.io
UPSTASH_REDIS_REST_TOKEN=your-token-here
```

---

### 15.3 Query Optimization

**Select projections:**

```typescript
// Good: Only fetch needed fields (reduces payload size)
const contracts = await db
  .select({
    id: contracts.id,
    status: contracts.status,
    price: contracts.price,
  })
  .from(contracts)

// Bad: SELECT * (fetches unnecessary data)
const contracts = await db.select().from(contracts)
```

**Batch operations:**

```typescript
// Good: Single batch update
await db
  .update(contracts)
  .set({ status: 'Active' })
  .where(inArray(contracts.id, contractIds))

// Bad: Loop with individual updates
for (const id of contractIds) {
  await db.update(contracts).set({ status: 'Active' }).where(eq(contracts.id, id))
}
```

**Eager loading vs N+1 queries:**

```typescript
// Good: Single query with join
const usersWithContracts = await db
  .select()
  .from(users)
  .leftJoin(contracts, eq(contracts.userId, users.id))

// Bad: N+1 queries
const users = await db.select().from(users)
for (const user of users) {
  const userContracts = await db.select().from(contracts).where(eq(contracts.userId, user.id))
}
```

### 15.4 Index Strategy

**Essential indexes from schema:**

```typescript
// User lookups
index('users_email_idx').on(table.email)

// Contract queries
index('contracts_user_id_idx').on(table.userId),
index('contracts_status_idx').on(table.status),
index('contracts_user_status_idx').on(table.userId, table.status), // Composite

// Outbox processing
index('outbox_events_status_idx').on(table.status),
index('outbox_events_created_at_idx').on(table.createdAt),
```

**When to add indexes:**

1. ✅ **Foreign keys** (for JOIN queries)
2. ✅ **WHERE clause filters** used frequently
3. ✅ **Composite indexes** for common filter combinations
4. ⚠️ **Avoid over-indexing** (slows INSERT/UPDATE)

**Analyzing query performance:**

```typescript
// Use Drizzle's query logging
const db = drizzle(sql, { schema, logger: true })

// Check execution plan in Neon console
// EXPLAIN ANALYZE SELECT ...
```

### 15.5 Expected Performance

With neon-http driver on Windmill:
- **Simple query** (get by ID): 10-20ms
- **Complex query** (joins): 30-50ms
- **Transaction** (multiple operations): 50-100ms
- **Batch operation** (100 rows): 100-200ms

**Note**: These are FASTER than WebSocket for Windmill's use case due to no connection overhead.

---

## 16. CI/CD & Deployment

### 16.1 Windmill Sync Workflow

```yaml
# .github/workflows/windmill-sync.yml

name: Windmill Sync

on:
  push:
    branches: [main]
    paths:
      - 'lifecycle/**'

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install Windmill CLI
        run: npm install -g windmill-cli

      - name: Sync to Windmill
        run: |
          wmill workspace add prod ${{ secrets.WINDMILL_WORKSPACE }} ${{ secrets.WINDMILL_TOKEN }}
          wmill sync push --workspace prod
        env:
          WMILL_TOKEN: ${{ secrets.WINDMILL_TOKEN }}
```

### 16.2 Schema Migration CI/CD

```yaml
# .github/workflows/migrate.yml

name: Database Migration

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy to'
        required: true
        type: choice
        options:
          - staging
          - production

jobs:
  migrate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4

      - name: Install dependencies
        run: npm install

      - name: Create Neon test branch
        run: |
          neon branch create --name migration-test-${{ github.sha }}
        env:
          NEON_API_KEY: ${{ secrets.NEON_API_KEY }}

      - name: Test migration on branch
        run: |
          export DATABASE_URL=$(neon connection-string migration-test-${{ github.sha }})
          npx drizzle-kit push
          npm test
        env:
          NEON_API_KEY: ${{ secrets.NEON_API_KEY }}

      - name: Apply migration to production (via Windmill)
        if: github.event.inputs.environment == 'production'
        run: |
          wmill run lifecycle/migrations/run-migration --data '{"dbResource": "$res:postgresql_production"}'
        env:
          WMILL_TOKEN: ${{ secrets.WINDMILL_TOKEN }}

      - name: Cleanup test branch
        if: always()
        run: |
          neon branch delete migration-test-${{ github.sha }}
        env:
          NEON_API_KEY: ${{ secrets.NEON_API_KEY }}
```

### 16.3 Type-Checking CI

```yaml
# .github/workflows/type-check.yml

name: Type Check

on:
  pull_request:
    paths:
      - 'lifecycle/**/*.ts'

jobs:
  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm install
      - run: npm run type-check

      # Fail PR if type errors exist
      - name: Annotate PR with errors
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '❌ Type check failed. Please fix TypeScript errors.'
            })
```

---

## 17. Complete Examples

### 17.1 Full Command Handler (Task Completion)

```typescript
// /lifecycle/task/complete-task.ts

import { eq } from 'drizzle-orm'
import { DrizzleQueryError } from 'drizzle-orm'
import { createDbConnection } from '../lib/db'
import type { PostgresqlResource } from '../lib/db'
import { tasks } from '../lib/schema'
import { emitEvent, createTaskCompletedEvent } from '../lib/events'
import { z } from 'zod'

/**
 * Business Intent: Complete Task
 *
 * Marks a task as completed and emits an event to resume the originating flow.
 * This is the callback mechanism for human-in-the-loop workflows.
 */

// ========== INPUT SCHEMA (Zod 4 with z.interface and metadata) ==========
const completeTaskSchema = z.interface({
  taskId: z.uuid().meta({
    title: "Task ID",
    description: "UUID of task to complete",
    context: "Must be existing pending task. Used for human-in-the-loop workflows.",
    validationPriority: "critical"
  }),

  resolutionData: z.object({
    outcome: z.enum(['completed', 'approved', 'rejected']).meta({
      title: "Resolution Outcome",
      description: "How the task was resolved",
      examples: ['completed', 'approved', 'rejected']
    }),
    notes: z.string().optional().meta({
      title: "Resolution Notes",
      description: "Optional notes from user"
    }),
    metadata: z.record(z.unknown()).optional()
  }).meta({
    title: "Resolution Data",
    description: "Data captured during task completion",
    context: "Structured outcome + optional notes/metadata"
  }),

  "completedBy?": z.uuid().meta({
    title: "Completed By",
    description: "UUID of user who completed task",
    context: "Optional - defaults to system if not provided"
  })
})

type CompleteTaskParams = z.infer<typeof completeTaskSchema>

export async function main(
  dbResource: PostgresqlResource,
  rawParams: unknown
) {
  // ========== STEP 1: VALIDATE INPUT (safeParse pattern - never throw) ==========
  const parseResult = completeTaskSchema.safeParse(rawParams)

  if (!parseResult.success) {
    return {
      success: false,
      errorType: 'VALIDATION_ERROR',
      message: 'Invalid input parameters',
      details: {
        summary: z.prettifyError(parseResult.error),
        fields: z.flattenError(parseResult.error).fieldErrors,
        received: rawParams
      },
      retryable: true,
      action: 'REQUIRE_INPUT_CORRECTION'
    }
  }

  const params = parseResult.data

  // ========== STEP 2: CREATE DATABASE CONNECTION ==========
  const db = createDbConnection(dbResource)

  // ========== STEP 3: EXECUTE TRANSACTIONAL LOGIC ==========
  try {
    const result = await db.transaction(async (tx) => {
      // Step 3a: Lock and fetch task (pessimistic locking)
      const [task] = await tx
        .select()
        .from(tasks)
        .where(eq(tasks.id, params.taskId))
        .for('update')

      if (!task) {
        throw new Error('NOT_FOUND')
      }

      // Step 3b: Validate business rules
      if (task.status === 'completed') {
        throw new Error('ALREADY_COMPLETED')
      }

      if (task.status === 'cancelled') {
        throw new Error('TASK_CANCELLED')
      }

      // Step 3c: Update task state
      const [completedTask] = await tx
        .update(tasks)
        .set({
          status: 'completed',
          resolutionData: params.resolutionData,
          completedAt: new Date(),
        })
        .where(eq(tasks.id, params.taskId))
        .returning()

      // Step 3d: Emit event
      const event = createTaskCompletedEvent({
        taskId: params.taskId,
        taskType: task.type,
        contractId: task.contractId,
        resolutionOutcome: params.resolutionData.outcome || 'completed',
        originatingProcessName: task.originatingProcessName,
        originatingFlowRunId: task.originatingFlowRunId,
        traceId: crypto.randomUUID(),
        eventSource: 'lifecycle/task/complete-task',
      })

      await emitEvent(tx, event)

      // Step 3e: OPTIONAL - Fire webhook immediately (fire-and-forget)
      if (task.originatingFlowRunId) {
        try {
          await fetch('https://windmill.example.com/webhooks/u/system/lifecycle-event-handler', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(event),
          })
        } catch (error) {
          // Log error but don't fail transaction (event is in outbox as backup)
          console.error('Failed to fire immediate webhook:', error)
        }
      }

      return completedTask
    })

    // ========== STEP 4: RETURN SUCCESS RESULT ==========
    return {
      success: true,
      data: { task: result }
    }

  } catch (error) {
    // ========== ERROR HANDLING (Windmill-friendly structured errors) ==========
    if (error instanceof DrizzleQueryError) {
      return {
        success: false,
        errorType: 'DATABASE_ERROR',
        message: 'Database query failed',
        details: {
          sql: error.sql,
          params: error.params,
          cause: error.cause
        },
        retryable: true,
        action: 'RETRY_WITH_BACKOFF'
      }
    }

    if (error instanceof Error) {
      if (error.message === 'NOT_FOUND') {
        return {
          success: false,
          errorType: 'NOT_FOUND',
          message: `Task with ID ${params.taskId} not found`,
          details: { entityType: 'Task', entityId: params.taskId },
          retryable: false,
          action: 'VERIFY_TASK_EXISTS'
        }
      }

      if (error.message === 'ALREADY_COMPLETED') {
        return {
          success: false,
          errorType: 'INVALID_STATE',
          message: 'Task is already completed',
          details: { currentStatus: 'completed' },
          retryable: false,
          action: 'NO_ACTION_REQUIRED'
        }
      }

      if (error.message === 'TASK_CANCELLED') {
        return {
          success: false,
          errorType: 'INVALID_STATE',
          message: 'Cannot complete a cancelled task',
          details: { currentStatus: 'cancelled' },
          retryable: false,
          action: 'VERIFY_TASK_STATUS'
        }
      }
    }

    // Unknown error
    return {
      success: false,
      errorType: 'INTERNAL_ERROR',
      message: error instanceof Error ? error.message : 'Unknown error occurred',
      details: { originalError: String(error) },
      retryable: false,
      action: 'CONTACT_SUPPORT'
    }
  }
}
```

### 17.2 Full Query with Computed Properties

```typescript
// /lifecycle/contract/get-details.ts

import { eq } from 'drizzle-orm'
import { createDbConnection } from '../lib/db'
import type { PostgresqlResource } from '../lib/db'
import { contracts } from '../lib/schema'
import { z } from 'zod'

// Input validation schema (Zod 4 with z.interface and metadata)
const inputSchema = z.interface({
  contractId: z.uuid().meta({
    title: "Contract ID",
    description: "UUID of contract to retrieve with computed properties",
    context: "Returns contract + computed business phase, eligibility, etc. Cached 10min.",
    validationPriority: "critical"
  })
})

export async function main(
  dbResource: PostgresqlResource,
  rawParams: unknown
) {
  // ========== STEP 1: VALIDATE INPUT (safeParse pattern) ==========
  const parseResult = inputSchema.safeParse(rawParams)

  if (!parseResult.success) {
    return {
      success: false,
      errorType: 'VALIDATION_ERROR',
      message: 'Invalid input parameters',
      details: {
        summary: z.prettifyError(parseResult.error),
        fields: z.flattenError(parseResult.error).fieldErrors
      },
      retryable: true
    }
  }

  const { contractId } = parseResult.data

  // ========== STEP 2: CREATE DATABASE CONNECTION WITH CACHING ==========
  const db = createDbConnection(dbResource, { cache: true })

  // ========== STEP 3: QUERY WITH CACHE ==========
  const results = await db
    .select()
    .from(contracts)
    .where(eq(contracts.id, contractId))
    .$withCache({
      tag: `contract-details:${contractId}`,
      config: { ex: 600 } // TTL: 10 minutes
    })

  const contract = results[0]

  if (!contract) {
    return {
      success: false,
      errorType: 'NOT_FOUND',
      message: `Contract with ID ${contractId} not found`,
      details: { entityType: 'Contract', entityId: contractId },
      retryable: false
    }
  }

  // ========== STEP 4: COMPUTE PROPERTIES (Pure, deterministic functions) ==========
  const now = new Date()
  const startDate = new Date(contract.startDate)
  const daysSinceStart = Math.floor((now.getTime() - startDate.getTime()) / (1000 * 60 * 60 * 24))

  const endDate = contract.endDate ? new Date(contract.endDate) : null
  const daysUntilEnd = endDate
    ? Math.floor((endDate.getTime() - now.getTime()) / (1000 * 60 * 60 * 24))
    : null

  const isBonusEligible = contract.status === 'active' && daysSinceStart >= 60

  const isWithinCancellationWindow =
    daysUntilEnd !== null && daysUntilEnd >= 30 && daysUntilEnd <= 90

  const currentBusinessPhase =
    contract.status === 'pending_verification'
      ? 'activation'
      : contract.status === 'price_increase_reported'
      ? 'optimization'
      : isWithinCancellationWindow
      ? 'renewal'
      : 'monitoring'

  // ========== STEP 5: RETURN SUCCESS WITH COMPUTED PROPERTIES ==========
  return {
    success: true,
    data: {
      ...contract,
      computed: {
        is_bonus_eligible: isBonusEligible,
        is_within_cancellation_window: isWithinCancellationWindow,
        days_until_renewal: daysUntilEnd,
        days_since_start: daysSinceStart,
        current_business_phase: currentBusinessPhase,
        effective_monthly_price: contract.pendingPriceChange || contract.price,
      },
    }
  }
}
```

---

## 18. Troubleshooting Guide

### 18.1 Common Errors

**Error: `relation "contracts" does not exist`**

**Cause**: Migrations not applied to database

**Solution**:
```bash
npx drizzle-kit push
# OR
wmill run lifecycle/migrations/run-migration
```

---

**Error: `Cannot find module 'drizzle-orm/neon-http'`**

**Cause**: Wrong Drizzle version or incorrect import path

**Solution**:
```bash
npm install drizzle-orm@latest @neondatabase/serverless@latest
```

Verify imports:
```typescript
import { drizzle } from 'drizzle-orm/neon-http' // ✅ CORRECT
import { drizzle } from 'drizzle-orm/postgres-js' // ❌ WRONG
```

---

**Error: `channel_binding not supported`**

**Cause**: Older Neon client version

**Solution**:
```bash
npm install @neondatabase/serverless@latest
```

Or remove `channel_binding=require` from connection string (not recommended).

---

**Error: `row-level security policy for table "contracts" violated`**

**Cause**: Using wrong database role or JWT not set

**Solution**:

1. Check Windmill resource is using correct role (NOT `neondb_owner`)
2. If using JWT auth, ensure token is passed correctly:

```typescript
const db = createAuthenticatedDbConnection(dbResource, authToken)
```

3. Temporarily disable RLS for debugging:

```sql
ALTER TABLE contracts DISABLE ROW LEVEL SECURITY;
```

---

**Error: `Zod validation failed` with unclear error message**

**Cause**: Input data doesn't match schema (Zod 4)

**Solution**: Use Zod 4 error formatting with safeParse pattern:

```typescript
const parseResult = schema.safeParse(rawParams)

if (!parseResult.success) {
  // Zod 4 enhanced error formatting
  console.error('Validation Summary:', z.prettifyError(parseResult.error))
  console.error('Field Errors:', z.flattenError(parseResult.error).fieldErrors)
  console.error('Tree Structure:', z.treeifyError(parseResult.error))

  // Return structured error (don't throw)
  return {
    success: false,
    errorType: 'VALIDATION_ERROR',
    message: 'Input validation failed',
    details: {
      summary: z.prettifyError(parseResult.error),
      fields: z.flattenError(parseResult.error).fieldErrors
    }
  }
}
```

---

**Error: `DrizzleQueryError: Query failed` (Drizzle 0.44.0+)**

**Cause**: Database query failed with enhanced error transparency

**Solution**: Extract SQL and params for debugging:

```typescript
import { DrizzleQueryError } from 'drizzle-orm'

try {
  const result = await db.select().from(contracts).where(...)
} catch (error) {
  if (error instanceof DrizzleQueryError) {
    console.error('Failed SQL:', error.sql)
    console.error('Query Params:', error.params)
    console.error('Underlying Cause:', error.cause)
    console.error('Stack Trace:', error.stack)

    // Return structured error
    return {
      success: false,
      errorType: 'DATABASE_QUERY_ERROR',
      message: 'Database query failed',
      details: {
        sql: error.sql,
        params: error.params,
        cause: error.cause
      },
      retryable: true
    }
  }
  throw error
}
```

---

**Error: `z.interface is not a function`**

**Cause**: Using Zod 3 instead of Zod 4

**Solution**: Upgrade to Zod 4.1.12+:

```bash
npm install zod@^4.1.12
```

Verify import:
```typescript
import { z } from 'zod'

// Zod 4 only - will fail on Zod 3
const schema = z.interface({
  id: z.uuid(),
  "optional?": z.string()
})
```

---

### 18.2 Performance Issues

**Symptom**: Queries taking >500ms

**Diagnosis**:
1. Check if indexes exist:

```sql
SELECT tablename, indexname, indexdef
FROM pg_indexes
WHERE schemaname = 'public'
AND tablename = 'contracts';
```

2. Analyze query plan:

```sql
EXPLAIN ANALYZE
SELECT * FROM contracts WHERE user_id = 'xxx' AND status = 'active';
```

**Solution**: Add missing indexes:

```typescript
// In schema.ts
index('contracts_user_status_idx').on(table.userId, table.status)
```

Then run migration.

---

**Symptom**: High database CPU usage

**Cause**: N+1 queries or missing batch operations

**Solution**: Use Drizzle query logging to identify:

```typescript
const db = drizzle(sql, { schema, logger: true })
```

Replace loops with batch operations (see Section 15.2).

---

### 18.3 Debugging Checklist

When a business intent fails:

1. ✅ **Check input validation** - Log raw input before parsing
2. ✅ **Check database connection** - Verify resource credentials
3. ✅ **Check RLS policies** - Ensure user has permission
4. ✅ **Check business rules** - Log entity state before validation
5. ✅ **Check transaction rollback** - Look for unhandled exceptions
6. ✅ **Check event emission** - Verify outbox has entry

---

## 19. Best Practices

### 19.1 Schema Design

1. ✅ **Always use UUIDs** for primary keys (distributed system safety)
2. ✅ **Include created_at/updated_at** on all entities (auditability)
3. ✅ **Use enums for status fields** (type safety)
4. ✅ **Add indexes on foreign keys** (query performance)
5. ✅ **Use jsonb for flexible metadata** (avoid schema churn)
6. ⚠️ **Avoid nullable foreign keys** (use separate junction tables)

### 19.2 Validation

1. ✅ **Use Zod 4 z.interface() for business intents** (precise optionality control)
2. ✅ **Add rich metadata with .meta()** (title, description, context, examples)
3. ✅ **Use safeParse() instead of parse()** (Windmill-friendly, no throws)
4. ✅ **Use top-level validators** (z.uuid(), z.email(), z.hash() - 14× faster)
5. ✅ **Validate at the edge** (earliest possible point)
6. ✅ **Use refinements for business rules** (not just type checks)
7. ✅ **Provide clear error messages** (z.prettifyError(), z.flattenError())
8. ✅ **Return structured errors** (not throwing - Windmill flows complete)
9. ⚠️ **Don't duplicate validation** (single source of truth in schema)
10. ⚠️ **Don't use z.object() for business intents** (use z.interface() instead)

### 19.3 Transactions

1. ✅ **Keep transactions short** (<100ms ideal)
2. ✅ **Use pessimistic locking** for state machines
3. ✅ **Emit events in same transaction** (atomicity)
4. ✅ **Validate before mutating** (fail fast)
5. ⚠️ **Avoid external API calls** in transactions (use outbox)

### 19.4 Error Handling

1. ✅ **Use typed errors** (NotFoundError, ValidationError, etc.)
2. ✅ **Include context in errors** (entity ID, current state)
3. ✅ **Log structured data** (JSON, not strings)
4. ✅ **Return user-friendly messages** (hide implementation details)
5. ⚠️ **Never expose stack traces** to end users

### 19.5 Performance

1. ✅ **Select only needed columns** (avoid SELECT *)
2. ✅ **Use batch operations** (avoid loops)
3. ✅ **Add indexes proactively** (before scaling)
4. ✅ **Monitor query execution times** (set up alerts)
5. ⚠️ **Don't over-index** (balance read vs write performance)

---

## 20. Reference

### 20.1 Configuration Files

**drizzle.config.ts:**

```typescript
import type { Config } from 'drizzle-kit'
import * as dotenv from 'dotenv'

dotenv.config()

export default {
  schema: './lifecycle/lib/schema.ts',
  out: './lifecycle/migrations/meta',
  dialect: 'postgresql',
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
  verbose: true,
  strict: true,
} satisfies Config
```

**package.json:**

```json
{
  "name": "switchup-windmill",
  "version": "1.0.0",
  "description": "SwitchUp Windmill Platform - Lifecycle Domain",
  "scripts": {
    "db:generate": "drizzle-kit generate",
    "db:push": "drizzle-kit push",
    "db:studio": "drizzle-kit studio",
    "test": "vitest",
    "lint": "eslint .",
    "type-check": "tsc --noEmit"
  },
  "dependencies": {
    "drizzle-orm": "^0.44.7",
    "@neondatabase/serverless": "^1.0.2",
    "zod": "^4.1.12",
    "drizzle-zod": "^0.8.3"
  },
  "devDependencies": {
    "drizzle-kit": "^0.31.6",
    "@types/node": "^22.0.0",
    "typescript": "^5.7.0",
    "vitest": "^2.0.0",
    "eslint": "^9.0.0",
    "@typescript-eslint/eslint-plugin": "^8.0.0",
    "@typescript-eslint/parser": "^8.0.0"
  },
  "engines": {
    "node": ">=19.0.0"
  }
}
```

### 20.2 Environment Variables

```bash
# .env

# Neon connection string with security parameters
DATABASE_URL=postgres://app_user:password@ep-xyz.us-east-2.aws.neon.tech:5432/switchup_production?sslmode=require&channel_binding=require

# For RLS testing (optional)
DATABASE_URL_ADMIN=postgres://neondb_owner:password@...
DATABASE_URL_AUTHENTICATED=postgres://app_user:password@...
DATABASE_URL_ANONYMOUS=postgres://anonymous_user:password@...
```

### 20.3 Key Commands

```bash
# Generate migration
npx drizzle-kit generate

# Push schema directly (dev only)
npx drizzle-kit push

# Run migration (via Windmill script)
wmill run lifecycle/migrations/run-latest

# Sync to Windmill
wmill sync push

# Run tests
npm test
```

### 20.4 Import Patterns

**Correct imports for Drizzle 0.44.7 + Zod 4.1.12:**

```typescript
// ========== DATABASE CONNECTION ==========
import { neon } from '@neondatabase/serverless'
import { drizzle } from 'drizzle-orm/neon-http'
import type { NeonHttpDatabase, DrizzleConfig } from 'drizzle-orm/neon-http'

// ========== DRIZZLE SCHEMA ==========
import { pgTable, varchar, uuid, integer, timestamp, jsonb } from 'drizzle-orm/pg-core'
import { sql, eq, and, or, gte, lte, inArray } from 'drizzle-orm'
import type { SQL } from 'drizzle-orm'

// ========== DRIZZLE 2025 FEATURES ==========
import { DrizzleQueryError } from 'drizzle-orm'  // Enhanced error handling (0.44.0+)
import { upstashCache } from 'drizzle-orm/cache/upstash'  // Native caching (0.44.0+)

// ========== NEON-SPECIFIC RLS HELPERS ==========
import { crudPolicy, authenticatedRole, authUid } from 'drizzle-orm/neon'

// ========== ZOD 4 VALIDATION ==========
import { z } from 'zod'
import { createSelectSchema, createInsertSchema, createUpdateSchema } from 'drizzle-zod'

// ========== TYPE INFERENCE ==========
import type { InferSelectModel, InferInsertModel } from 'drizzle-orm'
```

**Common mistakes to avoid:**

```typescript
// ❌ WRONG - Using postgres-js imports
import { drizzle } from 'drizzle-orm/postgres-js'

// ✅ CORRECT - Using neon-http
import { drizzle } from 'drizzle-orm/neon-http'

// ❌ WRONG - Old Zod 3 patterns
import { ZodError } from 'zod'
const schema = z.object({ ... })
const data = schema.parse(input)  // Throws

// ✅ CORRECT - Zod 4 patterns
import { z } from 'zod'
const schema = z.interface({ ... })
const result = schema.safeParse(input)  // Returns result object

// ❌ WRONG - Missing DrizzleQueryError import
try {
  await db.select()...
} catch (error) {
  // Can't handle DrizzleQueryError properly
}

// ✅ CORRECT - Import DrizzleQueryError
import { DrizzleQueryError } from 'drizzle-orm'
try {
  await db.select()...
} catch (error) {
  if (error instanceof DrizzleQueryError) {
    // Access error.sql, error.params, error.cause
  }
}
```

### 20.5 Links

- **Drizzle ORM**: https://orm.drizzle.team
- **Drizzle + Neon Guide**: https://orm.drizzle.team/docs/get-started/neon-new
- **Drizzle Zod**: https://orm.drizzle.team/docs/zod
- **Drizzle RLS**: https://orm.drizzle.team/docs/rls
- **Drizzle Caching**: https://orm.drizzle.team/docs/caching
- **Neon**: https://neon.tech
- **Neon Serverless Driver**: https://neon.tech/docs/serverless/serverless-driver
- **Neon + Drizzle Guide**: https://neon.com/docs/guides/drizzle
- **Neon RLS + Drizzle**: https://neon.com/docs/guides/rls-drizzle
- **Zod 4**: https://zod.dev
- **Zod Metadata**: https://zod.dev/?id=metadata
- **Windmill**: https://windmill.dev
- **Upstash Redis**: https://upstash.com/docs/redis

---

## 21. AI Tool Integration

### 21.1 Metadata-Driven Function Calling

Zod 4's metadata system enables automatic JSON Schema generation for AI agent tools. This allows business intents to be directly exposed as LLM function calls.

**Pattern** (`/lifecycle/lib/ai-tools.ts`):

```typescript
import { z } from 'zod'
import { reportPriceIncreaseSchema, createTaskSchema, completeTaskSchema } from './validation'

/**
 * Convert Zod schemas to OpenAPI 3.0 JSON Schema for LLM function calling
 */
export const LifecycleTools = {
  reportPriceIncrease: {
    name: "report_price_increase",
    description: "Record a provider-initiated price increase on an active contract. This triggers the optimization workflow.",
    parameters: z.toJSONSchema(reportPriceIncreaseSchema, {
      target: 'openapi-3.0',
      metadata: z.globalRegistry // Include all metadata
    })
  },

  createTask: {
    name: "create_task",
    description: "Create a human-in-the-loop task for manual intervention. Used when automation cannot proceed.",
    parameters: z.toJSONSchema(createTaskSchema, {
      target: 'openapi-3.0',
      metadata: z.globalRegistry
    })
  },

  completeTask: {
    name: "complete_task",
    description: "Mark a task as completed with resolution data. Resumes the originating workflow.",
    parameters: z.toJSONSchema(completeTaskSchema, {
      target: 'openapi-3.0',
      metadata: z.globalRegistry
    })
  }
}

/**
 * Format for LLM provider (OpenAI, Anthropic, etc.)
 */
export const tools = Object.values(LifecycleTools).map(tool => ({
  type: "function" as const,
  function: tool
}))
```

**Usage with LLM**:

```typescript
// In /case domain or AI agent script
import { tools } from '../../lifecycle/lib/ai-tools'

const completion = await openai.chat.completions.create({
  model: "gpt-4",
  messages: [
    {
      role: "system",
      content: "You are an AI agent managing contract lifecycle events. Use tools to interact with the system."
    },
    {
      role: "user",
      content: "Provider sent email: price increasing from £100 to £120 effective Jan 1st 2025"
    }
  ],
  tools: tools,
  tool_choice: "auto"
})

// LLM will call report_price_increase with parameters extracted from context
const toolCall = completion.choices[0].message.tool_calls[0]
const params = JSON.parse(toolCall.function.arguments)

// Execute the business intent
const result = await wmill.call('f/lifecycle/contract/report-price-increase', params)
```

**Why Metadata Matters**:

The `context` field in metadata provides AI agents with domain knowledge:

```typescript
contractId: z.uuid().meta({
  context: "Must be existing active contract. Query lifecycle to verify."
})
```

This enables the LLM to:
- Understand validation requirements
- Make informed decisions about field values
- Provide better error recovery suggestions
- Generate contextually appropriate test data

---

### 21.2 Structured Output Validation

Zod 4's JSON Schema generation also works for validating LLM responses:

```typescript
// Define expected LLM output structure
const strategyDecisionSchema = z.interface({
  decision: z.enum(['accept_increase', 'negotiate', 'switch_provider', 'cancel']),
  reasoning: z.string().min(50).meta({
    description: "Explanation for decision",
    context: "Should reference contract terms, market conditions, user preferences"
  }),
  "urgency?": z.enum(['immediate', 'within_week', 'within_month', 'no_rush']),
  "estimatedSavings?": z.number().int().meta({
    description: "Potential savings in cents if switching",
    examples: [5000, 10000, 15000]
  })
})

// Request structured output from LLM
const completion = await openai.beta.chat.completions.parse({
  model: "gpt-4-1106-preview",
  messages: [
    {
      role: "system",
      content: "Analyze price increase and recommend strategy"
    },
    {
      role: "user",
      content: JSON.stringify({
        contract: contractData,
        priceIncrease: priceIncreaseData,
        marketOffers: marketData
      })
    }
  ],
  response_format: {
    type: "json_schema",
    json_schema: {
      name: "strategy_decision",
      schema: z.toJSONSchema(strategyDecisionSchema, { target: 'openapi-3.0' })
    }
  }
})

// Validate LLM response with Zod
const parseResult = strategyDecisionSchema.safeParse(
  JSON.parse(completion.choices[0].message.content)
)

if (!parseResult.success) {
  // LLM output didn't match schema - retry with error feedback
  console.error("LLM output validation failed:", z.prettifyError(parseResult.error))
}

const strategy = parseResult.data
```

---

### 21.3 AI-Generated Test Data

Zod metadata with examples enables AI-assisted test data generation:

```typescript
// Test data generator script (AI-powered)
import { contractInsertSchema } from './lib/validation'
import { generateMock } from '@anatine/zod-mock'

// Generate realistic test data from schema + metadata
const mockContract = generateMock(contractInsertSchema, {
  seed: 12345 // Deterministic for tests
})

// Output uses examples and context from metadata:
// {
//   userId: 1001,
//   uuid: "550e8400-e29b-41d4-a716-446655440000",
//   status: "active",
//   providerId: "provider-abc",
//   serviceType: "electricity",
//   price: 15000, // From metadata examples
//   startDate: "2024-01-01T00:00:00Z",
//   ...
// }
```

---

**End of Specification**

This specification explores implementation patterns with Zod 4.1.12 and Drizzle ORM 0.44.7, researching latest features and potential best practices for Windmill's serverless execution model.

**Note**: These patterns are subject to refinement as practical implementation experience is gained.
