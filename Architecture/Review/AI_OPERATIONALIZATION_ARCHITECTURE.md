# AI Operationalization Architecture: Foundation Design

**Status**: Architecture Proposal
**Last Updated**: January 2025
**Context**: Greenfield platform design for AI-ready operational scalability
**Scope**: Foundation layers to enable step-by-step AI evolution

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Problem Statement](#problem-statement)
3. [Foundation Architecture](#foundation-architecture)
4. [How Existing Architecture Enables AI](#how-existing-architecture-enables-ai)
5. [Evolution Path](#evolution-path)
6. [Why This Foundation Wins](#why-this-foundation-wins)
7. [Appendices](#appendices)

---

## 1. Executive Summary

### The Challenge

SwitchUp's strategic vision includes AI-driven operational automation with the goal of achieving autonomous workflow execution. However, this vision currently lacks implementation specifications with gaps in AI operationalization.

The risk: Over-engineering NOW by building full multi-agent coordination, reinforcement learning pipelines, and A/B testing frameworks before proving value.

### The Approach

**Build the RIGHT FOUNDATION for greenfield design**, then evolve based on learnings.

This document proposes **4 foundation layers** to add during greenfield platform development:

1. **Schema Metadata Architecture** → Enable AI discovery and understanding
2. **Execution Audit Trail** → Track human vs AI command execution
3. **Validation-First Architecture** → Safety net for AI-generated commands
4. **Discoverable API Surface** → Enable AI agents to enumerate capabilities

### Key Insight

**Business Intents are already the right abstraction for AI**—they are:
- Atomic and self-contained
- Discoverable via registry
- Self-documenting with Zod schemas
- Validated at runtime
- Side-effect explicit

We don't need a separate "AI command layer"—we need to expose what we're already building with metadata that enables AI consumption.

### Foundation vs Future

| Component | Now (Greenfield) | Year 1-2 (Iteration) | Year 3+ (Vision) |
|-----------|------------------|----------------------|------------------|
| **Business Intents** | Full catalog with Zod schemas + metadata | Single-agent patterns proven | Multi-agent coordination |
| **Execution Tracking** | Audit trail (human/AI) | Success rate metrics | Learning loops |
| **API Discovery** | `/ai-catalog` endpoint | Agent playbook library | Autonomous planning |
| **Validation** | Runtime Zod validation | Expanded error recovery | Self-healing workflows |

---

## 2. Problem Statement

### The AI Operationalization Vision

The strategic vision for AI operationalization:

> "AI agents autonomously execute workflows end-to-end: classify user requests, fetch provider data, calculate savings, execute switches, respond to users—with minimal human intervention."

This requires:
1. **Discovery**: AI agents must enumerate available capabilities
2. **Understanding**: AI agents must understand what each capability does
3. **Execution**: AI agents must invoke capabilities with correct parameters
4. **Safety**: AI agents must not corrupt business state with hallucinated data
5. **Traceability**: Human operators must audit what AI agents did and why

### Current Architecture Gaps

**Gap 1: Schema Metadata for AI Context**
- Business Intent schemas exist (Zod validation)
- Missing: Metadata describing WHAT each field means, WHY it matters, WHEN to use it

**Gap 2: Execution Attribution**
- Commands are executed, state changes are tracked
- Missing: WHO executed (human operator vs AI agent)

**Gap 3: AI-Discoverable Catalog**
- Business Intents exist in `/lifecycle/lib/business-intents/`
- Missing: Programmatic enumeration for AI consumption

**Gap 4: Validation Recovery**
- Zod validation rejects invalid commands
- Missing: Structured error messages AI agents can parse and recover from

### What This Document DOES NOT Propose

Per user guidance: "don't go too far in terms of things that are not feasible as of now"

**Out of Scope for Foundation**:
- Multi-agent coordination frameworks
- Reinforcement learning pipelines
- A/B testing infrastructure for AI strategies
- Autonomous planning and goal decomposition
- Learning loops and feedback mechanisms

**Why**: These are Year 2-3 capabilities. Build foundation NOW, prove single-agent patterns in Year 1, THEN invest in advanced coordination based on learnings.

---

## 3. Foundation Architecture

### Layer 1: Schema Metadata Architecture

**Purpose**: Enable AI agents to understand WHAT fields mean and WHY they matter.

**Problem**: Current Zod schemas validate structure but don't explain semantics.

```typescript
// Current (validation only)
const confirmActivationSchema = z.object({
  contract_id: z.string().uuid(),
  activation_date: z.string().date(),
})

// Enhanced (validation + AI context)
const confirmActivationSchema = z.object({
  contract_id: z.string().uuid().meta({
    id: "contract_id",
    title: "Contract Identifier",
    description: "UUID of the contract to mark as activated",
    validationPriority: "critical",
    aiGuidance: "Use the contract_id from the provider confirmation email or portal scraping result",
    examples: ["550e8400-e29b-41d4-a716-446655440000"]
  }),

  activation_date: z.string().date().meta({
    id: "activation_date",
    title: "Activation Date",
    description: "Date when the provider confirmed activation (YYYY-MM-DD)",
    validationPriority: "critical",
    aiGuidance: "Extract from provider email subject line or portal 'Contract Start Date' field",
    examples: ["2025-02-01"],
    constraints: "Must be <= 30 days in the future, >= contract creation date"
  }),
})
```

**Schema Metadata Fields**:

```typescript
type SchemaMetadata = {
  id: string                    // Unique identifier
  title: string                 // Human-readable name
  description: string           // What this field represents
  validationPriority: 'critical' | 'high' | 'medium' | 'low'
  aiGuidance?: string           // How AI should populate this field
  examples?: string[]           // Sample valid values
  constraints?: string          // Business rules explanation
  relatedFields?: string[]      // Fields that affect validation
  context?: string              // Additional background
}
```

**Implementation in Service-Type Registry**:

Already designed in SERVICE_TYPE_ABSTRACTION_ARCHITECTURE.md:

```typescript
// ServiceTypeRegistry stores schemas with metadata
export const serviceTypeRegistry = pgTable('service_type_registry', {
  attributes_schema: jsonb('attributes_schema').$type<{
    schema: string,  // Stringified Zod schema WITH .meta() calls
    version: string,
  }>().notNull(),
})

// Example: German Energy schema with AI metadata
const energyElectricityAttributesSchema = z.interface({
  primary_identifier: z.string().regex(/^\d{11}$/).meta({
    id: "zaehlepunktnummer",
    title: "Zählpunktnummer",
    description: "11-digit electricity meter identifier (German energy market)",
    validationPriority: "critical",
    aiGuidance: "Extract from bill under 'Zählpunktnummer' or 'Zählernummer' heading",
    examples: ["12345678901"],
    context: "Required for all German electricity contracts since deregulation"
  }),
})
```

**Why This Enables AI**:
- AI agents can read `.meta()` to understand field purpose
- `aiGuidance` provides extraction instructions
- `examples` help with format validation
- `constraints` prevent business logic violations

**Greenfield Work**: Add `.meta()` calls to ALL Zod schemas (Business Intents + Service-Type schemas)

---

### Layer 2: Execution Audit Trail

**Purpose**: Track WHO executed each Business Intent (human operator vs AI agent).

**Problem**: Current state transitions don't attribute execution source.

**Minimal Schema Addition**:

```typescript
// Enhanced Business Intent execution signature
type BusinessIntentExecution = {
  intent_id: string              // 'confirm_activation', 'report_price_increase'
  input: Record<string, unknown> // Validated input

  // NEW: Execution attribution
  executed_by: 'human' | 'ai_agent'
  executor_id: string            // user_id or agent_id
  trace_id: string               // Distributed tracing ID

  // NEW: AI-specific metadata (optional)
  ai_metadata?: {
    model: string                // 'claude-3.5-sonnet'
    confidence: number           // 0.0 - 1.0
    reasoning?: string           // AI's explanation
    source_data_refs?: string[]  // What data AI used (email_id, portal_scrape_id)
  }
}
```

**Implementation Pattern**:

```typescript
// /lifecycle/lib/business-intents/confirm-activation.ts

export async function confirmActivation(
  input: z.infer<typeof confirmActivationSchema>,
  context: ExecutionContext  // NEW parameter
): Promise<Result<ContractState>> {

  // Log execution with attribution
  await auditLog.create({
    intent_id: 'confirm_activation',
    contract_id: input.contract_id,
    executed_by: context.executed_by,
    executor_id: context.executor_id,
    trace_id: context.trace_id,
    ai_metadata: context.ai_metadata,
    timestamp: new Date(),
  })

  // Execute state transition...
  const result = await db.transaction(async (tx) => {
    // Existing logic
  })

  return result
}
```

**Database Schema**:

```typescript
export const businessIntentAuditLog = pgTable('business_intent_audit_log', {
  id: uuid('id').primaryKey().defaultRandom(),

  intent_id: text('intent_id').notNull(),
  contract_id: uuid('contract_id').references(() => contracts.id),

  executed_by: text('executed_by').notNull().$type<'human' | 'ai_agent'>(),
  executor_id: text('executor_id').notNull(),
  trace_id: text('trace_id').notNull(),

  input: jsonb('input').notNull(),  // What was passed
  output: jsonb('output'),          // What was returned

  ai_metadata: jsonb('ai_metadata').$type<{
    model?: string
    confidence?: number
    reasoning?: string
    source_data_refs?: string[]
  }>(),

  created_at: timestamp('created_at').notNull().defaultNow(),

  // Performance index
}, (table) => [
  index('audit_contract_idx').on(table.contract_id),
  index('audit_executor_idx').on(table.executor_id),
  index('audit_trace_idx').on(table.trace_id),
])
```

**Why This Enables AI**:
- Human operators can audit AI decisions
- Debugging: "Why did AI mark this contract as activated?"
- Metrics: AI success rate vs human success rate
- Compliance: Prove who made which state changes

**Greenfield Work**: Add `ExecutionContext` parameter to all Business Intent handlers

---

### Layer 3: Validation-First Architecture

**Purpose**: Runtime Zod validation prevents AI hallucinations from corrupting business state.

**Problem**: AI models hallucinate (generate plausible but incorrect data).

**Solution**: ALREADY DESIGNED in Service-Type Registry Pattern.

```typescript
// Every Business Intent has Zod schema
export async function confirmActivation(
  input: unknown  // Untrusted input from AI
): Promise<Result<ContractState>> {

  // FIRST: Validate with Zod
  const validated = confirmActivationSchema.safeParse(input)

  if (!validated.success) {
    return {
      success: false,
      error: {
        code: 'VALIDATION_ERROR',
        message: 'Invalid activation data',
        details: validated.error.format(),  // Structured errors for AI
      }
    }
  }

  // ONLY THEN: Execute state transition
  return executeActivation(validated.data)
}
```

**Validation Error Format for AI Recovery**:

```typescript
// Zod validation error (structured)
{
  "code": "VALIDATION_ERROR",
  "message": "Invalid activation data",
  "details": {
    "activation_date": {
      "_errors": [
        "Date must be in format YYYY-MM-DD",
        "Date cannot be more than 30 days in the future"
      ]
    }
  }
}

// AI can parse this and retry with corrected data
```

**Service-Type Attribute Validation**:

```typescript
// Runtime validation against ServiceTypeRegistry schema
export async function createContract(input: {
  service_type_id: string
  service_attributes: unknown  // Untrusted AI-generated data
}) {

  // 1. Fetch schema from registry
  const serviceType = await db.query.serviceTypeRegistry.findFirst({
    where: eq(serviceTypeRegistry.service_type_id, input.service_type_id)
  })

  // 2. Parse stringified Zod schema
  const attributesSchema = parseZodSchema(serviceType.attributes_schema.schema)

  // 3. Validate at runtime
  const validated = attributesSchema.safeParse(input.service_attributes)

  if (!validated.success) {
    return {
      success: false,
      error: {
        code: 'SERVICE_ATTRIBUTES_INVALID',
        service_type: input.service_type_id,
        details: validated.error.format()
      }
    }
  }

  // 4. Store validated data
  await db.insert(contracts).values({
    service_type_id: input.service_type_id,
    service_attributes: validated.data,
  })
}
```

**Why This Enables AI**:
- Safety net: Invalid AI commands are rejected BEFORE state corruption
- Recovery: Structured errors enable AI to fix and retry
- Trust: Runtime validation means AI output is ALWAYS validated
- Extensibility: New service types get validation automatically

**Greenfield Work**: ALREADY DESIGNED—no additional work needed. Just follow existing pattern.

---

### Layer 4: Discoverable API Surface

**Purpose**: Enable AI agents to enumerate available capabilities programmatically.

**Problem**: Business Intents exist in `/lifecycle/lib/business-intents/` but no programmatic discovery.

**Solution**: `/lifecycle/lib/ai-catalog.ts` endpoint that enumerates all Business Intents.

```typescript
// /lifecycle/lib/ai-catalog.ts

export type BusinessIntentCatalogEntry = {
  intent_id: string              // 'confirm_activation'
  category: string               // 'lifecycle.activation'
  title: string                  // "Confirm Contract Activation"
  description: string            // "Mark contract as activated after provider confirmation"

  input_schema: {
    schema: string               // Stringified Zod schema with .meta()
    example: Record<string, unknown>
  }

  output_schema: {
    schema: string               // Result type
    example: Record<string, unknown>
  }

  side_effects: string[]         // ["contract.status → 'active'", "event.ContractActivated emitted"]
  preconditions: string[]        // ["contract.status === 'pending'"]
  related_intents: string[]      // ["cancel_activation", "report_activation_delay"]
}

export async function getAICatalog(): Promise<BusinessIntentCatalogEntry[]> {
  return [
    {
      intent_id: 'confirm_activation',
      category: 'lifecycle.activation',
      title: 'Confirm Contract Activation',
      description: 'Mark contract as activated after receiving provider confirmation (email, portal, or phone)',

      input_schema: {
        schema: confirmActivationSchema.toString(),  // Zod schema with .meta()
        example: {
          contract_id: '550e8400-e29b-41d4-a716-446655440000',
          activation_date: '2025-02-01'
        }
      },

      output_schema: {
        schema: 'Result<ContractState>',
        example: {
          success: true,
          data: {
            contract_id: '550e8400-e29b-41d4-a716-446655440000',
            status: 'active',
            activated_at: '2025-02-01T00:00:00Z'
          }
        }
      },

      side_effects: [
        "contract.status → 'active'",
        "contract.activated_at set to activation_date",
        "Event ContractActivated emitted to webhook consumers"
      ],

      preconditions: [
        "contract.status === 'pending'",
        "activation_date <= 30 days in future",
        "activation_date >= contract.created_at"
      ],

      related_intents: [
        'cancel_activation',
        'report_activation_delay',
        'update_activation_date'
      ]
    },

    // ... 50+ other Business Intents
  ]
}
```

**AI Consumption Pattern**:

```typescript
// AI agent queries catalog
const catalog = await fetch('/lifecycle/ai-catalog')
const intents = await catalog.json()

// AI finds relevant intent
const intent = intents.find(i =>
  i.category === 'lifecycle.activation' &&
  i.intent_id === 'confirm_activation'
)

// AI generates input based on schema metadata
const input = {
  contract_id: extractedFromEmail.contractId,
  activation_date: extractedFromEmail.activationDate
}

// AI invokes Business Intent
const result = await executeBusinessIntent('confirm_activation', input, {
  executed_by: 'ai_agent',
  executor_id: 'email-classifier-agent-v1',
  trace_id: currentTraceId,
  ai_metadata: {
    model: 'claude-3.5-sonnet',
    confidence: 0.92,
    reasoning: 'Email subject contains "Activation confirmed" and body has start date',
    source_data_refs: ['email:abc123']
  }
})
```

**Service-Type Catalog**:

```typescript
// /lifecycle/lib/ai-service-type-catalog.ts

export type ServiceTypeCatalogEntry = {
  service_type_id: string        // 'energy.electricity'
  market_id: string              // 'de'

  display_name: string           // "Electricity (Germany)"
  category: string               // "Energy"
  subcategory: string            // "Electricity"

  attributes_schema: {
    schema: string               // Stringified Zod schema with .meta()
    example: Record<string, unknown>
  }

  required_attributes: string[]  // ['primary_identifier', 'market_location_id']
  optional_attributes: string[]  // ['meter_type', 'tariff_zone']
}

export async function getServiceTypeCatalog(
  market_id?: string
): Promise<ServiceTypeCatalogEntry[]> {

  const serviceTypes = await db.query.serviceTypeRegistry.findMany({
    where: market_id
      ? eq(serviceTypeRegistry.market_id, market_id)
      : undefined
  })

  return serviceTypes.map(st => ({
    service_type_id: st.service_type_id,
    market_id: st.market_id,
    display_name: `${st.display_name_en} (${st.market_id.toUpperCase()})`,
    category: st.category,
    subcategory: st.subcategory,

    attributes_schema: {
      schema: st.attributes_schema.schema,  // Includes .meta()
      example: generateExampleFromSchema(st.attributes_schema.schema)
    },

    required_attributes: extractRequiredFields(st.attributes_schema.schema),
    optional_attributes: extractOptionalFields(st.attributes_schema.schema),
  }))
}
```

**Why This Enables AI**:
- Discovery: AI can enumerate ALL available capabilities
- Understanding: Catalog provides descriptions, examples, side effects
- Composition: AI can chain intents based on `related_intents`
- Self-documentation: Catalog is auto-generated from Zod schemas

**Greenfield Work**: Create `/lifecycle/lib/ai-catalog.ts` with registry of all Business Intents

---

## 4. How Existing Architecture Enables AI

### Service-Type Registry Pattern → Zero-Code Service Type Extensibility

**Current Architecture** (from SERVICE_TYPE_ABSTRACTION_ARCHITECTURE.md):
- Service types stored as DATA in `serviceTypeRegistry` table
- Each service type has Zod schema stored as JSONB
- Runtime validation against schema

**AI Enablement**:
```typescript
// AI agent can discover NEW service types without code changes
const serviceTypes = await getServiceTypeCatalog('de')  // German market

// AI finds telco.mobile schema
const telco = serviceTypes.find(st => st.service_type_id === 'telco.mobile')

// AI generates contract data based on schema
const contractData = {
  service_type_id: 'telco.mobile',
  service_attributes: {
    primary_identifier: '+49 30 12345678',  // Extracted from bill
    contract_start_date: '2025-01-15',
    monthly_fee_cents: 2999,  // €29.99
  }
}

// Validation happens at runtime (AI doesn't need to know schema at "compile time")
const result = await createContract(contractData)
```

**Key Insight**: AI agents consume schemas at RUNTIME, not compile-time. Service-Type Registry provides exactly this.

---

### Market Registry Pattern → Multi-Market AI Agents

**Current Architecture** (from GEOGRAPHIC_ABSTRACTION_ARCHITECTURE.md):
- Markets stored as DATA in `marketRegistry` table
- Service types scoped by (service_type_id, market_id)
- Market-specific regulatory config in JSONB

**AI Enablement**:
```typescript
// AI agent queries available markets
const markets = await db.query.marketRegistry.findMany()

// AI adapts behavior based on market config
const market = markets.find(m => m.market_id === 'de')

// AI uses market-specific notice period
const noticePeriodDays = market.regulatory_config.energy.notice_period_days_default

// AI generates cancellation date
const cancellationDate = addDays(new Date(), noticePeriodDays)

// AI invokes Business Intent with market-aware data
await submitCancellation({
  contract_id: contractId,
  cancellation_date: cancellationDate,
  market_id: 'de'
})
```

**Key Insight**: AI agents don't hardcode market rules—they query `marketRegistry` and adapt.

---

### Business Intent Catalog → Atomic, Self-Documenting Commands

**Current Architecture** (from Lifecycle Domain spec):
- Business Intents are atomic state-change commands
- Each has Zod schema for validation
- Side effects are explicit (state transitions + events)

**AI Enablement**:
```typescript
// AI queries catalog
const catalog = await getAICatalog()

// AI searches for intent matching task
const intent = catalog.find(i =>
  i.description.includes('price increase') &&
  i.category === 'lifecycle.price_change'
)

// AI reads preconditions
console.log(intent.preconditions)
// ["contract.status === 'active'", "price_increase > 0"]

// AI validates before execution
if (contract.status !== 'active') {
  throw new Error('Cannot report price increase for inactive contract')
}

// AI invokes
await executeBusinessIntent('report_price_increase', {
  contract_id: contractId,
  new_price_cents: 3499,
  effective_date: '2025-03-01',
  source: 'provider_email'
})
```

**Key Insight**: Business Intents are ALREADY the right abstraction—atomic, discoverable, self-documenting, validated.

---

### Validation-First → Safety Net for AI Hallucinations

**Current Architecture** (from Service-Type Registry spec):
- Runtime Zod validation on ALL state changes
- Structured error responses
- Type-safe after validation

**AI Enablement**:
```typescript
// AI generates data from email extraction
const extractedData = {
  contract_id: 'abc123',  // WRONG: Not a UUID
  activation_date: 'February 1st, 2025'  // WRONG: Not YYYY-MM-DD
}

// Validation catches hallucinations
const result = await confirmActivation(extractedData)

if (!result.success) {
  console.log(result.error.details)
  // {
  //   contract_id: { _errors: ['Invalid UUID format'] },
  //   activation_date: { _errors: ['Date must be YYYY-MM-DD format'] }
  // }

  // AI can retry with corrected data
  const correctedData = await aiAgent.fixValidationErrors(
    extractedData,
    result.error.details
  )

  // Second attempt with correct format
  await confirmActivation(correctedData)
}
```

**Key Insight**: Runtime validation means AI mistakes are caught BEFORE state corruption.

---

## 5. Evolution Path

### Phase 1: Foundation (Greenfield - Now)

**Timeline**: Q1-Q2 2025 (part of Tier 1 refactoring)

**What to Build**:

1. **Schema Metadata**
   - Add `.meta()` to all Business Intent schemas
   - Add `.meta()` to all Service-Type attribute schemas
   - Include: title, description, aiGuidance, examples, constraints

2. **Execution Audit Trail**
   - Add `ExecutionContext` parameter to Business Intent handlers
   - Create `business_intent_audit_log` table
   - Log all executions with attribution

3. **AI Catalog Endpoint**
   - Create `/lifecycle/lib/ai-catalog.ts`
   - Registry of all Business Intents with schemas
   - Create `/lifecycle/lib/ai-service-type-catalog.ts`

4. **Validation Error Formatting**
   - Ensure Zod errors are structured (already true)
   - Document error recovery patterns for AI consumption

**Deliverables**:
- All Zod schemas enhanced with `.meta()`
- Audit log capturing human vs AI execution
- `/ai-catalog` endpoint returning 50+ Business Intents
- Documentation: "AI Agent Integration Guide"

**Success Metrics**:
- 100% of Business Intents have complete metadata
- Audit log captures 100% of state changes
- AI catalog returns valid JSON schema for all intents

---

### Phase 2: Single-Agent Patterns (Year 1)

**Timeline**: Q3-Q4 2025

**What to Prove**:

1. **Email Classification Agent**
   - Reads provider emails
   - Classifies intent (activation confirmation, price increase, usage report)
   - Invokes correct Business Intent
   - **Success metric**: 90%+ classification accuracy

2. **Contract Data Extraction Agent**
   - Extracts contract attributes from bills/emails
   - Generates `service_attributes` based on Service-Type schema
   - Submits for validation
   - **Success metric**: 80%+ extraction accuracy on German energy contracts

3. **Simple Response Agent**
   - Reads user question
   - Queries contract state from /lifecycle
   - Generates response
   - **Success metric**: 70%+ user satisfaction (vs template responses)

**Learnings to Validate**:
- Do schema metadata fields provide sufficient context?
- Are validation errors parseable and recoverable by AI?
- What additional metadata is needed?
- Which Business Intents are AI-friendly vs human-only?

**Iteration**:
- Expand `.meta()` based on AI agent failures
- Add new fields like `aiDifficulty: 'easy' | 'medium' | 'hard'`
- Create AI-specific Business Intent variants if needed

---

### Phase 3: Coordination (Year 2-3)

**Timeline**: 2026+

**What to Build** (based on Phase 1-2 learnings):

1. **Multi-Step Workflows**
   - AI chains multiple Business Intents
   - Example: Extract contract → Validate → Confirm activation → Notify user
   - Requires: Workflow state machine, rollback capabilities

2. **Optimization Agents**
   - AI queries /offer domain for alternatives
   - AI invokes /optimisation for savings calculation
   - AI generates recommendation
   - Requires: Multi-domain coordination patterns

3. **Learning Loops**
   - Track AI success rates per Business Intent
   - A/B test AI strategies
   - Requires: Experimentation framework, metrics pipeline

**Key Principle**: Don't build these NOW—validate that Phase 1 foundation enables them.

---

## 6. Why This Foundation Wins

### Minimal, Not Maximal

**Alternative Rejected**: Build full multi-agent framework NOW
- Cost: 18-23 weeks, 4-5 engineers (estimated for full implementation)
- Risk: Over-engineering before proving value

**Foundation Approach**: Add metadata and audit trail NOW
- Cost: 2-3 weeks (part of Tier 1 refactoring)
- Risk: Low—metadata is useful even without AI

**Why Better**: Build what's needed for greenfield platform, enable AI later.

---

### Leverage Existing Patterns

**No New Abstractions Required**:
- Business Intents already atomic and self-documenting
- Service-Type Registry already provides runtime schemas
- Market Registry already provides market-specific config
- Validation already happens at runtime

**What's Added**:
- Metadata (`.meta()` calls) → Makes existing schemas AI-consumable
- Audit trail → Tracks WHO executed (1 new parameter)
- Catalog endpoint → Enumerates existing intents

**Why Better**: Enhances what we're already building, rather than parallel system.

---

### Step-by-Step Evolution

**Phase 1** (Foundation): Metadata + audit trail
- Useful even without AI (better docs, human audit trail)
- Low cost, low risk

**Phase 2** (Single-Agent): Prove value with email classification
- Learn what works, what doesn't
- Iterate on metadata based on real AI agent failures

**Phase 3** (Coordination): Build advanced features based on learnings
- Only invest if Phase 2 proves ROI
- Informed by actual AI agent behavior

**Why Better**: Avoids big upfront investment in unproven technology.

---

### AI-Agnostic Foundation

**Schema Metadata** is useful for:
- AI agents (primary use case)
- Human developers (better documentation)
- Auto-generated API docs (OpenAPI specs)
- Visual form builders (UI generation)

**Audit Trail** is useful for:
- AI accountability (track AI decisions)
- Compliance (SOC 2, GDPR audit requirements)
- Debugging (trace why state changed)
- Analytics (human vs AI performance)

**Catalog Endpoint** is useful for:
- AI discovery (primary use case)
- Developer onboarding (see all capabilities)
- API documentation generation
- Integration testing (enumerate all commands)

**Why Better**: Foundation provides value REGARDLESS of AI success.

---

## 7. Appendices

### A. Example: Full Business Intent with Metadata

```typescript
// /lifecycle/lib/business-intents/confirm-activation.ts

import { z } from 'zod'
import type { ExecutionContext, Result, ContractState } from '../types'

/**
 * Business Intent: Confirm Contract Activation
 *
 * Triggered when provider confirms contract activation via:
 * - Email confirmation
 * - Portal status update
 * - Phone call from provider
 *
 * Side Effects:
 * - contract.status → 'active'
 * - contract.activated_at set to activation_date
 * - Event ContractActivated emitted
 */

export const confirmActivationSchema = z.object({
  contract_id: z.string().uuid().meta({
    id: "contract_id",
    title: "Contract Identifier",
    description: "UUID of the contract to mark as activated",
    validationPriority: "critical",
    aiGuidance: "Use the contract_id from the provider confirmation email subject or body. Usually prefixed with 'Contract #' or 'Vertragsnummer'.",
    examples: ["550e8400-e29b-41d4-a716-446655440000"],
    constraints: "Must reference an existing contract in 'pending' status"
  }),

  activation_date: z.string().date().meta({
    id: "activation_date",
    title: "Activation Date",
    description: "Date when the provider confirmed activation (YYYY-MM-DD format)",
    validationPriority: "critical",
    aiGuidance: "Extract from email subject line ('starts on [DATE]'), email body ('Activation Date: [DATE]'), or portal 'Contract Start Date' field. Convert to YYYY-MM-DD format.",
    examples: ["2025-02-01", "2025-03-15"],
    constraints: "Must be <= 30 days in the future, >= contract.created_at",
    relatedFields: ["contract.created_at", "contract.desired_start_date"]
  }),

  confirmation_source: z.enum(['email', 'portal', 'phone', 'letter']).meta({
    id: "confirmation_source",
    title: "Confirmation Source",
    description: "Channel through which provider sent activation confirmation",
    validationPriority: "medium",
    aiGuidance: "Use 'email' if extracted from email, 'portal' if scraped from provider portal, 'phone' if reported by customer service agent, 'letter' if scanned from physical mail.",
    examples: ["email", "portal"],
    context: "Used for analytics and audit trail"
  }),

  provider_reference: z.string().optional().meta({
    id: "provider_reference",
    title: "Provider Reference Number",
    description: "Provider's internal reference for this activation (optional)",
    validationPriority: "low",
    aiGuidance: "Extract if present in email/letter. Common labels: 'Reference:', 'Ref:', 'Vorgangsnummer:', 'Confirmation #:'",
    examples: ["ACT-2025-12345", "VG-987654"],
    context: "Useful for customer service inquiries with provider"
  })
})

export async function confirmActivation(
  input: z.infer<typeof confirmActivationSchema>,
  context: ExecutionContext
): Promise<Result<ContractState>> {

  // 1. Validate input (Zod schema)
  const validated = confirmActivationSchema.safeParse(input)
  if (!validated.success) {
    return {
      success: false,
      error: {
        code: 'VALIDATION_ERROR',
        message: 'Invalid activation data',
        details: validated.error.format()
      }
    }
  }

  // 2. Log execution with audit trail
  await auditLog.create({
    intent_id: 'confirm_activation',
    contract_id: validated.data.contract_id,
    executed_by: context.executed_by,
    executor_id: context.executor_id,
    trace_id: context.trace_id,
    ai_metadata: context.ai_metadata,
    input: validated.data,
    timestamp: new Date()
  })

  // 3. Execute state transition
  const result = await db.transaction(async (tx) => {
    // Fetch current state
    const contract = await tx.query.contracts.findFirst({
      where: eq(contracts.id, validated.data.contract_id)
    })

    if (!contract) {
      return {
        success: false,
        error: {
          code: 'CONTRACT_NOT_FOUND',
          message: `Contract ${validated.data.contract_id} not found`
        }
      }
    }

    // Validate preconditions
    if (contract.status !== 'pending') {
      return {
        success: false,
        error: {
          code: 'INVALID_STATUS_TRANSITION',
          message: `Cannot activate contract in status '${contract.status}'. Expected 'pending'.`,
          context: { current_status: contract.status, required_status: 'pending' }
        }
      }
    }

    // Validate activation date constraints
    const activationDate = new Date(validated.data.activation_date)
    const maxFutureDate = addDays(new Date(), 30)

    if (activationDate > maxFutureDate) {
      return {
        success: false,
        error: {
          code: 'ACTIVATION_DATE_TOO_FAR',
          message: `Activation date cannot be more than 30 days in the future`,
          context: { activation_date: validated.data.activation_date, max_date: maxFutureDate }
        }
      }
    }

    if (activationDate < contract.created_at) {
      return {
        success: false,
        error: {
          code: 'ACTIVATION_DATE_BEFORE_CREATION',
          message: `Activation date cannot be before contract creation date`,
          context: {
            activation_date: validated.data.activation_date,
            created_at: contract.created_at
          }
        }
      }
    }

    // Update state
    const [updatedContract] = await tx.update(contracts)
      .set({
        status: 'active',
        activated_at: activationDate,
        confirmation_source: validated.data.confirmation_source,
        provider_reference: validated.data.provider_reference,
        updated_at: new Date()
      })
      .where(eq(contracts.id, validated.data.contract_id))
      .returning()

    // Emit event (fire-and-forget webhook)
    await emitEvent({
      event_type: 'ContractActivated',
      contract_id: updatedContract.id,
      trace_id: context.trace_id,
      payload: {
        activation_date: validated.data.activation_date,
        confirmation_source: validated.data.confirmation_source,
        executed_by: context.executed_by
      }
    })

    return {
      success: true,
      data: updatedContract
    }
  })

  // 4. Update audit log with result
  await auditLog.update({
    where: { trace_id: context.trace_id },
    data: {
      output: result,
      completed_at: new Date()
    }
  })

  return result
}
```

---

### B. Example: AI Agent Consumption

```typescript
// AI Agent: Email Classification and Execution

import Anthropic from '@anthropic-ai/sdk'

const client = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY
})

export async function processProviderEmail(email: {
  from: string
  subject: string
  body: string
  received_at: Date
}) {

  // 1. Fetch AI catalog
  const catalog = await getAICatalog()

  // 2. Classify email intent using Claude
  const classificationPrompt = `
You are an AI agent for SwitchUp contract management.

Email from: ${email.from}
Subject: ${email.subject}
Body: ${email.body}

Available Business Intents:
${catalog.map(intent => `
- ${intent.intent_id}: ${intent.description}
  Preconditions: ${intent.preconditions.join(', ')}
`).join('\n')}

Which Business Intent should be executed for this email? Return JSON:
{
  "intent_id": "confirm_activation",
  "confidence": 0.95,
  "reasoning": "Email confirms contract activation with start date"
}
`

  const classification = await client.messages.create({
    model: 'claude-3.5-sonnet',
    max_tokens: 1024,
    messages: [{
      role: 'user',
      content: classificationPrompt
    }]
  })

  const result = JSON.parse(classification.content[0].text)

  if (result.confidence < 0.8) {
    // Low confidence - escalate to human
    await escalateToHuman(email, result)
    return
  }

  // 3. Fetch intent schema from catalog
  const intent = catalog.find(i => i.intent_id === result.intent_id)

  // 4. Extract parameters using Claude
  const extractionPrompt = `
Email from: ${email.from}
Subject: ${email.subject}
Body: ${email.body}

Extract parameters for Business Intent: ${intent.title}

Schema with guidance:
${intent.input_schema.schema}

Example:
${JSON.stringify(intent.input_schema.example, null, 2)}

Return JSON matching the schema above.
`

  const extraction = await client.messages.create({
    model: 'claude-3.5-sonnet',
    max_tokens: 1024,
    messages: [{
      role: 'user',
      content: extractionPrompt
    }]
  })

  const params = JSON.parse(extraction.content[0].text)

  // 5. Execute Business Intent
  const executionResult = await executeBusinessIntent(
    result.intent_id,
    params,
    {
      executed_by: 'ai_agent',
      executor_id: 'email-classifier-v1',
      trace_id: generateTraceId(),
      ai_metadata: {
        model: 'claude-3.5-sonnet',
        confidence: result.confidence,
        reasoning: result.reasoning,
        source_data_refs: [`email:${email.id}`]
      }
    }
  )

  // 6. Handle result
  if (!executionResult.success) {
    // Check if validation error (AI can retry)
    if (executionResult.error.code === 'VALIDATION_ERROR') {
      // Retry with corrected data
      const correctedParams = await aiFixValidationErrors(
        params,
        executionResult.error.details,
        intent.input_schema.schema
      )

      return executeBusinessIntent(result.intent_id, correctedParams, context)
    }

    // Other error - escalate
    await escalateToHuman(email, executionResult.error)
    return
  }

  // Success - log metrics
  await logAISuccess({
    intent_id: result.intent_id,
    confidence: result.confidence,
    execution_time_ms: Date.now() - startTime,
    email_id: email.id
  })
}
```

---

### C. Schema Metadata Standards

**Required Fields** (all schemas):
```typescript
{
  id: string           // Unique identifier (snake_case)
  title: string        // Human-readable name
  description: string  // What this field represents
  validationPriority: 'critical' | 'high' | 'medium' | 'low'
}
```

**AI-Specific Fields** (high-priority for AI consumption):
```typescript
{
  aiGuidance?: string    // How AI should populate this field
  examples?: string[]    // Sample valid values (2-3 examples)
  constraints?: string   // Business rules explanation
  context?: string       // Additional background
}
```

**Relationship Fields** (complex validation):
```typescript
{
  relatedFields?: string[]    // Other fields that affect this one
  dependsOn?: string          // Field that must be set first
  invalidates?: string[]      // Fields that become invalid if this changes
}
```

**Difficulty Rating** (Phase 2 - after AI testing):
```typescript
{
  aiDifficulty?: 'easy' | 'medium' | 'hard'
  // easy: Direct extraction (dates, amounts, IDs)
  // medium: Interpretation required (status from prose)
  // hard: Multi-step reasoning (calculate from related data)
}
```

---

### D. Greenfield Implementation Checklist

**Week 1-2: Schema Metadata**
- [ ] Add `.meta()` to all Business Intent schemas (50+ intents)
- [ ] Add `.meta()` to all Service-Type attribute schemas (energy, telco, insurance)
- [ ] Document metadata standards
- [ ] Create metadata validation tests

**Week 2-3: Audit Trail**
- [ ] Create `business_intent_audit_log` table
- [ ] Add `ExecutionContext` type definition
- [ ] Update all Business Intent handlers to accept `ExecutionContext`
- [ ] Implement audit log creation/update in handlers
- [ ] Create audit log query utilities

**Week 3-4: AI Catalog**
- [ ] Create `/lifecycle/lib/ai-catalog.ts`
- [ ] Register all 50+ Business Intents with descriptions
- [ ] Create `/lifecycle/lib/ai-service-type-catalog.ts`
- [ ] Implement schema introspection (required vs optional fields)
- [ ] Create catalog API endpoint

**Week 4: Documentation**
- [ ] Document AI agent integration patterns
- [ ] Create example AI agent (email classifier)
- [ ] Document validation error recovery patterns
- [ ] Create AI agent development guide

**Total Effort**: 3-4 weeks (part of Tier 1 Service-Type Abstraction refactoring)

---

### E. Success Metrics

**Foundation Completeness** (Phase 1):
- 100% of Business Intents have complete `.meta()` (all required fields)
- 100% of Service-Type schemas have `.meta()` with aiGuidance
- Audit log captures 100% of state changes with attribution
- AI catalog returns valid schemas for all intents

**AI Agent Performance** (Phase 2):
- Email classification accuracy: >90%
- Contract data extraction accuracy: >80% (German energy)
- Validation pass rate (first attempt): >70%
- Human escalation rate: <20%

**Operational Impact** (Phase 2-3):
- AI handles 50%+ of routine email processing
- Human intervention time reduced 40%+
- AI execution audit trail 100% complete (no missing attributions)
- Zero state corruption incidents from AI hallucinations

---

## Document Version

**Version**: 1.0
**Created**: January 2025
**Last Updated**: January 2025
**Next Review**: End of Q1 2025
**Owner**: CTO / Lead Architect

---

## References

- `SERVICE_TYPE_ABSTRACTION_ARCHITECTURE.md`: Service-Type Registry Pattern
- `GEOGRAPHIC_ABSTRACTION_ARCHITECTURE.md`: Market Registry Pattern
- `Domains/Lifecycle/Lifecycle-Domain-The-System-of-Record.md`: Business Intent Catalog
- `Technical/ZOD-4-DRIZZLE-2025-UPDATES.md`: Zod 4 metadata capabilities
