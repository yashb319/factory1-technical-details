# Sequence: Production Order Lifecycle & Automatic Inventory Posting

The most mature domain module. Covers workflow definition, order execution
step-by-step, and the automatic stock posting that happens when a final step's
output is accepted.

```mermaid
sequenceDiagram
    actor M as Management/Admin
    participant FE as Frontend (ProductionPage)
    participant API as Backend ProductionController /<br/>ProductionPhase2Controller
    participant SVC as ProductionServiceImpl /<br/>ProductionPhase2ServiceImpl
    participant POST as ProductionOutputPostingService
    participant DB as PostgreSQL

    M->>FE: define workflow (steps, stations) + BOM + publish
    FE->>API: POST /api/production/workflows, /boms (draft -> publish)
    API->>DB: INSERT production_workflows, bom_lines

    M->>FE: create a production order against a published workflow
    FE->>API: POST /api/production/orders
    API->>DB: INSERT production_orders (planned_quantity, current_step)

    loop For each workflow step
        M->>FE: start step / assign workstation
        FE->>API: PUT /api/production/orders/{id}/steps/{stepId}/start
        M->>FE: record partial output (accepted/rejected quantities)
        FE->>API: PUT .../steps/{stepId}/complete {accepted, rejected}
        API->>SVC: only advance current_step when<br/>cumulative(accepted+rejected) reaches planned_quantity<br/>(fixed bug: previously advanced prematurely)
        SVC->>DB: UPDATE production_order_steps, production_order_step_snapshots
    end

    Note over SVC,POST: When the FINAL step's output is accepted
    SVC->>POST: postInventoryForAcceptedOutput(orderId, accepted)
    POST->>DB: check production_inventory_postings for<br/>existing (organization_id, source_type, source_id)<br/>-> idempotency guard against double-posting
    alt not already posted
        POST->>POST: for each BOM line:<br/>consumed = accepted * quantityPerUnit * (1 + wastePct/100)<br/>(BigDecimal scale 3, HALF_UP)
        POST->>DB: decrementStockIfAvailable(rawMaterialId, consumed)<br/>-- atomic conditional UPDATE, 0 rows affected = insufficient stock
        alt any raw material insufficient
            POST->>POST: throw -> whole posting rolled back
            POST-->>SVC: error, order step NOT marked complete
        else all sufficient
            POST->>DB: incrementStock(finishedGoodId, accepted)
            POST->>DB: INSERT production_inventory_postings (ledger row)
            POST-->>SVC: success
        end
    else already posted
        POST-->>SVC: no-op (idempotent replay)
    end

    API-->>FE: updated order/step state
    FE->>FE: kanban board, timeline, and analytics update
```
