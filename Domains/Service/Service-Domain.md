# Service Domain

## 1. Core Mandate & Philosophy

The /service domain is the exclusive **API Gateway and Experience Layer** for all external interactions, serving both end-users (via web/mobile apps) and internal colleagues (via admin tools). Its primary purpose is to translate the intents of external actors into well-defined, secure requests to the core business process layer (/case).

This domain operates under a **"Secure Facade"** philosophy. Its capabilities are designed to be:

1. **Intent-Focused:** The primary job is to understand the *goal* of an external request (e.g., "user wants to update address") not to execute the steps to achieve it.
2. **Stateless and Thin:** This domain is a "thin" layer. It holds no business state of its own and contains minimal business logic. Its main functions are authentication, authorization, request validation, and delegation to the /case domain.
3. **The Single Point of Entry:** All synchronous, external API calls from the Presentation Layer (Layer 5) **must** pass through this Service Layer (Layer 4). There is no direct access from the outside world to the /case or /lifecycle domains.
4. **Context-Agnostic (by design):** Unlike our previous models, this domain is now "dumber." It is no longer responsible for assembling a complex 360-degree view of the user. It simply passes the user/agent identifiers to the /case domain, which is the authoritative source of business context.

## 2. Architectural Structure & Implementation

- **Implementation:** The capabilities in this domain are best implemented as Windmill **Scripts** exposed as **Webhooks**. This creates a standard, secure, and scalable API layer. A framework like Python's FastAPI is ideal for building the logic within these scripts to handle request validation and response formatting.
- **Structure:** /service/<actor>/<capability>. The structure is organized by the primary actor being served (end-users or internal colleagues).

## 3. The Strict Interaction Rule (The Boundary)

Following our definitive "Hub and Spoke" architecture:

- The /service domain **ONLY CALLS** the /case domain (Layer 3).
- It is **STRICTLY FORBIDDEN** from calling /lifecycle, /provider, /optimisation, or any other "tool" domain directly. Its job is to initiate a business process, not to cherry-pick the tools for that process.

## 4. Detailed Capability Specifications

### 4.1. The Public "Self-Service" Capabilities (The End-User API)

These are the Windmill scripts exposed as webhooks that would power the SwitchUp customer portal and mobile app.

- **Purpose:** The single, universal, and secure entry point for any action an authenticated end-user can take.
- **Trigger:** Windmill Webhook (e.g., POST /api/v1/intent)
- **Input Schema (Request Body):**
    ```json
    {
      "intent_name": "UPDATE_ADDRESS" | "SUBMIT_METER_READING", // A structured, enumerated string
      "intent_payload": { // The data needed for this specific intent
        "new_address": { "street": "...", ... }
      }
    }
    ```
- **Output Schema (Success):** A response object designed to give the UI immediate feedback.
    ```json
    {
      "status": "PROCESS_INITIATED",
      "case_id": "flow_run_xyz", // The ID of the case flow that was started
      "user_message": "Thanks! We've received your request to update your address. You'll receive a confirmation once it's complete."
    }
    ```
- **Core Logic:**
    1. **Authentication & Authorization:** The script's first action is to validate the user's authentication token (e.g., JWT) to confirm their identity (user_id).
    2. **Request Validation:** It validates the incoming intent_payload against a schema for the given intent_name. If invalid, it returns a 400 Bad Request.
    3. **Delegation to the Brain:** It makes a single, authoritative call to the Process Dispatcher to start the appropriate business process.

         ```python
    from windmill_sdk import run_script_by_path

    case_run = run_script_by_path(
        "u/your_workspace/case/meta/dispatch-process_flow",
        args={
            "process_name": "address_management/update-user-address",  # Mapped from intent_name
            "user_context": {"user_id": authenticated_user_id},
            "input_args": intent_payload,
            "trace_id": generate_trace_id()
        }
    )
    ```

    1. **Formatting the Response:** It takes the run_id from the successfully initiated case flow and formats the user-friendly JSON response.

### 4.2. The Public "Support Augmentation" Capabilities (The Admin/Copilot API)

These are the webhooks that power the internal admin tools used by support colleagues.

- **Purpose:** To provide a complete, context-rich, and secure view of a user's situation for display in the admin tool.
- **Trigger:** Windmill Webhook (e.g., GET /api/internal/user-view?user_id=...)
- **Input Schema:** user_id (passed as a query parameter).
- **Output Schema (Success):** A rich "view model" object, designed for the admin UI. This is the object defined and produced by the /case/context/get-comprehensive-case-view capability.
- **Core Logic:**
    1. **Authentication & Authorization:** Validates the agent's credentials and checks if they have permission to view the requested user's data.
    2. **Delegation to the Brain:** This capability is a simple, secure pass-through. It makes a single call to the authoritative source of context.
    3. ```python
    user_view = run_script_by_path(
        "u/your_workspace/case/context/get-comprehensive-case-view",
        args={"user_id": user_id_from_query}
    )
    ```
    4. `return user_view`
- **Purpose:** A secure endpoint for an authenticated agent to initiate a business process on behalf of a user.
- **Trigger:** Windmill Webhook (e.g., POST /api/internal/agent-action)
- **Input Schema (Request Body):**
    ```json
    {
      "user_id": "string", // The target user
      "action_name": "FORCE_CONTRACT_CANCELLATION",
      "action_payload": { "reason": "User request via phone call." }
    }
    ```
- **Core Logic:** Identical to execute-user-intent, but with different authentication/authorization logic and a different mapping from action_name to process_name.

## 5. What DOES NOT Belong (The Definitive Anti-Pattern List)

- **Business Process Orchestration:** This domain **NEVER** contains logic like "if X, then call Y, then call Z." It makes a single call to a /case flow and its job is done.
- **Direct State Access:** This domain **NEVER** calls /lifecycle. It is forbidden from knowing the raw state of the system. It must get its context from the /case domain.
- **Direct External Interaction:** This domain **NEVER** calls /provider. It does not know how to talk to the outside world.
- **Complex Decision Making:** This domain **NEVER** calls /optimisation. It does not decide if a switch is good; it only triggers the /case process that is responsible for making that decision.

This detailed specification solidifies the /service domain's role as a clean, secure, and thin API gateway. It protects the core business logic from the complexities of external interaction and enforces the strict hierarchical data flow that is essential for the architecture's long-term stability and maintainability. It is the disciplined and professional "front door" to your entire operational system.

