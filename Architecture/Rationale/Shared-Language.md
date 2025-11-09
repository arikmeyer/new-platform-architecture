# Shared "Language"

### 1. The Principle of "Ubiquitous Language"

This is the absolute heart of DDD, and we have been using it implicitly, but we must now state it explicitly.

- **What it is:** The deliberate and rigorous use of a **single, shared language** for all concepts, shared between the technical team, the business experts, and the code itself. There is no "translation" between what a product manager says and what a developer codes. They are the same words.
- **Why it's Worth Highlighting:**
  - **Eliminates Ambiguity:** In a traditional project, the business might say "case," the developer might code a Ticket, and the database might have a ServiceRequest table. This creates constant, low-level friction and misunderstanding.
  - **In Our Architecture:** We have made this a core principle.
  - When we decided to rename the "orchestration domain" to /case, we were practicing Ubiquitous Language.
  - The **Business Intent Catalog** is the ultimate expression of this. The name of the command, ConfirmActivation, is the exact term a business operations manager would use. The code *is* the business language.
  - The names of the capabilities (generate-optimal-recommendation) and the domains (/offer, /growth) are chosen to reflect the business, not a technical implementation.
- **The Unstated Rule:** Every new concept, capability, or command **must** be named using the precise terminology of the business experts. If there is disagreement on the term, that disagreement must be resolved *before* any code is written. The shared language is non-negotiable.

### 2. The Concept of "Bounded Contexts" and "Anti-Corruption Layers"

We have designed these, but we haven't named them. Giving them their formal names elevates them from a good idea to a formal architectural principle.

- **What is a Bounded Context?** Each of our domains (/lifecycle, /provider, /case) is a **Bounded Context**. This means that within the boundary of that domain, every term has a specific, unambiguous meaning.
  - **Example:** Inside the /offer domain, the word "Price" might be a complex object with tiers, conditions, and validity dates. Inside the /lifecycle domain, a "Price" on a historical event might just be a simple integer (price_cents). This is perfectly fine. Each domain has its own specialized model that is fit for its purpose.
- **What is an Anti-Corruption Layer (ACL)?** A Bounded Context that is dedicated to translating between the messy outside world and the clean internal world.
  - **Our Implementation:** The **/provider domain implements the Anti-Corruption Layer pattern** (though we describe it more plainly as "provider interactions" in other docs). Its entire job is to deal with the ugly, inconsistent APIs and documents of hundreds of different providers and to present a single, clean, and unified set of "tool" capabilities to the rest of the system. It protects the /case and /lifecycle domains from being "corrupted" by the chaos of the outside world.
  - **In plain terms**: The /provider domain handles inbound provider communications (parsing emails, scraping portals) and outbound actions (executing via RPA bots, APIs, email).
- **Why it's Worth Highlighting:** Formally naming these concepts gives us a powerful tool for architectural debate. When a new integration is proposed, we can ask: "Should this be part of the existing Provider ACL, or is it a new Bounded Context that requires its own ACL?" It provides a formal framework for managing external dependencies.

### 3. The Role of "Aggregates" in the /lifecycle Domain

This concept explains *why* the /lifecycle domain is structured the way it is and justifies the "Guardian" pattern.

- **What is an Aggregate?** An Aggregate is a cluster of associated objects that we treat as a **single unit for the purpose of data changes**. It consists of a "root" entity and a boundary. Any reference from outside the Aggregate must only go to the root.
- **Our Implementation:** The Contract entity is the perfect example of an Aggregate Root.
  - A Contract might have associated BillingEvents, TermConditions, and BonusRecords. In a traditional model, you might be tempted to modify one of these child objects directly.
  - In our model, you **cannot**. The Contract is the Aggregate Root. To change anything related to the contract, you **must** go through the root by issuing a **Command** to a /lifecycle/contract/... capability.
- **Why it's Worth Highlighting:** This principle is the theoretical justification for our Business Intent Catalog. It explains *why* you can't just have a generic update-billing-event endpoint. To ensure the consistency of the entire Contract object, all changes must be funneled through the root entity's controlled, validated methods (our Command Handlers). This guarantees that a change to a billing event can never leave the overall contract in an invalid state. It enforces transactional consistency at the business level, not just the database level.

### Summary: The Unstated Pillars of Our Architecture

By explicitly highlighting these three core DDD concepts, we enrich our understanding of the "why" behind our design.

1. **Ubiquitous Language:** We are not just writing code; we are building a **living model of the business**. The language of the code *is* the language of the business.
2. **Bounded Contexts & ACLs:** We are not just creating services; we are building **fortresses of logic**. Each domain is a fortress with a well-defined perimeter, and domains like /provider are the specialized gatehouses that safely manage all traffic with the outside world.
3. **Aggregates:** We are not just manipulating database rows; we are managing **consistent business entities**. The Command Handler pattern on our Aggregate Roots (like Contract) ensures that the core concepts of our business can never become corrupted.

Stating these principles makes the architecture more than just a diagram of boxes and arrows. It transforms it into a deep, philosophical framework for how to build software that is, and remains, a true and effective reflection of the business it serves.
