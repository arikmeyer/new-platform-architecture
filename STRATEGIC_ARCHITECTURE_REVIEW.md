# Strategic Architecture Review: SwitchUp Platform
**Date:** 2025-11-09
**Scope:** Multi-service expansion (Energy → Telco → Insurance) + Geographic expansion (Germany → International) at 2-3M+ household scale

---

## Executive Summary

**Overall Assessment:** The architecture demonstrates **excellent foundational thinking** with strong separation of concerns, clean domain boundaries, and sophisticated Windmill integration. However, **critical gaps exist** in service-type abstraction and geographic parameterization that will create significant friction during expansion.

**Key Verdict:**
- ✅ **Architecture pattern is sound** - Brain/Spine/Limbs model scales conceptually
- ✅ **Windmill utilization is exemplary** - Maximizes platform strengths
- ⚠️ **Service-type abstraction is incomplete** - Energy assumptions embedded throughout
- ⚠️ **Geographic strategy is absent** - No visible multi-market architecture
- ⚠️ **Scale readiness needs validation** - Lifecycle domain bottleneck risks

---

## 1. Domain Model Analysis: Service Type Universality

### Current State Assessment

**Strengths:**
- `/offer` domain's "Universal Service Ontology" philosophy is conceptually service-agnostic
- Clean domain boundaries enable service-specific logic encapsulation
- Blueprint/Policy architecture in Offer domain supports diverse product types

**Critical Gaps:**

#### Gap 1.1: Energy-Specific Core Entities
**Location:** `/lifecycle` domain (Lifecycle-Domain-The-System-of-Record.md:73-77)

The core business entity model contains energy-specific assumptions:

```
Entity: MeterPoint
- /lifecycle/meter_point/register-meter-point
- /lifecycle/meter_point/update-meter-point-attributes
```

**Problem:** "MeterPoint" is energy-specific. Telco has "phone numbers" or "MSISDN allocations," insurance has "vehicles" or "properties."

**Impact:**
- Adding telco requires either awkward abstraction ("MeterPoint" for a phone number)
- OR forking the entire entity model (creating TelecomServicePoint, InsuranceAsset, etc.)
- Both approaches violate DRY and create maintenance burden

**Recommendation:**
```
Refactor to service-agnostic abstraction:
Entity: ServicePoint
- service_point_type_id (references ServicePointType definition)
- type_specific_attributes (JSONB with validated schemas per type)

ServicePointType examples:
- energy.meter_point (electricity/gas meters)
- telco.msisdn (mobile numbers)
- telco.broadband_line (DSL/fiber connections)
- insurance.vehicle (car insurance)
- insurance.property (home insurance)
```

#### Gap 1.2: Service-Type Dimension Missing from Business Intents
**Location:** `/lifecycle` domain Business Intent Catalog

Current commands like `report-price-increase`, `confirm-activation` appear universal, but their validation logic and state transitions may have energy-specific assumptions.

**Questions:**
- Does "activation" mean the same thing for energy (meter switch) vs telco (SIM activation) vs insurance (policy inception)?
- Are state machines truly universal or service-specific?

**Recommendation:**
```
Option A: Universal State Machine (preferred)
- Keep commands generic
- Add service_type_context to command inputs
- Validation rules parameterized by service type

Option B: Service-Scoped Commands
- /lifecycle/contract/energy/confirm-meter-activation
- /lifecycle/contract/telco/confirm-sim-activation
- Explicit service-type namespacing
```

I **recommend Option A** - maintain universal commands but make validation rules and computed properties service-type-aware through configuration, not code.

#### Gap 1.3: Provider Domain Knowledge Graph Structure
**Location:** `/provider` domain (Provider-Domain.md:20-28)

The Provider Knowledge Graph stores "operational rules" and "interaction methods" but doesn't clearly indicate how service-type-specific knowledge is modeled.

**Example:**
- Energy provider: "requires meter reading submission within 14 days"
- Telco provider: "requires porting authorization code (PAC)"
- Insurance provider: "requires vehicle registration documents"

**Recommendation:**
```
Provider Knowledge Graph Schema Enhancement:
Provider {
  provider_id
  service_types[] {  // A provider may offer multiple service types
    service_type_id
    capabilities[] {
      business_action (e.g., "initiate_cancellation")
      interaction_methods[] (API, RPA, Email)
      operational_rules[] (service-type-specific)
    }
  }
}
```

### Verdict on Domain Model Universality

**Current:** 60% universal, 40% energy-centric
**Required for multi-service:** 95% universal, 5% service-specific plugins

