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

    PRODUCT ||--o{ BOM_LINE : "consumes"
    PRODUCTION_ORDER ||--o{ PRODUCTION_ORDER_STEP : "executes"
    PRODUCTION_ORDER_STEP ||--o| PRODUCTION_ORDER_STEP_SNAPSHOT : "snapshots"
    PRODUCTION_ORDER }o--|| PRODUCT : "produces"
    PRODUCTION_ORDER ||--o{ PRODUCTION_INVENTORY_POSTING : "posts stock via"

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
  **V67** (sandbox trial orgs).
