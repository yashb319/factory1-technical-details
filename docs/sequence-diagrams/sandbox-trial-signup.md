# Sequence: Sandbox Trial Signup

The fast, no-approval path added to let prospects try Factory1 instantly. Contrast
with [`auth-and-signup.md`](auth-and-signup.md) — no OTP, no manual approval, and
the org is pre-seeded with realistic demo data across every module.

```mermaid
sequenceDiagram
    actor U as Prospective customer
    participant FE as Frontend (homepage "Try it free" CTA)
    participant API as Backend SandboxSignupController
    participant SEED as SandboxDataSeedingService
    participant SVC as Existing domain services<br/>(inventory, production, billing, ...)
    participant DB as PostgreSQL
    participant SCHED as SandboxCleanupScheduler (daily)

    U->>FE: name, organizationName, email (no password, no OTP)
    FE->>API: POST /api/public/sandbox/signup
    API->>DB: check email not already used
    API->>DB: INSERT organizations (status=ACTIVE, is_sandbox=true,<br/>sandbox_expires_at=now()+7d)
    API->>DB: INSERT organization_settings (defaults)
    API->>DB: INSERT users (role=OWNER, random temp password, never returned)

    rect rgb(235, 245, 255)
    Note over API,SVC: Seeding runs inside the same transaction,<br/>impersonating the new user so real side effects happen
    API->>SEED: seed(organizationId)
    SEED->>SVC: create suppliers, customers
    SEED->>SVC: create inventory items + BOM
    SEED->>SVC: create employees + attendance history + leave + payroll run
    SEED->>SVC: create production workflow + orders<br/>(one completed -> triggers real inventory posting)
    SEED->>SVC: create bills (mixed paid/unpaid) -> real accounting postings
    SVC->>DB: writes via normal service logic (not hand-rolled inserts)
    end

    alt seeding fails
        API->>DB: ROLLBACK entire signup
        API-->>FE: error (no half-seeded sandbox left behind)
    else success
        API-->>FE: AuthResponse {token, user} (same shape as /register, /login)
        FE->>FE: store token, redirect straight into dashboard (no extra step)
    end

    Note over FE: App shell calls GET /api/organization/sandbox-status once per load
    FE->>API: GET /api/organization/sandbox-status
    API-->>FE: {isSandbox: true, expiresAt}
    FE->>FE: render persistent banner "Sandbox trial — expires in Nd"

    opt Owner/Admin clicks "Activate real account"
        U->>FE: confirm activation
        FE->>API: POST /api/organization/sandbox/convert
        API->>DB: UPDATE organizations SET is_sandbox=false,<br/>sandbox_expires_at=null, status=PENDING_APPROVAL
        Note over API: Demo data is NOT stripped (deliberate).<br/>Org now follows the normal manual-approval path.
        API-->>FE: success
        FE->>FE: hide sandbox banner
    end

    loop Daily at 03:00 Asia/Kolkata
        SCHED->>DB: find organizations WHERE is_sandbox AND sandbox_expires_at < now()
        SCHED->>DB: delete across ~20 tables in FK-safe order<br/>(nulls production_orders.current_step_id first to break circular FK)
        SCHED->>SCHED: log cleanup summary
    end
```

## Deliberate scope decisions (from the implementing session)

- No captcha/rate-limiting on the public signup endpoint yet — flagged as a follow-up.
- `ORG_ADMIN` maps to the existing `Role.OWNER` — no separate role was introduced.
- Seeding reuses **real domain services**, not raw inserts, so all side effects
  (stock decrements, ledger postings) are realistic and consistent with production behavior.
- Converting to a real account does not delete the seeded demo data.