**Critical Path:**
1. Refactor `MeterPoint` → `ServicePoint` with type system (BREAKING CHANGE)
2. Add `service_type_id` foreign key to Contract, Offer, Provider entities
3. Create `/lifecycle/service_type/` subdomain to manage service type registry
4. Parameterize validation rules, state machines, computed properties by service type

---

## 2. Geographic Expansion Readiness

### Current State Assessment

**Finding:** Geographic expansion strategy is **conspicuously absent** from the architecture specifications.

**Critical Questions Unanswered:**

#### Question 2.1: Where Do Regulations Live?
- German energy market: 14-day cancellation right (Widerrufsrecht)
- German telco market: Different cancellation windows
- UK energy market: Different regulatory framework (Ofgem vs BNetzA)

**Missing Component:** Regulatory rules engine separate from business logic

**Recommendation:**
```
Create: /lifecycle/regulatory/ subdomain

Entities:
- MarketDefinition (country/region + service_type)
- RegulatoryRule (cancellation_windows, mandatory_disclosures, etc.)

Usage:
Business Intents in /lifecycle query regulatory rules:
- "What is the legal cancellation window for this contract?"
- Computed property: contract.is_within_legal_cancellation_window
  → Queries MarketDefinition based on contract.market_id
```

#### Question 2.2: How Are Locale-Specific Data Handled?
- Date formats: Germany (DD.MM.YYYY) vs UK (DD/MM/YYYY) vs US (MM/DD/YYYY)
- Currency: EUR (Germany) vs GBP (UK) vs PLN (Poland)
- Address schemas: German postal codes (5 digits) vs UK postcodes (alphanumeric)
- Provider document formats: German PDFs use different layouts than UK PDFs

**Missing Component:** Localization and normalization layer

**Recommendation:**
```
Enhancement to /provider/extraction/ capabilities:

Input Schema:
{
  "provider_id": "...",
  "market_id": "de_DE",  // NEW: Explicit market context
  "content": "..."
}

Logic:
1. Load market-specific parsing rules
2. Apply locale-aware normalization (dates, currency, addresses)
3. Validate against market-specific schemas
4. Return normalized data in internal canonical format (always ISO 8601, always EUR cents, always canonical address structure)
```

#### Question 2.3: How Is Provider Ecosystem Scoped?
**Fundamental Design Question:**

Are providers market-scoped or global?

**Scenario:** Vodafone operates in Germany, UK, Spain, Italy...
- Is `vodafone_de` a separate provider entity from `vodafone_uk`?
- OR is `Vodafone` one provider with market-specific configurations?

**Implication for Architecture:**

**Option A: Market-Scoped Providers (simpler)**
```
Provider {
  provider_id: "vodafone_de"
  market_id: "de_DE"
  global_brand_id: "vodafone" (optional, for analytics)
}
```
✅ Simpler queries (filter by market_id)
✅ No cross-market coupling
❌ Brand-level analytics harder

**Option B: Global Providers with Market Dimensions (complex)**
```
Provider {
  provider_id: "vodafone"
  market_operations[] {
    market_id: "de_DE"
    market_specific_config: {...}
  }
}
```
✅ Brand-level view
❌ All queries need market filtering
❌ Risk of cross-market data leakage

**Recommendation:** **Option A (market-scoped)**. Your architecture philosophy favors explicit separation over complex hierarchies. Treat each market as an isolated operational unit.

#### Question 2.4: Offer Domain Geographic Availability
**Location:** `/offer` domain (Offer-Domain.md)

The Offer domain has "Location Availability Lookup" in aggregation logic, but it's unclear how multi-country availability works.

**Current (assumed):** Offers have geographic availability rules (e.g., "available in postal codes 10000-19999")

**Multi-market question:** Is this:
- Per-market offer catalogs (Germany offers vs UK offers are separate)
- OR global offers with multi-market availability rules?

**Recommendation:** **Per-market offer catalogs** with market_id scoping:
```
OfferDefinition {
  offer_id
  market_id  // NEW: All offers are scoped to a market
  ...
}

Query pattern:
get-aggregate-offers(
  entityConfigVersionIds: [...],
  search_request_context: {
    session_context: {
      market_id: "de_DE"  // NEW: Explicit market
    }
  }
)
```

### Verdict on Geographic Expansion

**Current:** 10% ready (no explicit multi-market architecture)
**Required:** Full geographic dimension across all domains

