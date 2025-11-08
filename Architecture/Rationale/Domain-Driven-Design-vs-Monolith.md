# Domain Driven Design vs. Monolith

## The Two Worlds: A Tale of Two Architectures

We will compare two hypothetical versions of the SwitchUp backend:

1. **"SwitchUp Classic" (Traditional Architecture):** Built using a standard, layered, data-centric approach. This is the default way many systems are built.
2. **"SwitchUp NextGen" (DDD/Capability Architecture):** Built using the "Hub and Spoke," Command-driven, Capability-based model we have meticulously designed.

### Core Philosophy: The Fundamental Difference

### SwitchUp Classic (Data-Centric)

- **Philosophy:** "The database is the center of the universe."
- **How it thinks:** The architecture is modeled around the data entities (Users, Contracts, Providers). The system is a set of services that perform CRUD (Create, Read, Update, Delete) operations on these database tables. The business logic is often located in "Service" or "Manager" classes that are called by the API endpoints.
- **Analogy:** A public library. The books (data) are on the shelves. Anyone (any service) can walk in, pick up a book, read it, and (if they have permission) write in the margins. The "rules" are enforced by the librarian at the front desk (the API layer), but the logic of *what to write* is scattered among all the different library patrons (the services).

### SwitchUp NextGen (Domain-Centric)

- **Philosophy:** "The business process is the center of the universe."
- **How it thinks:** The architecture is modeled around the business's **capabilities** and **processes**. The database is an implementation detail, hidden behind a strict, intent-driven API (/lifecycle domain). The core of the system is the /case domain, which explicitly codifies the business's operational playbooks.
- **Analogy:** A high-end, bespoke watch. You interact with the simple, elegant interface (the crown and hands, our /service domain). You don't know or care about the intricate gears and springs (the _playbooks and "tool" capabilities) inside. Each component has a single, well-defined job, and they are orchestrated to produce a reliable outcome. You cannot reach in and move a gear by hand; you must use the official interface.

### Stress Test 1: The "Price Increase" Process

This is the core value proposition. Let's see how each architecture handles it.

### SwitchUp Classic (The Tangled Path)

1. **The Code:** A PriceIncreaseService.js is created.
2. **The Logic:** Inside a function like handlePriceIncrease(contractId), the developer writes the entire end-to-end logic:
- It connects to the database to fetch the contract details.
- It makes an HTTP call to an external MarketDataService to get alternative tariffs.
- It contains complex if/else logic to compare the prices and bonuses.
- If a better deal is found, it makes an HTTP call to a NotificationService to send an email.
- Finally, it connects to the database again to update the contract's status field to "AwaitingUserConfirmation".
3. **The Problems (The Cons):**
- **Low Cohesion:** The PriceIncreaseService does everything. It knows about database schemas, external APIs, business rules, and email formatting. It is a "god object."
- **High Coupling:** It is tightly coupled to the database schema (what if a column name changes?), the MarketDataService's API, and the NotificationService's API. A change in any of these external systems will break this service.
- **Implicit Business Process:** The "playbook" for handling a price increase is hidden inside this one monolithic function. A product manager cannot read it or understand it. To change the strategy, a developer has to perform risky surgery on this complex code.
- **Hard to Test:** To unit test this function, you have to mock the database, the market data service, and the notification service. It is a brittle and complex test.

### SwitchUp NextGen (The Clean Orchestration)

1. **The Code:** A **Flow** is created: /case/lifecycle_management/handle-price-increase_playbook_v1.
2. **The Logic:** The Flow is a simple, visual sequence of calls to independent, specialized "tool" capabilities:
- **Step 1:** Call /lifecycle/contract/get-details.
- **Step 2:** Call /offer/processing/get-aggregate-offers.
- **Step 3:** Call /optimisation/generate-optimal-recommendation.
- **Step 4:** A Branch node that calls either /service/notification/dispatch-notification or another action.
- **Step 5:** Call the **Business Intent Command**: /lifecycle/contract/report-price-increase (to record the event) and /lifecycle/contract/apply-contract-transition (to change the status).
3. **The Advantages (The Pros):**
- **High Cohesion:** Each capability does one thing. /lifecycle manages state. /offer knows the market. /optimisation makes decisions. /case only orchestrates.
- **Low Coupling:** The handle-price-increase Flow is only coupled to the *stable contracts* of the other domains, not their internal implementations. The Optimisation team can completely rewrite their recommendation engine, and as long as the API contract doesn't change, the Case Flow will not break.
- **Explicit Business Process:** The business process is now a first-class, visual, and auditable artifact. A product manager can literally look at the Flow and understand the company's strategy.
- **Easy to Test:** Each "tool" capability can be tested in perfect isolation. Testing the Flow itself is an integration test, but it's simplified because you're just testing the sequence of calls, not the complex logic within them.

