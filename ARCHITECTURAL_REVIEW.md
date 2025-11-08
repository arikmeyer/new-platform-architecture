# Comprehensive Architectural Review
## SwitchUp Next-Generation Platform Architecture

**Date:** 2025-11-08
**Reviewer:** Claude (Architectural Analysis)
**Scope:** Complete strategic architecture review focusing on operational scalability, Windmill.dev fit, and future-proofing

---

## Executive Summary

This architecture specification demonstrates **strong conceptual thinking** with excellent Domain-Driven Design principles, clear separation of concerns, and forward-thinking AI integration. The "Brain, Spine, and Limbs" model is intellectually sound and well-articulated.

**However, there is a critical gap between the conceptual architecture and its practical implementation on Windmill.dev.** The specification treats Windmill as a full application platform when it is fundamentally a workflow orchestration tool. This creates several unresolved questions about database management, service boundaries, and infrastructure that must be addressed before implementation.

### Critical Findings

1. ✅ **Strong:** Domain boundaries and separation of concerns
2. ✅ **Strong:** Configuration-driven process routing for agility
3. ✅ **Strong:** Business Intent Catalog for auditability
4. ⚠️  **CRITICAL:** Database architecture completely missing
5. ⚠️  **CRITICAL:** Windmill.dev platform capabilities misaligned with architectural needs
6. ⚠️  **CRITICAL:** Infrastructure layer undefined
7. ⚠️  **GAP:** AI capabilities described are aspirational, not currently achievable at scale
8. ⚠️  **GAP:** Security, error handling, and operational concerns underspecified

### Recommendation

**Do not proceed with implementation without first resolving the Windmill.dev architectural fit question.** You must decide:

- **Option A:** Windmill-Heavy (all domains as Windmill Scripts + external PostgreSQL)
- **Option B:** Hybrid Architecture (Windmill for /case orchestration + microservices for domains) ← **RECOMMENDED**
- **Option C:** Minimal Windmill (Windmill only for ingestion + Temporal/Camunda for orchestration)

---

## Part 1: Strategic Strengths

### 1.1 Domain-Driven Design Excellence

**Strength:** The domain partitioning is business-aligned and conceptually clean.

- `/lifecycle` as single source of truth is architecturally correct
- `/case` as exclusive orchestrator enforces process clarity
- `/provider` as anti-corruption layer properly isolates complexity
- `/offer`, `/optimisation`, `/service`, `/growth` have clear, non-overlapping mandates

**Why This Matters:** Clean boundaries reduce coupling, enable team autonomy, and make the system easier to reason about.

### 1.2 Orchestration as Explicit Artifact

**Strength:** Business processes are first-class, version-controlled entities, not emergent behavior.

- Process Dispatcher + Routing Manifest makes processes visible
- A/B testing and gradual rollouts become operational capabilities
- Process evolution is managed through configuration, not code changes
- Experiment lifecycle management enables data-driven optimization

**Why This Matters:** This directly addresses operational scalability by making process automation transparent and evolvable.

### 1.3 Business Intent Catalog

**Strength:** State mutations are explicit, intent-driven commands.

- `ConfirmActivation`, `ReportPriceIncrease`, `ApplyManualCredit` are semantic, not CRUD
- Creates perfect audit trail
- Prevents invalid state transitions
- Makes system behavior predictable and testable

**Why This Matters:** This is critical for a regulated industry (energy, insurance) where auditability and compliance are non-negotiable.

### 1.4 AI Integration Strategy

**Strength:** The three-tier cognitive stack (Strategies → Capabilities → Commands) is sound.

- Provides clear abstraction levels for AI agents
- "Playbook vs Agent" distinction allows gradual AI adoption
- Constraining AI to use Business Intents as atomic actions provides safety
- Recognizes that not all processes should be agentic

**Why This Matters:** This enables you to evolve from deterministic automation to agentic AI without architectural rewrites.

### 1.5 Configuration-Driven Agility

**Strength:** The routing configuration system enables business agility.

- Process variants can be deployed and tested independently
- Percentage rollouts reduce risk
- Federated ownership aligns with team structure
- Experiment tracking is built into the manifest

**Why This Matters:** This is how you achieve continuous improvement of operational processes without deployment friction.

---

## Part 2: Critical Architectural Gaps

### 2.1 DATABASE ARCHITECTURE IS MISSING ⚠️ **CRITICAL**

**The Problem:**

