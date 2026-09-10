# Per-Module Reference (Backend)

Quick reference for every backend package: purpose, main endpoints, and test coverage at a glance. See `docs/status/feature-status.md` for build-completeness/gap details.

| Package | Purpose | Main API base | Has tests? |
|---|---|---|---|
| `auth` | JWT auth, OTP flows, user management, roles | `/api/auth`, `/api/users` | ✅ |
| `organization` | Org lifecycle, settings, sandbox trial fields | `/api/organizations`, `/api/organization/settings` | ✅ |
| `employee` | Employee CRUD, CSV import, invitations | `/api/employees` | ❌ |
| `attendance` | Attendance records, device intake, reporting | `/api/attendance`, `/api/public/attendance-capture` | ✅ |
| `leave` | Leave types/balances/requests/calendar/holidays | `/api/leave` | ✅ |
| `payroll` | Salary calculation, payroll runs, payslips | `/api/payroll` | ❌ |
| `inventory` | Stock items, movements, dashboard | `/api/inventory` | ❌ |
| `product` | Products, BOM | `/api/products` | ❌ |
| `production` | Workflows, orders, execution, quality, kanban, notifications, vendor outsourcing, per-step deadlines, audit trail, employee self-service | `/api/production` (phase 1 + 2), `/api/production/orders/{id}/audit-log`, `/api/production/my-assignments` | ✅ (most-covered module) |
| `vendor` | Third-party vendor (outsourcing) CRUD, dashboard, insights — mirrors `supplier` | `/api/vendors` | ✅ |
| `supplier` | Supplier CRUD, insights, bulk import | `/api/suppliers` | ❌ |
| `customer` | Customer CRUD, insights, bulk import | `/api/customers` | ❌ |
| `billing` | Bills, GST, OCR import, e-way integration hooks | `/api/billing` | ❌ |
| `accounting` | Vouchers, ledgers, trial balance, P&L, balance sheet, GST | `/api/accounting` | ✅ |
| `ewaybill` | E-way bill provider integration (disabled by default) | `/api/ewaybill` | ❌ |
| `ai` | AI chat, business insights, benchmarks, action execution | `/api/ai` | ❌ |
| `feature` | Feature catalog + per-org overrides + gate filter | `/api/saas-admin/features`, `/api/organizations/features` | ✅ |
| `saasadmin` | Pricing catalog, offers, factories, marketing, public pricing | `/api/saas-admin/*`, `/api/public/plans` etc. | ✅ (pricing only) |
| `whitelabel` | Partner program, branding, domain resolution | `/api/public/whitelabel`, `/api/whitelabel/branding/me`, `/api/partner/whitelabel`, `/api/saas-admin/whitelabel` | ✅ |
| `importexport` | Generic import/export job tracking | `/api/import-export/jobs` | ❌ |
| `registration` | Early-registration/lead-capture questionnaire | `/api/public/early-registration` | ❌ |
| `notification` | Email service + schedulers (digest, renewal reminders, sandbox cleanup) | — (background only) | ❌ |
| `dashboard` | Org-level summary/trend dashboard | `/api/dashboard` | ✅ |
| `common` | BaseEntity, pagination, API response wrapper, global exception handling | — | ✅ (BaseEntity only) |
| `config` | Security, Swagger, cache configuration | — | — |

# Per-Feature Reference (Frontend)

| Feature dir | Purpose | Backend module it talks to | Notable status |
|---|---|---|---|
| `features/auth` | Login, signup, OTP, forgot-password, employee activation, partner-code field | `auth` | Working; partner-code field is recent |
| `features/employees` | Employee CRUD, import, invitations, pagination | `employee` | Working |
| `features/attendance` | Attendance dashboard, manual/bulk entry, monthly report, export, QR generation for capture station | `attendance` | Working; no offline support |
| `features/leave` | Leave types/balances/requests/calendar | `leave` | Working; dense page |
| `features/payroll` | Payroll runs, generate/approve/pay, payslips, insights | `payroll` | Working; financial actions untested |
| `features/inventory` | Stock CRUD, movements, dashboard, bulk import/export | `inventory` | Working; "delete" is actually disable |
| `features/products` | Product + BOM CRUD | `product` | Working |
| `features/production` | Orders, workflows, BOM, workstations, assignments (internal worker or vendor), Jira-style drag-drop kanban (fixed/step-view toggle, no horizontal scroll), inline partial-completion, audit trail tab, employee "My Orders" view, analytics | `production` | Most complete feature; very large page, now split with `KanbanBoard.tsx`, `PartialCompletionForm.tsx`, `MyAssignmentsPage.tsx` |
| `features/vendors` | Vendor CRUD, dashboard — mirrors `features/suppliers` | `vendor` | Working |
| `features/suppliers` | Supplier CRUD, dashboard, AI insights | `supplier` | Working |
| `features/customers` | Customer CRUD, dashboard, insights | `customer` | Working |
| `features/billing` | Invoices, e-way bill actions, GST, OCR import | `billing`, `ewaybill` | Working; some endpoints may be dead/unwired |
| `features/accounting` | Vouchers, ledgers, reports (trial balance, P&L, balance sheet, aging) | `accounting` | Working; most complex/dense page |
| `features/ai` | Assistant chat, insights, benchmarks | `ai` | Experimental/heuristic |
| `features/organization-settings` | Org settings, GST integration, access management, termination | `organization`, `feature` | Working |
| `features/public-pricing` | Public homepage pricing cards | `saasadmin` (public pricing) | Working, DB-driven |
| `features/saas-admin` | Factories, feature gating, insights, marketing, whitelabel admin, pricing catalog manager | `saasadmin`, `feature`, `whitelabel` | Working; broad admin surface |
| `features/whitelabel` | SaaS/partner branding admin, runtime branding resolution | `whitelabel` | Working; recently hardened |
| `features/dashboard` | Summary/trends/benchmarks | `dashboard` | Working |
| `features/help-center`, `features/docs` | In-app help/documentation | — | Static content |
| `features/import-export` | Generic import/export job UI, CSV utilities | `importexport` | Foundational |
| Tally-mode pages (`features/*/tally/*`) | Alternate Tally-style UI per module | various | **Disabled by default** (`NEXT_PUBLIC_ENABLE_TALLY_UI=false`) |
