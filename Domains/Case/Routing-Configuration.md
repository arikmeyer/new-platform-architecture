# Routing Configuration

## 1. Core Mandate & Philosophy

The Routing Configuration is the **declarative, strategic brain** of the /case domain. It is a set of version-controlled, human- and machine-readable files that explicitly define every orchestrated business process in the system, their contracts, and the rules for their execution.

It operates under a **"Configuration as Code"** philosophy, adhering to the following principles:

1. **Declarative, Not Imperative:** The configuration defines *what* the desired outcome is (e.g., "run the playbook v2 for 10% of Berlin users"), not *how* to achieve it. The "how" is encapsulated in the Process Dispatcher and Strategy Capabilities.
2. **Single Source of Truth:** This configuration is the one and only source of truth for which business processes exist, what their contracts are, and how they are routed.
3. **Human-Editable, Machine-Readable:** The structure (JSON) is simple enough for a product manager to understand and edit, but rigorous enough for automated validation and parsing.
4. **Agility Through Separation:** It completely decouples the **business decision** (which process to run, when) from the **technical implementation** (the code in the implementation flows). This allows business strategy to evolve independently of engineering deployment cycles.

### 2. File Structure & Governance: The Federated Model

To ensure team autonomy and reduce the blast radius of errors, the configuration is federated, not monolithic.

- **Repository:** A dedicated Git repository, e.g., switchup-process-config.
- **Structure:**

    |-- /onboarding/
    |   |-- routing.json
    |
    |-- /lifecycle_management/
    |   |-- routing.json
    |
    |-- /address_management/
    |   |-- routing.json
    |
    |-- /GLOBAL/
        |-- schema.json  <-- The master JSON Schema for all routing.json files

