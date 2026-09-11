# Per-Module Reference

Quick reference for every backend package and frontend feature directory, refreshed
from the current local source checkouts:

- backend: `/Users/yashb319/IdeaProjects/factory1-backend` (`main`)
- frontend: `/Users/yashb319/IdeaProjects/factory1-frontend` (`main`)

The separate attendance-capture checkout named in earlier notes is not present
locally, so this file documents only the public attendance-capture backend API
and the management frontend currently available in the main frontend app.

## Backend packages

| Package | Purpose | Main API base | Has tests? |
|---|---|---|---|
| `auth` | JWT login/register, OTP flows, password reset, employee activation, user administration, roles | `/api/auth`, `/api/users` | ✅ `AuthServiceImplTest` |
| `organization` | Organization CRUD, role lookup, org settings, accounting/statutory/payslip toggles, attendance capture key, plan-change request, termination, sandbox status/convert | `/api/organizations`, `/api/organization/settings`, `/api/organization/sandbox-status`, `/api/organization/sandbox/convert` | ✅ `OrganizationSettingsDefaultsTest` |
| `sandbox` | Public sandbox signup, demo data, rate limiting, cleanup/read-only filters | `/api/public/sandbox` | ✅ sandbox integration/service/controller/rate-limit/filter tests |
| `employee` | Employee CRUD, employee self lookup, CSV import preview/import, invitations, statutory profile (PAN/UAN/PF/TDS profile) | `/api/employees`, `/api/employees/{employeeId}/statutory-profile` | ✅ statutory profile coverage only (`EmployeeStatutoryProfileServiceImplTest`) |
| `attendance` | Attendance records, bulk/device intake, dashboard/monthly reports, leave-status resolver, public capture lookup/intake | `/api/attendance`, `/api/public/attendance-capture` | ✅ resolver coverage (`AttendanceLeaveResolverTest`) |
| `leave` | Leave types, balances, requests, approvals/rejections/cancel, calendar, holidays, public email actions | `/api/leave`, `/api/public/leave/requests` | ✅ scheduler coverage (`LeaveBalanceSchedulerTest`) |
| `payroll` | Payroll run generation/search/detail/dashboard, approve/pay/cancel, salary calculators, PF/TDS statutory calculation wiring, payroll statutory audit rows | `/api/payroll` | ✅ statutory/wiring coverage (`PayrollServiceImplStatutoryWiringTest`, `PfCalculatorTest`, `TdsCalculatorTest`); legacy salary calculators still have limited direct coverage |
| `payslip` | Generated payslip snapshots, template versions/defaults, secure share links, public token access, delivery orchestration | `/api/organization/payslips`, `/api/organization/payslip-templates`, `/api/public/payslips` | ✅ generation/template/share-link/notification/rate-limit tests |
| `inventory` | Inventory items, stock movements, dashboard, bulk import | `/api/inventory` | ❌ |
| `product` | Product CRUD, product BOM CRUD, production-product entry point | `/api/products` | ❌ |
| `production` | Versioned workflows/BOMs, orders, modal execution flow, assignments, workstations, quality, materials, Kanban, analytics, notifications, audit trail, employee self-service progress | `/api/production`, `/api/production/notification-preferences` | ✅ broadest coverage (`ProductionServiceImplTest`, `ProductionPhase2ServiceImplTest`, output posting/progress auth/deadline/event tests) |
| `vendor` | Third-party production vendor CRUD, active list, dashboard, insights | `/api/vendors` | ✅ `VendorServiceImplTest` |
| `supplier` | Supplier CRUD, active list, dashboard, insights, bulk import | `/api/suppliers` | ❌ |
| `customer` | Customer CRUD, active list, dashboard, insights, bulk import | `/api/customers` | ❌ |
| `billing` | Bills, number suggestions/availability, posting/cancel/payment, GST suggestions/report, OCR extraction/templates, e-way action hooks | `/api/billing`, `/api/billing/ocr`, `/api/billing/ocr-templates` | ❌ |
| `ewaybill` | E-way bill preview/sync/cancel/vehicle/update, billing e-way actions, provider credential management | `/api/eway-bills`, `/api/eway-bills/integration`, `/api/billing/bills/{billId}/eway/*` | ❌ |
| `ai` | AI chat, local business insight drilldowns, listed-company benchmarks, action execution, usage ledger | `/api/ai` | ❌ |
| `accounting` | Masters, tax sections, groups, ledgers, vouchers, GST summary, trial balance, P&L, balance sheet, aging | `/api/accounting` | ✅ report/voucher tests |
| `feature` | Feature catalog, org feature overrides, effective org features, gate filter | `/api/saas-admin/features`, `/api/organizations/features` | ✅ admin/service/filter tests |
| `saasadmin` | SaaS owner dashboard, factory paid/delete operations, pricing plans/add-ons, public plans/offers, marketing sends, insights | `/api/saas-admin`, `/api/saas-admin/pricing`, `/api/public/plans`, `/api/public/add-ons`, `/api/public/offers` | ✅ pricing catalog coverage |
| `whitelabel` | Public domain/partner-code lookup, current-org branding, partner-managed branding, SaaS-owner branding/partner admin | `/api/public/whitelabel`, `/api/whitelabel/branding`, `/api/partner/whitelabel`, `/api/saas-admin/whitelabel` | ✅ public/current/partner/admin service tests |
| `importexport` | Generic import/export job tracking | `/api/import-export/jobs` | ❌ |
| `registration` | Public early-registration questionnaire and approval endpoints | `/api/public/early-registration` | ❌ |
| `notification` | Email service plus scheduled digests/renewals/sandbox/production notification plumbing | — background/service layer | ❌ |
| `dashboard` | Org summary and trend dashboard | `/api/dashboard` | ✅ `DashboardServiceImplTest` |
| `common` | Base entity, health ping, pagination, API response wrapper, exceptions, security helpers, utilities | `/api/ping` | ✅ `BaseEntityTest` |
| `config` | Spring Security, Swagger/OpenAPI, cache and application configuration | — | ❌ |

