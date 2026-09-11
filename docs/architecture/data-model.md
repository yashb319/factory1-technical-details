# Core Data Model & Multi-Tenancy

## Tenancy model

Factory1 uses a **single shared PostgreSQL database** with row-level tenant
isolation: nearly every domain table carries an `organization_id` column, and
every service/repository call is scoped by the caller's organization. There is
no schema-per-tenant or database-per-tenant separation.

Two categories of exception exist, both used carefully:

1. **Partner-admin users** are provisioned with a *synthetic placeholder*
   `organization_id` (a random UUID that doesn't correspond to a real org) because
   `users.organization_id` has **no foreign-key constraint** in the schema. This is
   safe only because `PARTNER_ADMIN` identity resolution goes through
   `partners.user_id`, never through `organization_id`, and both
   `OrganizationApprovalFilter` and `FeatureGateFilter` explicitly skip
   `/api/partner/**` routes. This is flagged in the codebase as a known
   architectural shortcut worth revisiting (see `docs/status/feature-status.md`).
2. **Sandbox trial organizations** are real organizations (`is_sandbox = true`)
   with a normal `organization_id`, just auto-active instead of pending approval,
   and auto-deleted by a scheduled job after expiry.

## Core entity relationships

```mermaid
erDiagram
    ORGANIZATION ||--o{ USER : "employs"
    ORGANIZATION ||--|| ORGANIZATION_SETTINGS : "has"
    ORGANIZATION ||--o{ EMPLOYEE : "has"
    ORGANIZATION ||--o{ ROLE : "scopes"
    ORGANIZATION ||--o| ORGANIZATION_PARTNER : "linked via"
    ORGANIZATION ||--o| ORGANIZATION_BRANDING : "customizes"
    PARTNER ||--o{ ORGANIZATION_PARTNER : "manages many"
    PARTNER ||--o| USER : "has one PARTNER_ADMIN user"

    ORGANIZATION {
        uuid id PK
        string name
        enum status "PENDING_APPROVAL | ACTIVE | ..."
        boolean is_sandbox
        timestamp sandbox_expires_at
        string gst_number
        string industry_type
    }

    USER {
        uuid id PK
        uuid organization_id "no FK constraint (see note above)"
        string email UK
        enum role "OWNER|ADMIN|FINANCE|MANAGEMENT|EMPLOYEE|SAAS_OWNER|PARTNER_ADMIN"
        string password_hash
    }

    PARTNER {
        uuid id PK
        string name
        string contact_email
        string code UK "auto-generated, e.g. KAMBAN"
        boolean active
        uuid user_id FK "nullable, unique -> users.id"
    }

    ORGANIZATION_PARTNER {
        uuid organization_id FK
        uuid partner_id FK
    }

    ORGANIZATION_BRANDING {
        uuid organization_id FK
        enum domain_type "SUBDOMAIN|CUSTOM_DOMAIN|SHARED"
        string domain_value
        boolean domain_verified "server-controlled only"
        boolean active "server-controlled only"
        string logo_url
        string primary_color
    }

    EMPLOYEE ||--o{ ATTENDANCE_RECORD : "clocks"
    EMPLOYEE ||--o{ LEAVE_REQUEST : "requests"
    EMPLOYEE ||--o{ PAYROLL_RUN_LINE : "paid via"
    EMPLOYEE ||--o| EMPLOYEE_STATUTORY_PROFILE : "has"
    PAYROLL_RUN_LINE ||--o| PAYROLL_STATUTORY_CALCULATION : "audits PF/TDS"
    PAYROLL_RUN_LINE ||--o| PAYSLIP : "snapshotted into"
    PAYSLIP_TEMPLATE ||--o{ PAYSLIP : "renders"
    PAYSLIP ||--o{ PAYSLIP_ACCESS_TOKEN : "shared by"

    PRODUCT ||--o{ BOM_LINE : "consumes"
    PRODUCTION_ORDER ||--o{ PRODUCTION_ORDER_STEP : "executes"
    PRODUCTION_ORDER_STEP ||--o| PRODUCTION_ORDER_STEP_SNAPSHOT : "snapshots"
    PRODUCTION_ORDER }o--|| PRODUCT : "produces"
    PRODUCTION_ORDER ||--o{ PRODUCTION_INVENTORY_POSTING : "posts stock via"
    PRODUCTION_ORDER }o--|| USER : "has responsible_user_id"
    PRODUCTION_ORDER_STEP ||--o{ PRODUCTION_ASSIGNMENT : "assigned via"
    PRODUCTION_ASSIGNMENT }o--o| USER : "assignee_user_id (XOR vendor_id)"
    PRODUCTION_ASSIGNMENT }o--o| VENDOR : "vendor_id (XOR assignee_user_id)"
    PRODUCTION_ORDER ||--o{ PRODUCTION_ORDER_AUDIT_LOG : "logs every lifecycle event"

    PRODUCTION_ASSIGNMENT {
        uuid id PK
        uuid assignee_user_id FK "nullable"
        uuid vendor_id FK "nullable"
        timestamp deadline
        timestamp deadline_breach_notified_at "set once, prevents repeat emails"
    }

    VENDOR {
        uuid id PK
        uuid organization_id FK
        string name
        string contact_email
        string contact_phone
        string service_type "free text, e.g. Embroidery, Cutting"
        boolean active
    }

    PRODUCTION_ORDER_AUDIT_LOG {
        uuid id PK
        uuid production_order_id FK
        uuid order_step_snapshot_id FK "nullable"
        uuid assignment_id FK "nullable"
        enum event_type "CREATED|ASSIGNED|REASSIGNED|VENDOR_HANDOFF|STEP_STARTED|PARTIAL_COMPLETE|STEP_COMPLETED|DEADLINE_BREACHED|..."
        uuid actor_user_id "nullable - null for system-triggered events"
        timestamp occurred_at
        text details "JSON/text metadata, event-specific"
    }

    EMPLOYEE_STATUTORY_PROFILE {
        uuid id PK
        uuid organization_id
        uuid employee_id FK
        string pan_number
        string uan
        string pf_account_number
        boolean pf_enabled
        enum pf_calculation_type "STATUTORY_CEILING|ACTUAL_WAGES|CUSTOM"
        enum tax_regime "OLD|NEW"
        numeric previous_employer_income
        numeric declared_deductions_total
    }

    TAX_RULE ||--o{ TAX_SLAB : "contains"
    TAX_RULE ||--o{ TAX_REBATE_RULE : "contains"
    TAX_RULE ||--o{ CESS_RULE : "contains"
    TAX_RULE ||--o{ SURCHARGE_RULE : "contains"
    TAX_RULE ||--o{ PAYROLL_STATUTORY_CALCULATION : "referenced by"
    PF_RULE ||--o{ PAYROLL_STATUTORY_CALCULATION : "referenced by"

    TAX_RULE {
        uuid id PK
        string financial_year
        enum regime "OLD|NEW"
        date effective_from
        date effective_to
        numeric standard_deduction_amount
    }

    PF_RULE {
        uuid id PK
        date effective_from
        date effective_to
        numeric wage_ceiling
        numeric employee_rate
        numeric employer_rate
        numeric eps_rate
        numeric edli_rate
        numeric admin_charge_rate
    }

    PAYROLL_STATUTORY_CALCULATION {
        uuid id PK
        uuid payroll_item_id FK
        uuid employee_id FK
        numeric employee_pf
        numeric voluntary_pf
        numeric employer_pf
        numeric current_month_tds
        uuid tax_rule_id FK
        uuid pf_rule_id FK
        string calculation_version
        text calculation_snapshot_json
    }

    PAYSLIP_TEMPLATE {
        uuid id PK
        uuid organization_id
        uuid logical_template_id
        int version
        enum status "DRAFT|PUBLISHED|ARCHIVED"
        boolean default_template
        json template_data
    }

    PAYSLIP {
        uuid id PK
        uuid organization_id
        uuid employee_id FK
        uuid payroll_run_id FK
        uuid payroll_item_id FK
        uuid template_id FK
        int template_version
        int pay_period_month
        int pay_period_year
        text payslip_data_json
    }

    PAYSLIP_ACCESS_TOKEN {
        uuid id PK
        uuid payslip_id FK
        string token_hash
        timestamp expires_at
        int max_views
        int view_count
        boolean password_required
        timestamp revoked_at
    }

    SAAS_PLAN ||--o{ SAAS_ADDON : "offered alongside"
    ORGANIZATION }o--|| SAAS_PLAN : "subscribes to"
```

## Notable schema/design decisions

- **`BaseEntity`**: all entities extend a common base providing `id`, audit
  timestamps, and `organizationId` where applicable. A past bug (fixed) had
  `onCreate()` unconditionally overwriting manually-assigned IDs — now it only
  sets the ID when null.
- **Production inventory postings** use a unique `(organization_id, source_type,
  source_id)` constraint as an idempotency ledger, so a retried "accept output"
  action never double-posts stock.
- **`production_orders.current_step_id` ↔ `production_order_step_snapshots`**
  has a circular FK — the sandbox cleanup job must null the FK before deleting,
  a real schema wrinkle worth simplifying eventually.
- **Flyway** governs every schema change; migrations are strictly append-only
  (never renumber/reuse a version — this caused a real deployment collision
  once, fixed by renaming `V63` → `V64`). As of writing, the schema is at
  **V71** (payroll statutory foundation in `V70`, payslip templates/delivery in
  `V71`).
- **`production_assignments` vendor-or-user XOR**: `assignee_user_id` and
  `vendor_id` are both nullable, enforced mutually exclusive by a DB check
  constraint (`ck_production_assignment_target`) plus service-level
  validation — an assignment targets exactly one of an internal `User` or a
  `Vendor`, never both/neither.
- **`production_order_audit_log` is deliberately a separate table from
  `production_events`** — `production_events` is a retryable notification
  *outbox* (delivery status/retries/idempotency for emails), not an immutable
  business audit trail; conflating the two would have made audit history
  dependent on notification delivery state.
- **Production progress is derived per step from execution rows** — the modal
  execution flow records completed/rejected quantities with `/record`; `/complete`
  can advance only after that step's execution rows account for the full planned
  quantity. Order-level completed/rejected totals are updated only from final-step
  accepted/rejected output.
- **Payroll statutory calculations are audited per payroll item** —
  `payroll_statutory_calculations` stores PF/TDS amounts, referenced rule rows,
  a calculation version, and a JSON snapshot so a payslip can explain how the
  statutory deductions were produced.
- **Payslip links never store raw tokens** — `payslip_access_tokens` stores a
  SHA-256 hash, expiry/max-view/password controls, and public access deliberately
  returns the same generic 404 for unknown, expired, revoked, exhausted, or
  password-failed links.
- **Deadline breach scheduler bug found & fixed during implementation**: the
  original scheduler called its own `@Transactional` method via `forEach`,
  which bypasses the Spring AOP proxy and silently drops the transaction. Fixed
  by extracting a separate `ProductionDeadlineBreachNotifier` bean so
  self-invocation isn't possible.
