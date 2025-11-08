# Process Dispatcher

### 1. Overview & Core Mandate

The Process Dispatcher is a centralized, meta-driven system responsible for dynamically routing any incoming business process request to the correct, versioned implementation Flow. It is the single entry point for all orchestrated logic in the /case domain.

Its core mandate is to **decouple the invocation of a business process from its implementation**, enabling agility, experimentation, and safe, gradual rollouts.

### 2. Components

The system consists of three tightly integrated components:

1. **The Dispatcher Flow (/case/meta/dispatch-process_flow):** The generic, stateless engine.
2. **The Strategy Capabilities (/optimisation/routing_strategies/):** A library of reusable decision-making algorithms.
3. **The Process Manifest (routing_config.json):** The declarative, version-controlled configuration that drives the entire system.

### 3. The Process Manifest (routing_config.json) - The Configuration

This is the declarative "brain" of the dispatcher. It is a federated set of JSON files, validated by a global schema.

### 3.1. Top-Level Schema (for a single process)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "description": { "type": "string" },
    "owner": { "type": "string" },
    "input_schema": { "type": "object" },
    "strategy": { "$ref": "#/definitions/Strategy" },
    "variants": {
      "type": "array",
      "items": { "$ref": "#/definitions/Variant" }
    },
    "experiment": { "$ref": "#/definitions/Experiment" }
  },
  "required": ["description", "owner", "input_schema", "strategy", "variants"]
}
```

### 3.2. Strategy Object Schema (definitions/Strategy)

This object defines *how* to choose a variant.

```json
{
  "type": "object",
  "properties": {
    "path": { 
      "type": "string",
      "description": "The full Windmill path to the Strategy Capability Script."
    },
    "args": { 
      "type": "object",
      "description": "A JSON object of static arguments to pass to the strategy script."
    }
  },
  "required": ["path"]
}
```

### 3.3. Variant Object Schema (definitions/Variant)

This object defines a possible implementation to be executed.

```json
{
  "type": "object",
  "properties": {
    "id": { 
      "type": "string",
      "description": "A unique, human-readable identifier for this variant."
    },
    "path": { 
      "type": "string",
      "description": "The full Windmill path to the implementation Flow."
    },
    "weight": {
      "type": "number",
      "minimum": 0,
      "maximum": 1,
      "description": "The traffic allocation for this variant (0.0 to 1.0)."
    }
  },
  "required": ["id", "path"]
}
```

*(The Experiment block is for governance and is detailed separately.)*

### 4. The Strategy Capabilities (/optimisation/routing_strategies/) - The Algorithms

These are a library of pure, reusable Windmill **Scripts**. Each script must adhere to a standardized interface.

- **Standard Input:**
  - user_context: object (Contains user_id, session_id, contract_id, etc.)
  - variants: array (The list of Variant objects from the manifest)
  - static_args: object (The strategy.args from the manifest)
- **Standard Output:** A JSON object containing the full, chosen Variant object. {"id": "...", "path": "...", "weight": ...}.

### 4.1. Core Strategy Implementations:

- **/optimisation/routing_strategies/direct.py**
  - **Purpose:** The simplest strategy. Always chooses the first variant in the list.
  - **static_args:** None.
  - **Logic:** Returns variants[0].
- **/optimisation/routing_strategies/percentage_rollout.py**
  - **Purpose:** Deterministically assigns a user to a variant based on weights. The core of A/B testing and gradual rollouts.
  - **static_args:**
  - bucketing_key: string (e.g., "user_id", "session_id") - Tells the script which key from the user_context to use for hashing.
  - **Logic:**
  1. Extracts the value for the bucketing_key from user_context.
  2. Hashes the value to a number between 0 and 99.
  3. Iterates through the variants, summing their weights (e.g., v1=0.8, v2=0.2) to create ranges ([0-79], [80-99]).
  4. Returns the variant whose range the hash falls into.
- **/optimisation/routing_strategies/attribute_filter.py**
  - **Purpose:** Chooses a variant based on matching attributes in the user's context. Enables targeted rollouts.
  - **static_args:**
  - rules: array of rule objects [{ "attribute": "user_context.region", "is": "Berlin", "variant_id": "berlin_special" }]
  - default_variant_id: string
  - **Logic:**
  1. Iterates through the rules.
  2. For each rule, it checks if the specified attribute in user_context matches the value.
  3. If a match is found, it returns the variant with the corresponding variant_id.
  4. If no rules match, it returns the default_variant_id.
- **/optimisation/routing_strategies/multi_armed_bandit.py** (Advanced, Future)
  - **Purpose:** An adaptive strategy for optimization. It dynamically adjusts traffic weights based on real-time performance data to maximize a goal (e.g., conversion rate).
  - **Logic:** This would be a complex AI agent that interacts with an external data store (like Redis or a database) to track the performance of each variant and apply an algorithm like Thompson sampling to choose the next variant.

### 5. The Dispatcher Flow (/case/meta/dispatch-process_flow) - The Engine

This is a single Windmill **Flow**.

- **Input Schema (Arguments):**
  - process_name: string
  - user_context: object
  - input_args: object
  - trace_id: string (Mandatory for observability)
- **Internal Logic (as a Visual Flow):**
    1. **Node 1: Validate Inputs (Script).**
        - **Path:** /case/meta/helpers/validate-and-load-manifest_script
        - **Action:**<br>a. Fetches the appropriate routing.json manifest based on process_name.<br>b. Validates the received input_args against the input_schema defined in the manifest.<br>c. **Fails loudly** if validation fails.<br>d. **Output:** The full, validated manifest object.
    2. **Node 2: Resolve Strategy (Script).**
        - **Path:** /case/meta/helpers/resolve-strategy_script
        - **Action:**<br>a. Takes the manifest, user_context, and trace_id as input.<br>b. Extracts the strategy.path and strategy.args.<br>c. **Calls the specified Strategy Capability Script** using run_script_by_path.<br>d. **Logs the decision:** "trace_id: ..., strategy: ..., chosen_variant: ...".<br>e. **Output:** The chosen Variant object (which includes the target_path).
    3. **Node 3: Dynamic Execution (Resume/Dynamic Dispatch Node).**
        - **Action:**<br>a. This is a native Windmill feature. It is configured to execute the path provided by the output of the previous node (results.node2.path).<br>b. It passes the original input_args and trace_id down to the chosen implementation Flow.

This detailed specification provides a complete, robust, and extensible design for the Process Dispatcher. It balances the need for a simple, stable core engine with the infinite flexibility required by the business through its pluggable, configuration-driven strategy model. It is the heart of an agile and intelligent operational system.

```
+--------------------------+
|       CALLER             |
| (e.g., Ingestion Flow)   |
|                          |
|  - Knows the business    |
|    process name.         |
|  - Calls the Dispatcher. |
+--------------------------+
           |
           V
+--------------------------+
|   CENTRAL DISPATCHER     |
|  (/case/meta/dispatch)   |
|                          |
|  1. Reads config for     |
|     process name.        |
|  2. Runs strategy logic. |
|  3. Determines target    |
|     implementation path. |
|  4. Dynamically calls    |
|     the target.          |
+--------------------------+
           |
           | (Calls the path determined in Step 3)
           |
           V
+--------------------------+      +--------------------------+
|     IMPLEMENTATION A     |      |     IMPLEMENTATION B     |
| (…_playbook_v1)          |  OR  | (…_playbook_v2)          |
|                          |      |                          |
|  - Contains the actual   |      |  - Contains the actual   |
|    step-by-step business |      |    step-by-step business |
|    logic.                |      |    logic.                |
+--------------------------+      +--------------------------+
```

