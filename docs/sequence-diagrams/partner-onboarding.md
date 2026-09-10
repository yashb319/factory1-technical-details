# Sequence: Partner Onboarding & Auto-Provisioning

Covers how a SaaS owner creates a reseller "partner," how that partner gets an
auto-generated login + code, how a factory links itself to that partner at
signup, and how the public code-validation endpoint works.

## Partner creation (SaaS Admin UI)

```mermaid
sequenceDiagram
    actor SA as SaaS Owner
    participant FE as Frontend (SaasWhitelabelPage — Add Partner dialog)
    participant API as Backend WhitelabelAdminServiceImpl
    participant DB as PostgreSQL
    participant MAIL as EmailService (best-effort)

    SA->>FE: enter partner name + contact email only
    FE->>API: POST /api/saas-admin/whitelabel/partners {name, contactEmail}
    API->>API: derive unique code from name<br/>(first3+first3 of two words, e.g. "Kamal Bansal" -> "KAMBAN";<br/>collision -> counter suffix "KAMBAN2"; single word -> first6;<br/>random suffix only for fully-symbolic names)
    API->>DB: INSERT partners {name, contactEmail, code, active=true}
    API->>API: generate SecureRandom temp password
    API->>DB: INSERT users {role=PARTNER_ADMIN, organization_id=<synthetic UUID>,<br/>password=encode(tempPassword)}
    API->>DB: UPDATE partners SET user_id = new user's id
    API->>MAIL: send invite email (name, code, temp password, login instructions)
    Note over MAIL: Reuses existing forgot-password OTP flow<br/>for the partner's first real password reset —<br/>no new activation-token mechanism was built.
    API-->>FE: {partner, generatedCode}
    FE->>FE: show copyable success dialog with code + admin login email
```

## Factory signup with a partner code

```mermaid
sequenceDiagram
    actor U as Factory owner (signing up)
    participant FE as Frontend (SignupForm — optional "Partner code" field)
    participant API as Backend PublicWhitelabelController / AuthController
    participant DB as PostgreSQL

    U->>FE: types partner code (optional, 3+ chars)
    FE->>FE: debounce 400ms
    FE->>API: GET /api/public/whitelabel/partner-code?code=XXXX
    API->>DB: lookup partners WHERE code=XXXX AND active=true
    API-->>FE: {success:true, data:{valid, partnerName}} (always HTTP 200,<br/>never leaks partner id/email/userId)
    FE->>FE: show "Linked to <partnerName>" (valid) or<br/>"Code not recognized" (invalid) — never blocks submit

    U->>FE: submits signup with partnerCode (trimmed+uppercased)
    FE->>API: POST /api/auth/register {..., partnerCode}
    API->>API: linkPartnerIfPresent(organizationId, partnerCode)
    API->>DB: if valid+active -> INSERT organization_partners {organizationId, partnerId}
    Note over API: Silently no-ops if code is missing/invalid —<br/>never blocks or errors the signup.
```

## Partner-scoped restricted access

```mermaid
sequenceDiagram
    actor P as Partner (PARTNER_ADMIN)
    participant FE as Frontend (partner/whitelabel page)
    participant API as Backend PartnerWhitelabelController
    participant SEC as SecurityConfig + filters
    participant DB as PostgreSQL

    P->>FE: log in with partner admin email + password
    FE->>API: GET /api/partner/whitelabel/organizations
    API->>SEC: route matches /api/partner/** -> requires PARTNER_ADMIN role
    SEC->>SEC: OrganizationApprovalFilter & FeatureGateFilter skip this path<br/>(shouldNotFilter excludes /api/partner/**)
    API->>DB: resolve partner via partners.user_id = caller's user id<br/>(never via organization_id, since it's synthetic)
    API->>DB: SELECT organizations JOIN organization_partners WHERE partner_id = ...
    API-->>FE: list of linked organizations + their branding
    Note over API: Partner can edit branding display fields,<br/>but the server always derives/preserves<br/>domain_verified and active — a partner can never<br/>self-verify a domain or reactivate branding.
```
