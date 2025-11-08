# Overview on<br>DDD, Capabilities & Business Intents

## The Three Concepts: A Detailed Breakdown

### 1. Domain-Driven Design (DDD): The "Zoning Laws"

- **What it is:** DDD is the **overarching architectural philosophy** and **strategic framework**. It is a set of principles for how to partition a complex system into logical, independent parts.
- **Its Core Question:** "How do we draw the boundaries?"
- **In the Kitchen Analogy:** DDD is the **master blueprint and zoning plan** for the entire restaurant. It doesn't tell you which brand of oven to buy, nor does it tell you how to chop an onion. It dictates that there must be a clear separation between the "Hot Line" zone (for cooking), the "Cold Prep" zone (for salads), the "Pantry" zone (for storage), and the "Service Window" (for interacting with waiters). It enforces rules like "raw meat from the Pantry must never cross the finished plates at the Service Window."
- **In SwitchUp's Architecture:**
  - DDD is the decision to have a /case domain, a /lifecycle domain, a /provider domain, etc.
  - It is the "Hub and Spoke" principle that dictates the communication rules between these domains.
  - It is a **high-level, strategic design choice**. It is the "map" of our system.

### 2. The Capability: The "Appliance" or "Workstation"

- **What it is:** A Capability is a **specific, public-facing, and reusable business function** that lives *within* a Domain. It is a concrete implementation of a piece of business logic.
- **Its Core Question:** "What specific, valuable *work* can this part of the system do?"
- **In the Kitchen Analogy:** A capability is a high-level **appliance or a complete workstation**.
  - The /optimisation/generate-optimal-recommendation script is the "Rational Combi-Oven." It's a complex, powerful tool that takes in raw ingredients (data) and produces a perfectly cooked component (a decision).
  - The /provider/switching/initiate-cancellation script is the "Automated Order Submission Terminal." It's a specialized tool for communicating with the outside world.
  - A /case Flow is the "Head Chef's Orchestration Board," a tool for directing the sequence of work.
- **In SwitchUp's Architecture:**
  - Capabilities are the public **Scripts** and **Flows** we have defined within each domain.
  - They are the "tools" in our "Workshop" domains and the "assembly instructions" in our /case domain.
  - They are **mid-level, tactical implementations** that exist within the strategic boundaries set by DDD.

### 3. The Business Intent Catalog: The "Atomic Actions" or "Knife Cuts"

- **What it is:** The Business Intent Catalog is the **most granular, foundational, and explicit vocabulary of all possible state-changing actions** in the system. It is the exclusive API of the System of Record (/lifecycle).
- **Its Core Question:** "What are the fundamental, unbreakable 'verbs' of our business?"
- **In the Kitchen Analogy:** The Business Intent Catalog is the set of **fundamental, professional knife cuts**.
  - ConfirmActivation is a *julienne*.
  - ReportPriceIncrease is a *brunoise*.
  - ApplyManualCredit is a *chiffonade*.
  - These are not complete dishes. They are the atomic, precise, and non-negotiable actions that a chef (a /case Flow) uses to build a dish. You cannot ask a knife to "make a salad." You must tell it to *dice* the cucumber, *slice* the tomato, and *mince* the garlic.
- **In SwitchUp's Architecture:**
  - This is the list of **Command Handlers** within the /lifecycle domain.
  - They are the **low-level, atomic building blocks of state mutation**.
  - They are not complete business functions; they are the safest, most reliable way to execute a single, auditable change.

### The Relationship: A Perfect Hierarchy of Abstraction

Now, let's connect them in a clear, top-down chain of command.

```
+-----------------------------------------------------------------------------------------------------------------------------+
| 1. DOMAIN-DRIVEN DESIGN (The Philosophy / The Zoning Laws)                                                                  |
|    - Defines the existence of /case, /lifecycle, /provider, etc.                                                            |
|    - Sets the "Hub and Spoke" rule: The Brain calls the Limbs.                                                              |
|                                                                                                                             |
|    (This philosophy allows us to build…)                                                                                    |
+-----------------------------------------------------------------------------------------------------------------------------+
       |
       V
+-----------------------------------------------------------------------------------------------------------------------------+
| 2. CAPABILITIES (The Tools & Processes / The Appliances & Assembly Instructions)                                            |
|    - A "Process Capability" in /case, like handle-price-increase_playbook, is an                                            |
|      orchestration of "Tool Capabilities."                                                                                  |
|    - It calls…                                                                                                              |
|      - /optimisation/generate-optimal-recommendation (a tool)                                                               |
|      - /service/notification/dispatch-notification (a tool)                                                                 |
|                                                                                                                             |
|    (To perform work, these capabilities call…)                                                                              |
+-----------------------------------------------------------------------------------------------------------------------------+
       |
       V
+-----------------------------------------------------------------------------------------------------------------------------+
| 3. BUSINESS INTENT CATALOG (The Atomic Actions / The Knife Cuts)                                                            |
|    - The most granular, explicit vocabulary of state-changing commands.                                                     |
|    - Examples: ConfirmActivation, ReportPriceIncrease, ApplyManualCredit.                                                   |
|    - Used by the most critical capabilities (/case flows) to enact their final steps.                                       |
|    (To change the system's state, these capabilities call…)                                                                 |
+-----------------------------------------------------------------------------------------------------------------------------+
```

### Summary of Differences

**Domain-Driven Design**

- **What it is:** An architectural **philosophy**
- **Level of Abstraction:** Highest (Strategic)
- **Analogy:** The Restaurant Blueprint / Zoning Laws
- **Key Question:** "How do we partition the system?"

**Capability**

- **What it is:** A reusable **business function** (Script/Flow)
- **Level of Abstraction:** Middle (Tactical)
- **Analogy:** The Appliance / Assembly Instruction
- **Key Question:** "What valuable work can this service do?"

**Business Intent**

- **What it is:** An atomic **state-change command**
- **Level of Abstraction:** Lowest (Operational)
- **Analogy:** The Professional Knife Cut
- **Key Question:** "What is the most fundamental verb?"

You don't "choose" between them. You use **Domain-Driven Design** to structure your system into domains. You populate those domains with **Capabilities** that perform useful work. And you build your most foundational domain (/lifecycle) upon a bedrock of explicit **Business Intents** to ensure the ultimate safety and clarity of all state mutations.

This is the full, coherent picture. Each concept has a distinct, crucial role to play in creating an architecture that is robust from the highest strategic level down to the most granular operational detail.