- **Governance:**
  - Changes are made via Pull Requests only.
  - A CODEOWNERS file in Git maps each directory to a specific team (e.g., /onboarding/* is owned by @switchup/team-growth), requiring their approval for merges.
  - A mandatory CI/CD pipeline runs on every PR to validate the changed files against the master schema.json.

### 3. The Master Schema (/GLOBAL/schema.json)

This is the constitution. It defines the valid structure for any routing.json file. It is a dictionary where each key is a process_name.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Process Routing Manifest Schema",
  "description": "Defines the structure for all routing.json files.",
  "type": "object",
  "patternProperties": {
    "^[a-zA-Z0-9_-]+$": { "$ref": "#/definitions/ProcessManifest" }
  },
  "definitions": {
    "ProcessManifest": {
      "type": "object",
      "properties": {
        "description": { "type": "string" },
        "owner": { "type": "string", "pattern": "^@switchup/team-[a-z_]+$" },
        "input_schema": { "type": "object" },
        "strategy": { "$ref": "#/definitions/Strategy" },
        "variants": {
          "type": "array",
          "items": { "$ref": "#/definitions/Variant" },
          "minItems": 1
        },
        "experiment": { "$ref": "#/definitions/Experiment" }
      },
      "required": ["description", "owner", "input_schema", "strategy", "variants"]
    },
    // Definitions for Strategy, Variant, and Experiment follow...
  }
}
```

### 4. Detailed ProcessManifest Object Specification

This is the full specification for a single process entry in a routing.json file.

### 4.1. Example: "Handling an Ambiguous Provider Document"

**The Scenario:** The system ingests a PDF document from a provider. The /provider/extraction/interpret-communication capability successfully extracts some data but returns a low confidence score or multiple conflicting interpretations. For example, it might detect text that looks like *both* a final bill *and* a contract termination confirmation. A simple playbook cannot handle this ambiguity.

This is where the Process Dispatcher and a well-defined manifest shine.

### The routing.json Manifest for the "Document Interpretation" Process

This example will demonstrate how we can A/B test two different *automation strategies* to resolve an ambiguous situation, managed by the "Optimisation" team (in your sense of the word, meaning "team that optimizes processes").

**File:** /case/ingestion/routing.json

```json
{
  "resolve-ambiguous-provider-document": {
    "description": "The process for resolving provider communications where automated interpretation has low confidence or conflicting results.",
    "owner": "@switchup/team-optimisation",
    "input_schema": {
      "type": "object",
      "properties": {
        "document_id": { "type": "string" },
        "interpretation_results": {
          "type": "object",
          "description": "The full, ambiguous output from the /provider/interpret-communication capability."
        }
      },
      "required": ["document_id", "interpretation_results"]
    },
    "strategy": {
      "path": "/optimisation/routing_strategies/percentage_rollout",
      "args": {
        "bucketing_key": "document_id"
      }
    },
    "variants": [
      {
        "id": "playbook_v1_human_first",
        "path": "/case/ingestion/resolve-ambiguous-document_playbook_v1.flow",
        "weight": 0.5,
        "description": "The baseline, safe strategy: Immediately create a task for a human operator to classify the document."
      },
      {
        "id": "agent_v1_clarification_attempt",
        "path": "/case/ingestion/resolve-ambiguous-document_agent_v1.flow",
        "weight": 0.5,
        "description": "Experimental AI agent: Attempts to resolve the ambiguity by cross-referencing other data before escalating to a human."
      }
    ],
    "experiment": {
      "status": "ACTIVE",
      "hypothesis": "The AI clarification agent can autonomously resolve 30% of ambiguous documents, reducing manual workload without sacrificing accuracy.",
      "owner": "@optimisation-team-lead",
      "start_date": "2025-11-20T10:00:00Z",
      "resolution_policy": {
        "end_date": "2026-02-20T10:00:00Z",
        "success_metric": "manual_task_creation_rate < baseline * 0.7 AND error_rate <= baseline",
        "on_success": "PROMOTE_VARIANT",
        "on_failure": "CLEANUP_VARIANT"
      }
    }
  }
}
```

**The Scenario:** The system ingests a PDF document from a provider. The /provider/extraction/interpret-communication capability successfully extracts some data but returns a low confidence score or multiple conflicting interpretations. For example, it might detect text that looks like *both* a final bill *and* a contract termination confirmation. A simple playbook cannot handle this ambiguity.

This is where the Process Dispatcher and a well-defined manifest shine.

### The routing.json Manifest for the "Document Interpretation" Process

This example will demonstrate how we can A/B test two different *automation strategies* to resolve an ambiguous situation, managed by the "Optimisation" team (in your sense of the word, meaning "team that optimizes processes").

**File:** /case/ingestion/routing.json


### Deconstructing the Example: How it Fits the SwitchUp Model

**1. The Business Problem is Core to SwitchUp**

This is not a generic e-commerce problem. It's about dealing with the messy reality of provider communications, which is a central challenge in your "Operational Scalability" problem space. The goal is to reduce manual intervention.

**2. The Owner is the "Optimisation" Team**

The @switchup/team-optimisation owns this process. Their job is not "retention" but "process efficiency and accuracy." They are testing a hypothesis to improve the system's autonomy. This perfectly aligns with your definition of the team.

**3. The Variants are Competing *Operational Strategies***

This is the most important part. We are A/B testing two fundamentally different ways to run the business:

- **Variant A: The Safe Playbook (playbook_v1_human_first)**
  - This implementation Flow is incredibly simple and reliable.
  - **Step 1:** It receives the ambiguous document context.
  - **Step 2:** It immediately calls /operations/create-and-route-task, packaging up all the information and assigning it to the support-tier-1 queue for a human to look at.
  - **Outcome:** 100% of ambiguous documents go to a human. This is safe, but expensive and slow.
- **Variant B: The Experimental Agent (agent_v1_clarification_attempt)**
  - This is a more sophisticated AI agent Flow.
  - **Step 1:** It receives the ambiguous context (e.g., "might be a bill, might be a cancellation").
  - **Step 2 (Hypothesis Testing):** It calls /lifecycle/contract/get-details to check the contract's current status. If the status is already CANCELLATION_PENDING, the likelihood of the document being a cancellation confirmation increases dramatically.
  - **Step 3 (Cross-Referencing):** It might call /provider/data_exchange/request-latest-bill-status (a hypothetical capability) to see if a new bill is expected.
  - **Step 4 (Decision):** Based on this cross-referenced data, the agent makes a new, higher-confidence decision.
    - **If confidence is now > 95%:** The agent directly invokes the appropriate downstream process itself (e.g., calls the /case/lifecycle_management/process-contract-termination Flow). It has autonomously resolved the ambiguity.
    - **If confidence is still low:** The agent gives up and, as its final step, calls /operations/create-and-route-task, just like the simple playbook.
  - **Outcome:** A percentage of ambiguous documents are resolved automatically, while the rest are still safely routed to a human.

**4. The Success Metric is about Operational Efficiency**

The success_metric is not "revenue" or "retention." It is manual_task_creation_rate and error_rate. The goal is to prove that the new, more complex AI strategy can reduce the human workload (cost savings) without introducing more mistakes (maintaining quality). This is a perfect KPI for the Optimisation team.

This example is far more aligned with the core of SwitchUp's challenges. It demonstrates how the Meta Router architecture allows you to safely experiment with and evolve your core automation strategies, directly addressing the "Operational Scalability" problem by testing different approaches to reducing manual intervention.

### 4.2. Detailed Field Explanations

- A concise, human-readable summary of the business process's purpose.
- The unique identifier for the team responsible for this process (e.g., @switchup/team-retention). Used for CODEOWNERS and notifications.
- A valid JSON Schema object.
- **Mandate:** This is the public contract for the business process. The Process Dispatcher **must** validate all incoming input_args against this schema before proceeding. It is the primary guardrail against incorrect invocations.
- **path (string, required):** The full, absolute Windmill path to the Strategy Capability Script (e.g., /optimisation/routing_strategies/percentage_rollout).
- **args (object, optional):** A JSON object of static arguments that configure the behavior of the chosen strategy script. This is how the business logic (the "rules") are passed to the algorithm.
- An array containing at least one Variant object.
- **id (string, required):** A unique, human-readable identifier for this implementation within the context of this process (e.g., playbook_v1, agent_v1). This ID is used by the strategy rules.
- **path (string, required):** The full, absolute Windmill path to the implementation Flow (e.g., /case/onboarding/initiate-new-user-and-contract_playbook_v1.flow).
- **description (string, optional):** A brief explanation of what this specific implementation does or what makes it different.
- **weight (number, optional):** A float between 0.0 and 1.0, used by weighting-based strategies. The validation pipeline must ensure weights sum to 1.0 if the strategy requires it.
- If this block is present, the process is considered an active experiment.
- **status (string, required):** ACTIVE, PAUSED, CONCLUDED.
- **hypothesis (string, required):** A clear, plain-language statement of what the experiment is trying to prove. This is crucial for future context.
- **owner (string, required):** The product manager or business owner responsible for the experiment's outcome.
- **start_date / end_date (string, required):** ISO 8601 timestamps defining the experiment's lifecycle.
- **success_metric (string, required):** A description of the metric used to determine the winner.
- **on_success / on_failure (string, required):** Defines the automated cleanup policy (PROMOTE_VARIANT, CLEANUP_VARIANT). Used by the automated Experiment Lifecycle Management flow.

This comprehensive specification ensures that the routing configuration is not just a simple file, but a powerful, governable, and self-documenting **Process Manifest**. It is the central pillar of the system's agility, providing a single, safe, and auditable place to manage the evolution of every business process at SwitchUp.

