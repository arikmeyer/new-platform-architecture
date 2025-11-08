# Business Intent Catalog

## Entity: Contract

This is the most complex entity with the richest set of business intents.

### Initialization & Activation

- **RegisterPendingContract**
  - **Purpose:** To bring a new contract into existence. This is the first command in any onboarding process.
  - **Payload:** user_id, offer_id, provider_id, start_date, end_date, initial_terms.
  - **State Transition:** (Does Not Exist) -> PENDING_ACTIVATION.
- **ConfirmActivation**
  - **Purpose:** To confirm that a provider has accepted and activated the contract.
  - **Payload:** contract_id, activation_date, provider_reference_id.
  - **State Transition:** PENDING_ACTIVATION -> ACTIVE.
- **ReportActivationFailure**
  - **Purpose:** To report that a provider has rejected or failed to activate the contract.
  - **Payload:** contract_id, failure_reason_code, failure_details_text.
  - **State Transition:** PENDING_ACTIVATION -> ERRORED.

### Mid-Life Events & Modifications

- **ReportPriceIncrease**
  - **Purpose:** To record the fact that a price increase has been detected for this contract. This is an "event" command that may not change the primary status.
  - **Payload:** contract_id, effective_date, new_price_details.
  - **Action:** Appends a PRICE_INCREASE_DETECTED event to the contract's history.
- **AcceptNewTerms**
  - **Purpose:** To update a contract's terms, often as a result of accepting a new offer during a renewal or switch process.
  - **Payload:** contract_id, new_offer_id, new_terms_object, effective_date.
  - **Action:** Updates the contract's attributes and appends a TERMS_ACCEPTED event.
- **CorrectContractAttribute**
  - **Purpose:** To perform a data correction on a specific field, with a full audit trail.
  - **Payload:** contract_id, field_to_correct (e.g., "billing_start_date"), old_value, new_value, correction_reason.
  - **Action:** Updates the specified attribute and appends a DATA_CORRECTION event.
- **ApplyManualCredit**
  - **Purpose:** To record a manually applied credit or goodwill gesture.
  - **Payload:** contract_id, credit_amount_cents, credit_reason.
  - **Action:** Appends a CREDIT_APPLIED event.

### Termination & End-of-Life

- **InitiateUserCancellation**
  - **Purpose:** To record the user's intent to cancel the contract and move it into a pending state.
  - **Payload:** contract_id, requested_cancellation_date.
  - **State Transition:** ACTIVE -> CANCELLATION_PENDING.
- **ConfirmProviderTermination**
  - **Purpose:** To confirm that the provider has acknowledged and processed the termination.
  - **Payload:** contract_id, confirmed_termination_date.
  - **State Transition:** CANCELLATION_PENDING -> TERMINATED.
- **ReportTerminationFailure**
  - **Purpose:** To report that a provider has rejected a termination request.
  - **Payload:** contract_id, failure_reason_code, failure_details_text.
  - **State Transition:** CANCELLATION_PENDING -> ERRORED.
- **ExpireContract**
  - **Purpose:** To formally expire a contract that has reached its natural end date without being renewed or terminated.
  - **Payload:** contract_id.
  - **State Transition:** ACTIVE -> EXPIRED.
- **ArchiveContract**
  - **Purpose:** To logically delete or archive a contract, typically for data retention policy enforcement.
  - **Payload:** contract_id, archive_reason.
  - **State Transition:** TERMINATED | EXPIRED -> ARCHIVED.

## Entity: User

### Initialization & Lifecycle

- **RegisterUser**
  - **Purpose:** To create a new user account, typically in a pending state.
  - **Payload:** email, hashed_password, name.
  - **State Transition:** (Does Not Exist) -> PENDING_VERIFICATION.
- **ConfirmUserVerification**
  - **Purpose:** To confirm the user has verified their email address.
  - **Payload:** user_id, verification_timestamp.
  - **State Transition:** PENDING_VERIFICATION -> ACTIVE.