**Critical Path:**
1. Add `market_id` as first-class dimension to all geographic-dependent entities (Contract, Offer, Provider, Address)
2. Create `/lifecycle/market/` subdomain for market registry and regulatory rules
3. Implement locale-aware data validation and normalization in `/provider` and `/offer` domains
4. Add market filtering to all domain queries
5. Design data isolation strategy (multi-tenant DB vs per-market schemas vs per-market instances)

---

## 3. Scalability to 2-3M+ Households

### Current Strengths

✅ **Stateless tool domains** - /provider, /offer, /optimisation, /service scale horizontally
✅ **Event-driven architecture** - Decoupled reactions via webhooks
✅ **Configuration-driven routing** - Process changes don't require redeployment
✅ **Clear observability** - Trace ID propagation enables distributed debugging

### Bottleneck Analysis

#### Bottleneck 3.1: /lifecycle Domain Write Throughput
**Location:** Lifecycle-Domain-The-System-of-Record.md:9-12

**Current:** All state mutations funnel through `/lifecycle` Command Handlers

**Scale Math:**
- 2M households × 10 lifecycle events/year = 20M writes/year
- = ~55K writes/day = ~0.6 writes/second (average)
- Peak (e.g., price increase season): 10× average = 6 writes/second

**Assessment:** Low write volume (~6 writes/sec at peak) is **easily handled** by any modern database.

**However:**
- Each Command Handler may have complex validation logic (multi-table queries)
- Temporal queries (`get-timeline-projection`) can be expensive
- No discussion of database choice (Postgres? MySQL? MongoDB?)

**Recommendation:**
```
Explicit Scaling Strategy for /lifecycle:

1. Database Choice:
   - PostgreSQL with JSONB for type_specific_attributes
   - Write sharding by user_id hash (if needed at >>3M scale)
   - Read replicas for query capabilities

2. CQRS Pattern Consideration:
   - Command side: Optimized for writes, emit events
   - Query side: Materialized views optimized for get-details, get-timeline-projection
   - Event sourcing for full audit trail (optional but recommended)

3. Caching Strategy:
   - get-details responses cached (TTL: 5 minutes)
   - Cache invalidation on Command Handler execution
```

#### Bottleneck 3.2: Event Delivery Reliability
**Location:** Lifecycle-Domain-The-System-of-Record.md:184-190

**Current:** "Fire-and-forget" webhook with acknowledged 0.01% loss risk

**Scale Math:**
- 20M events/year × 0.01% loss = 2,000 lost events/year

**Assessment:** At 150K households, 150 lost events/year is tolerable with nightly reconciliation. At 2M households, 2,000 lost events/year becomes operationally significant.

**Recommendation:**
```
Replace fire-and-forget webhooks with proper event bus:

Option A: Managed Event Bus
- AWS EventBridge / Azure Event Grid / GCP Pub/Sub
- At-least-once delivery guarantees
- Built-in retry and dead-letter queues

Option B: Open-Source Event Bus
- Kafka (high throughput, complex ops)
- RabbitMQ (simpler, lower throughput)

Recommendation: Start with managed cloud event bus (EventBridge/Pub/Sub)
- Minimal ops overhead
- Built-in monitoring
- Windmill can consume via webhooks or native integrations
```

#### Bottleneck 3.3: Provider Knowledge Graph Query Hot Spot
**Location:** Provider-Domain.md:20-28

**Current:** Every provider interaction queries the Knowledge Graph

**Scale Math:**
- 2M households × 5 provider interactions/year = 10M queries/year
- = ~27K queries/day = ~0.3 queries/second (average)

**Assessment:** Very low read volume, **not a bottleneck**

**Recommendation:** Simple in-memory cache with 1-hour TTL (provider configs change infrequently)

#### Bottleneck 3.4: Windmill Platform Limits
**Location:** Not addressed in architecture docs

**Critical Unknown:** What are Windmill's concurrency and throughput limits?

**Questions:**
- How many concurrent flows can a Windmill instance handle?
- What is the latency for flow dispatch and execution?
- How does Windmill handle long-running flows (e.g., multi-day processes)?
- What is Windmill's pricing model at scale (per-execution? per-compute-time?)?

**Recommendation:**
```
Immediate Action Items:

1. Load Testing:
   - Simulate 100K concurrent flows
   - Measure dispatch latency, execution throughput
   - Identify breaking points

2. Windmill Scaling Plan:
   - Multi-instance deployment (is this supported?)
   - Horizontal scaling strategy
   - Cost projection at 2M households

3. Escape Hatch Design:
   - For critical paths that exceed Windmill limits
   - Consider hybrid architecture (Windmill for orchestration, custom services for high-throughput)
```

### Verdict on Scalability