### Stress Test 2: The Evolution from "Human First" to "AI Triage"

This is the "Admin as Supervisor" vision. How do the architectures adapt?

### SwitchUp Classic (The Painful Refactor)

1. **The Goal:** We want to stop sending every ambiguous document to a human and have an AI try to resolve it first.
2. **The Process:** A developer has to find the AmbiguousDocumentService.js. Inside, there's a function that just puts the item in a human task queue.
3. **The Surgery:** The developer now has to perform open-heart surgery on this service.
    - They inject a new AIClarificationService dependency.
    - They wrap the existing logic in a large if/else block: try { ai_result = aiService.clarify(doc) } catch (e) { ... }.
    - if (ai_result.is_confident) { // new logic to handle the AI's decision } else { // old logic to create a human task }.
4. **The Problems (The Cons):**
    - **High Risk:** The developer is modifying a critical, production code path. A bug in the new AI integration could break the existing, safe "human first" path.
    - **No Easy A/B Test:** How do you run both strategies side-by-side? This would require adding even more complexity, like a feature flag system, deep inside the service logic.
    - **Technical Debt:** The service becomes a tangled mess of conditional logic for different strategies. It's hard to read and even harder to clean up if the experiment fails.

### SwitchUp NextGen (The Elegant Evolution)

1. **The Goal:** The same. Stop sending every ambiguous document to a human.
2. **The Process (As we defined):**
    - **Step A:** The /case/ingestion/resolve-ambiguous-document_playbook_v1 Flow already exists. It contains one step: call /lifecycle/task/create. It is untouched.
    - **Step B:** The Optimisation team creates a new, independent Flow: /case/ingestion/resolve-ambiguous-document_agent_v1. This new Flow contains the AI-driven logic (call /lifecycle, etc.).
    - **Step C:** The Product Manager submits a one-line pull request to the routing_config.json file, changing the strategy from DIRECT to AB_TEST and adding the new _agent_v1 as a 50% weighted variant.
3. **The Advantages (The Pros):**
    - **Zero Risk:** The existing, stable _playbook_v1 was never modified. The experiment runs in parallel. If the _agent has a bug, it only affects its 50% of traffic, and it can be instantly disabled by reverting the configuration change.
    - **A/B Testing is a Native Feature:** The architecture is designed for this from the ground up. Experimentation is not a special, difficult task; it is the default way of evolving the system.
    - **Clean and Composable:** The two strategies (human-first vs. AI-triage) are separate, clean, and self-contained artifacts. The system's complexity is managed, not entangled.

### Final Conclusion: The Strategic Difference

| Aspect                | Traditional Architecture ("SwitchUp Classic")                                                    | DDD / Capability Architecture ("SwitchUp NextGen")                                                     |
| --------------------- | ------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------ |
| **Unit of Change**    | A line of code within a large service.                                                           | A versioned, independent Capability or Flow.                                                           |
| **Business Logic**    | Implicit, hidden inside service methods.                                                         | Explicit, codified as a first-class Orchestration Flow.                                                |
| **Coupling**          | High and brittle. Services are deeply aware of each other's implementation details.              | Low and contractual. Domains are only aware of each other's stable, public interfaces.                 |
| **Agility**           | Low. Changes are risky, slow, and require complex refactoring.                                   | High. New strategies can be developed in isolation and deployed via configuration.                     |
| **Testability**       | Difficult. Requires extensive mocking of coupled dependencies.                                   | Easy. "Tool" capabilities can be unit tested in isolation.                                             |
| **Alignment**         | Aligns with technical layers (Controller, Service, DAO).                                         | Aligns with the Business Domain (Offer, Case, Provider).                                               |
| **Long-Term Outcome** | A "Big Ball of Mud." A monolithic, entangled system that becomes progressively harder to change. | An "Evolvable Ecosystem." A clean, composable system that is built to adapt and be improved over time. |

The DDD/Capability approach is undeniably more work to set up. It requires more discipline and a deeper upfront understanding of the business. But this initial investment pays for itself a hundred times over by creating a system that is not just functional, but also agile, resilient, and comprehensible—the three qualities that are absolutely essential for long-term success and achieving your "Admin as Supervisor" vision.
