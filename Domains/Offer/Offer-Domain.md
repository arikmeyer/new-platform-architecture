# Offer Domain

## 1. Core Mandate & Philosophy

The /offer domain is the exclusive, authoritative owner of the **Universal Service Ontology**. Its sole purpose is to provide the capabilities necessary to model the entire consumer services market with high fidelity and to transform raw, heterogeneous offer data from any source into a clean, canonical, and contextually evaluated format.

This domain operates under a **"Model-Driven Pipeline"** philosophy, adhering to the following principles:

1. **Model is Law:** Every capability and data structure is a direct reflection of the rich, formally defined Offer Data Model (Entities, Elements, Policies, Contexts). The domain's primary function is to bring this model to life.
2. **Pipeline as an API:** The domain's public interface is structured as a series of composable, pipeline-like capabilities. This allows orchestrators in the /case domain to either run an end-to-end ingestion process or to enter the pipeline at a specific stage (e.g., directly evaluating a known offer).
3. **Strict Separation of Concerns:**
- **Description, not Prescription:** This domain's mandate is to *describe* what an offer *is* in a given context. It is strictly forbidden from making *prescriptive* judgments about which offer is "best." That is the exclusive responsibility of the /optimisation domain.
- **Data, not Process:** This domain provides the data and evaluations. It has zero knowledge of the end-to-end business processes that consume its output. Orchestration is the exclusive responsibility of the /case domain.
4. **Immutability and Versioning:** The core artifacts it manages (Blueprints, Definitions, Instances) are treated as immutable. Changes are managed through a rigorous versioning and lifecycle status (draft, published, archived), ensuring stability and auditability.

## 2. Architectural Structure & Implementation

- **Implementation:** **Python** is the designated language due to the requirements for complex data transformation (Pandas/Polars), rule engine implementation, and the potential for AI/ML in the matching and normalization capabilities.
- **Structure:** The domain is organized into two primary subdomains:
- /offer/management/: Contains the administrative capabilities for managing the ontology itself.
- /offer/processing/: Contains the public-facing pipeline capabilities for ingesting and evaluating offers.
- **Interaction Rules:**
- **Calls Down:** This domain is a Layer 1 "Tool." It can **ONLY** call downwards to Layer 0 (/lifecycle) to resolve details about core entities (e.g., getting the official name for a providerEntityId).

## 3. Detailed Capability Specifications

**(P) denotes a Public Capability.**

### 3.1. Subdomain: /offer/management/ (The Ontology Admin API)

These capabilities are the programmatic interface for the "Admin Workflows" and would be consumed by an internal admin UI.

- **Purpose:** To create a new Entity or Element blueprint, or to create a new version of an existing one.
- **Input Schema:** A full Blueprint definition object (e.g., defining a new RecurringFee type, its category, constraints, and value/unit types).
- **Output Schema (Success):** The full, versioned Blueprint object as created, including its new blueprintId and version.
- **Core Logic:** Validates the blueprint definition against the meta-model. Ensures that all referenced types (e.g., from Dictionaries) exist. Stores the immutable, versioned blueprint.
- **Purpose:** To create or update Policy, Region, Reference List, or Adjustment definitions.
- **Input Schema:** A full Definition object (e.g., an Eligibility policy definition with its ruleType and parameters).
- **Output Schema (Success):** The full, versioned Definition object as created.
- **Purpose:** To create or update specific, versioned Entity and Element instances.
- **Input Schema:** A full Instance definition object (e.g., a specific RecurringFee instance with value: 49.95, unit: EUR).
- **Output Schema (Success):** The full, versioned Instance object as created.
- **Purpose:** The critical function for creating or versioning the Offer Entity Configurations (the "recipe cards").
- **Input Schema:** A full Configuration object, including its primary entity, parent, dimensional context, and lists of elementInstanceIds and appliedPolicyIds.
- **Output Schema (Success):** The full, versioned Configuration object, including its new entityConfigVersionId.
- **Core Logic:** Performs deep validation to ensure all referenced entities, elements, and policies exist and are in a valid state (published, active).
- **Purpose:** To move any managed artifact (Blueprint, Definition, Instance, Configuration) through its lifecycle states (draft -> inReview -> published -> active -> inactive -> archived).
- **Input Schema:** { "artifact_id": "...", "artifact_type": "...", "new_status": "ACTIVE" }.
- **Core Logic:** Enforces the valid state transitions for each artifact type.

### 3.2. Subdomain: /offer/processing/ (The Offer Pipeline API)

These capabilities are the public interface for the functional processing pipeline.

