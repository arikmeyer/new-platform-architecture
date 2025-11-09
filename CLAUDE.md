# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a **strategic architecture specification repository** for SwitchUp's next-generation platform. It contains detailed design documents for a Domain-Driven Design (DDD) architecture aimed at achieving operational scalability through automation and AI integration. This is **not an active codebase** with executable code—it's a specification and planning repository.

## Core Business Context

SwitchUp is a subscription contract monitoring and optimization service covering:
- **Energy** (electricity, gas)
- **Telco** (broadband, mobile)
- **Insurance** (car, home, liability)

The primary challenge is **operational scalability**—eliminating manual processes in workflow execution, data extraction, and customer service to handle millions of contracts efficiently.

## Architectural Philosophy: "The Explicit Orchestrator"

The system follows a "Brain, Spine, and Limbs" model with three key principles:

1. **Orchestration is Explicit**: Business processes are readable, version-controlled artifacts
2. **Domains Provide Tools**: Modular domains provide pure, stateless, reusable capabilities
3. **State is Sacred**: Authoritative business state is managed by a dedicated transactional domain

## Domain Architecture

The system is organized into **7 primary domains**, each with exclusive mandates:

### Layer 0: The Spine (System of Record)
- **/lifecycle/**: Exclusive transactional owner of all core business entity state (Contract, User, Task, etc.)
  - Provides **Business Intent Catalog**: atomic state-change commands (e.g., `ConfirmActivation`, `ReportPriceIncrease`)
  - Implements **Command Handlers** for mutations and rich **Query capabilities** for reads
  - Generates "Lean State Transition" events via fire-and-forget webhooks
  - **Golden Rule**: Calls no one; everyone calls it for truth

### Layer 1: The Limbs (Tool Domains)
- **/provider/**: Provider interactions
  - **Inbound**: Parse provider emails, scrape provider portals, extract provider messages/documents
  - **Outbound**: Execute actions on provider systems (RPA bots: login, submit forms, meter readings, cancellations)
  - Stores: portal credentials, RPA execution logs, scraping results (JSONB)

- **/offer/**: Market data and tariff knowledge
  - Ingestion pipelines (fetch tariffs from multiple sources)
  - Normalization (canonical offer data model)
  - Matching rules (find relevant offers for contracts)
  - Stores: normalized offers, ingestion metadata, matching rules, market data (JSONB)

- **/optimisation/**: Decision engine
  - Savings calculations (compare current vs alternative tariffs)
  - ML models (churn risk, user behavior prediction)
  - Recommendation generation (rank alternatives)
  - A/B test routing strategies
  - Stores: ML model outputs, calculation results, A/B configurations, projections (JSONB)

- **/service/**: User interactions
  - **Inbound**: Classify user requests via email/chat/phone
  - **Outbound**: Respond to users (notifications, status updates, replies)
  - **Self-service dashboard**: Provider-agnostic view of contracts, usage, savings
  - Stores: conversation memory, LLM classification state, user session data (JSONB)
  - **Strict boundary**: Only calls /case domain, never direct to tools

- **/growth/**: User acquisition
  - Content generation (blog posts, landing pages using AI)
  - Funnel analytics (track conversion, optimize flows)
  - Personalized onboarding (RL/multi-armed bandits)
  - Stores: content generation results, funnel analytics, personalization state (JSONB)

### Layer 2-3: The Brain (Process Orchestrator)
- **/case/**: Exclusive owner of all end-to-end business processes
  - **Three-tier structure**:
    1. `/case/ingestion/`: Entry points for external triggers ("Airlocks")
    2. `/case/meta/`: Central nervous system with Process Dispatcher and Strategy Resolver
    3. `/case/<business_purpose>/`: Versioned implementation flows (Playbooks/Agents)
  - Orchestrates calls across multiple tool domains
  - Configuration-driven routing enables A/B testing and gradual rollouts

## Key Implementation Patterns

### The Windmill Platform
All capabilities are implemented as **Windmill Flows** (orchestration) or **Scripts** (atomic functions):
- Naming: `/domain/subdomain/capability-name_{playbook|agent}_v[N]`
- **Playbook**: Deterministic, rule-based process
- **Agent**: Dynamic, goal-oriented AI process

### Hub and Spoke Communication
Strict rules prevent domain coupling:
- **/lifecycle** (Layer 0): Calls no one
- **Tool domains** (Layer 1): Can only call /lifecycle
- **/case** (Layer 2-3): Can call any tool domain and /lifecycle
- No lateral communication between tool domains

### Configuration-Driven Agility
- Federated routing configuration in Git (`routing.json` files)
- Each process manifest defines: description, input schema, strategy, variants
- Strategy capabilities enable A/B testing, percentage rollouts, user-based routing
- **Implementation immutability**: Clone and version rather than modify

### Observability Requirements
- **Trace ID propagation**: Every ingestion flow generates unique ID
- **Structured logging**: All logs must be JSON with Trace ID
- **Context propagation**: Trace ID flows through entire call chain
- Central logging platform (Datadog/OpenSearch) for distributed tracing

## Document Structure

### `/Background/`
- `Business_Context.md`: Core business model and scalability challenge
- `Problem_Spaces.md`: Detailed problem definitions by domain (operational scaling challenge, offer modeling, workflow orchestration, etc.)

### `/Architecture/Rationale/`
Strategic reasoning documents:
- `Big-Picture.md`: "Brain, Spine, and Limbs" model overview
- `Overview-on-DDD-Capabilities-and-Business-Intents.md`: Hierarchy of abstraction (DDD → Capabilities → Business Intents)
- `AI-Operating-System.md`: AI integration strategy
- Comparison documents (DDD vs Microservices, vs Monolith)

### `/Domains/`
Detailed specifications for each domain:
- `Capability-Architecture.md`: Complete three-tier implementation pattern
- `Lifecycle/Lifecycle-Domain-The-System-of-Record.md`: Full Business Intent Catalog
- `Case/Case-Domain.md`: Process orchestration structure
- `Case/Process-Dispatcher.md`: Meta-routing mechanism
- `Case/Routing-Configuration.md`: Configuration system details
- Individual domain specs: `Provider-Domain.md`, `Offer-Domain.md`, `Optimisation-Domain.md`, `Service-Domain.md`, `Growth-Domain.md`

## Working with This Repository

### When Reading Documents
- Start with `Architecture/Rationale/Big-Picture.md` for system overview
- Review `Domains/Capability-Architecture.md` for implementation patterns
- Refer to specific domain docs for detailed capability specifications

### When Proposing Changes
- Maintain domain boundaries—never blur responsibilities
- Follow naming conventions for new capabilities
- Ensure new capabilities fit the Hub and Spoke communication rules
- Consider configuration-driven approaches before code changes
- Validate against the "What DOES NOT Belong" anti-patterns in each domain spec

### Key Terminology
- **Capability**: A public-facing, reusable business function (Script or Flow)
- **Business Intent**: Atomic state-change command in /lifecycle domain
- **Command Handler**: Implementation of a Business Intent (mutation)
- **Query**: Read operation that returns authoritative state
- **Aggregate Offer**: Fully evaluated offer with applied policies and context
- **Process Dispatcher**: Central router in /case/meta/ that executes strategy-driven process selection
- **Trace ID**: Unique identifier propagated through entire distributed process execution