**Current:** Architecture pattern supports scale, but **implementation unknowns** exist
**Confidence:** 70% (strong conceptual foundation, needs validation)

**Critical Path:**
1. Load test Windmill at target scale (immediate)
2. Implement proper event bus for lifecycle events (before 500K households)
3. Add CQRS/caching to /lifecycle domain (before 1M households)
4. Document database scaling strategy (before 2M households)

---

## 4. Maximum Windmill Leverage Assessment

### Strengths: Exemplary Alignment

✅ **Flow for orchestration** - /case domain uses Flows perfectly
✅ **Script for capabilities** - Tool domains use Scripts correctly
✅ **Process Dispatcher pattern** - Elegant use of dynamic flow dispatch (Capability-Architecture.md:38-45)
✅ **Configuration-driven routing** - Federated routing.json aligns with Windmill's git-sync
✅ **Visual process modeling** - Flows are inherently visible and traceable

**This is textbook Windmill usage.** Your architecture maximizes the platform's strengths.

### Opportunities: Deeper Integration

#### Opportunity 4.1: Windmill Resource Types
**Question:** Are you using Windmill's built-in resource types for databases, APIs, and credentials?

**Recommendation:**
```
Leverage Windmill Resources for:
- Database connections (PostgreSQL resource for /lifecycle)
- Provider API credentials (OAuth/API key resources)
- External service integrations (Datadog, OpenSearch)

Benefits:
- Centralized credential management
- Automatic credential rotation support
- Access control via Windmill RBAC
```

#### Opportunity 4.2: Windmill State Management
**Question:** How do long-running processes (e.g., multi-week contract lifecycle) persist state?

**Current (assumed):** State stored in /lifecycle domain via Commands

**Windmill Feature:** Flows can have persistent state, approval steps, human-in-the-loop

**Recommendation:**
```
Use Windmill's state management for:
- Multi-step approval workflows (e.g., high-value contract changes)
- Long-running async processes (e.g., provider response wait loops)
- Human task tracking (instead of separate Task entity?)

Question: Could /lifecycle/task entity be replaced by Windmill's native task/approval features?
```

#### Opportunity 4.3: Windmill Observability Integration
**Current:** Logs shipped to Datadog/OpenSearch (Capability-Architecture.md:81-84)

**Question:** Does Windmill provide native observability integrations?

**Recommendation:**
```
Investigate Windmill's built-in observability:
- Flow execution history (likely exists)
- Automatic trace correlation (does it support Trace ID propagation?)
- Performance metrics (flow duration, failure rates)
- Integration with Datadog/OpenSearch (native vs custom)

If Windmill has native OpenTelemetry support, use it instead of custom logging.
```

### Gaps: Windmill Constraints

#### Gap 4.1: Windmill Vendor Lock-In Risk
**Assessment:** Architecture is **deeply coupled** to Windmill's execution model

**Risk Scenarios:**
- Windmill pricing becomes prohibitive at scale
- Windmill performance doesn't meet requirements
- Windmill platform reliability issues
- Windmill company discontinues product

**Mitigation:** Design **escape hatches** for critical paths

**Recommendation:**
```
Abstraction Layer Strategy:

1. Define standard interfaces for capabilities (OpenAPI specs)
2. Implement capabilities as standalone functions wrapped in Windmill Scripts
3. For critical high-throughput paths, allow dual implementation:
   - Primary: Windmill Script
   - Fallback: Standalone service (FastAPI, Node.js)

Example:
/lifecycle/contract/confirm-activation
- Implemented as Python function with type hints
- Wrapped in Windmill Script
- Can be deployed as FastAPI endpoint if needed

This "Windmill-native with escape hatch" approach balances leverage with risk.
```

#### Gap 4.2: Windmill Multi-Tenancy
**Question:** How does Windmill handle multi-tenancy for geographic expansion?

**Options:**
- **Single Windmill workspace:** All markets in one workspace (data isolation via market_id filtering)
- **Per-market workspaces:** Germany workspace, UK workspace (operational complexity)
- **Per-market instances:** Separate Windmill deployments (maximum isolation, high overhead)

**Recommendation:**
```
Start with single workspace, market_id scoping:
- Simpler operations
- Shared code/flow library
- Data isolation via query filtering

Migrate to per-market workspaces IF:
- Regulatory requirements demand physical data isolation
- Scale requires geographic distribution
- Market-specific customizations become too complex
```

### Verdict on Windmill Leverage

**Current:** 90% excellent, 10% unexplored opportunities
**Recommendation:** Continue Windmill-first approach with documented escape hatches

