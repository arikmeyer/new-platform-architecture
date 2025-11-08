# Lifecycle Domain (The System of Record)

## 1. Core Mandate & Philosophy

The /lifecycle domain is the **exclusive, transactional, and authoritative owner of the state, history, and relationships of all core business entities**. It acts as the system's verifiable ledger of truth and the ultimate guardian of data integrity.

This domain operates under a **"Guardian of the Digital Twin"** philosophy. Its capabilities are designed to be:

1. **Intent-Driven:** State mutations are executed via explicit **Command Handlers** (e.g., ConfirmActivation), not generic CRUD operations. This ensures every change is based on a clear business intent.
2. **Authoritative & Self-Governing:** This domain encapsulates the business rules and state machine logic for its entities. It is responsible for rejecting invalid commands or state transitions, thus protecting its own integrity.
3. **Active State Propagator:** Upon any successful state change, this domain is responsible for emitting a **Rich State Transition Event**. These events provide "before and after" context, enabling decoupled, intelligent reactions from other parts of the system.
4. **Temporal & Auditable:** Every entity's history is an immutable, first-class citizen. The domain provides capabilities to project an entity's state over time (get-timeline-projection), ensuring perfect auditability and enabling predictive analysis.

## 2. Architectural Structure & Implementation

- **Implementation:** The capabilities in this domain are implemented as Windmill **Scripts**. **Go** or **Rust** are the recommended languages due to their strong type safety and performance, which are critical for this foundational layer.
- **Structure:** /lifecycle/<entity>/<capability>. The structure is strictly organized by the business entity.
- **Interaction Rules (Hub and Spoke - Layer 0):**
  - **Who it Calls:** **No one.** This domain is the bedrock of the architecture and has zero upstream dependencies on any other domain.
  - **Who Calls It:** All other domains (/case, /provider, /optimisation, etc.) call this domain to read and mutate the authoritative state of the business.

## 3. Detailed Capability Specifications (The Business Intent Catalog)

These are the public **(P)** capabilities, implemented as Command Handlers for mutations and Queries for reads.

### 3.1. Entity: Contract

- /lifecycle/contract/register-pending-contract (P Command)
- /lifecycle/contract/confirm-activation (P Command)
- /lifecycle/contract/report-activation-failure (P Command)
- /lifecycle/contract/report-price-increase (P Command)
- /lifecycle/contract/accept-new-terms (P Command)
- /lifecycle/contract/correct-contract-attribute (P Command)
- /lifecycle/contract/apply-manual-credit (P Command)
- /lifecycle/contract/initiate-user-cancellation (P Command)
- /lifecycle/contract/confirm-provider-termination (P Command)
- /lifecycle/contract/report-termination-failure (P Command)
- /lifecycle/contract/expire-contract (P Command)
- /lifecycle/contract/archive-contract (P Command)
- /lifecycle/contract/get-details (P Query) - *Returns the rich object with computed properties.*
- /lifecycle/contract/get-timeline-projection (P Query) - *Returns the historical and future state projection.*

### 3.2. Entity: User

- /lifecycle/user/register-user (P Command)
- /lifecycle/user/confirm-user-verification (P Command)
- /lifecycle/user/suspend-user (P Command)
- /lifecycle/user/reactivate-user (P Command)
- /lifecycle/user/archive-user (P Command)
- /lifecycle/user/update-user-details (P Command)
- /lifecycle/user/change-user-password (P Command)
- /lifecycle/user/get-details (P Query)
- /lifecycle/user/get-login-credentials (P Query)

### 3.3. Entity: Task

- /lifecycle/task/create-task (P Command)
- /lifecycle/task/assign-task (P Command)
- /lifecycle/task/begin-task-progress (P Command)
- /lifecycle/task/complete-task (P Command) - *This handler is responsible for firing the fire-and-forget webhook to trigger the /case/ingestion/handle-task-completion_flow.*
- /lifecycle/task/fail-task (P Command)
- /lifecycle/task/cancel-task (P Command)
- /lifecycle/task/get-details (P Query)

