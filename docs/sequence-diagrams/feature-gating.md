# Sequence: Feature Gating

Controls which modules/screens an organization can access based on its plan
and any SaaS-owner overrides — independent of the (frontend-only, always-off-
by-default) Tally UI flag.

```mermaid
sequenceDiagram
    actor SA as SaaS Owner
    participant AFE as Frontend (saas-admin feature-gate page)
    participant API as Backend SaasFeatureController /<br/>OrganizationFeatureController
    participant FILT as FeatureGateFilter
    participant DB as PostgreSQL
    actor U as Org user
    participant FE as Frontend (any authenticated page)

    SA->>AFE: define feature catalog (production_tracking, leave_management,<br/>inventory, billing, accounting, payroll, ai_assistant, ...)
    AFE->>API: POST /api/saas-admin/features (catalog CRUD)
    API->>DB: INSERT/UPDATE feature_catalog

    SA->>AFE: override a specific org's features<br/>(e.g. disable ai_assistant for a trial org)
    AFE->>API: PUT /api/organizations/features/{orgId}/overrides
    API->>DB: INSERT/UPDATE organization_feature_overrides

    U->>FE: navigate to a gated page (e.g. /production)
    FE->>API: any request under that module's API path
    API->>FILT: FeatureGateFilter runs (skipped for /api/partner/**, /api/public/**)
    FILT->>DB: resolve effective feature set =<br/>org's plan defaults + organization_feature_overrides
    alt feature disabled for this org
        FILT-->>FE: 403 feature-gated response
        FE->>FE: FeatureGate component shows<br/>"not included in your plan" / upgrade prompt
    else feature enabled
        FILT->>API: request proceeds normally
        API-->>FE: normal response
    end
```

## Relationship to the Tally UI flag

`NEXT_PUBLIC_ENABLE_TALLY_UI` is a **separate, frontend-only, build-time/env
flag** (default `false`) — it is not part of the backend feature-gating system.
It exists purely because the Tally-style UI mode was half-built and needed to
be hidden from all users regardless of plan, not because of any per-org
entitlement.
