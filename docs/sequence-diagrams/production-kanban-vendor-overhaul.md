# Sequence: Production Kanban, Vendor Outsourcing, Deadlines & Audit Trail

Covers the production-tracking overhaul around the current modal execution
lifecycle: vendor outsourcing, per-step deadlines with breach notification, the
full audit trail, board/list Kanban, and employee self-service "My Orders" view.

## Order creation with a responsible person

```mermaid
sequenceDiagram
    actor M as Management/Admin
    participant FE as Frontend (order creation form)
    participant API as Backend ProductionController
    participant DB as PostgreSQL

    M->>FE: pick workflow/BOM or product, quantity,<br/>and a required "Responsible person" (notified on deadline breach)
    FE->>API: POST /api/production/orders {..., responsibleUserId}
    API->>API: validate responsibleUserId is an active user in caller's org
    API->>DB: INSERT production_orders (responsible_user_id, ...)
    API->>DB: INSERT production_order_audit_log (eventType=CREATED)
    API-->>FE: order created
```

## Assigning a step to an internal worker OR a vendor

```mermaid
sequenceDiagram
    actor M as Management/Admin
    participant FE as Frontend (assignment dialog —<br/>Internal worker / Vendor toggle)
    participant API as Backend ProductionPhase2Controller
    participant DB as PostgreSQL

    M->>FE: choose assignee type
    alt Internal worker
        FE->>FE: show user picker
        M->>FE: pick user + deadline (datetime)
    else Third-party vendor
        FE->>FE: show vendor picker (from GET /api/vendors/active)
        M->>FE: pick vendor + deadline (datetime)
    end
    FE->>FE: client-side validate exactly one of user/vendor is set
    FE->>API: POST /api/production/orders/{id}/steps/{stepId}/assignments<br/>{assigneeUserId | vendorId, deadline}
    API->>API: DB check constraint + service validation:<br/>reject if both or neither set (400)
    API->>DB: close prior active assignment for this order/step/role (if any)
    API->>DB: INSERT production_assignments (new row)
    API->>DB: INSERT production_order_audit_log<br/>(eventType=ASSIGNED | REASSIGNED | VENDOR_HANDOFF)
    API-->>FE: updated assignment
```

## Deadline breach detection & notification

```mermaid
sequenceDiagram
    participant SCHED as ProductionDeadlineBreachScheduler (daily)
    participant NOTIF as ProductionDeadlineBreachNotifier<br/>(separate bean — avoids self-invocation<br/>bypassing @Transactional)
    participant DB as PostgreSQL
    participant MAIL as EmailService (best-effort)

    loop Daily
        SCHED->>DB: find assignments WHERE deadline < now()-1day<br/>AND order NOT IN (COMPLETED, PARTIALLY_COMPLETED, CANCELLED)<br/>AND deadline_breach_notified_at IS NULL
        loop each overdue assignment
            SCHED->>NOTIF: notify(assignment)
            NOTIF->>DB: resolve order.responsible_user_id (same org)
            NOTIF->>MAIL: send breach email (order ref, step, assignee/vendor,<br/>original deadline, days overdue)
            alt email succeeds
                NOTIF->>DB: SET deadline_breach_notified_at = now()
                NOTIF->>DB: INSERT production_order_audit_log (eventType=DEADLINE_BREACHED)
            else email fails
                NOTIF->>NOTIF: log failure, continue to next assignment<br/>(eligible for retry next run — not marked notified)
            end
        end
    end
```

Note: no manager/reporting-line concept exists in `User`/`Employee` today — the
email goes only to the order's `responsibleUserId`. Flagged as a known gap if
per-assignee-manager escalation is needed later.

## Jira-style kanban board (drag-and-drop)

```mermaid
sequenceDiagram
    actor U as Management/Admin
    participant FE as Frontend (KanbanBoard.tsx, @dnd-kit)
    participant API as Backend ProductionController

    U->>FE: toggle view mode
    alt Fixed view (default)
        FE->>FE: columns = To Do / In Progress / Done
    else Step view
        FE->>FE: columns = union of orders' current workflow step names,<br/>ordered by step sequence number
    end

    U->>FE: drag a card to another column
    FE->>FE: validate transition is forward-only (no skipping/backward)
    alt valid transition
        FE->>API: PUT .../steps/{stepId}/start  (or complete, per target column)
        API-->>FE: updated order/step state
        FE->>FE: card moves, audit log updated server-side
    else invalid transition
        FE->>FE: toast explanation, snap card back — no API call made
    end

    U->>FE: click a card to open order modal
    FE->>FE: Execution tab shows PartialCompletionForm
    FE->>API: POST .../steps/{stepId}/record<br/>{completedQuantity, rejectedQuantity}
    API-->>FE: updated card/modal quantities<br/>(record only, does not advance)
    U->>FE: click Move to next step / Complete step
    FE->>API: POST .../steps/{stepId}/complete
    API-->>FE: updated order if current step's<br/>completed + rejected total reaches planned quantity

    Note over FE: Cards lazily fetch their own assignment via the existing<br/>per-order assignments query (no bulk endpoint exists) to show<br/>overdue badges and vendor-vs-worker avatars.
```

Layout note: the current production orders tab supports **board/list** views,
and board columns use a wrapping grid so the workspace can stay full-width
without the old always-horizontal board layout.

## Audit trail view

```mermaid
sequenceDiagram
    actor U as Management/Admin
    participant FE as Frontend (order detail — Audit trail tab)
    participant API as Backend ProductionController

    U->>FE: open an order, switch to "Audit trail" tab
    FE->>API: GET /api/production/orders/{orderId}/audit-log?page=&size=
    API-->>FE: paginated AuditLogResponse[] (newest-first)
    FE->>FE: render timeline: icon/color per eventType,<br/>actor (or "System" for scheduler-triggered events),<br/>relative timestamp, human-readable summary from details
    U->>FE: scroll / load more
    FE->>API: next page
```

## Employee "My Orders" self-service view

```mermaid
sequenceDiagram
    actor SA as SaaS/Org Owner
    participant SFE as Frontend (Organization settings)
    participant API as Backend OrganizationSettingsController
    actor E as Employee
    participant EFE as Frontend (My Orders page)
    participant PAPI as Backend ProductionController

    SA->>SFE: toggle "Allow employees to update their own step progress"
    SFE->>API: PUT /api/organization/settings {employeeSelfProgressUpdateEnabled: true}
    API->>API: persist per-org setting (default false)

    E->>EFE: open "My Orders"
    EFE->>PAPI: GET /api/production/my-assignments
    PAPI-->>EFE: assignments for caller's userId across all orders<br/>(order ref, product, step, deadline, status, quantities)
    alt org setting enabled
        EFE->>EFE: show inline PartialCompletionForm (same component as kanban card)
        E->>PAPI: start / log partial / complete own assignment
        PAPI->>PAPI: authorize: EMPLOYEE role + own active assignment<br/>+ org setting enabled (else 403)
    else org setting disabled
        EFE->>EFE: read-only status view, no action controls
    end
```