- **SuspendUser**
  - **Purpose:** To temporarily suspend a user's account.
  - **Payload:** user_id, suspension_reason.
  - **State Transition:** ACTIVE -> SUSPENDED.
- **ReactivateUser**
  - **Purpose:** To lift a suspension from a user's account.
  - **Payload:** user_id, reactivation_reason.
  - **State Transition:** SUSPENDED -> ACTIVE.
- **ArchiveUser**
  - **Purpose:** To permanently archive a user account and anonymize their data.
  - **Payload:** user_id, archive_reason.
  - **State Transition:** ACTIVE | SUSPENDED -> ARCHIVED.

### Attribute Management

- **UpdateUserDetails**
  - **Purpose:** To change user profile information like name or phone number.
  - **Payload:** user_id, updated_fields (object with new values).
  - **Action:** Updates attributes and records a DETAILS_UPDATED event.
- **ChangeUserPassword**
  - **Purpose:** To update a user's hashed password.
  - **Payload:** user_id, new_hashed_password.
  - **Action:** Updates password and records a security event.

## Entity: Task

### Lifecycle Management

- **CreateTask**
  - **Purpose:** To create a new task for a human or AI agent.
  - **Payload:** title, description, priority, due_date, context, assignment.
  - **State Transition:** (Does Not Exist) -> OPEN or ASSIGNED.
- **AssignTask**
  - **Purpose:** To assign or re-assign a task to a specific actor.
  - **Payload:** task_id, assignee_type, assignee_id.
  - **State Transition:** OPEN -> ASSIGNED.
- **BeginTaskProgress**
  - **Purpose:** To signal that an actor has started working on the task.
  - **Payload:** task_id, actor_id.
  - **State Transition:** ASSIGNED -> IN_PROGRESS.
- **CompleteTask**
  - **Purpose:** To mark a task as successfully completed.
  - **Payload:** task_id, resolution_data.
  - **State Transition:** IN_PROGRESS -> COMPLETED.
- **FailTask**
  - **Purpose:** To mark a task as failed after being attempted.
  - **Payload:** task_id, failure_reason.
  - **State Transition:** IN_PROGRESS -> FAILED.
- **CancelTask**
  - **Purpose:** To cancel a task that is no longer needed.
  - **Payload:** task_id, cancellation_reason.
  - **State Transition:** OPEN | ASSIGNED -> CANCELLED.

## Entity: Address

The Address entity is treated as a reusable, canonical object to ensure data consistency and avoid duplication. Its commands are focused on creation and correction.

### Lifecycle & Attribute Management

- **RegisterCanonicalAddress**
  - **Purpose:** To create a new, unique, and standardized address record in the system. This is typically called by a smart "create-or-get" capability.
  - **Payload:** street_address, postal_code, city, country_code, normalized_hash (a hash of the standardized address components to prevent duplicates).
  - **Action:** Creates a new address record with a unique address_id.
- **CorrectAddressDetails**
  - **Purpose:** To correct a typo or update a component of an existing address (e.g., a postal code correction). This is an audited action.
  - **Payload:** address_id, field_to_correct, old_value, new_value, correction_reason.
  - **Action:** Updates the address record and appends a DETAILS_CORRECTED event to its history.
- **MergeDuplicateAddresses**
  - **Purpose:** A special administrative command to merge two address records that have been identified as duplicates.
  - **Payload:** source_address_id (the one to be merged and archived), target_address_id (the one to keep).
  - **Action:** Re-links all entities associated with the source_address_id to the target_address_id and then archives the source address. This is a high-impact, audited operation.

## Entity: MeterPoint

The MeterPoint entity represents the physical or logical endpoint of a service, such as an electricity meter (Zählpunkt).

### Lifecycle & Attribute Management

- **RegisterMeterPoint**
  - **Purpose:** To create a new, unique record for a meter point.
  - **Payload:** meter_id_string (e.g., "DE..."), energy_type (ELECTRICITY, GAS), initial_address_id.
  - **State Transition:** (Does Not Exist) -> ACTIVE.
