# Sequence: Production Order Modal Execution Lifecycle

This reflects the current production-order UX and backend lifecycle: the primary
workspace is ordered as **Production orders → Workflow templates → BOM
definitions → Workstations → Analytics**. Selecting an order opens a large modal
with **Execution**, **Assignments**, **Quality**, **Materials**, **Timeline**, and
**Audit trail** sections.

The current execution model is record-first. Users no longer need visible
Start/Pause controls to make normal progress: the first `/record` call implicitly
starts the step when needed, `/record` only accumulates completed/rejected
quantities for that step, and `/complete` advances only once that step has
processed the order's planned quantity.

> See also [`production-kanban-vendor-overhaul.md`](production-kanban-vendor-overhaul.md)
> for vendor/internal assignment, deadlines, employee self-service, and Kanban
> board behavior built around this lifecycle.

## Modal-based execution and step advancement

```mermaid
sequenceDiagram
    actor U as Management/Admin
    participant FE as Frontend<br/>ProductionPage + modal
    participant API as ProductionController
    participant SVC as ProductionServiceImpl
    participant POST as ProductionOutputPostingService
    participant DB as PostgreSQL

    U->>FE: Open Production orders tab
    FE->>API: GET /api/production/orders<br/>GET /api/production/kanban
    API-->>FE: orders + board items<br/>(hasActiveAssignment included)
    FE->>FE: Render board/list full-width

    U->>FE: Click order card/row
    FE->>API: GET /api/production/orders/{orderId}
    FE->>API: GET /api/production/orders/{orderId}/assignments
    FE->>API: GET /api/production/orders/{orderId}/execution-batches
    FE->>API: GET /api/production/orders/{orderId}/timeline
    FE->>API: GET /api/production/orders/{orderId}/audit-log
    API-->>FE: detail payloads
    FE->>FE: Open large modal with Execution,<br/>Assignments, Quality, Materials,<br/>Timeline, Audit trail

    U->>FE: Enter completed/rejected quantity for current step
    FE->>API: POST /api/production/orders/{orderId}/steps/{stepId}/record<br/>{completedQuantity, rejectedQuantity, expected versions}
    API->>SVC: recordProduction(orderId, stepId, request)
    SVC->>DB: Sum this step's prior execution rows
    SVC->>SVC: Reject if step processed + new quantity > planned quantity
    alt step was not already STARTED
        SVC->>DB: INSERT production_step_executions(action=STARTED)
    end
    SVC->>DB: INSERT production_step_executions(action=COMPLETED,<br/>completedQuantity, rejectedQuantity)
    SVC->>DB: INSERT production_order_audit_log(eventType=PARTIAL_COMPLETE,<br/>recordOnly=true)
    SVC->>DB: Set order status IN_PROGRESS
    API-->>FE: updated order
    FE->>FE: Refresh modal quantities and board card

    U->>FE: Click Move to next step / Complete step
    FE->>API: POST /api/production/orders/{orderId}/steps/{stepId}/complete
    API->>SVC: completeStep(orderId, stepId, request)
    SVC->>DB: Sum this step's execution rows only
    alt completed + rejected < order planned quantity
        SVC-->>API: 400 Current step quantity is not fully recorded
        API-->>FE: show gate message
    else step fully accounted for
        alt next active step exists
            SVC->>DB: production_orders.current_step_id = next step
            SVC->>DB: status = IN_PROGRESS
        else final step
            SVC->>DB: current_step_id = null
            alt rejected quantity is zero
                SVC->>DB: status = COMPLETED
            else rejected output exists
                SVC->>DB: status = PARTIALLY_COMPLETED
            end
        end
        SVC->>DB: INSERT production_order_audit_log(eventType=STEP_COMPLETED)
        API-->>FE: updated order
    end
```

## Final-step inventory posting

```mermaid
sequenceDiagram
    participant SVC as ProductionServiceImpl
    participant POST as ProductionOutputPostingService
    participant DB as PostgreSQL

    Note over SVC: Only final-step accepted output<br/>becomes finished stock.
    SVC->>POST: postFinalStepOutput(order, step, completed,<br/>sourceType=STEP_EXECUTION, sourceId=executionId)
    POST->>DB: Check production_inventory_postings unique<br/>(organization_id, source_type, source_id)
    alt already posted
        POST-->>SVC: no-op idempotent replay
    else not posted
        POST->>DB: Read published BOM lines for order product
        POST->>DB: Atomically decrement raw-material stock<br/>using BOM quantity + waste percentage
        alt any material insufficient
            POST-->>SVC: throw; transaction rolls back
        else all material available
            POST->>DB: Increment finished-good stock by accepted quantity
            POST->>DB: INSERT production_inventory_postings ledger row
            POST-->>SVC: posted
        end
    end
```

## Kanban placement rule

```mermaid
flowchart LR
    A[Kanban item] --> B{Status}
    B -->|COMPLETED or CANCELLED| D[Done]
    B -->|PLANNED or RELEASED| C{hasActiveAssignment?}
    C -->|true| E[In Progress]
    C -->|false| F[To Do]
    B -->|IN_PROGRESS / ON_HOLD / PARTIALLY_COMPLETED| E
```

The important backend correction behind the UX is that progress is now derived
from execution rows for the **specific step being viewed or advanced**. The old
order-level-only accumulation was too coarse: it could make a later modal action
look complete because the order had output elsewhere, even when the current
step's own completed + rejected rows did not yet account for the planned
quantity.
