# Sequence: Auth, Signup & Approval

Covers the standard (non-sandbox) organization signup flow, login, and the
approval gate that blocks new organizations until a SaaS owner approves them.

## Standard signup → pending approval → active

```mermaid
sequenceDiagram
    actor U as Prospective customer
    participant FE as Frontend (SignupForm)
    participant API as Backend AuthController
    participant OTP as EmailOtpService
    participant DB as PostgreSQL

    U->>FE: Fill signup form (org details, email, optional partnerCode)
    FE->>API: POST /api/auth/signup-otp {email}
    API->>OTP: generate + email OTP
    OTP-->>U: OTP email
    U->>FE: Enter OTP
    FE->>API: POST /api/auth/register {..., otp, partnerCode?}
    API->>OTP: verifyOtp(email, otp, SIGNUP)
    OTP-->>API: valid
    API->>DB: INSERT organizations (status=PENDING_APPROVAL)
    API->>DB: INSERT organization_settings (defaults)
    alt partnerCode present & valid
        API->>DB: linkPartnerIfPresent() -> INSERT organization_partners
    end
    API->>DB: INSERT users (role=OWNER)
    API-->>FE: AuthResponse {token, user}
    FE->>FE: store token, redirect to app

    Note over FE,API: Every subsequent authenticated request passes through<br/>OrganizationApprovalFilter
    FE->>API: any authenticated request
    API->>API: OrganizationApprovalFilter checks org.status
    alt status == PENDING_APPROVAL
        API-->>FE: 403 {"success":false,"message":"ORGANIZATION_PENDING_APPROVAL"}
        FE->>FE: show "registration pending" screen
    else status == ACTIVE
        API-->>FE: normal response
    end

    actor SA as SaaS Owner
    SA->>API: PUT /api/saas-admin/factories/{id} (approve)
    API->>DB: UPDATE organizations SET status=ACTIVE
    Note over U,SA: Org can now be used normally. This manual step<br/>is exactly what the sandbox flow (see next doc) bypasses.
```

## Login

```mermaid
sequenceDiagram
    actor U as User
    participant FE as Frontend (LoginForm)
    participant API as Backend AuthController

    U->>FE: email + password
    FE->>API: POST /api/auth/login
    API->>API: verify password, check org status/role
    API-->>FE: AuthResponse {token, user}
    FE->>FE: store token (authSlice), redirect to dashboard
    Note over FE: No refresh token exists in this system today —<br/>a single long-lived access token is stored client-side.
```