- **UpdateMeterPointAttributes**
  - **Purpose:** To update technical or administrative details of a meter point that are not part of its core identity.
  - **Payload:** meter_point_id, updated_fields (e.g., { "network_operator_id": "new_op_123" }).
  - **Action:** Updates the specified attributes and records a DETAILS_UPDATED event.
- **DecommissionMeterPoint**
  - **Purpose:** To mark a meter point as decommissioned or no longer in service.
  - **Payload:** meter_point_id, decommission_date, reason.
  - **State Transition:** ACTIVE -> DECOMMISSIONED.

## Entity: ProviderProfile

This entity within the /lifecycle domain represents the minimal, core identity of a provider, distinct from the rich, operational knowledge graph in the /provider domain. It's the central "rolodex" entry.

### Lifecycle & Attribute Management

- **RegisterProvider**
  - **Purpose:** To create the master record for a new provider entity.
  - **Payload:** provider_name, provider_type (SERVICE, NETWORK, METERING), legal_entity_details.
  - **State Transition:** (Does Not Exist) -> ACTIVE.
- **UpdateProviderDetails**
  - **Purpose:** To update core, static information about a provider, such as its legal name or parent company.
  - **Payload:** provider_id, updated_fields.
  - **Action:** Updates attributes and records a DETAILS_UPDATED event.
- **DeactivateProvider**
  - **Purpose:** To mark a provider as inactive or no longer operating (e.g., due to a merger).
  - **Payload:** provider_id, reason.
  - **State Transition:** ACTIVE -> INACTIVE.

## Entity: OfferDefinition

Similar to ProviderProfile, this is the minimal, core record of an offer's existence in the /lifecycle domain, linking it to the much richer model in the /offer domain.

### Lifecycle & Attribute Management

- **RegisterOffer**
  - **Purpose:** To create the master record for a canonical offer.
  - **Payload:** offer_name, provider_id, service_type (ELECTRICITY, TELCO), offer_domain_reference_id (the link to the rich model in /offer).
  - **State Transition:** (Does Not Exist) -> ACTIVE.
- **UpdateOfferMetadata**
  - **Purpose:** To update high-level metadata about the offer.
  - **Payload:** offer_id, updated_fields.
  - **Action:** Updates attributes and records a DETAILS_UPDATED event.
- **RetireOffer**
  - **Purpose:** To mark an offer as no longer available on the market.
  - **Payload:** offer_id.
  - **State Transition:** ACTIVE -> RETIRED.

## Entity Associations (/lifecycle/associations/)

### 1. User ⟷ Contract

This is the primary ownership link.

- **LinkContractToUser**
  - **Purpose:** To establish a user as the primary holder or beneficiary of a contract. This is a fundamental step in contract creation.
  - **Payload:** contract_id, user_id, role (e.g., PRIMARY_HOLDER, AUTHORIZED_USER).
  - **Action:** Creates a record in the user_contract_associations table. Records an event on both the User's and the Contract's history logs.
- **UnlinkContractFromUser**
  - **Purpose:** A corrective or administrative action to sever the link between a user and a contract. This is a high-impact, audited command.
  - **Payload:** contract_id, user_id, reason_code.
  - **Action:** Marks the association as inactive or deletes it. Records a "Contract Unlinked" event on both entities' histories.

### 2. User ⟷ Address

Manages where a user lives or receives mail.

- **LinkUserToAddress**
  - **Purpose:** To associate a user with a canonical address for a specific purpose.
  - **Payload:** user_id, address_id, address_type (Enum: PRIMARY_RESIDENCE, BILLING, MAILING), effective_from (date).
  - **Action:** Creates a new, active association. If another address of the same address_type existed, it may be marked as INACTIVE. Records an event on the User's history.
- **UpdateUserToAddressLink**
  - **Purpose:** To change the metadata of an existing link, such as making a PRIMARY address PREVIOUS.
  - **Payload:** user_id, address_id, new_address_type (PREVIOUS), effective_until (date).
  - **Action:** Updates the existing association record.

### 3. Contract ⟷ MeterPoint