The entire `/lifecycle` domain is described as the "System of Record" with "transactional guarantees," "temporal state projections," and "immutable history." Yet there is **zero specification of how data is actually stored, queried, or managed.**

**What's Missing:**

1. **Database Technology Choice**
   - PostgreSQL? (Most likely for transactional guarantees)
   - Event store? (If true event sourcing)
   - Graph database? (For complex entity relationships)
   - Time-series DB? (For temporal projections)

2. **Schema Design**
   - Table structures for Contract, User, Task, Address, MeterPoint, etc.
   - Relationship models (1:1, 1:N, M:N)
   - Indexing strategy for performance
   - Partitioning strategy for scale

3. **Data Modeling Decisions**
   - Normalized vs denormalized
   - How is "immutable history" stored? Separate audit table? Event log?
   - How are "computed properties" materialized or calculated?
   - How is temporal state queried efficiently?

4. **Connection Management**
   - How do Windmill Scripts connect to the database?
   - Connection pooling? (PgBouncer, pgpool?)
   - Connection string management (Windmill Resources?)
   - Transaction boundaries across multiple Script calls

5. **Migration Strategy**
   - Schema versioning (Flyway, Liquibase?)
   - Backward compatibility
   - Zero-downtime deployments

6. **Backup and Recovery**
   - Point-in-time recovery
   - Disaster recovery RTO/RPO
   - Data retention policies (GDPR compliance)

**Impact:** Without this, the `/lifecycle` domain is just an interface specification with no implementation plan. This is the foundation of the entire system.

**Recommendation:** Create a complete data architecture document including:
- ER diagrams for all entities
- Database technology selection with rationale
- Schema definitions with migration scripts
- Connection management and pooling strategy
- Performance and scalability projections

### 2.2 WINDMILL.DEV PLATFORM MISMATCH ⚠️ **CRITICAL**

**The Problem:**

The architecture assumes Windmill.dev is a full application platform. It is not. Windmill is a **workflow orchestration tool** with script execution capabilities. This creates fundamental misalignments.

#### 2.2.1 Windmill is NOT a Microservices Platform

**Current Assumption:** Each domain (`/lifecycle`, `/provider`, `/offer`, etc.) is treated as a separate "service" with enforced boundaries.

**Reality:**
- Windmill Scripts and Flows all run in a **single workspace**
- There are no network boundaries between domains
- Any Script can call any other Script
- The "Hub and Spoke" rule is a **convention**, not an enforcement
- There's no way to prevent `/provider` from directly calling `/offer`

**Implications:**
- Domain boundaries rely entirely on developer discipline
- No automated enforcement of architectural rules
- Testing in isolation is difficult
- Versioning and deployment are workspace-wide

#### 2.2.2 Windmill is NOT a Database Platform

**Current Assumption:** `/lifecycle` domain will be implemented as Go/Rust Scripts providing transactional database operations.

**Reality:**
- Windmill Scripts are **ephemeral**, stateless executions
- Windmill does not manage databases
- A Script must connect to an **external database** on every execution
- Connection pooling is not built-in
- Transaction management across multiple Script calls is manual

**Implications:**
- You need to deploy and manage PostgreSQL (or equivalent) **separately**
- Scripts need database connection configuration
- Need connection pooling layer (PgBouncer, RDS Proxy)
- Transaction boundaries must be carefully designed
- Database is **outside** Windmill's control plane

#### 2.2.3 Windmill Flow Complexity Limits

**Current Assumption:** Complex `/case` orchestration flows can be implemented as Windmill Flows.

**Reality:**
- Windmill Flows are **visual workflow editors**
- Great for 5-10 step linear processes
- Become unwieldy for 30+ step processes with complex branching
- Error handling, retries, and compensation logic clutters visual flows
- Difficult to version control complex visual flows effectively

**Implications:**
- Complex orchestration might be better as code (Temporal, Camunda, Conductor)
- Or implement complex flows as Python/TypeScript Scripts (losing visual benefits)
- Visual flows are great for documentation but limited for complexity

#### 2.2.4 Windmill Resource Limits

**Questions to Answer:**
- How many Scripts/Flows will this architecture create? (Potentially 200-500+)
- What are Windmill's workspace limits?
- What are execution time limits per Script?
- What are concurrency limits?
- What happens if you need multiple workspaces?
- Can workspaces call across boundaries? (Yes, but adds complexity)

**Recommendation:**

**You must make a fundamental architectural choice:**

**Option A: Windmill-Heavy Architecture**

