# AI "Operating System"

## The AI Agent's "Cognitive Stack": Perceiving the Capability Layers

An AI agent operating within our architecture doesn't just see a flat list of APIs. It perceives a hierarchical "stack" of tools, each with a different level of abstraction and a different strategic purpose. It will learn to navigate this stack from the top down.

This is the AI's "Cognitive Stack," from most abstract to most granular.

### Level 3: The "Strategic Playbook" Layer (/case/ Routers)

- **What the AI sees:** A catalog of high-level, end-to-end business processes, defined in the Process Manifest. It sees onboard-new-user, handle-price-increase, resolve-ambiguous-document.
- **What this represents to the AI:** **"Known, successful solutions to major business problems."** These are the company's established, human-vetted "best practices."
- **The AI's Default Decision Rule (Principle of Least Astonishment):**

> **"If my assigned goal directly matches the purpose of a pre-existing Strategic Playbook, my safest and most efficient first action is to invoke that playbook."**

- **Example Scenario:**
  - **Goal:** "Handle this price increase for Contract C123."
  - **AI's Thought Process:** "My goal is handle-price-increase. I see a playbook in the /case/lifecycle_management/ directory with the exact same name. This playbook represents the company's official procedure. I will not reinvent the wheel. I will call the handle-price-increase router flow and pass it the required context. My job is to initiate the correct, established process."
- **When to use this layer:** This is the AI's **default, primary, and preferred** mode of operation. It acts as an intelligent dispatcher, leveraging existing, trusted automation.

### Level 2: The "Business Capability" Layer (The "Workshop" Tools)

- **What the AI sees:** A library of powerful, specialized "tool" capabilities from domains like /optimisation, /provider, and /service. It sees generate-optimal-recommendation, initiate-cancellation, dispatch-notification.
- **What this represents to the AI:** **"A toolbox of expert sub-routines for solving specific parts of a larger problem."** These are the high-level verbs of the business.
- **The AI's Decision Rule (Principle of Composition):**

> **"If my assigned goal does not have a pre-existing playbook, OR if I am the implementation of an _agent flow myself, I must decompose the problem and select the correct sequence of Business Capabilities to achieve the goal."**

- **Example Scenario:**
  - **Goal:** "The user is reporting a billing error that also seems to be related to a recent, unconfirmed move. Resolve this complex issue."
  - **AI's Thought Process:** "This is a novel problem. There is no handle-billing-error-during-move playbook. I must create a plan.
        1. **Problem:** I need to understand the full situation.
            - **Tool:** I will call /case/context/get-comprehensive-case-view.
        2. **Problem:** I need to verify the provider's last bill.
            - **Tool:** I will call /provider/data_exchange/request-document with document_type: "LAST_BILL".
        3. **Problem:** The bill arrived. I need to understand it.
            - **Tool:** I will call /provider/extraction/interpret-communication.
        4. **Problem:** The bill total seems wrong. I need to calculate what it *should* have been.
            - **Tool:** I will call /optimisation/calculate-expected-cost.
        5. **Problem:** My calculation shows a discrepancy. I need to create a task for a human in the finance team to review my findings.
            - **Tool:** I will call /lifecycle/task/create..."
- **When to use this layer:** The AI uses this layer when it is acting as a **true orchestrator**, either for a novel problem or as the implementation of a reusable _agent flow. It is composing a new "play" from the available "actors."

### Level 1: The "Atomic Command" Layer (The Business Intent Catalog)

- **What the AI sees:** The granular, immutable "verbs" of state change from the /lifecycle domain. It sees ConfirmActivation, ApplyManualCredit, ReportTerminationFailure.
- **What this represents to the AI:** **"The final, non-negotiable, and safest way to commit a change to the System of Record."** These are not tools for solving a problem; they are the tools for recording the solution.
- **The AI's Decision Rule (Principle of Finality):**

> **"After I have used the Business Capabilities (Layer 2) to gather information and make a decision, I will use the most specific possible Atomic Command (Layer 1) to record the outcome of that decision in the ledger."**

- **Example Scenario:**
- **Goal:** The AI agent from the previous example has finished its investigation and concluded that the user was indeed overcharged by €10.
- **AI's Thought Process:** "My investigation is complete. The resolution is to apply a €10 credit."
- **WRONG thought:** "I will find the contracts table and subtract 10 from the balance column." (This is impossible in our architecture).
- **CORRECT thought:** "I need to record this resolution in the System of Record. I will consult the Business Intent Catalog for the Contract entity. The ApplyManualCredit command is the perfect tool for this. I will dispatch this command with the contract_id, credit_amount_cents: 1000, and reason: 'Correction for billing error discovered by Agent XYZ'."
- **When to use this layer:** The AI uses this layer at the **end of a reasoning chain**. It is the final step where a decision is committed to the system's memory.

### The Full Cognitive Loop of a Sophisticated AI Agent

This hierarchy creates a clear, top-down cognitive loop for the AI:

1. **GOAL RECEIVED:** "Resolve this complex issue for Contract C123."
2. **CHECK LAYER 3 (STRATEGY):** "Is there an existing playbook for this exact goal?"
   - **If YES:** "Invoke /case/playbooks/resolve-issue-standard_flow." -> **DONE.**
   - **If NO:** "I must construct a new plan."
3. **ENTER LAYER 2 (TACTICS):** "Decompose the goal. What information do I need? What decisions must I make?"
   - "Call get-comprehensive-case-view..."
   - "Call generate-optimal-recommendation..."
   - "Call project-interaction-timeline..."
   - The AI composes a sequence of "Tool" capabilities until it has a final decision.
4. **COMMIT WITH LAYER 1 (EXECUTION):** "My decision is to apply a credit and close the issue."
   - "Dispatch ApplyManualCredit command to /lifecycle."
   - "Dispatch CompleteTask command to /lifecycle for the original task."
5. **REPORT:** "My work is complete. I will now output my chain_of_thought for auditing."

This hierarchical approach provides the perfect blend of freedom and safety. It allows the AI to be creative and dynamic in its use of the "Workshop" tools (Layer 2), but it provides powerful guardrails by encouraging it to use established processes first (Layer 3) and forcing it to use safe, atomic commands for all final state changes (Layer 1). This is how you build an AI-powered system that is both intelligent and trustworthy.
