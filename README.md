# SwitchUp DDD Architecture Specification

Strategic architecture specification for SwitchUp's next-generation platform following Domain-Driven Design principles.

## Overview

This repository contains detailed design documents for a Domain-Driven Design (DDD) architecture aimed at achieving operational scalability through automation and AI integration. This is **not an active codebase** with executable code—it's a specification and planning repository.

### Key Components

- **7-domain architecture**: Lifecycle, Provider, Offer, Optimisation, Service, Growth, Case
- **"Brain, Spine, and Limbs" orchestration model**: Clear separation between state management, capabilities, and orchestration
- **Hub and Spoke communication patterns**: Prevents domain coupling
- **Configuration-driven routing**: Enables A/B testing and gradual rollouts
- **Complete business intent catalog**: Atomic state-change commands for all business operations

## Quick Start

Start with these documents to understand the architecture:

1. [Architecture/Rationale/Big-Picture.md](Architecture/Rationale/Big-Picture.md) - System overview
2. [Domains/Capability-Architecture.md](Domains/Capability-Architecture.md) - Implementation patterns
3. [CLAUDE.md](CLAUDE.md) - Repository guide and terminology

## Repository Structure

```
├── Architecture/
│   └── Rationale/          # Strategic reasoning documents
├── Background/              # Business context and problem spaces
├── Domains/                 # Detailed domain specifications
│   ├── Case/               # Process orchestration domain
│   └── Lifecycle/          # System of record domain
└── CLAUDE.md               # Repository guide
```

## License

Proprietary - SwitchUp