```
- All domains implemented as Windmill Scripts/Flows
- PostgreSQL deployed separately (RDS, self-hosted, etc.)
- /lifecycle Scripts connect to PostgreSQL directly
- Simple infrastructure (Windmill + PostgreSQL)
- Reliance on Windmill for ALL orchestration and execution
```

**Pros:** Simpler infrastructure, unified platform, single deployment model
**Cons:** Limited by Windmill's capabilities, boundaries are conventional, hard to scale independently

**Option B: Hybrid Architecture** ← **RECOMMENDED**

```
- /case domain: Windmill Flows (orchestration layer)
- /lifecycle, /provider, /offer, /optimisation: FastAPI/Go microservices
- /service: API Gateway (Kong, Tyk, or FastAPI)
- Windmill Flows call microservice APIs
- Microservices own their databases
```

**Pros:** Best tool for each job, clear boundaries, independent scaling, easier testing
**Cons:** More complex infrastructure, need container orchestration (K8s, ECS)

**Option C: Minimal Windmill**

```
- Windmill only for /case/ingestion (webhook receivers)
- Temporal.io or Camunda for orchestration
- All domains as microservices
- Windmill becomes an adapter layer
```

**Pros:** Industrial-strength orchestration, proven at scale
**Cons:** Most complex infrastructure, steeper learning curve

**My Strong Recommendation: Option B (Hybrid)**

Use Windmill for what it's excellent at (visual orchestration of high-level processes) and build the complex domain logic as proper services. This gives you:

- Visual process documentation (/case Flows)
- Proper service boundaries (microservices)
- Independent scaling (critical for /provider domain)
- Technology choice per domain (Go for /lifecycle, Python for /offer)
- Standard deployment and testing practices

### 2.3 INFRASTRUCTURE LAYER UNDEFINED ⚠️ **CRITICAL**

**The Problem:**

The architecture describes 7 domains but doesn't show the infrastructure that hosts them.

**What's Missing:**

1. **Deployment Architecture**
   - Where do Windmill Scripts run? (Windmill Cloud? Self-hosted?)
   - Where does PostgreSQL run? (RDS? Self-hosted?)
   - Where do other services run? (If hybrid architecture)
   - Container orchestration? (Kubernetes, ECS, Cloud Run?)

2. **Networking**
   - How do external systems call into the platform? (API Gateway?)
   - How do domains communicate? (HTTP, gRPC, message queue?)
   - VPC architecture, subnets, security groups?

3. **Supporting Services**
   - Authentication service (Keycloak, Auth0, custom?)
   - Secret management (Vault, AWS Secrets Manager?)
   - Message broker (if event-driven: Kafka, RabbitMQ, NATS?)
   - Caching layer (Redis, Memcached?)
   - Search engine (Elasticsearch, for offer matching?)
   - File storage (S3, for documents?)

4. **Observability Stack**
   - Logging platform (Datadog, OpenSearch, Loki?)
   - Metrics (Prometheus, CloudWatch?)
   - Tracing (Jaeger, Zipkin, Datadog APM?)
   - Dashboards (Grafana?)

5. **Development and Deployment**
   - CI/CD pipelines (GitHub Actions, GitLab CI?)
   - Infrastructure as Code (Terraform, Pulumi?)
   - Environment strategy (dev, staging, production)
   - Database migration pipelines

**Recommendation:** Create an infrastructure architecture document showing:
- Deployment diagram (C4 model level 2-3)
- Technology choices for each infrastructure component
- Network architecture
- Deployment and CI/CD strategy

### 2.4 EVENT ARCHITECTURE IS CONFUSED

**The Problem:**

The spec describes two different event models:

1. **"Lean State Transition" events** from `/lifecycle` (fire-and-forget webhooks)
2. **"Immutable history" and "temporal projections"** within `/lifecycle`

**Questions:**

- Are events the source of truth (Event Sourcing)?
- Or is state the source of truth, with events as notifications?
- If events are "best effort," what happens when they're lost?
- How do you rebuild state from history?
- How do you ensure consistency between state and events?

**Current Spec Says:**

> "This is a best-effort system. We consciously accept the minimal (e.g., <0.01%) risk of a lost event"

**Problem:** For financial/contractual data, even 0.01% data loss is problematic.

**Two Clear Options:**

**Option 1: Event Sourcing** (Events are Truth)
```
- All state changes are stored as immutable events
- Current state is derived by replaying events
- Events are the source of truth, database is a projection
- Guarantees consistency, perfect audit trail
- More complex implementation (need event store)
```