---

## 5. Extensibility Architecture: Adding Service Types & Markets

### Current Extensibility Model

**Strengths:**
- Domain boundaries enable isolated changes
- Offer domain's Blueprint/Policy architecture is inherently extensible
- Configuration-driven routing supports process variations

**Gap:** No explicit extension model documented

### Proposed: Hybrid Extension Architecture

#### 5.1: Service Type Extension Model

**Principle:** New service types should be **configuration-first, code-only-when-necessary**

**Extension Mechanism:**

```
Service Type Package Structure:
/service_types/
  energy/
    lifecycle_entity_schemas.json        # ServicePoint schema for meter points
    offer_blueprints.json                # Energy-specific Element types (kWh pricing, bonuses)
    provider_interaction_patterns.json   # Meter readings, portal scraping
    optimisation_rules.json              # Savings calculation specifics
  telco/
    lifecycle_entity_schemas.json        # ServicePoint schema for MSISDNs
    offer_blueprints.json                # Telco-specific Elements (data allowances, minutes)
    provider_interaction_patterns.json   # Porting, SIM activation
    optimisation_rules.json              # Coverage quality, data speed modeling
  insurance/
    lifecycle_entity_schemas.json        # ServicePoint schema for vehicles/properties
    offer_blueprints.json                # Insurance-specific Elements (deductibles, coverage limits)
    provider_interaction_patterns.json   # Claims submission, policy documents
    optimisation_rules.json              # Risk assessment, premium calculations
```

**Core Platform Changes Required:**

```
1. /lifecycle domain:
   - Refactor entities to use type_specific_attributes (JSONB)
   - Load validation schemas from service type packages
   - Computed properties become configurable rules

2. /offer domain:
   - Already has Blueprint architecture (minimal changes)
   - Add service_type_id to OfferDefinition
   - Load Element/Policy blueprints from service type packages

3. /provider domain:
   - Provider Knowledge Graph adds service_type_id dimension
   - Load interaction patterns from service type packages
   - RPA/extraction logic may need custom code (allow plugins)

4. /optimisation domain:
   - Load calculation rules from service type packages
   - Allow custom calculation scripts (plugin architecture)

5. /case domain:
   - Process implementations (Playbooks/Agents) may be service-specific
   - Routing config can direct by service_type_id
```

**Code Plugin Interface (for complex service-specific logic):**

```python
# Core platform defines interface
class ServiceTypePlugin(Protocol):
    def validate_service_point(self, data: dict) -> ValidationResult:
        """Custom validation logic for service-specific attributes"""

    def calculate_service_value(self, contract: Contract, offers: list[Offer]) -> ValueComparison:
        """Custom value calculation (e.g., insurance risk modeling)"""

    def extract_provider_data(self, content: str, provider_id: str) -> dict:
        """Custom extraction logic for provider-specific documents"""

# Service types can implement plugins for complex logic
class TelcoPlugin(ServiceTypePlugin):
    def calculate_service_value(self, contract, offers):
        # Telco-specific: factor in network coverage quality, data speed
        ...
```

**Onboarding a New Service Type:**

```
Steps:
1. Create service type package (JSONs + optional Python plugin)
2. Register service type in /lifecycle/service_type/register-service-type
3. Load schemas/blueprints into respective domains
4. Test with sample data
5. Deploy to production (all domains auto-detect new type via registry)

Goal: 80% configuration, 20% code for a new service type
```

#### 5.2: Market Extension Model

**Principle:** New markets should be **data packages** loaded into the platform

**Market Package Structure:**

```
/markets/
  de_DE/
    regulatory_rules.json         # Cancellation windows, mandatory disclosures
    locale_config.json            # Date formats, currency, address schema
    provider_registry.json        # Initial provider list for this market
    offer_catalog_seeds.json      # Starter offers (if applicable)
  gb_UK/
    regulatory_rules.json
    locale_config.json
    provider_registry.json
    offer_catalog_seeds.json
```

**Core Platform Changes Required:**

```
1. /lifecycle/market/ subdomain (NEW):
   - register-market Command
   - get-regulatory-rules Query
   - get-locale-config Query

2. All geographic-dependent entities:
   - Add market_id foreign key
   - Add market_id to all queries

3. /provider domain:
   - Providers scoped by market_id
   - Extraction/validation uses locale_config

4. /offer domain:
   - Offers scoped by market_id
   - Availability rules within-market only
```

**Onboarding a New Market:**

