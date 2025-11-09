# Case Domain

## 1. Core Mandate & Philosophy

The /case domain is the **exclusive owner and executor of all end-to-end business processes**. Its primary purpose is to act as the intelligent orchestrator that combines the pure "tool" capabilities from all other domains (/lifecycle, /provider, /optimisation, etc.) to create business value.

This domain operates under the **"Explicit Orchestrator"** philosophy. Its components are not black boxes; they are readable, versioned, and auditable representations of the company's operational playbook.

### 2. Architectural Structure

The /case domain is implemented as a library of **Windmill Flows**, organized into a clear, three-tier structure that separates initiation, dispatching, and implementation.

### 2.1. Subdomain: /case/ingestion/ (The "Airlocks")

- **Purpose:** To be the sole, controlled entry point for external triggers and data to enter the case management system. These Flows are the "system boundary."
- **Implementation:** A collection of Windmill Flows, each triggered by a specific external event source (e.g., Webhook, Schedule, Email).
- **Key Responsibility:** To perform initial data validation, interpretation (by calling /provider or /service capabilities), and then make a single, standardized call to the /case/meta/dispatch-process_flow to initiate the correct core business process.
- **Example Flows:**
  - process-inbound-provider-communication_flow: Triggered by a new email/document, interprets it, and dispatches a core process like handle-price-increase.
  - handle-inbound-user-request_flow: Triggered by user communication (email/chat/phone), interprets the user's intent, and dispatches a core process like update-user-address.

### 2.2. Subdomain: /case/meta/ (The "Central Nervous System")

- **Purpose:** To provide the single, generic, and centralized mechanism for routing and executing any business process. This subdomain is the embodiment of our configuration-driven agility.
- **Components:**
    1. **/case/meta/dispatch-process_flow (The Meta Router):**
        - A single, highly stable Windmill **Flow**.
        - It is the universal entry point for *all* orchestrated business processes called by the ingestion flows.
        - Its logic is simple: it calls the "Strategy Resolution Script" and then dynamically executes the target path that the script returns.
    2. **/case/meta/resolve-process-target_script (The Strategy Resolver):**
        - A single Windmill **Script**.
        - Its logic is to load the federated routing.json configuration for a given process_name, execute the defined strategy (by calling a strategy capability from /optimisation/routing_strategies/), and return the final target_path of the implementation flow to be executed.

### 2.3. Subdomain: /case/<business_purpose>/ (The "Playbooks & Agents")

- **Purpose:** To contain the actual, versioned implementation logic for every business process. This is the library of "workers."
- **Structure:** Organized by business purpose (e.g., /case/onboarding/, /case/lifecycle_management/).
- **Implementation:** A collection of Windmill **Flows** with a strict naming convention: <process_name>_playbook_v[N].flow or <process_name>_agent_v[N].flow.
- **Key Responsibility:** To contain the detailed, step-by-step orchestration logic. A Flow in this category will contain a graph of nodes that make direct, synchronous calls to the "tool" capabilities in the other domains (/lifecycle, /provider, /optimisation, etc.).

### 3. Key Public-Facing Business Processes (Capabilities)

These are the primary processes the /case domain is responsible for orchestrating. They are identified by their process_name, which is used by the Process Dispatcher.

- onboarding/initiate-new-user-and-contract
- lifecycle_management/handle-price-increase
- lifecycle_management/execute-annual-renewal-check
- lifecycle_management/process-contract-termination
- address_management/update-user-address
- metering/submit-meter-reading
- billing/process-annual-bill
- exception_handling/resolve-data-conflict

### 4. Governance and Best Practices

1. **The Interface Principle:** The /case domain's public interface is defined by the process_name strings and the input_schema for each process within the routing_config.json manifests.
2. **Strict Separation of Concerns:**
   - This domain **MUST NOT** contain pure state (/lifecycle).
   - This domain **MUST NOT** contain provider-specific interaction logic (/provider).
   - This domain **MUST NOT** contain pure calculation or decision logic (/optimisation).
   - Its sole purpose is to **orchestrate** the capabilities from those domains.
3. **Configuration-Driven Evolution:** All changes to routing strategy, A/B tests, or gradual rollouts **MUST** be managed through changes to the routing_config.json file, not by modifying the dispatch-process_flow.
4. **Implementation Immutability:** A specific, versioned implementation flow (e.g., _playbook_v1) should be considered immutable. To change a process, you **MUST** clone it, create a new version (e.g., _playbook_v2), and update the routing configuration to direct traffic to the new version. This ensures that in-flight processes are not affected and provides a perfect audit trail.
5. **Auditability Mandate:**
   - The dispatch-process_flow **MUST** produce clear logs indicating which process_name it is handling, which strategy it is executing, and which target_path was chosen.
   - Any _agent implementation **MUST** produce a chain_of_thought output that explains its decision-making process.

### 5. Observability Requirements

- **Trace ID Propagation:** Any Flow originating in /case/ingestion/ **MUST** generate a unique Trace ID. This Trace ID **MUST** be passed as part of the user_context to the Process Dispatcher and down into every subsequent capability call for the entire duration of the process.
- **Structured Logging:** All scripts and flows within this domain must use structured JSON logging and include the Trace ID in every log entry.

This specification provides a complete and thorough definition of the /case domain. It establishes its purpose, its internal structure, its core responsibilities, and the strict rules of engagement that will allow it to function as the powerful, flexible, and maintainable brain of the SwitchUp operational system.

