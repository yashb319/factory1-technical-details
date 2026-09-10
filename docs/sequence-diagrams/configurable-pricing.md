# Sequence: Configurable SaaS Pricing

Plans and add-ons are fully DB-driven (Flyway V66 seeded 5 plans + 10 add-ons)
and manageable from SaaS Admin, with the public homepage rendering live data —
no hardcoded pricing in the frontend.

```mermaid
sequenceDiagram
    actor SA as SaaS Owner
    participant AFE as Frontend (SaaS Admin pricing catalog)
    participant API as Backend SaasPricingAdminController
    participant DB as PostgreSQL
    actor V as Website visitor
    participant PFE as Frontend (PublicPricingCards, homepage)
    participant PUB as Backend PublicPricingController

    SA->>AFE: create/edit a plan or add-on<br/>(pricing, badges, allowances, GST copy, ordering)
    AFE->>API: POST/PUT /api/saas-admin/pricing/plans (or /add-ons)
    API->>DB: INSERT/UPDATE saas_plans / saas_addons
    SA->>AFE: reorder whole list (drag/drop)
    AFE->>API: PUT /api/saas-admin/pricing/plans/reorder {orderedIds}
    API->>DB: UPDATE display_order for each row
    SA->>AFE: deactivate or hard-delete
    AFE->>API: DELETE /api/saas-admin/pricing/plans/{id}
    API->>DB: DELETE (confirmed hard delete, not soft-disable)

    Note over V,PUB: Public homepage has no build-time knowledge of pricing
    V->>PFE: visit homepage
    PFE->>PUB: GET /api/public/plans
    PFE->>PUB: GET /api/public/add-ons
    PFE->>PUB: GET /api/public/offers
    PUB->>DB: SELECT ... WHERE active=true ORDER BY display_order
    PUB-->>PFE: active, ordered plans/add-ons/offers only
    alt request fails (network/server error)
        PFE->>PFE: render minimal failure-only fallback<br/>(a "contact us" card) — never fabricates<br/>pricing or promises unlimited users
    else success
        PFE->>PFE: render plan cards with monthly/annual toggle,<br/>badges (Most Popular/Best Value), allowances,<br/>GST and scale copy, add-on cards
    end
```

## Legacy compatibility

`/api/public/plans` is kept as a **legacy-compatible alias** so any existing
integration or cached client referencing the old path keeps working while the
catalog itself is now fully configurable.
