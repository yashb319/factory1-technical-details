# Sequence: India Payroll Statutory Engine & Payslip Sharing

This covers the current backend statutory payroll and payslip modules plus the
frontend surfaces that exist in the current main checkout. PF/TDS is opt-in per
organization and employee; payslip templates/share links are implemented in the
backend, while the current frontend main branch exposes statutory settings,
employee statutory profiles, and payroll payslip statutory breakdowns but does
not yet include a payslip-template admin panel, "Share securely" action, or
unauthenticated `/payslip/[token]` viewer route.

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
