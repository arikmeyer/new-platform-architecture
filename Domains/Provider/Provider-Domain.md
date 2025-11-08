# Provider Domain

## 1. Core Mandate & Philosophy

The /provider domain is the exclusive owner and executor of all **interactions with and knowledge about external provider systems**. Its primary purpose is to act as a robust **Anti-Corruption Layer**, providing a stable, unified, and business-oriented interface to the rest of the SwitchUp system, while internally managing the immense complexity of heterogeneous provider interfaces.

This domain operates under a **"Resilient Adapter"** philosophy. Its capabilities are designed to be:

1. **Action-Oriented:** The public interface is defined by the *business action* to be performed (e.g., initiate-cancellation), not by the technical method (e.g., call-api, run-bot).
2. **Adaptive:** The internal logic is built to handle a wide variety of provider-specific methods (API, RPA, Email) and to be adaptable to their frequent changes.
3. **Self-Contained:** All provider-specific credentials, URLs, CSS selectors, and interaction logic are completely encapsulated within this domain. No other domain should ever know what a "provider API key" is.
4. **Translational:** It is responsible for both outbound translation (converting SwitchUp's canonical data model into a provider-specific format) and inbound translation (parsing a provider's messy communication into a clean, structured internal event).

## 2. Architectural Structure & Implementation

- **Structure:** /provider/<business_function>/<capability>. Private helpers are prefixed with an underscore.

## 3. Key Components

### 3.1. The Provider Knowledge Graph

- **What it is:** The central "brain" of this domain. It's a database (likely a graph database like Neo4j or a relational DB with a well-designed schema) that stores a rich, dynamic model of every provider.
- **Content:** For each provider, it stores:
  - provider_id, name.
  - **Interaction Methods:** A prioritized list of methods for each business action (e.g., for cancellation: 1. api_client_v2, 2. rpa_bot_v1, 3. formatted_email_v1).
  - **Configuration:** API endpoints, credentials (stored securely in a vault and referenced here), web portal URLs, target email addresses.
  - **Operational Rules:** Business constraints like "requires 30-day notice for cancellation," "service area is limited to postal codes X-Y."
- **This is NOT a public component.** It is the private, internal state of the /provider domain.

## 4. Detailed Capability Specifications

### 4.1. The Public "Action" Capabilities (The Stable Interface)

These are the only capabilities in this domain that should be called by the /case domain orchestrators.

- **Purpose:** The single, authoritative way to execute a contract cancellation with an old provider.
- **Input Schema:** A canonical cancellation object: { "provider_id": "...", "contract_details": {...}, "user_details": {...}, "cancellation_date": "..." }
- **Output Schema (Success):** { "status": "SUCCESS" | "PENDING_CONFIRMATION", "provider_reference_id": "string (optional)", "method_used": "API_V2" }
- **Core Logic (The "Router" Pattern):**
    1. Fetches the provider’s profile from the **Provider Knowledge Graph**.
    2. Reads the prioritized list of interaction methods for the cancellation action.
    3. Tries the highest priority method (e.g., _interact-via-api-client).
    4. If it fails with a retryable error, it proceeds to the next method in the list (e.g., _interact-via-adaptive-bot).
    5. If all methods fail, it returns a definitive failure that can be escalated by the calling /case flow.
    6. Logs every attempt and the final outcome.
- **Purpose:** The single, authoritative way to register a new contract with a new provider.
- **Input Schema:** A canonical registration object.
- **Output Schema (Success):** { "status": "SUCCESS", "new_provider_contract_id": "string (optional)", "method_used": "RPA_V1" }
- **Core Logic:** Follows the same "Router" pattern as initiate-cancellation.
- **Purpose:** To request a specific document (e.g., ANNUAL_BILL) from a provider.
- **Input Schema:** { "provider_id": "...", "contract_id": "...", "document_type": "ANNUAL_BILL" }
- **Output Schema (Success):** { "status": "REQUEST_SUBMITTED" }
- **Core Logic:** Follows the same "Router" pattern.

### 4.2. The Public "Interpretation" Capabilities

- **Purpose:** To be the single entry point for understanding any unstructured or semi-structured communication from a provider.
- **Input Schema:** { "provider_id": "string (optional)", "content_type": "EMAIL" | "PDF", "content": "..." }
- **Output Schema (Success):**
```json
{
  "structured_data": { // All extracted entities like prices, dates, etc.
    "total_due_cents": 54321,
    "due_date": "2025-12-20"
  },
  "detected_events": [ // Business events for the /case domain
    { "event_type": "ANNUAL_BILL_RECEIVED", "confidence": 0.99, ... },
    { "event_type": "PRICE_INCREASE_DETECTED", "confidence": 0.85, ... }
  ]
}
```
- **Core Logic:** This is a complex orchestration of private helpers.
    1. Calls _extract-text-from-content to get clean text.
    2. Calls _classify-communication-type to guess if it's a bill, a confirmation, etc.
    3. Based on the classification, it calls one or more specialized LLM-based extraction agents (_extract-billing-details,_extract-contract-terms).
    4. It aggregates, validates, and normalizes the results.
    5. It runs a final analysis to infer the high-level detected_events.

### 4.3. The Public "Knowledge" Capabilities

- **Purpose:** To provide other domains with access to the provider "digital twin."
- **Input Schema:** { "provider_id": "...", "query": "CANCELLATION_POLICY" | "SERVICE_AREA" }
- **Output Schema (Success):** An object containing the requested information, e.g., { "notice_period_days": 30 }.
- **Core Logic:** A straightforward query against the Provider Knowledge Graph.

### 4.4. The Private "Helper" Capabilities (The "Workers")

These are the numerous, volatile, and highly specific scripts that do the actual work. They are **never** called directly from outside the /provider domain.

- _interact-via-api-client_v[N] (H): A script that speaks a specific provider's REST/SOAP API.
- _interact-via-adaptive-bot_v[N] (H): A Python/RPA script that logs into and navigates a specific provider's web portal.
- _extract-billing-details_agent (H): An LLM-powered agent fine-tuned to extract data from bills.
- _update-knowledge-graph-from-website_agent (H): A scheduled AI agent that monitors provider websites for changes and proposes updates to the Knowledge Graph, to be validated by a human in the /operations domain.

## 5. What DOES NOT Belong (The Anti-Pattern List)

- **Business Process Orchestration:** This domain knows *how* to cancel with Vattenfall, but it does **not** know the multi-step business process of *what to do after* a cancellation is confirmed. That belongs in /case.
- **User-Facing Communication:** This domain may receive communications, but it **never** sends them directly to an end-user. It provides its output to a /case flow, which then calls the /service domain.
- **Authoritative State:** This domain may have its own cache or knowledge graph, but the single source of truth for a *user's contract state* is always the /lifecycle domain. This domain might update its knowledge based on a failed interaction, but the /case flow is responsible for telling /lifecycle that a contract is now in an ERRORED state.

This detailed specification defines the /provider domain as a powerful, sophisticated, and essential insulation layer. It is a complex system in its own right, but its purpose is to absorb that complexity so that the rest of the SwitchUp architecture can remain clean, stable, and focused on business logic.