Connects a service agreement to a physical location of service.

- **LinkContractToMeterPoint**
  - **Purpose:** To specify that a particular contract provides service to a specific meter point for a defined period.
  - **Payload:** contract_id, meter_point_id, start_date, end_date (can be null for ongoing).
  - **Action:** Creates a record in the contract_meterpoint_associations table. Performs validation to ensure a meter point is not linked to multiple active contracts for the same service type during the same time period. Records an event on both entities' histories.
- **UpdateContractToMeterPointLink**
  - **Purpose:** To modify an existing link, most commonly to set an end_date when a contract is terminated or a user moves.
  - **Payload:** association_id (or contract_id and meter_point_id), new_end_date.
  - **Action:** Updates the existing association record.

### 4. Contract ⟷ OfferDefinition

Creates a permanent, historical link from a live contract back to the specific, versioned offer it was based on.

- **LinkContractToOffer**
  - **Purpose:** To create an immutable link from a new contract to the canonical offer definition that defines its terms. This is critical for future analysis and auditing.
  - **Payload:** contract_id, offer_definition_id (the ID from /lifecycle/offer/), entityConfigVersionId (the more detailed ID from the /offer domain's model).
  - **Action:** Creates a record in the contract_offer_associations table. This is typically done once during contract creation.

### 5. Contract ⟷ Document (Conceptual)

Links a contract to relevant documents, like the original PDF agreement or subsequent bills. (Assuming a Document entity exists).

- **LinkContractToDocument**
  - **Purpose:** To associate a document with a contract's official record.
  - **Payload:** contract_id, document_id, document_type (e.g., INITIAL_AGREEMENT, ANNUAL_BILL, CANCELLATION_CONFIRMATION).
  - **Action:** Creates an association in the contract_document_associations table.

### 6. Contract ⟷ Task

Links a contract to a unit of work that needs to be performed on its behalf.

- **LinkTaskToContract**
  - **Purpose:** To formally associate a task with the contract it pertains to. This is often done implicitly via the task's context, but an explicit link can be valuable for querying.
  - **Payload:** task_id, contract_id.
  - **Action:** Creates a record in the task_contract_associations table.

### 7. MeterPoint ⟷ Address

Manages the physical location of a meter point.

- **LinkMeterPointToAddress**
  - **Purpose:** To specify the service address of a meter point.
  - **Payload:** meter_point_id, address_id, effective_from.
  - **Action:** Creates a new association. If a previous address existed, it is marked as INACTIVE.

### 8. ProviderProfile ⟷ OfferDefinition

Connects an offer to the provider who sells it.

- **LinkOfferToProvider**
  - **Purpose:** To establish the "sold by" relationship between an offer and a provider.
  - **Payload:** offer_definition_id, provider_id.
  - **Action:** Creates an association, typically done when the offer is first registered.

### Query-Based Association Capabilities

While the commands above are for mutation, a set of query-based capabilities is also essential for navigating these relationships. These do not change state.

- **/lifecycle/associations/get-user-contracts** (P)
  - **Purpose:** To retrieve all contracts linked to a specific user.
  - **Input:** user_id, status_filter (optional).
  - **Output:** Array of Contract summary objects.
- **/lifecycle/associations/get-contract-user** (P)
  - **Purpose:** To find the primary user for a specific contract.
  - **Input:** contract_id.
  - **Output:** User summary object.
- **/lifecycle/associations/get-meter-point-history** (P)
  - **Purpose:** To retrieve a chronological history of all contracts that have been associated with a specific meter point.
  - **Input:** meter_point_id.
  - **Output:** Array of Contract and date range objects.
- **/lifecycle/associations/get-address-history** (P)
  - **Purpose:** To retrieve all users and meter points that have ever been associated with a specific address.
  - **Input:** address_id.
  - **Output:** An object containing arrays of associated users and meter points.

By defining these granular, intent-driven commands for every core entity and their relationships, we create a complete, robust, and semantically rich API for all state mutations. This provides the foundation of safety, auditability, and clarity for the entire SwitchUp architecture.

