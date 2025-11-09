# SwitchUp DDD Architecture Specification

Strategic architecture specification for SwitchUp's next-generation platform following Domain-Driven Design principles. This repository contains **work-in-progress thoughts and evolving patterns** meant as a basis for further thinking and implementation.

## Overview

This repository contains comprehensive design documents for a Domain-Driven Design (DDD) architecture aimed at achieving operational scalability through automation and AI integration. This is **not an active codebase** with executable code—it's a specification and planning repository.

**Important**: All documents represent evolving ideas and technical explorations. They are **by no means complete or final**, but rather serve as a foundation for discussion, refinement, and implementation planning.

### Key Components

- **Domain architecture**: Lifecycle, Provider, Offer, Optimisation, Service, Growth, Case
- **"Brain, Spine, and Limbs" orchestration model**: Clear separation between state management, capabilities, and orchestration
- **Hub and Spoke communication patterns**: Prevents domain coupling
- **Configuration-driven routing**: Enables A/B testing and gradual rollouts
- **Complete business intent catalog**: Atomic state-change commands for all business operations
- **Modern tech stack exploration**: Windmill + Drizzle ORM + Zod 4 + Neon PostgreSQL

## Repository Structure

```
├── Architecture/
│   └── Rationale/                    # Strategic reasoning documents
│       ├── Big-Picture.md            # "Brain, Spine, Limbs" model
│       ├── AI-Operating-System.md    # AI integration strategy
│       ├── Overview-on-DDD-Capabilities-and-Business-Intents.md
│       ├── Domain-Driven-Design-vs-*.md  # Architecture comparisons
│       └── ...
│
├── Background/
│   ├── Business_Context.md           # Core business model
│   └── Problem_Spaces.md             # Problem definitions by domain
│
├── Domains/                          # Detailed domain specifications
│   ├── Capability-Architecture.md    # Three-tier implementation pattern
│   │
│   ├── Lifecycle/                    # System of Record (Layer 0)
│   │   ├── Lifecycle-Domain-The-System-of-Record.md
│   │   ├── Business-Intent-Catalog.md  # Complete catalog of state mutations
│   │   └── Tasks.md                  # Human-in-the-loop task system
│   │
│   ├── Case/                         # Process Orchestrator (Layer 2-3)
│   │   ├── Case-Domain.md            # Orchestration architecture
│   │   ├── Process-Dispatcher.md     # Central routing mechanism
│   │   └── Routing-Configuration.md  # Configuration-driven routing
│   │
│   ├── Provider/                     # Provider interactions (Layer 1)
│   │   └── Provider-Domain.md        # RPA bots, portal scraping
│   │
│   ├── Offer/                        # Market data (Layer 1)
│   │   └── Offer-Domain.md           # Tariff knowledge
│   │
│   ├── Optimisation/                 # Decision engine (Layer 1)
│   │   └── Optimisation-Domain.md    # Savings calculations & ML
│   │
│   ├── Service/                      # User interactions (Layer 1)
│   │   └── Service-Domain.md         # Email/chat/phone, dashboard
│   │
│   └── Growth/                       # User acquisition (Layer 1)
│       └── Growth-Domain.md          # Acquisition & activation
│
├── Technical/                        # Technical implementation patterns (WIP)
│   ├── Windmill-Drizzle-Zod-Neon-Integration-Spec.md  # ⭐ Main technical spec (draft)
│   ├── ZOD-4-DRIZZLE-2025-UPDATES.md                  # Latest patterns exploration
│   └── Naming-Convention-Strategy.md                   # Code conventions
│
├── CLAUDE.md                         # Repository guide for AI assistants
└── README.md                         # This file
```

## Technology Stack (2025)

### Core Platform
- **Orchestration**: Windmill 1.573.1+ (pre-bundling optimization, 60% performance gain)
- **Database**: Neon PostgreSQL (serverless-first, HTTP-based connections)
- **ORM**: Drizzle ORM 0.44.7 (native caching, enhanced error handling)
- **Validation**: Zod 4.1.12 (14× faster, metadata system, z.interface())
- **Integration**: drizzle-zod 0.8.3 (full Zod 4 compatibility)