**Option 2: State-First** (Database is Truth)
```
- PostgreSQL state is source of truth
- Events are notifications (can be lost without data loss)
- Simpler implementation
- Events enable async reactions but are optional
- Lost events only affect downstream notifications
```

**Recommendation:**

For SwitchUp's use case, **State-First is more pragmatic** initially:

- Store authoritative state in PostgreSQL
- Emit events for async reactions (notifications, analytics)
- Events are "best effort" notifications, not source of truth
- Later, consider event sourcing for specific high-value entities (Contract?)

### 2.5 AI CAPABILITIES ARE ASPIRATIONAL

**The Problem:**

The architecture describes AI capabilities that are bleeding-edge or don't exist reliably at scale.

**Examples from the Spec:**

1. **Self-Healing RPA Bots** (`/provider` domain):
   > "Adaptive AI agents automatically detect interface changes, attempt automated script repair or adaptation"

   **Reality:** This doesn't exist reliably. Current state-of-the-art:
   - Computer vision + LLMs can navigate simple UIs
   - But "self-healing" automation that adapts to arbitrary UI changes?
   - This is active research, not production-ready at scale

2. **Autonomous AI Planning Agents** (`/case` domain):
   > "AI planning agents decompose high-level goals into actionable task sequences, dynamically replan to handle exceptions"

   **Reality:** Agentic AI orchestration is experimental:
   - Works for constrained domains with clear boundaries
   - Unreliable for open-ended, multi-step business processes
   - Requires extensive prompt engineering and guardrails
   - High risk of unexpected behavior

3. **Generative Offer Ontology** (`/offer` domain):
   > "Generative AI models analyze diverse examples and automatically generate the necessary schema extensions"

   **Reality:** LLMs can propose schemas, but:
   - Require significant human validation
   - Prone to inconsistencies across generations
   - Hard to ensure logical consistency
   - Better as human augmentation, not automation

**Recommendation:**

**Be honest about AI maturity in the spec:**

1. **Phase 1 (Now):**
   - LLM-based extraction and classification (/provider extraction)
   - LLM-based summarization and customer support (/service)
   - Traditional ML for optimization (cost calculations, churn prediction)
   - Deterministic Playbooks for all processes

2. **Phase 2 (6-12 months):**
   - Agentic AI for simple, well-defined processes (document classification)
   - A/B testing Playbook vs Agent implementations
   - Human-in-the-loop for all AI decisions

3. **Phase 3 (12-24 months):**
   - Autonomous agents for specific high-value processes
   - Self-healing automation for select provider integrations
   - Generative capabilities with strong validation

**Specify clear criteria for Playbook vs Agent:**

```
Use Playbook when:
- Process is well-defined and deterministic
- Failure modes are understood
- Compliance/auditability is critical
- Performance/cost is constrained

Use Agent when:
- Process requires contextual judgment
- Many exception paths exist
- Process logic evolves rapidly
- Cost of human intervention is high
```

---

## Part 3: Significant Gaps

### 3.1 Security Architecture Missing

**Critical Security Concerns:**

1. **Authentication & Authorization**
   - Who authenticates users? (Auth0, Keycloak, custom?)
   - How are JWTs issued and validated?
   - How is SSO handled?
   - How are internal service-to-service calls authenticated?

2. **API Security**
   - Rate limiting, throttling?
   - API key management?
   - OAuth 2.0 flows?
   - CORS policies?

3. **Secret Management**
   - Provider credentials (hundreds of API keys)
   - Database credentials
   - Encryption keys
   - How are secrets rotated?

4. **Data Protection**
   - Encryption at rest (database, file storage)
   - Encryption in transit (TLS everywhere)
   - PII handling (pseudonymization, anonymization)
   - GDPR compliance (right to deletion, data portability)

5. **Audit Logging**
   - Security-relevant events (login, access, deletion)
   - Tamper-proof audit trail
   - Retention policies

**Recommendation:** Create a security architecture document covering all above points.

### 3.2 Error Handling and Resilience

**Missing Patterns:**

1. **Retry Policies**
   - Exponential backoff for provider API calls
   - Idempotency keys for all mutations
   - Circuit breakers for failing dependencies

2. **Compensation Logic**
   - What happens when a multi-step process fails mid-way?
   - How do you roll back state changes?
   - Saga pattern for distributed transactions?

3. **Fallback Strategies**
   - What happens when /lifecycle is down?
   - What happens when /provider integration fails?
   - Graceful degradation paths?