## Frontend feature directories

| Feature dir | Purpose | Backend module it talks to | Notable status |
|---|---|---|---|
| `features/auth` | Login, signup, OTP, forgot-password, employee activation, partner-code path | `auth`, `organization`, `sandbox` | Working; no frontend tests |
| `features/organization-settings` | Org settings, accounting toggles, PF/TDS toggles, employee production self-progress toggle, attendance capture key, plan/termination actions | `organization`, `payroll`, `production` | Working; payslip delivery/share settings are backend-only in current frontend main |
| `features/organization`, `features/access`, `features/organization-features` | Organization profile/access and effective feature-gate surfaces | `organization`, `feature` | Working/admin-oriented |
| `features/employees` | Employee CRUD/import/export, invitations, edit drawer, statutory profile form | `employee`, `employee/statutory` | Working; statutory form covers PAN/UAN/PF/tax-regime declaration data |
| `features/attendance` | Attendance dashboard/register/manual/bulk entry, monthly report/export, QR/capture-key management hooks | `attendance` | Working; no offline queue/service worker found |
| `features/leave` | Leave types/balances/requests/calendar/holidays | `leave` | Working; dense page |
| `features/payroll` | Payroll runs, generate/approve/pay/delete, details dialog, statutory deduction breakdown, HTML payslip preview, JPG/ZIP export | `payroll` | Working; no PDF rendering library and no frontend automated tests |
| `features/payslip` | Tenant payslip template/share/public viewer UI | `payslip` | **Not present in current frontend main**; backend APIs exist, but the local frontend checkout does not contain the admin panel, "Share securely" action, or `/payslip/[token]` viewer |
| `features/inventory` | Stock CRUD, stock movements, dashboard, import/export, Tally-mode variant | `inventory` | Working; delete UI disables items rather than hard-deleting |
| `features/products` | Product and product BOM management, production entry point, Tally-mode export view | `product` | Working |
| `features/production` | Production orders first, workflows, BOM definitions, workstations, analytics, modal order execution/details, board/list Kanban, assignments, quality, materials, timeline, audit trail, employee My Assignments | `production`, `vendor`, `inventory`, `product`, `customer` | Mature; current UX uses modal-based record/advance flow rather than Start/Pause-centric controls |
| `features/vendors` | Vendor CRUD/dashboard/insights | `vendor` | Working |
| `features/suppliers` | Supplier CRUD/dashboard/insights/bulk import/export | `supplier` | Working |
| `features/customers` | Customer CRUD/dashboard/insights/bulk import/export | `customer` | Working |
| `features/billing` | Bills, invoice print, OCR import dialog, e-way bill actions, GST suggestions/report | `billing`, `ewaybill` | Working surface with external-provider-dependent OCR/e-way edges |
| `features/gst-integration` | GST/e-way integration settings panel | `ewaybill`, `organization` | Working settings surface |
| `features/accounting` | Masters, ledgers, vouchers, reports, aging, GST summary, Tally-mode variant | `accounting` | Working; very dense/complex page |
| `features/ai` | Floating assistant, assistant page, insight drilldowns, export helper | `ai` | Experimental/heuristic |
| `features/public-pricing` | Public homepage pricing cards | `saasadmin` public pricing | Working, DB-driven |
| `features/saas-admin` | SaaS owner factories/dashboard/pricing/marketing/whitelabel/feature admin | `saasadmin`, `feature`, `whitelabel` | Working broad admin surface |
| `features/whitelabel` | Current-org branding, partner branding, SaaS-owner white-label admin helpers/hooks/config | `whitelabel` | Working |
| `features/dashboard` | Summary, trends, benchmarks/dashboard views | `dashboard`, `ai` | Working |
| `features/import-export` | Generic job table/hooks/types/CSV utilities | `importexport` | Foundational |
| `features/docs`, `features/help-center`, `features/legal` | In-app docs/help/legal content | static / mixed | Static content |
| `features/*/tally/*` | Alternate Tally-style UI per module | various | Disabled by default through `NEXT_PUBLIC_ENABLE_TALLY_UI=false` |