### Key Technical Patterns
- **z.interface()** for business intent schemas (precise optionality control)
- **safeParse()** for Windmill scripts (structured error responses)
- **Metadata system** for AI agent context and LLM function calling
- **Native caching** with Upstash Redis (3-5× performance improvement)
- **Identity columns** for PostgreSQL primary keys
- **Generated columns** for computed values

## Document Index

### Strategic Architecture
- [Big-Picture.md](Architecture/Rationale/Big-Picture.md) - System overview and "Brain, Spine, Limbs" model
- [AI-Operating-System.md](Architecture/Rationale/AI-Operating-System.md) - AI integration strategy
- [Capability-Architecture.md](Domains/Capability-Architecture.md) - Three-tier implementation pattern

### Business Context
- [Business Context](Background/Business_Context.md) - Core business model and scalability challenge
- [Problem Spaces](Background/Problem_Spaces.md) - Detailed problem definitions by domain

### Domain Specifications
- [Lifecycle Domain](Domains/Lifecycle/Lifecycle-Domain-The-System-of-Record.md) - System of Record (Layer 0)
- [Case Domain](Domains/Case/Case-Domain.md) - Process Orchestrator (Layer 2-3)
- [Provider Domain](Domains/Provider/Provider-Domain.md) - Provider interactions (inbound/outbound)
- [Offer Domain](Domains/Offer/Offer-Domain.md) - Market data and tariff knowledge
- [Optimisation Domain](Domains/Optimisation/Optimisation-Domain.md) - Decision engine
- [Service Domain](Domains/Service/Service-Domain.md) - User interactions (inbound/outbound)
- [Growth Domain](Domains/Growth/Growth-Domain.md) - User acquisition

### Technical Implementation (Work-in-Progress)
- [Windmill-Drizzle-Zod-Neon Integration](Technical/Windmill-Drizzle-Zod-Neon-Integration-Spec.md) - Technical specification draft
- [Zod 4 & Drizzle 2025 Updates](Technical/ZOD-4-DRIZZLE-2025-UPDATES.md) - Latest patterns exploration
- [Naming Conventions](Technical/Naming-Convention-Strategy.md) - Proposed code naming strategy

**Note**: These technical documents capture current thinking on implementation approaches. They represent research and exploration, not final production specifications.

## Key Concepts

### Domain Layers

**Layer 0 (The Spine)**
- **/lifecycle**: Exclusive transactional owner of all core business state
- Provides Business Intent Catalog (atomic state-change commands)
- **Golden Rule**: Calls no one; everyone calls it for truth

**Layer 1 (The Limbs - Tool Domains)**
- **/provider**: Provider interactions (inbound: parse provider emails/portals/messages; outbound: execute actions on provider systems via RPA bots)
- **/offer**: Market data and tariff knowledge (ingestion pipelines, offer normalization, matching rules)
- **/optimisation**: Decision engine (savings calculations, ML models, recommendations, A/B test routing)
- **/growth**: User acquisition (content generation, funnel analytics, personalized onboarding)
- **/service**: User interactions (inbound: classify user requests via email/chat/phone; outbound: respond to users; provides self-service dashboard)
- **Communication**: Most tool domains call only /lifecycle; /service calls /case (external entry point)

**Layer 2-3 (The Brain - Process Orchestrator)**
- **/case**: Exclusive owner of all end-to-end business processes
- Orchestrates calls across multiple tool domains
- Configuration-driven routing for A/B testing

### Hub and Spoke Communication
- **/lifecycle** (Layer 0): Calls no one
- **Tool domains** (Layer 1): Most can only call /lifecycle; /service is special and calls /case
- **/case** (Layer 2-3): Can call any tool domain and /lifecycle
- **No lateral communication** between tool domains

## License

Proprietary - SwitchUp