4. **Timeouts**
   - Maximum execution time per process
   - Timeouts for all external calls
   - Dead letter queues for failed messages

**Recommendation:** Add an "Error Handling and Resilience Patterns" section to each domain spec.

### 3.3 Operational Monitoring

**Missing Specifications:**

1. **SLOs (Service Level Objectives)**
   - What is acceptable latency for /lifecycle queries?
   - What is acceptable availability for /case processes?
   - What is acceptable error rate for /provider integrations?

2. **Business Metrics**
   - Contract activation success rate
   - Average savings per switch
   - Time to process price increase
   - Automation rate (manual vs automated tasks)

3. **Alerting**
   - When to page on-call engineer
   - When to create a ticket
   - When to auto-remediate

4. **Dashboards**
   - Real-time system health
   - Business KPIs
   - Cost monitoring

**Recommendation:** Define SLOs for each domain and specify monitoring/alerting strategy.

### 3.4 Provider Domain Complexity Underestimated

**The Reality of Provider Integrations:**

The spec describes a "Router Pattern" (try API, fallback to RPA, fallback to email). This is sound in theory but:

1. **Maintenance Burden**
   - Each provider might need 3+ implementations (API v1, API v2, RPA, Email)
   - Providers change APIs with little notice
   - RPA scripts break with every UI change
   - This multiplies quickly (100 providers × 3 methods = 300 integrations)

2. **The "Knowledge Graph" is Underspecified**
   - Described as "likely Neo4j or relational DB"
   - But no schema, no examples, no data model
   - How does it integrate with Windmill?
   - Is it a separate service? (If so, need microservices architecture)

3. **RPA Complexity**
   - Requires specialized tools (Playwright, Selenium, or commercial RPA)
   - Needs headless browser infrastructure
   - Captchas and anti-bot measures
   - Very brittle, high maintenance

**Recommendation:**

1. **Start API-First:**
   - Prioritize providers with APIs
   - Build solid API client library pattern
   - Only use RPA for critical providers without APIs

2. **Knowledge Graph as Separate Service:**
   - If using Neo4j, deploy as separate service
   - Expose via REST/GraphQL API
   - /provider domain calls this service
   - This implies microservices architecture

3. **Dedicated Provider Integration Team:**
   - This is a massive ongoing effort
   - Needs specialized skills (RPA, web scraping, API integration)
   - Consider building tooling to make integration creation faster

### 3.5 Offer Domain Ontology Complexity

**The Universal Service Ontology is ambitious:**

The spec describes a sophisticated model (Entities, Elements, Policies, Contexts) that can handle Energy, Telco, Insurance, and future markets.

**Concerns:**

1. **Complexity Trade-off**
   - Energy tariffs are complex (time-of-use, bonuses, guarantees)
   - Telco offers are different (data limits, throttling, roaming)
   - Insurance is different again (coverage, deductibles, riders)
   - Can ONE ontology handle all three without becoming overly abstract?

2. **Maintenance Burden**
   - Who maintains the ontology?
   - How often does it evolve?
   - How do you ensure consistency across markets?

3. **Performance**
   - "Evaluate every offer with full policy resolution"
   - For 1000s of offers per query?
   - Need caching, precomputation, or materialized views

**Recommendation:**

1. **Start with One Market:**
   - Build ontology for Energy first
   - Validate it works for real offers
   - Identify abstraction patterns
   - Extend to Telco/Insurance later

2. **Performance Strategy:**
   - Precompute offer eligibility where possible
   - Cache evaluated offers per user segment
   - Use materialized views for common queries
   - Consider search engine (Elasticsearch) for offer matching

3. **Governance:**
   - Assign domain experts per market (Energy expert, Telco expert)
   - Formal review process for ontology changes
   - Version the ontology itself

### 3.6 Configuration Governance Burden

**The Routing Manifest system is powerful but:**

**Cognitive Load:**
- Product managers need to understand:
  - Process names, variant paths, strategy capabilities
  - Input JSON schemas, experiment tracking
- This is a lot of technical detail for non-engineers

**Configuration Drift:**
- With federated configs, how do you ensure consistency?
- How do you deprecate old variants?
- How do you detect unused or dead code?

**Testing:**
- How do you test a new variant before production?
- The spec doesn't mention staging/testing environments
- How do you do canary deployments safely?

**Recommendation:**

1. **Tooling for Non-Engineers:**
   - Build a UI for managing process manifests
   - Visual experiment dashboard
   - Variant performance comparison

