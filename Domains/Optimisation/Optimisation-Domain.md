# Optimisation Domain

## 1. Core Mandate & Philosophy

The /optimisation domain is the exclusive owner of all **complex business calculations, predictive models, and decision-making algorithms**. Its purpose is to act as a "brain-as-a-service," consuming raw data and context, and producing insightful, actionable recommendations or evaluations.

This domain operates under a **"Stateless Oracle"** philosophy. Its capabilities are designed to be:

1. **Pure Functions:** Every capability is a pure, stateless function. Given the same input, it will always produce the same output. It has no memory and does not persist any state of its own.
2. **Context-Rich:** The capabilities in this domain are designed to receive rich, comprehensive context from the calling orchestrator (/case). They do not fetch their own data from /lifecycle; they expect to be given it.
3. **Model-Driven:** The logic within this domain is often based on mathematical, statistical, or machine learning models. It is the home for SwitchUp's core intellectual property related to cost calculation and user behavior prediction.
4. **Process-Agnostic:** This domain knows *how* to calculate the best tariff, but it has no knowledge of the end-to-end business process for proposing or executing a switch. It simply provides an answer to a question.

## 2. Architectural Structure & Implementation

- **Implementation:** Capabilities in this domain are implemented as Windmill **Scripts** for data transformation, statistical models, and machine learning pipelines. Performance-critical calculation logic should be optimized for speed and efficiency.
- **Structure:** /optimisation/<function>/<capability>. The structure is organized by the type of question being answered.

## 3. Detailed Capability Specifications (Applied to SwitchUp Use Cases)

### 3.1. The "Core Savings" Use Case: Finding the Best Deal

This is the absolute heart of the SwitchUp value proposition.

- **Purpose:** To analyze a universe of possible offers for a given user and determine the single best, long-term optimal strategy.
- **Input Schema:**
```json
{
  "user_context": { // Rich object with user's preferences, risk tolerance, etc.
    "preferences": { "prefers_green_energy": true }
  },
  "current_contract": { ... }, // Full object for the user's current contract
  "available_offers": [ ... ] // An array of all available, normalized offer objects
}
- **Output Schema (Success):**
```json
{
  "optimal_offer_id": "string",
  "recommendation_reason": "Switches to Green Energy Inc. for a projected annual savings of €150, factoring in all bonuses.",
  "projected_savings_cents": 15000,
  "comparison_details": [ // Top N offers and why they were/weren't chosen
    { "offer_id": "...", "rank": 1, ... },
    { "offer_id": "...", "rank": 2, "reason_for_lower_rank": "Lower savings despite higher bonus." }
  ]
}
```
- **Core Logic:** This is a complex orchestration of private helpers.
    1. **Calls _calculate_long_term_value (H)** for the current_contract and for *every single offer* in available_offers. This helper contains the core, battle-tested logic for factoring in bonuses, price guarantees, etc.
    2. **Calls _score_non_financial_factors (H)** for each offer, applying a scoring model based on the user_context.preferences (e.g., green energy, provider rating, contract length).
    3. **Calls _synthesize_final_ranking (H)** to combine the financial value and non-financial scores into a single, final utility score for each offer.
    4. It selects the offer with the highest utility score as the optimum and formats the final output object.

### 3.2. The "Go/No-Go" Use Case: Is a Switch Worth It?

This is a simpler, but high-frequency, decision.

- **Purpose:** To make a quick, definitive " go/no-go " decision on switching from a specific current contract to a specific new offer.
- **Input Schema:**
```json
{
  "current_contract": { ... },
  "proposed_offer": { ... },
  "business_rules": { // Dynamic rules passed by the case flow
    "minimum_savings_threshold_euros": 50,
    "minimum_provider_rating": 3.5
  }
}
```
- **Output Schema (Success):**
```json
{
  "is_switch_recommended": "boolean",
  "reason_code": "BELOW_SAVINGS_THRESHOLD" | "PROVIDER_RATING_TOO_LOW" | "SWITCH_RECOMMENDED",
  "details": {
    "projected_savings_cents": 4500 // Can be negative
  }
}
```
- **Core Logic:**
    1. Calls the same _calculate-long-term-value (H) helper for both the current and proposed contracts.
    2. Calculates the savings.
    3. Applies the business rules passed in the input.
    4. Returns a simple boolean and a reason code.

### 3.3. The "Risk Management" Use Case: Predicting User Behavior

This is for proactive, intelligent operations.

- **Purpose:** To score a user's likelihood to churn (leave the SwitchUp service) in the near future.
- **Input Schema:** { "user_context": { ... }, "recent_activity": [ ... ] } (A rich object containing everything known about the user and their recent interactions)
- **Output Schema (Success):**
```json
{
  "churn_probability": "float (0.0 to 1.0)",
  "contributing_factors": [
    { "factor": "NEGATIVE_SUPPORT_INTERACTION", "impact": 0.25 },
    { "factor": "SAVINGS_THIS_YEAR_LOW", "impact": 0.15 }
  ]
}
```
- **Core Logic:**
    1. This capability is a wrapper around a trained Machine Learning model.
    2. It performs "feature engineering" on the raw user_context and recent_activity to create the inputs the model expects.
    3. It invokes the ML model to get the probability score.
    4. It determines the top contributing factors for the prediction, providing crucial explainability.

### 3.4. The "Intelligent Routing" Use Case

This supports the Process Dispatcher and other operational routing needs.

- As defined in the Dispatcher spec, this subdomain contains a library of pure routing algorithms.
- **Example: percentage_rollout (H)**
  - **Purpose:** To deterministically assign a user to an experimental group.
  - **Input:** user_context, variants, static_args.
  - **Output:** The chosen Variant object.

## 5. What DOES NOT Belong (The Anti-Pattern List)

- **State:** This domain is **aggressively stateless**. It never saves or persists any information. It is a pure calculation engine.
- **Orchestration:** It never calls other capabilities in a sequence to manage a business process. A /case flow calls it, gets an answer, and then the /case flow decides what to do next.
- **External Interactions:** It **never** calls a provider API or sends a user an email.
- **Data Fetching:** It expects to be **given** all the data it needs. It does not call /lifecycle to fetch its own data. This makes it incredibly easy to test, as you can simply pass it mock data without needing a database connection.

This specification defines the /optimisation domain as the powerful, intelligent, but ultimately subservient brain of the operation. It provides the critical insights and decisions, but it is the /case domain that acts upon them, ensuring a clean separation between thinking and doing.

