# Tasks

## 1. Core Mandate & Entity Definition

The Task entity, managed exclusively by the /lifecycle domain, represents a discrete, auditable, and assignable unit of work that must be performed by a specific actor (human or AI). It is the canonical "to-do item" of the SwitchUp operational system.

**A Task is created when an automated process cannot, or should not, proceed without external input or action.**

## 2. The Task Entity: State Machine

A Task moves through a simple but strict lifecycle. Transitions are enforced by the update-status capability.

- OPEN: The task has been created and is in a queue, awaiting assignment or action.
- ASSIGNED: The task has been explicitly assigned to a specific actor.
- IN_PROGRESS: The assigned actor has actively started working on the task.
- PENDING_INFO: The actor cannot proceed and is waiting for more information.
- COMPLETED: The actor has successfully completed the task and submitted resolution data.
- CANCELLED: The task is no longer relevant and has been cancelled before completion.
- FAILED: The task could not be completed successfully after being attempted.

## 3. The Task Entity: Data Schema

This is the canonical data structure for a Task object stored in the System of Record.

```json
{
  "task_id": "string (UUID)",
  "status": "string (Enum: OPEN, ASSIGNED, ...)",
  "title": "string (A short, human-readable summary)",
  "description": "string (Detailed instructions for the actor)",
  "priority": "integer (e.g., 1-5, 1 being highest)",
  "created_at": "string (ISO 8601 timestamp)",
  "updated_at": "string (ISO 8601 timestamp)",
  "due_date": "string (ISO 8601 timestamp, optional)",

  "context": {
    "entity_refs": [
      { "type": "CONTRACT", "id": "C123" },
      { "type": "USER", "id": "U456" }
    ],
    "originating_process": {
      "case_flow_path": "/case/playbooks/...",
      "case_flow_run_id": "flow_run_xyz",
      "trace_id": "a1b2-c3d4-e5f6"
    },
    "payload": {
      // Rich, arbitrary JSON object containing all the data// needed by the actor to perform the task.
    }
  },
  "assignment": {
    "assignee_type": "string (Enum: INTERNAL_TEAM, EXTERNAL_USER, AI_AGENT)",
    "assignee_id": "string (e.g., 'support-tier-2', 'U456', 'billing-resolution-agent-v1')",
    "assigned_at": "string (ISO 8601 timestamp, optional)"
  },

  "resolution": {
    "completed_at": "string (ISO 8601 timestamp, optional)",
    "completed_by_id": "string (The specific user/agent who completed it)",
    "resolution_data": {
      // The structured data submitted by the actor upon completion.
    }
  },
  "history": [
    // An immutable log of all state changes and actions on this task.
  ]
}
```

## 4. Detailed Capability Specifications

These are the public **(P)** Windmill Scripts that provide the exclusive interface for interacting with Task entities.

### /lifecycle/task/create (P)

- **Purpose:** The single, authoritative way to create a new task.
- **Input Schema:** A subset of the Task schema, requiring title, description, context, and assignment.
- **Output Schema (Success):** The full, created Task object in the OPEN or ASSIGNED state.

- **Core Logic:**
    1. Validates the input against the schema.
    2. Generates a task_id.
    3. Creates the task record in the database.
    4. Records a "TASK_CREATED" event in the task's history log.
    5. **Crucially, it does NOT trigger any notifications.** It simply creates the record. Notifying the assignee is the responsibility of a separate, decoupled process.

### /lifecycle/task/get-details (P)

- **Purpose:** To retrieve the complete, current state of a specific task.
- **Input Schema:** { "task_id": "string" }
- **Output Schema (Success):** The full Task object.

### /lifecycle/task/update-status (P)

- **Purpose:** The state machine enforcer for a task.
- **Input Schema:** { "task_id": "string", "new_status": "IN_PROGRESS", "actor": "..." }
- **Output Schema (Success):** The updated Task object.
- **Core Logic:**
    1. Fetches the current task.
    2. Validates that the transition from the current status to new_status is allowed. Fails loudly if not.
    3. Updates the status and records the transition in the history log.

### /lifecycle/task/assign (P)

- **Purpose:** To assign or re-assign a task to a specific actor.
- **Input Schema:** { "task_id": "string", "assignee_type": "...", "assignee_id": "...", "actor": "..." }
- **Output Schema (Success):** The updated Task object.
- **Core Logic:** Updates the assignment block and the task status to ASSIGNED. Records the assignment in the history.

### /lifecycle/task/complete (P)

- **Purpose:** To mark a task as successfully completed and store its result. This is a critical, high-impact capability.
- **Input Schema:**
    ```json
    {
      "task_id": "string",
      "resolution_data": { ... }, // The structured output from the actor
      "actor": "string (The user/agent completing the task)"
    }
    ```
- **Output Schema (Success):** The updated Task object with status: "COMPLETED".
- **Core Logic:**
    1. Fetches the task and validates it can be completed (e.g., it is ASSIGNED or IN_PROGRESS).
    2. Updates the task's status to COMPLETED.
    3. Populates the resolution block with the submitted data and timestamp.
    4. Records the "TASK_COMPLETED" event in the history log.
    5. **Final Decoupled Trigger:** As its very last action, it makes a **fire-and-forget call** to a single, dedicated Windmill webhook: POST /webhooks/u/system/trigger-task-completed-handler. The payload is the full, completed Task object. This is the crucial hook for resuming business processes.

### /lifecycle/task/find-by-assignee (P)

- **Purpose:** The primary query capability for UIs. To get a list of tasks for a specific actor.
- **Input Schema:** { "assignee_id": "string", "status_filter": ["OPEN", "IN_PROGRESS"] (optional) }
- **Output Schema (Success):** An array of Task summary objects. [ { "task_id": "...", "title": "...", "due_date": "..." } ]

## 5. What DOES NOT Belong (The Anti-Pattern List)

- **Process Orchestration:** This domain has zero knowledge of what should happen *after* a task is created or completed. It only records the state and fires a dumb trigger. The logic for resuming a /case flow belongs in the /case/ingestion domain.
- **User Notification:** The create and assign capabilities do **not** send emails or Slack messages. That is the job of a separate, decoupled "sidecar" Flow that is triggered by the TASK_CREATED or TASK_ASSIGNED event (published via a similar webhook pattern). This keeps the core lifecycle logic fast and clean.
- **UI Formatting:** The get-details capability returns pure, structured data. It does not contain any HTML or user-friendly text. The /service domain is responsible for presentation.

This comprehensive specification defines the "Task" as a robust, central entity in the SwitchUp system. It provides the necessary foundation for a truly auditable, scalable, and universal system for managing work across all actors, both internal and external, human and AI.