### 3.4. Entity: Address

- /lifecycle/address/register-canonical-address (P Command)
- /lifecycle/address/correct-address-details (P Command)
- /lifecycle/address/merge-duplicate-addresses (P Command, Administrative)
- /lifecycle/address/get-details (P Query)

### 3.5. Entity: MeterPoint

- /lifecycle/meter_point/register-meter-point (P Command)
- /lifecycle/meter_point/update-meter-point-attributes (P Command)
- /lifecycle/meter_point/decommission-meter-point (P Command)
- /lifecycle/meter_point/get-details (P Query)

### 3.6. Entity: ProviderProfile

- /lifecycle/provider_profile/register-provider (P Command)
- /lifecycle/provider_profile/update-provider-details (P Command)
- /lifecycle/provider_profile/deactivate-provider (P Command)
- /lifecycle/provider_profile/get-details (P Query)

### 3.7. Entity: OfferDefinition

- /lifecycle/offer_definition/register-offer (P Command)
- /lifecycle/offer_definition/update-offer-metadata (P Command)
- /lifecycle/offer_definition/retire-offer (P Command)
- /lifecycle/offer_definition/get-details (P Query)

### 3.8. Entity Associations (/lifecycle/associations/)

- /lifecycle/associations/link-contract-to-user (P Command)
- /lifecycle/associations/unlink-contract-from-user (P Command)
- /lifecycle/associations/link-contract-to-meter-point (P Command)
- /lifecycle/associations/update-contract-to-meter-point-link (P Command)
- /lifecycle/associations/link-user-to-address (P Command)
- /lifecycle/associations/update-user-to-address-link (P Command)
- /lifecycle/associations/link-contract-to-offer (P Command)
- /lifecycle/associations/link-task-to-contract (P Command)
- /lifecycle/associations/get-user-contracts (P Query)
- /lifecycle/associations/get-contract-user (P Query)
- /lifecycle/associations/get-meter-point-history (P Query)

## 4. Core Querying & Projection Capabilities

While Command Handlers manage state changes, these Query capabilities provide the rich, contextual views of that state, which are essential for all upstream domains.

### 4.1. The "Current State" Query Pattern

- **Capability:** get-details (e.g., /lifecycle/contract/get-details)
- **Mandate:** Every core entity **must** have a get-details capability.
- **Core Responsibility:** To return the single, authoritative, and complete representation of the entity's state *at the present moment*.
- **Key Feature (Computed Properties):** The output of get-details **must** include a computed block. This block contains authoritative, business-relevant properties derived from the entity's raw state. This is a crucial feature that prevents this logic from being duplicated in every consumer.
  - **Example for Contract:** The computed block includes is_bonus_eligible, is_within_cancellation_window, days_until_renewal, current_business_phase.
  - **Logic:** The logic to calculate these properties is pure, deterministic, and entirely contained within this capability, using only the entity's own data and the current time.

### 4.2. The "Temporal State" Query Pattern (The Timeline Projection)

- **Capability:** /lifecycle/<entity>/get-timeline-projection (e.g., /lifecycle/contract/get-timeline-projection)
- **Mandate:** Complex entities with a defined lifecycle (especially Contract) **must** have a get-timeline-projection capability.
- **Core Responsibility:** To provide a complete, time-series view of an entity's lifecycle, combining its recorded past with its predictable future.
- **Strict Boundary:** The projection logic **must be 100% deterministic.** It can only project future events that are a direct, logical consequence of the entity's *existing, written terms* (e.g., renewal dates, bonus expirations). It is **strictly forbidden** from including any probabilistic or ML-based predictions (which belong to other domains).
- **Output Schema:** A chronologically sorted array of StateSnapshot objects. Each snapshot **must** include:
  - timestamp: The ISO 8601 time of the event.
  - event_type: The name of the event (e.g., ACTIVATION_CONFIRMED, PROJECTED_RENEWAL).
  - source: An enum (HISTORY or PROJECTION) indicating if the event actually occurred or is a future calculation.
  - state_snapshot: The full entity object, including its computed properties, as they were or will be at that point in time.