2. **Automated Cleanup:**
   - Detect variants with zero traffic
   - Automated deprecation warnings
   - Scheduled cleanup of old experiments

3. **Testing Strategy:**
   - Staging environment with copy of production routing config
   - Ability to override routing for specific test users
   - Automated integration tests per variant

### 3.7 Service Domain Too Thin

**Missing Responsibilities:**

The `/service` domain is described as "thin" but critical concerns are undefined:

1. **Who Owns Authentication?**
   - Spec says /service validates tokens
   - But who issues tokens?
   - Where's the auth service?

2. **API Gateway Concerns:**
   - Rate limiting, throttling
   - Request/response transformation
   - API versioning strategy
   - CORS handling
   - Request validation

3. **The Intent Mapping:**
   - User intent names map to process names
   - Where is this mapping maintained?
   - Hardcoded? Configuration?

**Recommendation:**

1. **Separate Auth Service:**
   - Use Keycloak, Auth0, or equivalent
   - /service delegates to auth service
   - Document the auth flow

2. **Use API Gateway:**
   - Kong, Tyk, AWS API Gateway, or Traefik
   - Handles cross-cutting concerns (rate limiting, logging)
   - /service capabilities are backend services

3. **Intent Catalog:**
   - Create an "Intent Catalog" similar to Business Intent Catalog
   - Maps user/agent intents to process names
   - Version controlled, validated

### 3.8 Missing Operational Domains

**The spec mentions but doesn't specify:**

1. **Operations/Support Domain:**
   - Task management for human agents
   - Escalation workflows
   - Internal tooling

2. **Data/Analytics Domain:**
   - Data warehouse strategy
   - ETL pipelines
   - Business intelligence
   - ML model training and deployment

3. **Compliance/Legal Domain:**
   - GDPR data subject requests
   - Right to deletion, data portability
   - Consent management
   - Regulatory reporting

**Recommendation:** Add specifications for these domains or explicitly defer them to Phase 2.

---

## Part 4: Recommendations

### 4.1 Immediate Actions (Must Address Before Implementation)

#### 1. Resolve the Windmill Architecture Question

**Decision Point:** Windmill-Heavy, Hybrid, or Minimal Windmill?

**My Recommendation: Hybrid Architecture**

```
┌─────────────────────────────────────────────────┐
│              External Systems                   │
│   (Users, Providers, Payment, Email, etc.)      │
└─────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────┐
│            API Gateway (Kong/Tyk)               │
│          - Authentication (JWT validation)      │
│          - Rate Limiting, CORS                  │
└─────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────┐
│         /service Domain (FastAPI)               │
│     - User intent interpretation                │
│     - API request/response handling             │
└─────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────┐
│    /case Domain (Windmill Flows)                │
│   - Ingestion flows (webhooks)                  │
│   - Process dispatcher                          │
│   - Orchestration flows (playbooks/agents)      │
└─────────────────────────────────────────────────┘
                      ↓
        ┌─────────────┼─────────────┐
        ↓             ↓             ↓
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│  /lifecycle │ │  /provider  │ │   /offer    │
│   (FastAPI/ │ │  (FastAPI/  │ │  (FastAPI)  │
│    Go)      │ │   Python)   │ │             │
│             │ │             │ │             │
│  PostgreSQL │ │  Neo4j?     │ │  PostgreSQL │
└─────────────┘ └─────────────┘ └─────────────┘
        ↓
┌─────────────────────────────────────────────────┐
│        /optimisation Domain (Python)            │
│     - Stateless calculation engine              │
│     - ML models, decision algorithms            │
└─────────────────────────────────────────────────┘
```

**Why:**
- Windmill excels at visual orchestration (/case flows)
- Complex domains benefit from proper service architecture
- Clear deployment and testing boundaries
- Independent scaling (critical for /provider)
- Technology choice per domain (Go for /lifecycle, Python for /offer)

#### 2. Create Complete Database Architecture

**Required Deliverable:** Database Architecture Document

**Contents:**
- ER diagrams for all /lifecycle entities
- PostgreSQL schema definitions
- Indexing strategy
- Migration scripts (Flyway/Liquibase)
- Connection pooling strategy (PgBouncer)
- Backup/recovery procedures
- Performance projections

**Example Schema Snippet:**