```
Steps:
1. Create market package (JSONs)
2. Register market in /lifecycle/market/register-market
3. Seed initial providers via /lifecycle/provider_profile/register-provider
4. Configure offer ingestion sources
5. Deploy market-specific RPA bots (if needed)
6. Test end-to-end with pilot users

Goal: New market operational in 2-4 weeks (data + config, minimal code)
```

### Verdict on Extensibility

**Current:** 40% ready (clean boundaries, unclear extension model)
**Required:** Formal extension architecture with documentation

**Critical Path:**
1. Design and document service type plugin interface (before first non-energy expansion)
2. Design and document market package structure (before first non-Germany expansion)
3. Refactor /lifecycle to support type/market dimensions (BREAKING CHANGE, do early)
4. Create developer guides: "How to Add a Service Type," "How to Add a Market"

---

## 6. Platform Evolution Path: Deterministic → AI-Agentic

### Strengths: Excellent AI-Ready Foundation

✅ **3-Layer Cognitive Stack** (AI-Operating-System.md) - Clear hierarchy: Playbooks → Capabilities → Commands
✅ **Playbook vs Agent implementation strategy** - Versioned, A/B testable
✅ **Configuration-driven routing** - AI agents can be rolled out gradually
✅ **Trace ID propagation** - AI decision chains are auditable

**Assessment:** Architecture is **ahead of most platforms** in AI-readiness

### Gaps: AI Operationalization

#### Gap 6.1: AI Learning Feedback Loop
**Current:** AI agents execute processes, but no visible mechanism for learning from outcomes

**Questions:**
- How does the platform measure AI agent success/failure?
- How does outcome data feed back into model training?
- How are successful agent strategies promoted to playbooks?

**Recommendation:**
```
Add AI Performance Tracking to /lifecycle:

Entity: AgentExecution
- agent_version_id
- input_context
- decision_trace (chain of thought)
- outcome (SUCCESS | FAILURE | ESCALATED)
- outcome_metrics (time_to_resolution, cost, user_satisfaction)
- human_review_rating (if reviewed)

Usage:
1. Every _agent flow logs execution to /lifecycle/agent_execution/log-execution
2. Analytics pipeline computes agent performance metrics
3. Successful agent patterns analyzed for playbook generation
4. Low-performing agents trigger retraining or rollback
```

#### Gap 6.2: Agent Safety Constraints
**Current:** No documented guardrails for AI agent actions

**Risk Scenarios:**
- Agent approves a high-cost action (e.g., €1000 credit) incorrectly
- Agent cancels a contract without proper user confirmation
- Agent makes irreversible provider API calls based on misinterpreted data

**Recommendation:**
```
Multi-Layer Safety Architecture:

Layer 1: Input Validation (Pre-execution)
- Schema validation on all agent inputs
- Plausibility checks (e.g., credit_amount < €500)

Layer 2: Decision Constraints (During execution)
- High-risk actions require human approval
  Example: apply-manual-credit > €100 → create approval task
- Configurable risk thresholds per process

Layer 3: Audit & Rollback (Post-execution)
- All agent decisions logged with full chain_of_thought
- Human review queue for random sampling
- Rollback mechanism for incorrect decisions

Layer 4: Rate Limiting
- Agent cannot execute > N high-risk actions per hour
- Circuit breaker: If agent failure rate > X%, pause and alert
```

#### Gap 6.3: Agent Performance Metrics
**Current:** No defined KPIs for agent vs playbook comparison

**Recommendation:**
```
Agent Performance Scorecard (per process):

Efficiency Metrics:
- Time to resolution (agent vs playbook vs human)
- Cost per execution (AI API costs + error correction)
- Automation rate (% of cases fully automated)

Quality Metrics:
- Success rate (correct outcome on first attempt)
- Escalation rate (% requiring human intervention)
- Error rate (incorrect decisions requiring rollback)

Business Metrics:
- User satisfaction (CSAT/NPS for agent interactions)
- Operational savings (human hours saved)

Rollout Decision Framework:
- Agent graduates from experiment to primary ONLY if:
  - Success rate > 95%
  - Escalation rate < 10%
  - User satisfaction >= playbook baseline
```

#### Gap 6.4: Playbook Generation from Agent Patterns
**Current:** No mechanism to "crystallize" successful agent behaviors into deterministic playbooks

**Vision:** AI agents discover better process strategies → automatically generate playbooks

**Recommendation:**
```
Agent-to-Playbook Pipeline (future enhancement):

1. Pattern Mining:
   - Analyze successful agent executions
   - Identify common decision trees
   - Extract reusable patterns

2. Playbook Synthesis:
   - Generate Windmill Flow from pattern
   - Human review and refinement
   - Deploy as new playbook version

3. Continuous Improvement Loop:
   - Agents experiment with novel approaches
   - Successful patterns become playbooks
   - Playbooks become baseline for next agent generation

Timeline: Year 2-3 capability (requires significant ML ops maturity)
```

