# Sequence: India Payroll Statutory Engine & Payslip Sharing

This covers the current backend statutory payroll and payslip modules together
with the frontend surfaces that drive them. PF/TDS is opt-in per organization and
per employee. Payslip templates, secure share links, and public token viewing are
shipped end to end: organization settings host the templates admin panel and
delivery/link policy, the payroll payslip dialog exposes "Share securely", and
`/payslip/[token]` renders the unauthenticated viewer.

## PF/TDS settings, employee profile, and payroll generation

```mermaid
sequenceDiagram
    actor Admin as Owner/Admin
    actor HR as HR/Admin
    participant FE as Frontend<br/>settings + employee + payroll
    participant ORG as OrganizationSettingsController
    participant EMP as EmployeeStatutoryProfileController
    participant PAY as PayrollController
    participant SVC as PayrollServiceImpl
    participant STAT as PayrollStatutoryCalculationServiceImpl
    participant PF as PfCalculator
    participant TDS as TdsCalculator
    participant DB as PostgreSQL

    Admin->>FE: Enable statutory payroll toggles
    FE->>ORG: PUT /api/organization/settings<br/>{pfEnabled, payrollTdsEnabled}
    ORG->>DB: UPDATE organization_settings<br/>(defaults are false)

    HR->>FE: Maintain employee statutory details
    FE->>EMP: POST/PUT /api/employees/{employeeId}/statutory-profile
    EMP->>DB: UPSERT employee_statutory_profiles<br/>(PAN, UAN, PF account, PF type,<br/>tax regime, declared income/deductions)

    HR->>FE: Generate payroll run
    FE->>PAY: POST /api/payroll {month, year}
    PAY->>SVC: generatePayroll(request)
    SVC->>DB: Load active employees, attendance,<br/>organization settings, statutory profiles

    loop each active employee
        SVC->>SVC: Run legacy salary calculator<br/>(hourly/daily/monthly)
        SVC->>STAT: calculate(employee, profile, settings, payrollItem)
        STAT->>DB: Load prior FY payroll items<br/>for projected tax and prior TDS
        STAT->>DB: Resolve effective tax_rules/tax_slabs/<br/>tax_rebate_rules/cess_rules/surcharge_rules
        STAT->>DB: Resolve effective pf_rules
        STAT->>PF: calculate PF if org pfEnabled<br/>and employee profile pfEnabled
        STAT->>TDS: calculate TDS if org payrollTdsEnabled
        STAT-->>SVC: PayrollStatutoryCalculation<br/>with version + JSON snapshot
        SVC->>DB: INSERT payroll_items<br/>(employeePf, voluntaryPf, tds, netSalary)
        SVC->>DB: INSERT payroll_statutory_calculations<br/>(PF/TDS amounts, rule refs, snapshot)
    end

    PAY-->>FE: payroll run detail
    FE->>FE: Payroll payslip dialog shows<br/>Employee PF, Voluntary PF, TDS,<br/>and expandable calculation details
```

## Statutory rule and audit tables

```mermaid
erDiagram
    TAX_RULE ||--o{ TAX_SLAB : has
    TAX_RULE ||--o{ TAX_REBATE_RULE : has
    TAX_RULE ||--o{ CESS_RULE : has
    TAX_RULE ||--o{ SURCHARGE_RULE : has
    PF_RULE ||--o{ PAYROLL_STATUTORY_CALCULATION : "referenced by"
    TAX_RULE ||--o{ PAYROLL_STATUTORY_CALCULATION : "referenced by"
    EMPLOYEE_STATUTORY_PROFILE ||--o{ PAYROLL_STATUTORY_CALCULATION : "inputs"
    PAYROLL_ITEM ||--o| PAYROLL_STATUTORY_CALCULATION : "has"
```

The statutory audit row stores the numbers shown in the payslip breakdown:
`pfWages`, `employeePf`, `voluntaryPf`, `employerPf`, `employerEpf`, `eps`,
`edli`, `adminCharge`, `projectedAnnualIncome`, `taxableIncome`, `baseTax`,
`annualTax`, `taxAlreadyDeducted`, `currentMonthTds`, `taxRuleId`, `pfRuleId`,
`calculationVersion`, and `calculationSnapshotJson`.

## Template administration (organization settings)

```mermaid
sequenceDiagram
    actor Admin as Owner/Admin/Management
    participant FE as PayslipTemplatesPanel +<br/>PayslipTemplateFormDialog
    participant API as PayslipTemplateController
    participant DB as PostgreSQL

    Admin->>FE: Open Organization settings → Payslip Templates
    FE->>API: GET /api/organization/payslip-templates
    API-->>FE: templates with status + version
    Note over FE: Empty state explains a system default<br/>is used until a template is created

    Admin->>FE: Create or edit a draft (form-based template data)
    FE->>API: POST or PUT /api/organization/payslip-templates
    API->>DB: Persist DRAFT template version

    Admin->>FE: Publish draft
    FE->>API: POST /api/organization/payslip-templates/{id}/publish
    API->>DB: status = PUBLISHED

    Admin->>FE: Set default / start new draft version
    FE->>API: POST .../set-default or .../new-draft-version
    API->>DB: Enforce one default per org<br/>or clone published version into a new draft
```

