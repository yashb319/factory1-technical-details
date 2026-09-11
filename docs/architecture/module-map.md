# Module Map — Backend ↔ Frontend

Maps each backend Java package to its frontend feature directory and the main
API base paths connecting them.

| Domain | Backend package | Frontend feature | Primary API base path(s) |
|---|---|---|---|
| Auth & onboarding | `auth` | `features/auth` | `/api/auth/*`, `/api/users` |
| Organization lifecycle & settings | `organization` | `features/organization-settings` | `/api/organizations`, `/api/organization/settings` |
| Sandbox trial | `organization` (fields) + `sandbox` signup path | homepage CTA + app-shell banner | `/api/public/sandbox/signup`, `/api/organization/sandbox-status`, `/api/organization/sandbox/convert` |
| Employees | `employee` | `features/employees` | `/api/employees` |
| Attendance | `attendance` | `features/attendance` (management) + external capture app (device) | `/api/attendance/*`, `/api/public/attendance-capture/*` |
| Leave | `leave` | `features/leave` | `/api/leave/*` |
| Payroll | `payroll` | `features/payroll` | `/api/payroll/*` |
| Employee statutory profiles | `employee.statutory` + `payroll.statutory` | `features/employees` statutory form, `features/payroll` statutory breakdown | `/api/employees/{employeeId}/statutory-profile`, `/api/payroll/*` |
| Payslips | `payslip` | Backend APIs only in current frontend main; payroll dialog still renders legacy HTML/JPG payslips from payroll data | `/api/organization/payslips`, `/api/organization/payslip-templates`, `/api/public/payslips/{token}` |
| Inventory | `inventory` | `features/inventory` | `/api/inventory/*` |
| Products / BOM | `product` | `features/products` | `/api/products/*` |
| Production tracking (modal execution, kanban, vendors, deadlines, audit trail) | `production` (+ `vendor` package) | `features/production`, `features/vendors` | `/api/production/*`, `/api/production/orders/{id}/audit-log`, `/api/production/my-assignments`, `/api/vendors/*` |
| Suppliers | `supplier` | `features/suppliers` | `/api/suppliers/*` |
| Customers | `customer` | `features/customers` | `/api/customers/*` |
| Billing / invoicing / OCR | `billing` | `features/billing` | `/api/billing/*` |
| Accounting / GST reports | `accounting` | `features/accounting` | `/api/accounting/*` |
| E-way bill integration | `ewaybill` | `features/billing`, `features/gst-integration` | `/api/eway-bills/*`, `/api/eway-bills/integration/*`, `/api/billing/bills/{billId}/eway/*` (provider disabled by default) |
| AI assistant | `ai` | `features/ai` | `/api/ai/*` |
| Feature gating | `feature` | `featureGating.ts`, saas-admin feature-gate page | `/api/saas-admin/features`, `/api/organizations/features` |
| SaaS pricing catalog | `saasadmin` (`SaasPricingAdminController`, `PublicPricingController`) | `features/public-pricing`, `features/saas-admin` | `/api/saas-admin/pricing`, `/api/public/plans`, `/api/public/add-ons`, `/api/public/offers` |
| SaaS platform admin (factories, marketing, insights) | `saasadmin` | `features/saas-admin` | `/api/saas-admin/*` |
| White-labeling / partners | `whitelabel` | `features/whitelabel` | `/api/public/whitelabel/*`, `/api/whitelabel/branding/me`, `/api/partner/whitelabel/*`, `/api/saas-admin/whitelabel/*` |
| Import/export jobs | `importexport` | `features/import-export` | `/api/import-export/jobs` |
| Early registration / lead capture | `registration` | `app/registration-pending` | `/api/public/early-registration/*` |
| Notifications (background only) | `notification` | — (no direct UI; drives emails) | — |
| Dashboard/insights | `dashboard` | `features/dashboard` | `/api/dashboard/*` |

## Cross-cutting frontend infrastructure

| Concern | File |
|---|---|
| API base client, JWT attach, 401/403 handling | `src/services/baseApi.ts` |
| Auth token/session state | `src/features/auth/authSlice.ts` |
| Runtime branding resolution | `src/features/whitelabel/hooks/useActiveBranding.ts` |
| Org-level feature gate checks | `src/config/featureGating.ts` |
| Tally UI mode flag | `src/config/features.ts` (`NEXT_PUBLIC_ENABLE_TALLY_UI`, default `false`) |
| App shell (nav, mode switch, feature gates) | `src/components/layout/AppShell.tsx` |

## Current frontend/source caveat

The current backend main branch contains the complete `payslip` API surface
(templates, generation snapshots, secure token links, public access, delivery
orchestration). The current frontend main branch contains payroll statutory
settings/profile/breakdown UI, but no payslip-template admin panel, "Share
securely" action, or unauthenticated `/payslip/[token]` viewer route. Treat
those as a frontend gap until the source checkout includes them.

## Security boundary summary (backend)

Defined centrally in `SecurityConfig`:
- `/api/public/**` — no auth (signup, OTP, public pricing, public whitelabel lookup, sandbox signup, attendance-capture public intake, public payslip token access)
- `/api/partner/**` — `PARTNER_ADMIN` only; excluded from `OrganizationApprovalFilter` and `FeatureGateFilter`
- `/api/saas-admin/**` — `SAAS_OWNER` only
- everything else — authenticated, role-checked per endpoint (`@PreAuthorize`), and gated by `OrganizationApprovalFilter` (blocks `PENDING_APPROVAL` orgs) + `FeatureGateFilter` (blocks orgs without the relevant plan feature)