### Verdict on AI Evolution Path

**Current:** 75% ready (excellent foundation, needs operationalization)
**Recommendation:** Add safety constraints and performance tracking **before** first agent deployment

**Critical Path:**
1. Implement agent safety constraints (immediate, before any agent goes to production)
2. Add agent execution logging and performance tracking (before first agent deployment)
3. Define agent graduation criteria (before first experiment → production promotion)
4. Build human review interface for agent decisions (first 6 months of agent use)
5. Design agent-to-playbook synthesis (Year 2+)

---

## 7. Critical Gaps Summary

### Tier 1: Blockers for Multi-Service/Multi-Geography (Address Immediately)

| Gap | Impact | Effort | Priority |
|-----|--------|--------|----------|
| Service type abstraction missing from /lifecycle | Blocks telco/insurance expansion | High (breaking change) | **P0** |
| Geographic dimension missing from core entities | Blocks international expansion | High (breaking change) | **P0** |
| No documented extension model (service types, markets) | Ad-hoc expansion creates tech debt | Medium (design + docs) | **P0** |

### Tier 2: Scale Readiness (Address Before 1M Households)

| Gap | Impact | Effort | Priority |
|-----|--------|--------|----------|
| Windmill scale limits unknown | Potential performance cliff | Low (testing) | **P1** |
| Fire-and-forget event delivery at scale | 1000s of lost events/year | Medium (infra) | **P1** |
| /lifecycle domain scaling strategy undefined | Potential bottleneck | Medium (architecture) | **P2** |

### Tier 3: AI Operationalization (Address Before First Agent Deployment)

| Gap | Impact | Effort | Priority |
|-----|--------|--------|----------|
| No agent safety constraints | Risk of incorrect high-impact actions | Medium (design + impl) | **P1** |
| No agent performance tracking | Cannot validate agent effectiveness | Low (logging + metrics) | **P1** |
| No agent learning feedback loop | AI cannot improve over time | High (ML ops pipeline) | **P2** |

---

## 8. Recommended Action Plan

### Phase 1: Foundation for Multi-Service/Multi-Geography (Months 1-3)

**Goal:** Refactor core domain model to support service types and markets

**Deliverables:**
1. **Service Type Abstraction**
   - Design service type plugin architecture
   - Refactor MeterPoint → ServicePoint with type system
   - Add service_type_id to Contract, Offer, Provider entities
   - Create /lifecycle/service_type/ subdomain
   - Document "How to Add a Service Type" guide

2. **Geographic Dimension**
   - Add market_id to all location-dependent entities
   - Create /lifecycle/market/ subdomain for market registry
   - Design locale-aware validation architecture
   - Document "How to Add a Market" guide

3. **Extension Model Documentation**
   - Service type package specification
   - Market package specification
   - Plugin interface definitions
   - Onboarding checklists

**Success Criteria:**
- Telco service type can be added via config + minimal code
- UK market can be added via config + minimal code
- Breaking changes completed before production scale

### Phase 2: Scale Validation & Infrastructure Hardening (Months 3-6)

**Goal:** Validate architecture at target scale, harden infrastructure

**Deliverables:**
1. **Windmill Load Testing**
   - Simulate 100K concurrent flows
   - Identify performance bottlenecks
   - Document scale limits and mitigation strategies
   - Design escape hatches if needed

2. **Event Infrastructure Upgrade**
   - Replace fire-and-forget webhooks with event bus (EventBridge/Pub/Sub)
   - Implement at-least-once delivery guarantees
   - Add dead-letter queues and monitoring

3. **Lifecycle Domain Optimization**
   - Add caching layer for high-frequency queries
   - Implement read replicas if needed
   - Design CQRS pattern for future scale (optional)

**Success Criteria:**
- Platform handles 3M household simulation load
- Event delivery reliability >99.99%
- /lifecycle query latency <100ms at p99

### Phase 3: AI Safety & Operationalization (Months 4-8)

**Goal:** Prepare platform for production AI agent deployment

**Deliverables:**
1. **Agent Safety Framework**
   - Multi-layer safety constraints (validation, approval, audit)
   - Risk-based action classification
   - Human review workflows for high-risk decisions

2. **Agent Performance Tracking**
   - AgentExecution logging in /lifecycle
   - Performance metrics dashboard (success rate, escalation rate, etc.)
   - Agent graduation criteria definition