## Payslip generation, secure share link, and public access

```mermaid
sequenceDiagram
    actor Fin as Finance/Admin
    actor Emp as Employee/Public recipient
    participant PAY as PayrollServiceImpl
    participant GEN as PayslipGenerationServiceImpl
    participant NOTIF as PayslipNotificationServiceImpl
    participant SHARE as PayslipShareLinkServiceImpl
    participant PUB as PublicPayslipController
    participant MAIL as EmailService
    participant SMS as NoopSmsSender / NoopWhatsAppSender
    participant DB as PostgreSQL

    Fin->>PAY: PUT /api/payroll/{id}/approve
    PAY->>DB: Mark payroll run APPROVED

    loop each payroll item
        PAY->>GEN: generateForItem(payrollItem)<br/>(REQUIRES_NEW)
        GEN->>DB: Resolve default/published payslip template<br/>or seed Standard Payslip default
        GEN->>DB: INSERT payslips<br/>(template version + payslip_data_json snapshot)
        PAY->>NOTIF: notifyEmployee(payslip)<br/>(REQUIRES_NEW, best effort)
        NOTIF->>DB: Read delivery/share settings
        alt no delivery channel or share links disabled
            NOTIF-->>PAY: skip
        else delivery enabled
            NOTIF->>SHARE: create(payslipId, defaults)
            SHARE->>DB: INSERT payslip_access_tokens<br/>(SHA-256 token hash, expiry,<br/>max views, optional password hash)
            alt email enabled and employee email exists
                NOTIF->>MAIL: send secure link email
            end
            alt SMS/WhatsApp enabled and phone exists
                NOTIF->>SMS: log no-op send<br/>(no real provider integration)
            end
        end
    end

    Emp->>PUB: POST /api/public/payslips/{token}<br/>{optional password}
    PUB->>PUB: Rate-limit by token fingerprint + IP
    PUB->>SHARE: resolve(raw token, password)
    SHARE->>DB: Find token by SHA-256 hash
    SHARE->>SHARE: Validate not revoked, not expired,<br/>max views not exhausted,<br/>password matches if required
    alt valid
        SHARE->>DB: Increment view_count
        SHARE-->>PUB: payslip + template
        PUB-->>Emp: Structured JSON<br/>(templateData + payslip data)
    else unknown/expired/revoked/exhausted/wrong password
        PUB-->>Emp: 404 "This payslip link is invalid or has expired"
    end
```

## Manual sharing and public viewing in the UI

```mermaid
sequenceDiagram
    actor Fin as Finance/Admin
    actor Emp as Employee/Public recipient
    participant DLG as PayrollPayslipDialog +<br/>ShareLinkPanel
    participant VIEW as /payslip/[token] →<br/>PublicPayslipViewer
    participant API as Backend

    Fin->>DLG: Open a payroll item's payslip
    DLG->>API: GET /api/organization/payslips?employeeId=...
    Fin->>DLG: "Share securely" → optionally customize<br/>expiry days, max views, password
    DLG->>API: POST /api/organization/payslips/{id}/share-link
    API-->>DLG: token URL + expiry + max views + password flag
    DLG->>DLG: Show one-time-reveal warning and copy button<br/>("shown only once and cannot be retrieved again")

    Fin->>Emp: Send the link out of band
    Emp->>VIEW: Open /payslip/{token}
    VIEW->>API: POST /api/public/payslips/{token}<br/>(auto-attempt without password)
    alt access granted
        API-->>VIEW: templateData + payslipData
        VIEW->>VIEW: Render the template sections,<br/>net pay in words, browser Print action
    else rejected
        API-->>VIEW: generic 404
        VIEW->>VIEW: Show one combined message covering<br/>invalid/expired/password-required, then<br/>offer a password field and retry
    end
```

Because the backend deliberately returns the same generic 404 for every failure
mode, the viewer cannot distinguish "wrong password" from "expired" or "unknown
token" — so it intentionally shows a single combined message and offers a
password retry rather than guessing the cause.

## Default template shape

`PayslipTemplateDefaults.DEFAULT_TEMPLATE_JSON` defines the built-in template
seeded for an organization when no template exists:

| Section | Shape |
|---|---|
| `header` | `showLogo`, `companyName`, `companyAddress` |
| `employeeInfo` | `fields` array (`employee.name`, `employee.employeeCode`, `employee.designation`, `employee.department`, `payPeriod`) |
| `earnings` | rows for `basic`, `grossSalary`, `overtimeAmount` |
| `deductions` | rows for `employeePf`, `voluntaryPf`, `tds` |
| `employerContributions` | rows for `employerPf`, `eps`, `edli`, `adminCharge` |
| `netPay` | `field: netSalary`, `amountInWords: true` |
| `signatureBlock` | `showSignature`, `text` |

Supported placeholders include employee identity fields, organization name and
address, pay period, earnings, statutory deductions, employer PF components, and
net salary. The backend returns these as structured JSON snapshots; it does not
generate PDF bytes.