- **Purpose:** A composite capability performing the initial "dirty work" of ingestion. It is a convenience wrapper around the first few pipeline steps.
- **Input Schema:** { "source_id": "verivox_api_v3", "raw_data_payload": "{...}" }.
- **Output Schema (Success):** An array of fully normalized and validated offer data objects, ready for matching. Each object is a rich structure representing a potential set of Element Instances.
- **Core Logic:**
    1. **Fetching:** Retrieves the raw data based on source_id.
    2. **Slicing:** Separates the response into individual offer records.
    3. **Mapping:** Translates source-specific fields to internal Element hints.
    4. **Normalization:** Parses values, standardizes units and formats.
    5. **Validation:** Checks the normalized data against Element Blueprint constraints.
    6. Returns an array of the resulting clean data objects.
- **Purpose:** The core matching engine. It links a single, clean data object to a specific, versioned entityConfigVersionId.
- **Input Schema:** A single, normalized offer data object (output from the previous capability).
- **Output Schema (Success):**
```json
{
  "match_status": "CONFIDENT_MATCH" | "AMBIGUOUS_MATCH" | "NO_MATCH",
  "matches": [
    { "entityConfigVersionId": "cfg_voda_red_m_v1.2", "confidence_score": 0.99, "reason": "Matched on provider ID and key static elements." },
    { "entityConfigVersionId": "cfg_voda_red_m_promo_v1.0", "confidence_score": 0.75, "reason": "Name similarity high, but one static term mismatched." }
  ],
  "original_input": { ... } // The input data is passed through for context.
}
```
- **Core Logic:** Implements the matching logic using a combination of deterministic checks on hard identifiers and static elements, and AI/ML models for semantic similarity on names and features. It returns all potential matches with confidence scores. It is the responsibility of the calling /case flow to interpret the match_status and decide whether to proceed or escalate to a human.
- **Purpose:** The main "evaluation engine." It takes a list of known entityConfigVersionIds and a rich runtime context, and returns the fully evaluated state of those offers. This is the most complex and important capability in the domain.
- **Input Schema:**
```json
{
  "entityConfigVersionIds": ["cfg_oc_magentamobil_m_standard_v1.0", ...],
  "search_request_context": {
    "user_context": { "age": 35, ... },
    "contract_context": { "contractId": "E789", ... },
    "session_context": { "location": { "postcode": "12047" }, "effectiveDateTime": "..." }
  }
}
```
- **Output Schema (Success):** An array of Aggregate Offer objects.
```json
[
  {
    "entityConfigVersionId": "cfg_oc_magentamobil_m_standard_v1.0",
    "policy_status": "OK", // or "FAILED"
    "failed_policies": [
      { "policyId": "elig_age_under28_v1", "reason": "User age 35 is not < 28." }
    ],
    "effective_elements": { // A structured object of the final, resolved elements
      "features": [...],
      "pricing": [...],
      "terms": [...]
    },
    "context_used": { // For auditability
      "location_context_result": { ... },
      "availability_results": [...]
    }
  }
]
```
- **Core Logic (The Aggregation Engine):**
    1. **Context Lookup Phase:** Calls private helpers to perform the Geographic Context Lookup and Location Availability Lookup based on the session_context.
    2. **Aggregation Phase (iterates per entityConfigVersionId):**
        a. **Inheritance Resolution:** Traverses the parentEntityConfigVersionId chain to build the full set of applicable elementInstanceIds and appliedPolicyIds, handling overrides.
        b. **Element Aggregation:** For each element, it resolves its value. If it's a Dynamic Element Instance, it queries the necessary internal tables using the runtime context. It applies all functional Modifier and Condition dependencies. The output is the set of effective_elements.
        c. **Policy Evaluation:** For each policy, it invokes the Policy Evaluator with the full search_request_context and the results from the context lookup phase. The output is the policy_status.
    3. It constructs and returns the final Aggregate Offer objects.
- **Purpose:** To apply post-aggregation modifications, typically for evaluating an existing, in-force contract against new rules.
- **Input Schema:**
```json
{
  "aggregate_offer": { ... }, // The output from the get-aggregate-offers capability
  "contract_context": { ... } // Provides the necessary context to scope the adjustments
}
```
- **Output Schema (Success):** The Adjusted Offer State, containing the modified effective_elements and a log of which adjustments were applied.

This detailed specification provides a complete blueprint for the /offer domain, creating a powerful, model-driven engine for understanding the market while maintaining the strict architectural boundaries that ensure system-wide stability and maintainability.