3. **First Agent Deployment**
   - Select low-risk process for first agent (e.g., simple data extraction)
   - Deploy as A/B experiment (10% traffic)
   - Validate performance metrics
   - Graduate to primary if criteria met

**Success Criteria:**
- First agent achieves >95% success rate in production
- Human review interface operational
- Agent performance dashboard live

### Phase 4: First Expansion (Telco or UK Market) (Months 6-12)

**Goal:** Validate extension architecture with real expansion

**Deliverables:**
1. **Telco Service Type OR UK Market**
   - Create service type/market package
   - Onboard via documented process
   - Identify and fix extension friction points

2. **Extension Process Refinement**
   - Update onboarding guides based on learnings
   - Improve plugin interfaces
   - Optimize time-to-market

**Success Criteria:**
- New service type/market operational in <4 weeks
- 90% configuration-driven (10% code)
- Extension process documented and validated

---

## 9. Final Assessment: Strengths & Risks

### Architectural Strengths (Preserve These)

1. **Brain/Spine/Limbs model** - Clean separation of orchestration, state, and capabilities
2. **Windmill-native design** - Maximizes platform strengths, excellent tooling alignment
3. **Configuration-driven agility** - Process Dispatcher + routing.json enables safe experimentation
4. **Intent-driven API** - Business Intent Catalog provides semantic clarity
5. **3-layer AI cognitive stack** - Sophisticated framework for AI agent evolution
6. **Observability-first** - Trace ID propagation, structured logging from day one

### Architectural Risks (Mitigate These)

1. **Service-type assumptions embedded in core** - Energy-centricity will create friction in expansion
   - **Mitigation:** Refactor to service-agnostic core (Phase 1)

2. **Geographic expansion strategy absent** - No clear path to multi-market
   - **Mitigation:** Add market dimension to all domains (Phase 1)

3. **Windmill vendor lock-in** - Deep coupling to platform creates switching cost
   - **Mitigation:** Design escape hatches, abstract critical paths (ongoing)

4. **Event delivery reliability at scale** - Fire-and-forget acceptable for 150K, risky at 2M+
   - **Mitigation:** Upgrade to proper event bus (Phase 2)

5. **Unknown Windmill scale limits** - Performance cliff risk
   - **Mitigation:** Load testing and validation (Phase 2)

6. **AI agent safety not addressed** - Risk of incorrect high-impact actions
   - **Mitigation:** Multi-layer safety framework (Phase 3)

---

## 10. Conclusion

**Summary Verdict:**

Your architecture demonstrates **sophisticated thinking** and **strong foundational design**. The Brain/Spine/Limbs model is elegant, the Windmill integration is exemplary, and the AI evolution path shows strategic foresight.

**However**, the architecture is currently **60% ready for stated goals**:

- ✅ **Single-service, single-geography at current scale:** Excellent
- ⚠️ **Multi-service expansion:** 60% ready (needs service type abstraction)
- ⚠️ **Multi-geography expansion:** 40% ready (needs geographic dimension)
- ⚠️ **2-3M household scale:** 70% ready (needs validation + infrastructure hardening)
- ✅ **AI evolution path:** 75% ready (needs safety + operationalization)

**The most critical gaps are not in the pattern, but in the parameterization:**

- Core entities assume energy-specific models (MeterPoint)
- No explicit service type or geographic abstraction layers
- Extension model is implicit, not documented

**Strategic Recommendation:**

**Invest 3-6 months in foundational refactoring (Phase 1-2) before scaling or expanding.** The breaking changes required (MeterPoint → ServicePoint, adding market_id dimensions) are easier to make at 150K households than at 1M+.

**The good news:** Your domain boundaries and separation of concerns mean these refactors are **localized and manageable**. The architecture's strength is that it can absorb these changes without systemic rewrites.

**Final Guidance:**

You're building an **operating system for subscription contract management**. Think like a platform architect, not an application developer:

- **Universal core** (lifecycle, case) - Must work for any service type, any market
- **Parameterized domains** (provider, offer) - Configured per service/market, not hardcoded
- **Pluggable extensions** - New services/markets added via data packages + minimal code

**This review's purpose:** Identify the gaps **now**, while you're at spec stage, so you can design them out **before** they become technical debt at scale.

You have an excellent foundation. Make it truly universal, and you'll build something remarkable.

---

**Prepared by:** Claude (Anthropic)
**Review Date:** 2025-11-09
**Next Review Recommended:** After Phase 1 completion (foundational refactoring)