```sql
CREATE TABLE contracts (
    contract_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(user_id),
    provider_id UUID NOT NULL REFERENCES providers(provider_id),
    offer_id UUID REFERENCES offers(offer_id),
    status VARCHAR(50) NOT NULL,
    activation_date TIMESTAMP,
    termination_date TIMESTAMP,
    terms JSONB NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE contract_history (
    history_id BIGSERIAL PRIMARY KEY,
    contract_id UUID NOT NULL REFERENCES contracts(contract_id),
    event_type VARCHAR(100) NOT NULL,
    event_timestamp TIMESTAMP NOT NULL,
    event_data JSONB NOT NULL,
    actor VARCHAR(255),
    trace_id VARCHAR(100)
);

CREATE INDEX idx_contract_history_contract_id ON contract_history(contract_id);
CREATE INDEX idx_contract_history_trace_id ON contract_history(trace_id);
```

#### 3. Create Infrastructure Architecture Document

**Required Deliverable:** Infrastructure & Deployment Architecture

**Contents:**
- C4 model diagrams (System Context, Container, Component)
- Technology choices with rationale
- Network architecture (VPCs, subnets, security groups)
- Supporting services (auth, secrets, caching, messaging)
- CI/CD pipeline design
- Environment strategy (dev, staging, prod)
- Cost estimates

#### 4. Add Security Architecture

**Required Deliverable:** Security Architecture Document

**Contents:**
- Authentication and authorization model
- Secret management strategy
- API security (rate limiting, OAuth)
- Data protection (encryption, PII handling)
- GDPR compliance approach
- Audit logging strategy
- Security monitoring and incident response

### 4.2 Short-Term Actions (Address in Parallel with Development)

#### 5. Refine AI Integration Strategy

**Action:** Create an AI Maturity Roadmap

**Contents:**
- Phase 1: Deterministic playbooks + LLM extraction
- Phase 2: Hybrid (playbook with AI steps)
- Phase 3: Full agentic automation
- Clear criteria for playbook vs agent
- Testing and validation strategies
- Cost management (LLM API costs can be significant)

**Include:** Realistic expectations document acknowledging current AI limitations.

#### 6. Add Error Handling Patterns

**Action:** Document standard resilience patterns for each domain

**Contents:**
- Retry policies (exponential backoff, max retries)
- Circuit breakers (Hystrix pattern)
- Timeouts and deadlines
- Idempotency requirements
- Saga patterns for distributed transactions
- Fallback strategies

#### 7. Clarify Provider Domain Implementation

**Action:** Create detailed Provider Integration Guide

**Contents:**
- API client template and best practices
- RPA bot development framework
- Knowledge graph schema (if using Neo4j)
- Integration testing framework
- Provider onboarding checklist
- Maintenance and monitoring approach

**Be Realistic:** Start with API-first approach. RPA only for critical providers without APIs.

#### 8. Add Monitoring and Observability

**Action:** Define SLOs and Monitoring Strategy

**Contents:**
- SLOs per domain (latency, availability, error rate)
- Business metrics to track
- Alerting rules and escalation policies
- Dashboard requirements
- Log aggregation and search
- Distributed tracing setup
- Cost monitoring

#### 9. Specify Data and Analytics Strategy

**Action:** Data Architecture Document

**Contents:**
- Data warehouse technology (BigQuery, Snowflake, Redshift?)
- ETL/ELT pipeline design
- Analytics data model (star schema, etc.)
- BI tool integration (Metabase, Tableau, Looker?)
- ML model training infrastructure
- Data governance and quality

#### 10. Governance and Lifecycle Management

**Action:** Establish Architectural Governance

**Contents:**
- Architecture Review Board (ARB) charter
- Architecture Decision Record (ADR) template
- Process for proposing new capabilities
- Deprecation and versioning policies
- Experiment lifecycle automation
- Regular architecture review schedule

### 4.3 Long-Term Considerations

#### 11. Evaluate and Optimize Offer Domain

**Recommendation:** Start simple, evolve the ontology

- Build for Energy market first
- Validate with real offers
- Extend to Telco, then Insurance
- Don't over-engineer upfront
- Performance testing with 10,000+ offers

#### 12. Consider Event-Driven Architecture Evolution

**Future Direction:** If scaling challenges emerge

- Introduce message broker (Kafka, NATS)
- Move to true event-driven architecture
- Enables better decoupling and async processing
- But adds complexity—defer until needed

#### 13. Provider Integration Tooling

**Investment Area:** Build internal platform for provider integrations

- Template-based integration creation
- Automated testing framework
- Monitoring and alerting per integration
- Self-service for adding new providers
- This is a force multiplier for the team

#### 14. Continuous Architectural Evolution

