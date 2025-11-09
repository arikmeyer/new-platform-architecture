# Naming Convention Strategy

**Version:** 1.0
**Last Updated:** 2025-11-09
**Purpose:** Definitive naming conventions for SwitchUp's Windmill + Drizzle + Neon stack
**Scope:** /lifecycle domain and all future domains

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Core Principles](#2-core-principles)
3. [Language Layer Separation](#3-language-layer-separation)
4. [TypeScript Naming Conventions](#4-typescript-naming-conventions)
5. [Database Naming Conventions](#5-database-naming-conventions)
6. [Event System Naming](#6-event-system-naming)
7. [Special Patterns](#7-special-patterns)
8. [Critical Issues & Migration](#8-critical-issues--migration)
9. [Decision Trees](#9-decision-trees)
10. [Anti-Patterns](#10-anti-patterns)
11. [Examples Reference](#11-examples-reference)

---

## 1. Executive Summary

### Purpose

This document establishes **unambiguous naming conventions** to ensure consistency across the SwitchUp platform. Naming clarity is critical for:

- **AI-assisted development** (85% code generation accuracy depends on predictable patterns)
- **Multi-developer collaboration** (avoiding "what should I name this?" debates)
- **Long-term maintainability** (code is read 10× more than written)
- **Cross-domain consistency** (lifecycle, case, provider, offer domains)

### Key Findings from Current Specification

✅ **Strengths:**
- TypeScript conventions are consistent (camelCase for variables/functions, PascalCase for types)
- File/path naming is consistent (kebab-case)
- Database column mapping follows Drizzle best practices

🚨 **Critical Issues:**
1. **Enum values use PascalCase** instead of PostgreSQL-standard snake_case or SCREAMING_SNAKE_CASE
2. **LifecycleEvent interface uses snake_case** properties, inconsistent with TypeScript conventions

⚠️ **Documentation Gaps:**
- Abbreviation strategy not documented
- Constant naming convention not established
- Dual naming system (TS vs SQL) not explicit

---

## 2. Core Principles

### Principle 1: Language-Appropriate Conventions

**Never force TypeScript conventions into SQL, or SQL conventions into TypeScript.**

- **TypeScript layer**: Follow JavaScript/TypeScript community standards (camelCase, PascalCase)
- **Database layer**: Follow PostgreSQL community standards (snake_case, lowercase)
- **Drizzle ORM**: Automatically maps between these layers

**Why:** Each ecosystem has established norms. Fighting them creates friction.

### Principle 2: Consistency Over Personal Preference

**Once a pattern is established, it's sacred.**

- Individual preference is irrelevant
- AI code generation depends on predictability
- Code reviews should enforce naming conventions mechanically

**Why:** Inconsistency is more costly than any specific choice.

### Principle 3: Explicit Over Implicit

**Never assume a naming pattern is "obvious."**

- Document prefixes, suffixes, and special cases
- Provide decision trees for ambiguous scenarios
- Make abbreviation rules explicit

**Why:** What's obvious to one developer is mysterious to another (or to AI).

### Principle 4: Optimize for Reading, Not Writing

**Code is read 10× more than written.**

- Favor descriptive names over short names
- Use full words unless abbreviation is universal
- Accept longer names for clarity

**Why:** Developer time spent reading code dwarfs time spent typing.

### Principle 5: Future-Proof Conventions

**Design for scale and evolution.**

- Avoid patterns that break with new entity types
- Use namespacing to prevent collisions
- Plan for 100+ business intents, not 10

**Why:** Today's shortcut becomes tomorrow's technical debt.

---

## 3. Language Layer Separation

### The Dual Naming System

SwitchUp uses **two parallel naming systems** that coexist:

| Layer          | Convention           | Example                 | Rationale               |
| -------------- | -------------------- | ----------------------- | ----------------------- |
| **TypeScript** | camelCase properties | `userId`, `createdAt`   | JavaScript/TS standard  |
| **PostgreSQL** | snake_case columns   | `user_id`, `created_at` | SQL database standard   |
| **Mapping**    | Automatic (Drizzle)  | N/A                     | ORM handles translation |

### Why This Exists

**TypeScript Code:**
```typescript
const contract = {
  id: 'abc-123',
  userId: 'user-456',
  createdAt: new Date(),
}
```

**PostgreSQL Table:**
```sql
CREATE TABLE contracts (
  id UUID,
  user_id UUID,
  created_at TIMESTAMP
);
```

**Drizzle Schema (The Bridge):**
```typescript
export const contracts = pgTable('contracts', {
  id: uuid('id'),                    // TS: id, SQL: id
  userId: uuid('user_id'),           // TS: userId, SQL: user_id
  createdAt: timestamp('created_at'), // TS: createdAt, SQL: created_at
})
```

### Critical Rule

**NEVER mix conventions within a layer:**

❌ **WRONG (mixing in TypeScript):**
```typescript
interface User {
  userId: string      // camelCase
  first_name: string  // snake_case ← WRONG!
}
```

❌ **WRONG (mixing in SQL):**
```sql
CREATE TABLE users (
  user_id UUID,       -- snake_case
  firstName VARCHAR   -- camelCase ← WRONG!
);
```

✅ **CORRECT:**
```typescript
// TypeScript layer
interface User {
  userId: string
  firstName: string
}

// Drizzle schema (bridge)
export const users = pgTable('users', {
  userId: uuid('user_id'),
  firstName: varchar('first_name'),
})
```

---

## 4. TypeScript Naming Conventions

### 4.1 Files and Paths

**Rule: Use kebab-case for file names, lowercase for directories**

```
✅ CORRECT:
/lifecycle/contract/report-price-increase.ts
/lifecycle/lib/db.ts
/lifecycle/task/complete-task.ts
/case/ingestion/lifecycle-event-handler.ts

❌ WRONG:
/lifecycle/contract/reportPriceIncrease.ts   (camelCase)
/lifecycle/contract/report_price_increase.ts (snake_case)
/lifecycle/Contract/report-price-increase.ts (PascalCase directory)
```

**Rationale:**
- Kebab-case is URL-friendly (important for Windmill paths)
- Lowercase directories avoid case-sensitivity issues on different filesystems
- Consistent with web development standards

### 4.2 Variables and Parameters

**Rule: Use camelCase for all variables and function parameters**

```typescript
✅ CORRECT:
const dbResource: PostgresqlResource
const rawParams: unknown
const params: ReportPriceIncreaseParams
const connectionString: string
const priceHistory: PriceIncreaseEvent[]

❌ WRONG:
const db_resource: PostgresqlResource  // snake_case
const RawParams: unknown               // PascalCase
const Params: CompleteTaskParams       // PascalCase
```

**Special Prefixes:**
- `raw` = unvalidated input (e.g., `rawParams`)
- No prefix = validated/processed (e.g., `params`)

### 4.3 Functions

**Rule: Use camelCase, verb-first naming**

```typescript
✅ CORRECT:
function createDbConnection()
function emitEvent()
function handleZodError()
function computeBonusEligibility()
function validatePriceIncrease()

❌ WRONG:
function CreateDbConnection()  // PascalCase
function emit_event()          // snake_case
function bonusEligibility()    // missing verb
function db_connection()       // snake_case, missing verb
```

**Common Verb Prefixes:**
- `create` = factory function
- `get` / `fetch` = retrieve data
- `compute` / `calculate` = derive value
- `validate` = check validity
- `handle` = error/event handler
- `with` = higher-order function wrapper

### 4.4 Types and Interfaces

**Rule: Use PascalCase for all types, interfaces, and type aliases**

```typescript
✅ CORRECT:
interface LifecycleEvent
type PostgresqlResource
type ReportPriceIncreaseParams
interface StateSnapshot
type ContractInsert

❌ WRONG:
interface lifecycleEvent   // camelCase
type postgresql_resource   // snake_case
type reportPriceIncreaseParams  // camelCase
```

**Suffix Conventions:**
- `Params` = input parameter types (e.g., `CreateTaskParams`)
- `Insert` = Drizzle insert schema type (e.g., `ContractInsert`)
- `Update` = Drizzle update schema type (e.g., `ContractUpdate`)
- `Select` = Drizzle select schema type (e.g., `Contract` or `ContractSelect`)
- `Error` = error class types (e.g., `NotFoundError`)
- No suffix = base entity types (e.g., `Contract`, `User`)

### 4.5 Classes

**Rule: Use PascalCase for class names**

```typescript
✅ CORRECT:
class LifecycleError extends Error
class NotFoundError extends LifecycleError
class InvalidCommandError extends LifecycleError

❌ WRONG:
class lifecycleError       // camelCase
class not_found_error      // snake_case
```

### 4.6 Enums and Enum Values

**Rule: Enum constants use camelCase, enum VALUES must use snake_case**

🚨 **CRITICAL CHANGE FROM CURRENT SPEC:**

```typescript
✅ CORRECT (NEW STANDARD):
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

❌ WRONG (CURRENT SPEC - NEEDS MIGRATION):
export const contractStatus = pgEnum('contract_status', [
  'PendingVerification',  // PascalCase ← WRONG
  'Active',               // PascalCase ← WRONG
  'InProgress',           // PascalCase ← WRONG
])
```

**Rationale:**
- PostgreSQL convention uses lowercase for enum values
- Mixing TypeScript conventions (PascalCase) into database layer violates Principle 1
- SQL queries will use lowercase: `WHERE status = 'active'` (standard)
- See [Section 8](#8-critical-issues--migration) for migration guide

**Alternative (if uppercase preferred):**
```typescript
export const contractStatus = pgEnum('contract_status', [
  'PENDING_VERIFICATION',
  'ACTIVE',
  'SWITCH_PENDING',
])
```
Both lowercase and SCREAMING_SNAKE_CASE are acceptable PostgreSQL conventions. **Choose one and be consistent.**

### 4.7 Schema Objects (Zod)

**Rule: Use camelCase with purpose-based suffixes**

```typescript
✅ CORRECT:
// Entity CRUD schemas (with refinements built in)
const contractSelectSchema = createSelectSchema(contracts)
const contractInsertSchema = createInsertSchema(contracts, {
  serviceType: (schema) => schema.serviceType.max(50),
  price: (schema) => schema.price.positive(),
})
const contractUpdateSchema = createUpdateSchema(contracts)

// Business intent schemas (purpose-specific)
const reportPriceIncreaseSchema = z.object({ ... })

// Specialized schemas (different context)
const contractInsertApiSchema = contractInsertSchema.omit({ id: true })

❌ WRONG:
const ContractSelectSchema  // PascalCase
const contract_select_schema  // snake_case
const contractSelect  // missing "Schema" suffix
const contractInsertRefined  // "Refined" suffix (anti-pattern)
const contractInsertV2  // version suffix (anti-pattern)
```

**Naming Patterns:**

1. **Entity CRUD Schemas**: `{entity}{Operation}Schema`
   - Always include ALL validation upfront
   - Build refinements directly into `createInsertSchema`
   - Examples: `contractInsertSchema`, `userUpdateSchema`

2. **Business Intent Schemas**: `{verbNoun}Schema`
   - Named after the specific operation
   - Examples: `reportPriceIncreaseSchema`, `createTaskSchema`

3. **Specialized Schemas**: `{entity}{Operation}{Context}Schema`
   - Only when genuinely different use case (not just better validation)
   - Examples: `contractInsertApiSchema`, `userSelectPublicSchema`

**CRITICAL: Never use these suffixes:**
- ❌ `Refined` - suggests parallel versions (anti-pattern)
- ❌ `Enhanced` - same issue
- ❌ `Extended` - same issue
- ❌ `V2`, `V3` - versioning schemas is an anti-pattern
- ❌ `Better`, `Improved` - obviously wrong

**Golden Rule**: If you're creating a "better" version of an existing schema, **delete the old version and improve the original**.

### 4.8 Constants

**Rule: Use SCREAMING_SNAKE_CASE for module-level constants**

```typescript
✅ CORRECT:
const MAX_RETRY_ATTEMPTS = 3
const DEFAULT_BATCH_SIZE = 50
const WEBHOOK_TIMEOUT_MS = 5000
const POSTGRES_CONNECTION_TIMEOUT = 30

❌ WRONG:
const maxRetryAttempts = 3  // camelCase
const default_batch_size = 50  // lowercase snake_case
const DefaultBatchSize = 50  // PascalCase
```

**Exception:** Configuration objects can use camelCase:
```typescript
const defaultConfig = {
  maxRetryAttempts: 3,
  batchSize: 50,
  timeoutMs: 5000,
}
```

### 4.9 Abbreviations

**Rule: Only abbreviate universally understood terms**

| Term                      | Abbreviation | Usage                                           |
| ------------------------- | ------------ | ----------------------------------------------- |
| database                  | `db`         | ✅ Use `db` (universal)                          |
| transaction               | `tx`         | ✅ Use `tx` (universal in DB contexts)           |
| Structured Query Language | `sql`        | ✅ Use `sql` (universal)                         |
| identifier                | `id`         | ✅ Use `id` (universal)                          |
| parameter                 | N/A          | ❌ Use `param` or `parameter`, not `prm`         |
| resource                  | N/A          | ❌ Use `resource`, not `res`                     |
| configuration             | `config`     | ✅ Use `config` (common)                         |
| environment               | `env`        | ✅ Use `env` (common)                            |
| context                   | `ctx`        | ⚠️ Avoid unless framework convention (e.g., Koa) |

**Examples:**
```typescript
✅ CORRECT:
const db = createDbConnection(dbResource)
const tx = db.transaction()
const userId = params.id
const config = loadConfig()

❌ WRONG:
const dbRes = createDbConnection(dbRes)  // "Res" is ambiguous
const trans = db.transaction()  // "trans" is unclear
const usrId = params.id  // "usr" is unnecessary
```

---

## 5. Database Naming Conventions

### 5.1 Table Names

**Rule: Use lowercase, plural nouns, snake_case for multi-word names**

```sql
✅ CORRECT:
CREATE TABLE users
CREATE TABLE contracts
CREATE TABLE price_increase_events
CREATE TABLE outbox_events

❌ WRONG:
CREATE TABLE User           -- PascalCase, singular
CREATE TABLE priceIncreaseEvents  -- camelCase
CREATE TABLE CONTRACTS      -- UPPERCASE
```

**Drizzle Mapping:**
```typescript
export const users = pgTable('users', { ... })
export const contracts = pgTable('contracts', { ... })
export const priceIncreaseEvents = pgTable('price_increase_events', { ... })
```

### 5.2 Column Names

**Rule: Use lowercase, snake_case for all columns**

```sql
✅ CORRECT:
CREATE TABLE contracts (
  id UUID,
  user_id UUID,
  created_at TIMESTAMP,
  pending_price_change INTEGER
);

❌ WRONG:
CREATE TABLE contracts (
  ID UUID,                -- UPPERCASE
  userId UUID,            -- camelCase
  created_at TIMESTAMP,   -- OK
  PendingPriceChange INT  -- PascalCase
);
```

**Drizzle Mapping:**
```typescript
export const contracts = pgTable('contracts', {
  id: uuid('id'),
  userId: uuid('user_id'),
  createdAt: timestamp('created_at'),
  pendingPriceChange: integer('pending_price_change'),
})
```

### 5.3 Foreign Key Columns

**Rule: Use `{singular_entity}_id` pattern**

```sql
✅ CORRECT:
user_id UUID REFERENCES users(id)
contract_id UUID REFERENCES contracts(id)
provider_id UUID REFERENCES providers(id)

❌ WRONG:
users_id UUID        -- plural
id_user UUID         -- reversed order
user UUID            -- missing "_id" suffix
fk_user_id UUID      -- unnecessary "fk_" prefix
```

### 5.4 Timestamps

**Rule: Use `{action}_at` pattern**

```sql
✅ CORRECT:
created_at TIMESTAMP
updated_at TIMESTAMP
deleted_at TIMESTAMP
completed_at TIMESTAMP
published_at TIMESTAMP

❌ WRONG:
creation_date TIMESTAMP     -- use "_at" suffix
last_updated TIMESTAMP      -- use "updated_at"
deletion_timestamp TIMESTAMP -- use "_at" suffix
```

### 5.5 Boolean Columns

**Rule: Use `is_{adjective}` or `has_{noun}` pattern**

```sql
✅ CORRECT:
is_active BOOLEAN
is_deleted BOOLEAN
has_verified_email BOOLEAN
has_pending_tasks BOOLEAN

❌ WRONG:
active BOOLEAN          -- missing "is_" prefix
verified_email BOOLEAN  -- missing "has_" prefix
deleted BOOLEAN         -- ambiguous (use "is_deleted")
```

### 5.6 Enum Columns

**Rule: Use singular noun without "_type" suffix**

```sql
✅ CORRECT:
status contract_status
priority task_priority

❌ WRONG:
status_type contract_status  -- unnecessary "_type"
statuses contract_status     -- plural
contractStatus contract_status -- camelCase
```

### 5.7 Indexes

**Rule: Use `{table}_{columns}_idx` pattern**

```sql
✅ CORRECT:
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX contracts_user_id_idx ON users(user_id);
CREATE INDEX contracts_user_status_idx ON contracts(user_id, status);

❌ WRONG:
CREATE INDEX idx_users_email ON users(email);  -- prefix instead of suffix
CREATE INDEX users_email ON users(email);      -- missing "_idx"
CREATE INDEX email_index ON users(email);      -- table name missing
```

**Drizzle Syntax:**
```typescript
export const users = pgTable('users', {
  email: varchar('email'),
}, (table) => [
  index('users_email_idx').on(table.email),
])
```

---

## 6. Event System Naming

### 6.1 Event Type Names

**Rule: Use `{domain}.{entity}.{past_tense_action}` with snake_case**

```typescript
✅ CORRECT:
'lifecycle.contract.price_increase_reported'
'lifecycle.task.status_updated'
'lifecycle.contract.activated'
'lifecycle.user.created'
'case.optimization.started'

❌ WRONG:
'lifecycle.contract.PriceIncreaseReported'  -- PascalCase
'lifecycle.contract.reportPriceIncrease'    -- present tense, camelCase
'contract.price_increase_reported'          -- missing domain
'lifecycle-contract-price_increase_reported' -- inconsistent separators
```

**Structure:**
1. **Domain**: `lifecycle`, `case`, `provider`, `offer`
2. **Entity**: `contract`, `user`, `task`, `offer`
3. **Action**: Past tense verb + object (if needed)

**Action Naming:**
- Use past tense: `created`, `updated`, `reported`, `activated` (NOT `create`, `update`)
- Be specific: `price_increase_reported` > `updated`
- Avoid generic names: `changed` (too vague)

### 6.2 Event Interface Properties

🚨 **CRITICAL CHANGE FROM CURRENT SPEC:**

**Rule: Use camelCase for ALL TypeScript interface properties**

```typescript
✅ CORRECT (NEW STANDARD):
export interface LifecycleEvent {
  eventId: string          // camelCase
  eventType: string        // camelCase
  eventSource: string      // camelCase
  eventVersion: string     // camelCase
  timestamp: string        // camelCase
  traceId: string          // camelCase
  entity: {
    id: string
    type: string
  }
  payload: {
    transition?: {
      fromStatus?: string   // camelCase
      toStatus?: string     // camelCase
    }
    context: Record<string, unknown>
  }
}

❌ WRONG (CURRENT SPEC - NEEDS MIGRATION):
export interface LifecycleEvent {
  event_id: string         // snake_case ← WRONG
  event_type: string       // snake_case ← WRONG
  trace_id: string         // snake_case ← WRONG
  payload: {
    transition?: {
      from_status?: string  // snake_case ← WRONG
    }
  }
}
```

**Rationale:**
- TypeScript interfaces should use TypeScript conventions (camelCase)
- Mixing snake_case in TS code violates [Language Layer Separation](#3-language-layer-separation)
- JSON serialization layer can handle conversion if external APIs require snake_case
- See [Section 8.2](#82-lifecycle-event-interface-snake_case-to-camelcase) for migration

### 6.3 Event Payload Structure

**Rule: Payload contents should match entity property naming (camelCase)**

```typescript
✅ CORRECT:
payload: {
  context: {
    userId: 'user-123',
    providerId: 'provider-abc',
    oldPrice: 10000,
    newPrice: 12000,
    priceDelta: 2000,
  }
}

❌ WRONG:
payload: {
  context: {
    user_id: 'user-123',  // snake_case
    old_price: 10000,     // snake_case
  }
}
```

---

## 7. Special Patterns

### 7.1 Prefix Conventions

| Prefix    | Meaning                | Example               | Usage                       |
| --------- | ---------------------- | --------------------- | --------------------------- |
| `raw`     | Unvalidated input      | `rawParams`           | Before Zod validation       |
| `is`      | Boolean value          | `isActive`, `isValid` | Boolean variables/functions |
| `has`     | Possession check       | `hasPermission`       | Boolean check for existence |
| `get`     | Synchronous retrieval  | `getContract`         | Fetch from memory/cache     |
| `fetch`   | Asynchronous retrieval | `fetchContract`       | Fetch from database/API     |
| `create`  | Factory function       | `createDbConnection`  | Constructor-like function   |
| `compute` | Derived calculation    | `computeEligibility`  | Pure calculation            |
| `handle`  | Error/event handler    | `handleZodError`      | Handles specific case       |
| `with`    | Higher-order wrapper   | `withErrorHandling`   | Wrapper function            |

### 7.2 Suffix Conventions

| Suffix   | Meaning               | Example                     | Usage                     |
| -------- | --------------------- | --------------------------- | ------------------------- |
| `Schema` | Zod validation schema | `contractSelectSchema`      | Validation definition     |
| `Params` | Input parameter type  | `ReportPriceIncreaseParams` | Function input type       |
| `Insert` | Drizzle insert type   | `ContractInsert`            | Database insert operation |
| `Update` | Drizzle update type   | `ContractUpdate`            | Database update operation |
| `Error`  | Error class           | `NotFoundError`             | Error type                |
| `At`     | Timestamp             | `createdAt`, `updatedAt`    | Time of action            |
| `Id`     | Identifier            | `userId`, `contractId`      | Foreign key / reference   |
| `Idx`    | Index                 | `users_email_idx`           | Database index            |

**Note:** Never use suffixes like `Refined`, `Enhanced`, `Extended`, `V2`, etc. for schemas. See Section 4.7 for schema naming strategy.

### 7.3 Singular vs Plural

**Rule: Use plurals for collections, singular for types/schemas**

```typescript
✅ CORRECT:
const contracts: Contract[]           // plural variable, singular type
const users = pgTable('users', ...)   // plural table name
const contract: Contract              // singular variable, singular type
type Contract = { ... }               // singular type name

❌ WRONG:
const contract: Contracts[]           // plural type name
const user = pgTable('users', ...)    // singular constant for plural table
```

**Table Naming:**
- Always plural: `users`, `contracts`, `tasks`
- Even for junction tables: `user_contracts` (not `user_contract`)

**Type Naming:**
- Always singular: `User`, `Contract`, `Task`
- Exception: If type represents collection: `UserList` or `Users` (rare)

---

## 8. Critical Issues & Migration

### 8.1 Enum Values: PascalCase to snake_case

**Current State (INCORRECT):**
```typescript
export const contractStatus = pgEnum('contract_status', [
  'PendingVerification',
  'Active',
  'SwitchPending',
])
```

**Migration Target (CORRECT):**
```typescript
export const contractStatus = pgEnum('contract_status', [
  'pending_verification',
  'active',
  'switch_pending',
])
```

**Migration Steps:**

**Step 1: Create new enum with correct values**
```typescript
// /lifecycle/lib/schema.ts
export const contractStatusNew = pgEnum('contract_status_new', [
  'pending_verification',
  'active',
  'switch_pending',
  'price_increase_reported',
  'cancelled',
  'expired',
  'archived'
])
```

**Step 2: Add new column to contracts table**
```typescript
export const contracts = pgTable('contracts', {
  // ... existing columns
  status: contractStatus('status').notNull(),  // OLD
  statusNew: contractStatusNew('status_new'),  // NEW
})
```

**Step 3: Backfill data with migration**
```sql
-- migration.sql
UPDATE contracts SET status_new = LOWER(
  REGEXP_REPLACE(
    REGEXP_REPLACE(status::TEXT, '([A-Z])', '_\1', 'g'),
    '^_', ''
  )
);
-- PendingVerification → pending_verification
-- Active → active
```

**Step 4: Update application code**
```typescript
// Before:
contract.status === 'Active'

// After:
contract.status === 'active'
```

**Step 5: Drop old column, rename new column**
```typescript
export const contracts = pgTable('contracts', {
  status: contractStatusNew('status').notNull(),
})
```

**Impact Analysis:**
- **Files affected**: All business intent handlers (60+ files)
- **Database queries**: All status comparisons
- **Time estimate**: 4-6 hours with find-replace + testing

### 8.2 LifecycleEvent Interface: snake_case to camelCase

**Current State (INCORRECT):**
```typescript
export interface LifecycleEvent {
  event_id: string
  event_type: string
  trace_id: string
  // ...
}
```

**Migration Target (CORRECT):**
```typescript
export interface LifecycleEvent {
  eventId: string
  eventType: string
  traceId: string
  // ...
}
```

**Migration Steps:**

**Step 1: Create new interface**
```typescript
// /lifecycle/lib/events.ts
export interface LifecycleEventNew {
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
```

**Step 2: Update event factory functions**
```typescript
// Before:
export function createPriceIncreaseEvent(...): LifecycleEvent {
  return {
    event_id: crypto.randomUUID(),
    event_type: 'lifecycle.contract.price_increase_reported',
    trace_id: params.traceId,
    // ...
  }
}

// After:
export function createPriceIncreaseEvent(...): LifecycleEventNew {
  return {
    eventId: crypto.randomUUID(),
    eventType: 'lifecycle.contract.price_increase_reported',
    traceId: params.traceId,
    // ...
  }
}
```

**Step 3: Update outbox table mapping (if needed)**
```typescript
// If external webhooks require snake_case, add serialization layer:
function serializeForWebhook(event: LifecycleEventNew): Record<string, unknown> {
  return {
    event_id: event.eventId,
    event_type: event.eventType,
    trace_id: event.traceId,
    // ... map all properties
  }
}
```

**Step 4: Replace all usages**
```bash
# Find all references
rg "LifecycleEvent" --type ts

# Update imports
# Update type annotations
# Update property access
```

**Impact Analysis:**
- **Files affected**: 5-10 files (event utilities, command handlers, webhook handlers)
- **Breaking change**: Yes (interface change)
- **Time estimate**: 2-3 hours

---

## 9. Decision Trees

### 9.1 Choosing a Name for a New Function

```
START
  ├─ Does it return a boolean?
  │    ├─ YES → Use "is" or "has" prefix (e.g., isActive, hasPermission)
  │    └─ NO → Continue
  │
  ├─ Does it create/instantiate something?
  │    ├─ YES → Use "create" prefix (e.g., createDbConnection, createEvent)
  │    └─ NO → Continue
  │
  ├─ Does it fetch data asynchronously?
  │    ├─ YES → Use "fetch" or "get" (e.g., fetchContract, getUser)
  │    └─ NO → Continue
  │
  ├─ Does it compute a derived value?
  │    ├─ YES → Use "compute" or "calculate" (e.g., computeTotal)
  │    └─ NO → Continue
  │
  ├─ Does it handle errors or events?
  │    ├─ YES → Use "handle" prefix (e.g., handleError, handleEvent)
  │    └─ NO → Continue
  │
  └─ Generic action
       └─ Use verb-first naming (e.g., validateInput, updateStatus)
```

### 9.2 Choosing a Name for a New Type

```
START
  ├─ Is it an input parameter object?
  │    ├─ YES → Use "{VerbNoun}Params" (e.g., ReportPriceIncreaseParams)
  │    └─ NO → Continue
  │
  ├─ Is it for database operations?
  │    ├─ INSERT → Use "{Entity}Insert" (e.g., ContractInsert)
  │    ├─ UPDATE → Use "{Entity}Update" (e.g., ContractUpdate)
  │    ├─ SELECT → Use "{Entity}" or "{Entity}Select" (e.g., Contract)
  │    └─ NO → Continue
  │
  ├─ Is it an error class?
  │    ├─ YES → Use "{Description}Error" (e.g., NotFoundError)
  │    └─ NO → Continue
  │
  └─ Generic type
       └─ Use descriptive PascalCase noun (e.g., StateSnapshot, LifecycleEvent)
```

### 9.3 Choosing Between Abbreviation and Full Name

```
START
  ├─ Is the term universally understood in software?
  │    ├─ YES (e.g., db, id, url, api) → Use abbreviation
  │    └─ NO → Continue
  │
  ├─ Is the term used 20+ times per file?
  │    ├─ YES → Consider abbreviation (e.g., tx for transaction)
  │    └─ NO → Continue
  │
  ├─ Would abbreviation cause ambiguity?
  │    ├─ YES → Use full name (e.g., "resource" not "res")
  │    └─ NO → Continue
  │
  └─ Default to full name for clarity
```

### 9.4 Singular vs Plural for Table Names

```
START
  └─ Is this a database table?
       ├─ YES → ALWAYS use plural (users, contracts, tasks)
       │
       └─ NO → Is this a TypeScript type?
              ├─ YES → ALWAYS use singular (User, Contract, Task)
              └─ NO → Does variable hold multiple items?
                     ├─ YES → Use plural (contracts, users)
                     └─ NO → Use singular (contract, user)
```

---

## 10. Anti-Patterns

### 10.1 Mixing Naming Conventions

❌ **NEVER mix camelCase and snake_case in the same layer:**

```typescript
// TypeScript layer
interface User {
  userId: string      // camelCase
  first_name: string  // snake_case ← WRONG
}
```

✅ **CORRECT:**
```typescript
interface User {
  userId: string
  firstName: string
}
```

### 10.2 Database Conventions in TypeScript

❌ **NEVER use snake_case in TypeScript code:**

```typescript
const user_id = params.user_id  // ← WRONG
const created_at = new Date()   // ← WRONG
```

✅ **CORRECT:**
```typescript
const userId = params.userId
const createdAt = new Date()
```

### 10.3 TypeScript Conventions in SQL

❌ **NEVER use camelCase in SQL:**

```sql
CREATE TABLE users (
  userId UUID,        -- ← WRONG
  createdAt TIMESTAMP -- ← WRONG
);
```

✅ **CORRECT:**
```sql
CREATE TABLE users (
  user_id UUID,
  created_at TIMESTAMP
);
```

### 10.4 Unnecessary Abbreviations

❌ **AVOID abbreviating common domain terms:**

```typescript
const usr = getUser()      // ← WRONG
const ctr = getContract()  // ← WRONG
const prm = params         // ← WRONG
```

✅ **CORRECT:**
```typescript
const user = getUser()
const contract = getContract()
const params = params
```

### 10.5 Ambiguous Prefixes

❌ **AVOID unclear prefixes:**

```typescript
function doValidation()    // "do" is vague
function processData()     // "process" is vague
function manageUser()      // "manage" is vague
```

✅ **CORRECT:**
```typescript
function validateInput()
function transformData()
function updateUser()
```

### 10.6 Hungarian Notation

❌ **AVOID type prefixes (Hungarian notation):**

```typescript
const strUserName: string = 'John'  // ← WRONG
const arrContracts: Contract[] = [] // ← WRONG
const objConfig: Config = {}        // ← WRONG
```

✅ **CORRECT (TypeScript has type annotations):**
```typescript
const userName: string = 'John'
const contracts: Contract[] = []
const config: Config = {}
```

### 10.7 Redundant Naming

❌ **AVOID redundant context in names:**

```typescript
// In file: /lifecycle/contract/report-price-increase.ts
function reportPriceIncreaseContract() { }  // "Contract" is redundant

// In class: ContractRepository
class ContractRepository {
  getContract() { }  // Should be just "get()"
}
```

✅ **CORRECT:**
```typescript
// In file: /lifecycle/contract/report-price-increase.ts
function reportPriceIncrease() { }

// In class: ContractRepository
class ContractRepository {
  get() { }
  create() { }
  update() { }
}
```

---

## 11. Examples Reference

### 11.1 Complete File Example

```typescript
// /lifecycle/contract/report-price-increase.ts

import { eq } from 'drizzle-orm'
import { createDbConnection } from '../lib/db'
import type { PostgresqlResource } from '../lib/db'
import { contracts, priceIncreaseEvents } from '../lib/schema'
import { reportPriceIncreaseSchema } from '../lib/validation'
import type { ReportPriceIncreaseParams } from '../lib/validation'
import { emitEvent, createPriceIncreaseEvent } from '../lib/events'
import { NotFoundError, InvalidCommandError, handleZodError } from '../lib/errors'

const MAX_PRICE_INCREASE_PERCENT = 50

export async function main(
  dbResource: PostgresqlResource,
  rawParams: unknown
) {
  // Validate
  let params: ReportPriceIncreaseParams
  try {
    params = reportPriceIncreaseSchema.parse(rawParams)
  } catch (error) {
    handleZodError(error)
  }

  // Connect
  const db = createDbConnection(dbResource)

  // Execute transaction
  const result = await db.transaction(async (tx) => {
    // Lock and fetch
    const [contract] = await tx
      .select()
      .from(contracts)
      .where(eq(contracts.id, params.contractId))
      .for('update')

    if (!contract) {
      throw new NotFoundError('Contract', params.contractId)
    }

    // Validate business rules
    if (contract.status === 'cancelled') {
      throw new InvalidCommandError(
        'Cannot report price increase for cancelled contract',
        contract.status,
        'report-price-increase'
      )
    }

    const increasePercent = ((params.newPrice - contract.price) / contract.price) * 100
    if (increasePercent > MAX_PRICE_INCREASE_PERCENT) {
      throw new InvalidCommandError(
        `Price increase of ${increasePercent}% exceeds maximum allowed ${MAX_PRICE_INCREASE_PERCENT}%`
      )
    }

    // Update state
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

    // Emit event
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

  return {
    success: true,
    contract: result,
  }
}
```

**Naming Conventions Demonstrated:**

- ✅ File: `report-price-increase.ts` (kebab-case)
- ✅ Function: `main`, `handleZodError` (camelCase, verb-first)
- ✅ Variables: `dbResource`, `rawParams`, `params` (camelCase, `raw` prefix)
- ✅ Types: `PostgresqlResource`, `ReportPriceIncreaseParams` (PascalCase, `Params` suffix)
- ✅ Constants: `MAX_PRICE_INCREASE_PERCENT` (SCREAMING_SNAKE_CASE)
- ✅ Database: `contracts.id`, `contracts.userId` (camelCase in TS)
- ✅ Abbreviations: `db`, `tx` (universal terms only)

### 11.2 Schema Definition Example

```typescript
// /lifecycle/lib/schema.ts

import { pgTable, uuid, varchar, integer, timestamp, pgEnum } from 'drizzle-orm/pg-core'
import { crudPolicy, authenticatedRole, authUid } from 'drizzle-orm/neon'

// Enums (snake_case values)
export const contractStatus = pgEnum('contract_status', [
  'pending_verification',
  'active',
  'switch_pending',
  'price_increase_reported',
  'cancelled',
  'expired',
  'archived'
])

// Tables (plural names, snake_case SQL columns)
export const contracts = pgTable('contracts', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').notNull().references(() => users.id),
  status: contractStatus('status').notNull(),
  providerId: uuid('provider_id').notNull(),

  serviceType: varchar('service_type', { length: 50 }).notNull(),
  startDate: timestamp('start_date').notNull(),
  endDate: timestamp('end_date'),
  price: integer('price').notNull(),

  pendingPriceChange: integer('pending_price_change'),
  pendingPriceEffectiveDate: timestamp('pending_price_effective_date'),

  createdAt: timestamp('created_at').notNull().defaultNow(),
  updatedAt: timestamp('updated_at').notNull().defaultNow(),
}, (table) => [
  index('contracts_user_id_idx').on(table.userId),
  index('contracts_status_idx').on(table.status),

  crudPolicy({
    role: authenticatedRole,
    read: authUid(table.userId),
    modify: authUid(table.userId),
  })
])
```

**Naming Conventions Demonstrated:**

- ✅ Table constant: `contracts` (camelCase, plural)
- ✅ SQL table name: `'contracts'` (lowercase, plural)
- ✅ TS properties: `userId`, `createdAt` (camelCase)
- ✅ SQL columns: `'user_id'`, `'created_at'` (snake_case)
- ✅ Enum: `contractStatus` (camelCase constant)
- ✅ Enum values: `'pending_verification'` (snake_case)
- ✅ Indexes: `'contracts_user_id_idx'` (table_columns_idx pattern)

### 11.3 Validation Schema Example

```typescript
// /lifecycle/lib/validation.ts

import { z } from 'zod'
import { createSelectSchema, createInsertSchema, createUpdateSchema } from 'drizzle-zod'
import { contracts } from './schema'

// Entity CRUD schemas (with all validation built in)
export const contractSelectSchema = createSelectSchema(contracts)
export const contractInsertSchema = createInsertSchema(contracts, {
  serviceType: (schema) => schema.serviceType.max(50),
  price: (schema) => schema.price.positive('Price must be positive'),
})
export const contractUpdateSchema = createUpdateSchema(contracts, {
  serviceType: (schema) => schema.serviceType.max(50),
  price: (schema) => schema.price.positive('Price must be positive'),
})

// Business intent schema (purpose-specific)
export const reportPriceIncreaseSchema = z.object({
  contractId: z.string().uuid('Invalid contract ID format'),
  newPrice: z.number().int().positive('Price must be positive'),
  effectiveDate: z.coerce.date().refine(
    (date) => date > new Date(),
    { message: 'Effective date must be in the future' }
  ),
  notificationMethod: z.enum(['email', 'letter', 'provider_portal']),
  reportedBy: z.string().uuid('Invalid reporter ID'),
})

// Type inference
export type Contract = z.infer<typeof contractSelectSchema>
export type ContractInsert = z.infer<typeof contractInsertSchema>
export type ContractUpdate = z.infer<typeof contractUpdateSchema>
export type ReportPriceIncreaseParams = z.infer<typeof reportPriceIncreaseSchema>
```

**Naming Conventions Demonstrated:**

- ✅ Entity CRUD: `{entity}{Operation}Schema` with refinements built in
- ✅ Business intent: `{verbNoun}Schema` pattern
- ✅ Types: PascalCase with `Params` suffix for input types
- ❌ **Never use**: `{entity}{Operation}Refined` pattern (creates parallel versions)

---

## Appendix A: Quick Reference Checklist

### Before Committing Code

- [ ] All TypeScript variables/parameters use camelCase
- [ ] All TypeScript types/interfaces use PascalCase
- [ ] All TypeScript functions use camelCase, verb-first
- [ ] All file names use kebab-case
- [ ] All SQL table names are lowercase, plural
- [ ] All SQL column names use snake_case
- [ ] All enum values use snake_case (not PascalCase)
- [ ] Event types use `domain.entity.past_tense_action` format
- [ ] Event interface properties use camelCase (not snake_case)
- [ ] Constants use SCREAMING_SNAKE_CASE
- [ ] Abbreviations are only used for universal terms (db, tx, id)
- [ ] No mixing of camelCase and snake_case within same layer

### AI Code Review Prompt

```
Review this code for naming convention compliance:

1. Are all TypeScript identifiers using camelCase (variables) or PascalCase (types)?
2. Are all SQL identifiers using snake_case?
3. Do enum values use snake_case (not PascalCase)?
4. Do event interface properties use camelCase (not snake_case)?
5. Are file names using kebab-case?
6. Are constants using SCREAMING_SNAKE_CASE?
7. Are abbreviations limited to universal terms (db, tx, id, sql)?
8. Are function names verb-first?
9. Are boolean functions/variables prefixed with "is" or "has"?
10. Are table names plural and lowercase?
```

---

## Appendix B: Conversion Examples

### snake_case → camelCase

| snake_case                | camelCase              |
| ------------------------- | ---------------------- |
| `user_id`                 | `userId`               |
| `created_at`              | `createdAt`            |
| `pending_price_change`    | `pendingPriceChange`   |
| `originating_flow_run_id` | `originatingFlowRunId` |
| `is_active`               | `isActive`             |

### PascalCase → snake_case (Enum Values)

| PascalCase              | snake_case                |
| ----------------------- | ------------------------- |
| `PendingVerification`   | `pending_verification`    |
| `Active`                | `active`                  |
| `InProgress`            | `in_progress`             |
| `PriceIncreaseReported` | `price_increase_reported` |

### camelCase → kebab-case (File Names)

| camelCase                | kebab-case                 |
| ------------------------ | -------------------------- |
| `reportPriceIncrease.ts` | `report-price-increase.ts` |
| `completeTask.ts`        | `complete-task.ts`         |
| `createUser.ts`          | `create-user.ts`           |

---

**End of Naming Convention Strategy**

This document is the authoritative source for all naming decisions in the SwitchUp platform. All code—human-written or AI-generated—must comply with these conventions. When in doubt, refer to the decision trees in Section 9.
