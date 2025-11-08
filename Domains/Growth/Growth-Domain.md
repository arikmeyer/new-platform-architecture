# Growth Domain

## 1. Core Mandate & Philosophy

The /growth domain is the exclusive owner of the **strategies and capabilities related to user acquisition and initial activation**. Its purpose is to create a compelling, personalized, and efficient journey for prospective users, from their first touchpoint to their successful first switch.

This domain operates under an **"Autonomous Conversion Engine"** philosophy. Its capabilities are designed to be:

1. **Data-Driven & Experimental:** Every capability is built with experimentation (A/B testing, multi-armed bandits) as a core principle. The goal is to continuously learn and optimize conversion strategies.
2. **Personalized:** It avoids a "one-size-fits-all" approach, dynamically tailoring content, messaging, and user flows based on the user's context, source, and behavior.
3. **Self-Contained Funnel:** It manages the entire pre-customer journey. Once a user successfully completes their first switch, the responsibility is handed over to the /case and /service domains.
4. **Content-Led:** It leverages high-quality, data-rich content to establish brand authority, drive organic traffic, and educate users.

### 2. Architectural Structure & Implementation

- **Implementation:** **Python** is the primary language for this domain, given its strengths in data analysis, machine learning (for personalization and reinforcement learning), and web frameworks. **TypeScript/Node.js** may also be used for capabilities that are heavily reliant on the front-end ecosystem.
- **Structure:** /growth/<function>/<capability>. The structure is organized by the stage of the acquisition funnel.

### 3. The Strict Interaction Rule (The Boundary)

- **Who it Calls:** The /growth domain is a "top-level" business capability domain. To do its job, it can call:
  - **Layer 1 Primitives:** It might call /offer/get-available-tariffs to populate a landing page with real-time examples.
  - **Layer 0 System of Record:** It calls /lifecycle/user/create at the point of registration.
- **Crucial Boundary:** Once a user is fully activated and their first contract is created, the /growth domain's primary job is done. It **delegates** the ongoing management of that user to the /case domain by invoking a process like /case/onboarding/initiate-new-contract. It does not manage the long-term contract lifecycle itself.

### 4. Detailed Capability Specifications

### 4.1. The "Top of Funnel" Use Case: Attracting and Educating Users

- **Purpose:** To automate the creation of high-quality, data-driven articles, guides, and reports that establish SwitchUp as a market expert and drive organic traffic.
- **Input Schema:**
```json
{
  "topic_brief": "A detailed brief for the content, including target keywords, audience, and desired angle.",
  "content_type": "BLOG_POST" | "MARKET_ANALYSIS_REPORT",
  "data_requirements": [ // Instructions for internal data to be included
    { "query": "AVERAGE_SAVINGS_IN_BERLIN_2025" }
  ]
}
```
- **Output Schema (Success):** A structured content object: { "draft_html": "...", "title": "...", "sources": [...], "internal_data_insights": [...] }
- **Core Logic (AI Agent):**
    1. **Research Phase:** An AI agent performs web searches to gather external data and understand the topic.
    2. **Internal Data Phase:** It calls capabilities in the /optimisation or /offer domains to fulfill the data_requirements.
    3. **Drafting Phase:** A generative AI synthesizes the internal and external research into a coherent, well-structured draft.
    4. **Review Trigger:** It calls /operations/create-and-route-task to assign the draft to a human content editor for review, fact-checking, and publication.

### 4.2. The "Mid-Funnel" Use Case: Converting Visitors to Users

This is the heart of the domain's responsibility.

- **Purpose:** To act as the real-time "brain" for the user onboarding funnel. For any given user at any given step, it decides the single best next action to present.
- **Input Schema:** A rich context object from the user's session:
```json
{
  "session_id": "string",
  "user_context": {
    "source": "GOOGLE_ADS_KAMPAGNE_X",
    "device": "MOBILE",
    "geo_location": "BERLIN",
    "steps_completed": ["LANDING_PAGE", "ENTERED_POSTCODE"]
  }
}
```
- **Output Schema (Success):** A JSON object that tells the frontend exactly what to render next.
```json
{
  "next_step": {
    "ui_component": "SHOW_SAVINGS_ESTIMATE" | "ASK_FOR_EMAIL" | "DISPLAY_TESTIMONIAL",
    "content_payload": { "estimated_savings": 150, ... }
  }
}
```
- **Core Logic (Reinforcement Learning Agent):**
    1. This capability is a wrapper around a trained Reinforcement Learning (RL) model (a "multi-armed bandit" is a good starting point).
    2. It takes the user_context as its state.
    3. The RL model chooses an "action" (the next_step) from a predefined set of possibilities, with the goal of maximizing the long-term probability of the user completing the funnel.
    4. The system learns over time by observing which sequences of actions lead to the highest conversion rates for different types of users.

- **Purpose:** The final step of the growth funnel. To take the user's details, create their account, and officially hand them off to the core business process.
- **Input Schema:** { "email": "...", "password": "...", "onboarding_session_id": "..." }
- **Output Schema (Success):** { "status": "SUCCESS", "user_id": "..." }
- **Core Logic:**
    1. Validates the inputs.
    2. **Calls /lifecycle/user/create** to create the authoritative user record.
    3. **Calls /case/onboarding/initiate-new-user-and-contract** to trigger the core, long-running business process of setting up their first contract. This is the crucial hand-off.
    4. Records the successful conversion event for analytics.

### 5. What DOES NOT Belong (The Anti-Pattern List)

- **Long-Term Contract Management:** This domain's responsibility **ends** when it successfully calls the /case domain to start the first contract onboarding. It does not manage price increases, renewals, or cancellations.
- **User Support:** It does not handle "service" interactions. If a user has a problem during onboarding, this domain might trigger a /case flow for exception handling, but it does not contain the support logic itself.
- **Authoritative State:** It may have its own analytics databases, but the single source of truth for a User entity is always the /lifecycle domain.
- **Provider Interactions:** It has zero knowledge of provider systems.

This specification defines the /growth domain as a highly specialized, data-driven, and autonomous engine focused on a single, critical business goal: turning anonymous visitors into active SwitchUp customers. It operates at the top of the funnel and cleanly hands off successful users to the core operational domains for their long-term lifecycle management.

