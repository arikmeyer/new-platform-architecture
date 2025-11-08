# Big Picture

## The SwitchUp Architecture: A "Brain, Spine, and Limbs" Model

At the highest level, our system is not a collection of services, but a model of a living organism, designed for intelligence, stability, and coordinated action. It consists of three primary layers.

### 1. The Spine: The /lifecycle Domain (The System of Record)

- **What it is:** The single, unyielding source of truth for all core business data (Contracts, Users, Tasks). It is the bedrock foundation of the entire system.
- **Its Personality:** It is a **"Guardian."** It is conservative, transactional, and relentlessly consistent. Its primary job is to protect the integrity of the business's core data.
- **How it Works:** It exposes a granular, intent-driven API called the **Business Intent Catalog**. These are the atomic "verbs" of the business (ConfirmActivation, ApplyCredit). It guarantees that no action can be taken that would violate the core rules of the business's state. It also provides rich, temporal "Projection" queries that allow other domains to understand an entity's past, present, and deterministic future.
- **The Golden Rule:** It calls no one. Everyone calls it for the truth.

### 2. The Limbs: The "Tool" Domains (The Business Capabilities)

- **What they are:** A collection of independent, specialized, and pure "tool" domains.
  - **/provider/:** The expert on interacting with providers.
  - **/service/:** The expert on user and agent interaction.
  - **/offer/:** The expert on modeling the market.
  - **/optimisation/:** The expert on making optimal choices.
  - **/growth/:** The expert on user acquisition.
- **Their Personality:** They are **"Specialized Experts."** Each one is a master of its own domain. /provider knows everything about Vattenfall's API but nothing about user experience. /service knows everything about user intents but nothing about provider APIs.
- **How they Work:** They provide a library of reusable, stateless **Capabilities** (the "tools"). They are forbidden from talking to each other directly. To perform their work, they can query the "Spine" (/lifecycle) for the state of the world.
- **The Golden Rule:** A Limb only performs its specialized function. It never orchestrates a multi-domain process.

### 3. The Brain: The /case Domain (The Process Orchestrator)

- **What it is:** The single, centralized orchestrator for all end-to-end business processes.
- **Its Personality:** It is the **"Director"** or **"Chief Executive."** It doesn't have deep specialized knowledge, but it is the only part of the system that understands the complete "story" of how to achieve a business goal.
- **How it Works:** Its capabilities are **Flows** (the "Playbooks" and "Agents"). These flows do not contain complex logic themselves. Instead, they are a sequence of calls that **orchestrate** the "Limbs" and direct the "Spine." A handle-price-increase Flow tells the Optimisation limb to think, the Service limb to talk, and the Lifecycle spine to record the final decision.
- **The Golden Rule:** It is the only domain allowed to make calls across multiple "Limb" domains to execute a process. It is the single source of truth for all business process logic.

### How It All Ties Together: The Flow of Work

This architecture creates a clean, top-down flow of command for every action.

1. **A Trigger Occurs:** An external event (like a user click or a provider email) is received by a dedicated **Ingestion Flow** in the /case domain.
2. **The Brain is Activated:** The Ingestion Flow interprets the trigger and invokes the **Process Dispatcher** (the Meta Router), a central capability within the /case domain.
3. **A Strategy is Chosen:** The Dispatcher consults its **Routing Configuration** (the "Process Manifest") to decide which specific version of the business process to run (e.g., the stable _playbook_v1 or the experimental _agent_v1). This is the heart of the system's agility and allows for A/B testing of entire business processes.
4. **The Brain Directs the Limbs:** The chosen implementation Flow (the "Playbook" or "Agent") executes. It makes a series of calls to the specialized "Limb" domains to gather information, make decisions, and interact with the world.
- *"Hey /offer domain, what's on the market?"*
- *"Hey /optimisation domain, what's the best choice?"*
- *"Hey /provider domain, execute this cancellation."*
5. **The Spine Records the Truth:** As the final step of any significant action, the "Brain" (/case) issues a precise, atomic **Command** from the Business Intent Catalog to the "Spine" (/lifecycle). The Spine validates and records the state change, creating an immutable, auditable record of the action.
6. **The System Reacts (Optional):** The Spine (/lifecycle) emits a "Lean State Transition" **Event**. This allows decoupled "sidecar" processes (like analytics or non-critical notifications) to react to the change without being part of the core orchestration.

### The Strategic Advantages of This Model

This architecture is not just a technical choice; it is a business strategy.

- **Clarity:** The business processes are explicit, readable artifacts, not hidden in code.
- **Agility:** The system is built for experimentation. New strategies and AI agents can be safely tested and deployed via configuration, without risky code changes.
- **Resilience:** The clean separation of concerns and strict communication rules dramatically reduce the "blast radius" of failures.
- **Scalability (Human & Technical):** It allows teams to work autonomously within their domains of expertise without creating system-wide dependencies.
- **AI-Readiness:** It provides a perfect, three-tiered "Cognitive Stack" of tools (Processes, Capabilities, and Commands) that allows an AI agent to operate safely and effectively.

In essence, we have moved from building a simple application to building an **intelligent, evolvable, and resilient operational platform.** This is the definitive big-picture vision.

