# Sequence: White-Label Branding Resolution

Two independent resolution paths exist because a hostname alone cannot identify
a tenant on a **shared** domain (many orgs share `factory1.in`).

```mermaid
sequenceDiagram
    participant FE as Frontend (useActiveBranding hook)
    participant PUB as Backend PublicWhitelabelController
    participant AUTH as Backend CurrentOrganizationBrandingController
    participant DB as PostgreSQL

    Note over FE: On every page load, before login is known
    FE->>FE: read window.location.hostname
    FE->>PUB: GET /api/public/whitelabel/by-domain?domain=<hostname>
    alt hostname matches a SUBDOMAIN or CUSTOM_DOMAIN org
        PUB->>DB: SELECT organization_branding WHERE domain_value = hostname
        PUB-->>FE: {success:true, data:{logo, colors, ...}}
        FE->>FE: apply branding immediately (works even pre-login)
    else hostname is the shared base domain (factory1.in) or unmatched
        PUB-->>FE: {success:false, data:null}
        FE->>FE: fall back to hardcoded Factory1 default branding for now
    end

    Note over FE: After login, a token exists
    FE->>AUTH: GET /api/whitelabel/branding/me (authenticated)
    AUTH->>DB: resolve organization_branding via caller's organizationId<br/>(works regardless of SHARED/SUBDOMAIN/CUSTOM_DOMAIN)
    AUTH-->>FE: {success:true, data:{...}} or {data:null} if none configured
    FE->>FE: prefer this result once available —<br/>this is what actually fixes shared-domain branding,<br/>since hostname alone is ambiguous there
```

## Domain type UX (SaaS/Partner admin editing branding)

```mermaid
sequenceDiagram
    actor A as SaaS Owner / Partner Admin
    participant FE as Frontend (Whitelabel admin page)
    participant API as Backend SaasAdminWhitelabelController /<br/>PartnerWhitelabelController
    participant DB as PostgreSQL

    A->>FE: select domain type: SUBDOMAIN | CUSTOM_DOMAIN | SHARED
    alt SHARED selected
        FE->>FE: clear + disable the domain value field,<br/>display the REAL configured shared domain<br/>(env NEXT_PUBLIC_FACTORY1_SHARED_DOMAIN),<br/>not a hardcoded example like acme.factory1.in
    else SUBDOMAIN or CUSTOM_DOMAIN selected
        FE->>FE: enable domain value field, show a real example
    end
    A->>FE: edit colors/logo, save
    FE->>API: PUT /api/{saas-admin|partner}/whitelabel/organizations/{id}/branding
    API->>API: server ALWAYS derives/overwrites domain_verified and active —<br/>ignores any value the caller sends for these two fields
    API->>DB: UPDATE organization_branding
    API-->>FE: updated branding
    Note over FE: Because color edits are read back via<br/>GET /api/whitelabel/branding/me (org-scoped, not hostname-scoped),<br/>changes to a SHARED-domain org's branding now reflect correctly in-app.
```
