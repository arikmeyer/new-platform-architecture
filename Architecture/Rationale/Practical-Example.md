# Practical Example

## The Story: "Ensuring Jane Doe is on the Best Deal"

This is the end-to-end story of how the system fulfills one of its core "contract guard" promises.

### Act I: The Strategic Blueprint (Domain-Driven Design)

**The Scene:** Before any code is written, the architect applies the principles of **Domain-Driven Design**.

**The Action:** The architect makes a series of high-level, strategic decisions that define the "stage" for our play:

1. **Partitioning:** "We will not build one giant 'SwitchUp' service. We will have separate, specialized services. There will be one for managing the core truth of a contract's state (/lifecycle), one for orchestrating business processes (/case), one for making smart decisions (/optimisation), one for talking to users (/service), and so on."
2. **Defining Boundaries:** "The /case domain is the 'Brain' and the only part of the system allowed to orchestrate. All other domains are 'Limbs' or 'Tools' that are forbidden from talking to each other. They can only be called by the Brain, and they can all reference the foundational 'Spine' of truth, /lifecycle."
3. **Setting the Stage:** DDD has drawn the map of the world. It has defined the domains and the fundamental rules of interaction. It has built the restaurant with its separate kitchen zones.

**The Outcome:** We have a stable, logical, and well-defined architectural framework.

### Act II: The Tactical Playbook (Composing Capabilities)

**The Scene:** The "Optimisation" team is tasked with implementing the annual renewal check. They work within the /case domain to create the orchestration.

**The Action:** The team builds a **Process Capability**, the /case/lifecycle_management/execute-annual-renewal-check_flow. This Flow is the "director" of the play. It does not perform the actions itself; it calls upon the specialized "actors"—the **Tool Capabilities**—from the other domains.

The Flow's script reads like this:

1. **Director:** "I need to know which contracts to check today."
   - **Calls Actor (Tool Capability):** /lifecycle/associations/get-contracts-nearing-renewal.
2. **Director:** "Great, I have a list. For this first contract, C123, I need to build a complete picture."
   - **Calls Actor (Tool Capability):** /lifecycle/contract/get-details for C123.
   - **Calls Actor (Tool Capability):** /offer/processing/get-aggregate-offers for the market alternatives.
3. **Director:** "I have all the facts. Now I need an expert opinion. Is a switch worth it?"
   - **Calls Actor (Tool Capability):** /optimisation/generate-optimal-recommendation, passing all the context.
4. **Director:** "The expert says yes, a switch is recommended. I need to inform the user."
   - **Calls Actor (Tool Capability):** /service/notification/dispatch-notification with the recommendation details.
5. **Director:** "The user has been notified. I must now record that we are waiting for their decision."
   - This is a critical, state-changing action. The Director needs to use the safest, most precise, and most auditable method possible. It looks into the foundational script...

**The Outcome:** The Case domain has successfully orchestrated a complex business process by composing a series of high-level, reusable **Capabilities**.

### Act III: The Foundational Action (Invoking a Business Intent)

**The Scene:** The Case Flow needs to update the state of Contract C123 to "AWAITING_USER_CONFIRMATION".

**The Action:** The Flow does **not** perform a generic database update. It selects a precise, atomic "verb" from the **Business Intent Catalog** provided by the /lifecycle domain.

1. **Director:** "I need to formally change the state of this contract because I am now waiting for the user. I will issue a command."
    - **Calls Actor (Command Capability):** /lifecycle/contract/apply-contract-transition.
    - **With Payload (The Command):** It sends a clear, unambiguous command object:

```json
{
  "contract_id": "C123",
  "new_status": "AWAITING_USER_CONFIRMATION",
  "reason_code": "PROACTIVE_RENEWAL_PROPOSAL_SENT",
  "actor": "/case/lifecycle_management/execute-annual-renewal-check_flow"
}
```

1. **The Guardian's Response:** The /lifecycle domain receives this command. Its "Command Handler" capability acts as the ultimate guardian. It checks its own internal rules ("Is this contract in a state that *can* be moved to AWAITING_USER_CONFIRMATION?"). If the rule passes, it performs the atomic state change, writes it to the history log, and emits the "Lean State Transition" event.

**The Outcome:** The business process has successfully and safely mutated the system's core state by using a verb from the explicit, foundational **Business Intent Catalog**.

### The Final, Unified Picture

- **Domain-Driven Design** is the **WHY**. It's the strategic framework that gives us a stable, partitioned world (the domains and their interaction rules). It's the reason our system is not a monolith.
- The **Capability** is the **WHAT**. It's the tactical unit of business function that lives within those domains. It's the "tool" that performs a specific job (generate-recommendation) or the "playbook" that defines a sequence of jobs (handle-price-increase).
- The **Business Intent** is the **HOW**. It's the most granular, operational instruction for how to safely and audibly change the foundational state of the system. It is the vocabulary that the most important Capabilities (/case flows) use to enact their final, critical steps.

They are not separate ideas. They are a **nested hierarchy of control and abstraction**. DDD provides the structure for the Capabilities to live in, and the Business Intent Catalog provides the foundational grammar for the most critical Capabilities to speak. This is how they all tie together to create a single, coherent, and robust system.