- **"What-If" Analysis:** The capability **should** support a hypothetical_terms input object, allowing consumers like /optimisation to request a projection for a potential new contract, enabling direct, apples-to-apples comparison of long-term value.

## 5. Event Generation & Transport ("Lean State Transitions")

This section defines the non-negotiable rules for how the /lifecycle domain communicates state changes to the rest of the system.

### 5.1. Mandate & Philosophy

- **Mandate:** Every successful Command Handler that results in a meaningful state change **must** generate and publish a single, well-defined event.
- **Philosophy ("Lean but Sufficient"):** The event payload is designed to be **"smart enough to route, but dumb enough to stay stable."** It must provide enough context for an ingestion flow to make an intelligent routing decision without needing an immediate callback, but it avoids the complexity of including full "before and after" state objects as a baseline requirement.

### 5.2. The Canonical Event Schema

All events published by this domain **must** adhere to this standardized envelope.

```json
{
  "event_id": "string (UUID)",
  "event_type": "string (e.g., 'lifecycle.task.status_updated')",
  "event_source": "string (The path of the generating capability)",
  "event_version": "string (e.g., '1.0')",
  "timestamp": "string (ISO 8601 UTC)",
  "trace_id": "string (Propagated from the initial call)",
  "entity": { "id": "string", "type": "string" },
  "payload": { /* The event-specific, "lean" payload */ }
}
```

### 5.3. The "Lean" Payload Specification

The payload object **must** contain:

- **transition (object):** For state machine changes, this describes the change.
  - Example: { "from_status": "IN_PROGRESS", "to_status": "COMPLETED" }
- **context (object):** A curated, stable set of key-value pairs essential for routing and immediate reaction. This is the critical component that prevents callbacks.
  - **Example for TaskCompleted:**
    ```json
    "context": {
      "originating_process_name": "lifecycle_management/handle-price-increase",
      "originating_flow_run_id": "flow_run_xyz",
      "resolution_outcome": "APPROVE_SWITCH" // From the resolution_data
    }
    ```
  - **Example for ContractStatusUpdated (to ACTIVE):**
    ```json
    "context": {
      "user_id": "U456",
      "provider_id": "prov_abc"
    }
    ```

### 5.4. The Transport Mechanism & Reliability

- **Mechanism:** The Command Handler **must** publish the event via a **best-effort, fire-and-forget call** to a single, dedicated Windmill Webhook (e.g., POST /webhooks/u/system/lifecycle-event-handler).
- **Reliability (The Pragmatic Compromise):**
  - This is a **best-effort** system. We consciously accept the minimal (e.g., <0.01%) risk of a lost event due to a transient network failure in exchange for avoiding the massive complexity of a Transactional Outbox pattern at launch.
  - **Mitigation is Mandatory:**
        1. **Logging:** The Command Handler **must** log a structured "EventPublished" or "EventPublishFailed" message immediately after the webhook call attempt.
        2. **Monitoring & Alerting:** A monitoring system **must** be in place to track the success/failure rate of calls to the event webhook. An alert **must** be configured to fire if the failure rate exceeds a defined threshold.
        3. **Reconciliation:** A periodic reconciliation process (e.g., a nightly batch job) should be considered to compare the state change history log with the event log to detect any missed events.

## 6. What DOES NOT BELONG (The Definitive Anti-Pattern List) - Reaffirmed

- **Orchestration Logic:** No multi-step processes.
- **Probabilistic Predictions:** The get-timeline-projection is strictly deterministic.
- **External System Interactions:** No third-party network calls.
- **Data Interpretation:** Expects clean, validated data.
- **Presentation Logic:** Returns pure, structured data only.