**Ongoing Practice:**

- Quarterly architecture reviews
- Regular ADR creation
- Metrics-driven optimization
- Team feedback loops
- Keep the spec as a living document

---

## Part 5: Architectural Decision Records (Immediate Decisions Needed)

### ADR-001: Windmill Architecture Pattern

**Status:** MUST DECIDE
**Decision:** [Windmill-Heavy | Hybrid | Minimal Windmill]
**Rationale:** [To be filled after team discussion]
**Consequences:** Affects all implementation decisions

### ADR-002: Database Technology for /lifecycle

**Status:** MUST DECIDE
**Decision:** [PostgreSQL | Other]
**Rationale:** Transactional guarantees, JSON support, mature ecosystem
**Consequences:** Schema design, migration strategy, hosting costs

### ADR-003: Event Architecture Pattern

**Status:** MUST DECIDE
**Decision:** [State-First with Events | Event Sourcing]
**Rationale:** State-first is simpler for MVP, can evolve later
**Consequences:** Data consistency model, audit trail implementation

### ADR-004: Authentication Provider

**Status:** MUST DECIDE
**Decision:** [Auth0 | Keycloak | AWS Cognito | Custom]
**Rationale:** [To be determined]
**Consequences:** Integration effort, ongoing costs, feature set

### ADR-005: Container Orchestration (if Hybrid)

**Status:** MUST DECIDE (if choosing Hybrid architecture)
**Decision:** [Kubernetes | ECS | Cloud Run | None]
**Rationale:** [To be determined]
**Consequences:** Deployment complexity, operational overhead, scaling model

---

## Part 6: Risk Assessment

### High-Risk Areas

| Risk | Impact | Likelihood | Mitigation |
|------|--------|-----------|-----------|
| Windmill platform limitations discovered mid-implementation | HIGH | MEDIUM | Make architecture decision now with prototype |
| Provider integrations more brittle than expected | HIGH | HIGH | Start with API-first, build strong monitoring |
| Offer ontology too complex for real-world offers | MEDIUM | MEDIUM | Start with single market, validate early |
| AI capabilities don't meet expectations | MEDIUM | MEDIUM | Set realistic expectations, human-in-loop |
| Database performance issues at scale | HIGH | LOW | Design for performance, load test early |
| Team cannot maintain complex configuration system | MEDIUM | MEDIUM | Build tooling, simplify where possible |

### Critical Success Factors

1. **Clear architectural decision on Windmill usage** (immediately)
2. **Database architecture defined and validated** (immediately)
3. **Infrastructure plan with cost estimates** (before implementation)
4. **Team alignment on AI maturity roadmap** (avoid over-promising)
5. **Strong monitoring and observability from day one** (non-negotiable)

---

## Conclusion

### The Good News

This architectural specification demonstrates **excellent strategic thinking**. The domain boundaries are clean, the separation of concerns is sound, and the vision for operational scalability through automation and AI is compelling.

### The Critical Issue

There is a **fundamental gap between the conceptual architecture and its practical implementation on Windmill.dev**. Windmill is being treated as a full application platform when it is fundamentally a workflow orchestration tool.

### The Path Forward

**Before writing any implementation code, you MUST:**

1. ✅ **Decide:** Windmill-Heavy, Hybrid, or Minimal Windmill?
2. ✅ **Define:** Complete database architecture
3. ✅ **Design:** Infrastructure and deployment architecture
4. ✅ **Document:** Security, error handling, and monitoring strategies

**My Recommendation:**

Adopt a **Hybrid Architecture** where:
- Windmill excels at visual process orchestration (/case domain)
- Core domains are proper microservices (FastAPI, Go)
- This gives you the best of both worlds: visual workflows + proper engineering

**Once these foundations are solid, this architecture can absolutely achieve the operational scalability goals.** The conceptual model is sound. The implementation path needs clarity.

---

## Next Steps

1. **Review this document with the team**
2. **Make the critical architectural decisions (ADRs)**
3. **Create the missing architecture documents** (database, infrastructure, security)
4. **Build a prototype** (one simple flow end-to-end) to validate the architecture
5. **Iterate the specification** based on prototype learnings
6. **Proceed with confidence** once foundations are solid

---

**End of Review**

---

*This review was conducted with the goal of ensuring SwitchUp builds a truly scalable, maintainable, and future-proof platform. The conceptual architecture is strong. With the gaps addressed, this can become an exemplar of modern, AI-enabled, domain-driven platform engineering.*
