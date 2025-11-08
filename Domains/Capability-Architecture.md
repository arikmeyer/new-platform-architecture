# Capability Architecture

## Part 1: The Central Philosophy: "The Explicit Orchestrator"

Our system is architected around a central philosophy that prioritizes clarity, maintainability, and evolvability.

1. **Orchestration is Explicit:** Core business processes are not hidden or emergent. They are explicitly defined as readable, version-controlled artifacts.
2. **Domains Provide Tools:** The system is composed of modular domains that provide pure, stateless, reusable "tools" (capabilities). These domains are experts in their function but are agnostic to the business processes that use them.
3. **State is Sacred and Separate:** The authoritative record of business state is managed by a dedicated, transactional Lifecycle domain, completely decoupled from process logic.
4. **Configuration over Code for Agility:** Business agility (A/B testing, gradual rollouts, strategy changes) is managed through external configuration, not by modifying core orchestration logic.
5. **AI is an Implementation Strategy:** The architecture seamlessly accommodates both deterministic, rule-based processes (Playbooks) and dynamic, goal-oriented Agents. The choice is a versioned implementation detail, not a top-level architectural concern.

## Part 2: The Definitive Capability Atlas (The Domain Map)

The system is partitioned into a set of clear, business-aligned domains, each with an exclusive mandate.

- **/lifecycle/ (The Ledger of State):** The exclusive, transactional owner of the state and history of all core business entities (Contract, User, MeterPoint, etc.). It provides rich, computed properties on-demand.
- **/provider/ (The External Interface):** The anti-corruption layer that provides a stable, reliable interface to all chaotic external provider systems. It owns all provider-specific interaction logic.
- **/offer/ (The Market Knowledge Base):** The single source of truth for all products and tariffs available in the market. It owns the logic for ingesting, modeling, and normalizing market data.
- **/optimisation/ (The Decision Engine):** A pure, stateless "brain-as-a-service" that provides intelligent, data-driven decisions, evaluations, and recommendations.
- **/service/ (The User & Agent Service Domain):** Manages all direct interactions and provides the tools for a seamless service experience for both end-users and internal support colleagues.
- **/growth/ (The Acquisition & Activation Domain):** Responsible for attracting, onboarding, and activating new users through content and personalized experiences.
- **/case/ (The Business Process Orchestrator):** The exclusive home of orchestration. It contains the explicit, end-to-end business processes that combine the "tools" from all other domains to create business value.

## Part 3: The Windmill Implementation Pattern: A Three-Tier System

This architecture is implemented in Windmill using a clean, three-tier pattern of Flows and Scripts.

### Tier 1: The Implementation Flows (The "Workers")

- **What they are:** A library of Windmill **Flows** that contain the actual, step-by-step logic for a specific version of a business process.
- **Naming Convention:** /case/<purpose>/<process_name>_playbook_v[N].flow or ..._agent_v[N].flow.
- **Location:** Reside in the appropriate subdirectory within the /case domain (e.g., /case/onboarding/).
- **Responsibility:** To orchestrate a sequence of **direct, synchronous calls** to the "Tool" capabilities (Scripts) in the other domains (/lifecycle, /provider, etc.). They are the embodiment of a single, versioned business process.

### Tier 2: The Process Dispatcher (The "Meta Router")

- **What it is:** A single, generic, and highly stable Windmill **Flow** that acts as the central router for the entire /case domain.
- **Path:** /case/meta/dispatch-process_flow
- **Responsibility:**
    1. Receive a standardized request containing process_name, user_context, and input_args.
    2. Execute a "Strategy Resolution" **Script** (/case/meta/resolve-process-target_script) which reads the central routing configuration.
    3. Based on the strategy (A/B test, direct, etc.), the script determines the exact path of the Implementation Flow to execute.
    4. The Flow then uses Windmill's dynamic dispatch (resume) feature to execute that target path, passing through the input_args.
    5. It transparently returns the result of the implementation flow to the original caller.

### Tier 3: The Callers (The "Initiators")

- **What they are:** Any other part of the system that needs to start a core business process. These are typically the "Ingestion Flows."
- **Example:** The Flow /case/ingestion/process-inbound-provider-communication, which is triggered by a new email.
- **Responsibility:** To perform the initial interpretation and then make a **single, standardized call** to the Process Dispatcher Flow. Their knowledge of the downstream process is limited to knowing the correct process_name to request.

## Part 4: The Configuration & Governance System

This system for managing agility is as important as the code itself.

### 1. The Federated Routing Configuration

- **What it is:** A Git repository (e.g., switchup-process-config) synced to the Windmill workspace. It contains a directory structure that mirrors the /case domain.
- **Structure:**
  - /onboarding/routing.json
  - /lifecycle_management/routing.json
  - /GLOBAL/schema.json
- **Content (The Process Manifest):** Each routing.json file contains the manifests for the processes in that domain. Each manifest includes:
  - description and owner.
  - input_schema: A JSON Schema defining the required inputs.
  - strategy: A reference to the strategy capability and its arguments (e.g., path: "/optimisation/routing_strategies/percentage_rollout").
  - variants: A list of all available implementation paths and their metadata.
  - experiment: An optional block for managing the lifecycle of A/B tests.

### 2. Governance Workflows

- **CI/CD Pipeline for Configuration:** A mandatory CI/CD pipeline on the config repo that validates all routing.json files against the global schema and checks for logical errors before any change can be merged.
- **Capability Proposal Process:** A lightweight but formal process, managed by an Architectural Review Board (ARB), for approving the creation and location of any new **Public Capability**. This prevents domain drift.
- **Experiment Lifecycle Management:** An automated, scheduled Windmill Flow that monitors the experiment blocks in the routing configs and automates the process of reminding owners and cleaning up concluded experiments.

## Part 5: The Observability Mandate

To make this distributed system manageable, a strict observability policy is non-negotiable.

1. **Distributed Tracing is Mandatory:** Every top-level "Ingestion Flow" **must** generate a unique Trace ID.
2. **Context Propagation:** This Trace ID **must** be propagated through every single call in the chain: from the Caller, through the Dispatcher, into the Implementation Flow, and down into every atomic Capability Script it calls.
3. **Structured Logging:** Every log line emitted by any Script or Flow in the entire system **must** be a structured JSON log and **must** include the Trace ID.
4. **Centralized Logging Platform:** All logs are shipped to a central platform (e.g., Datadog, OpenSearch) that allows for querying and filtering by Trace ID, enabling the complete reconstruction of any distributed process.

This final, holistic blueprint represents a socio-technical system. It combines a robust, flexible, and platform-native **technical architecture** with the necessary **governance and operational frameworks** to ensure it remains clean, understandable, and evolvable as SwitchUp scales. It is an engine for managed, observable, and continuous evolution.
