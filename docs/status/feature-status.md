# Feature Status Snapshot

_Snapshot from a full-system audit against the current local backend/frontend
main checkouts. Update this file whenever a significant gap is closed or a new
one is discovered — treat it as a living technical-debt register, not a one-time
report._

## ✅ Solid & working end-to-end

- Core ERP: employees, attendance, leave, payroll, inventory, billing,
  accounting, production, customers, suppliers, vendors — full CRUD or
  operational flows with tenant scoping on backend and frontend.
- Platform layer: white-labeling, configurable SaaS pricing (plans + add-ons),
  partner onboarding + auto-provisioning, feature gating, sandbox trial signup
  — all wired frontend ↔ backend with backend coverage for the highest-risk
  pieces.
- Production order modal execution overhaul: tabs now prioritize Production
  orders, Workflow templates, BOM definitions, Workstations, and Analytics;
  order click-through opens a large modal with Execution, Assignments, Quality,
  Materials, Timeline, and Audit trail sections. Execution records completed and
  rejected quantities through `/record`, auto-starts on first record, gates
  `/complete` until the current step's processed quantity reaches the planned
  quantity, and places assigned PLANNED/RELEASED cards into In Progress via
  `hasActiveAssignment`. See
  [`production-order-lifecycle.md`](../sequence-diagrams/production-order-lifecycle.md).
- Production tracking phase 2 remains mature: internal/vendor assignments,
  deadlines with once-only breach email, immutable audit trail, board/list
  Kanban, material consumption, quality results, and employee My Assignments.
  See
  [`production-kanban-vendor-overhaul.md`](../sequence-diagrams/production-kanban-vendor-overhaul.md).
- India payroll PF/TDS backend engine and frontend statutory views are wired:
  org-level `pfEnabled` / `payrollTdsEnabled` flags default off, employee
  statutory profiles hold PAN/UAN/PF/tax declarations, payroll generation writes
  PF/TDS fields and `payroll_statutory_calculations`, and the payroll payslip
  dialog shows the statutory breakdown. See
  [`payroll-statutory-and-payslip-sharing.md`](../sequence-diagrams/payroll-statutory-and-payslip-sharing.md).
- Payslip backend is implemented end-to-end: template versions/defaults,
  generated payslip snapshots, secure one-time-reveal share links, generic 404
  public token access, email delivery, and no-op SMS/WhatsApp stubs behind org
  toggles. See
  [`payroll-statutory-and-payslip-sharing.md`](../sequence-diagrams/payroll-statutory-and-payslip-sharing.md).

## 🟡 Half-cooked / rough edges

| Area | Issue | Where |
|---|---|---|
| Payslip frontend layer | Backend has template/share/public-access APIs, but current frontend main does **not** contain a payslip-template admin panel, "Share securely" action, or unauthenticated `/payslip/[token]` viewer page | `factory1-frontend` main; backend `payslip/controller/*` |
| Payslip rendering | Backend endpoints return structured JSON snapshots/templates, not PDF bytes; current frontend renders HTML and exports JPG/ZIP, with no PDF rendering library found | `payslip` backend DTOs/services; `features/payroll/utils/payrollPayslipDownload.utils.ts` |
| Payslip SMS/WhatsApp delivery | Explicitly wired as logging no-op stubs until real providers are chosen; toggles exist server-side but no provider integration exists | `payslip/delivery/NoopSmsSender.java`, `NoopWhatsAppSender.java` |
| Tally UI | Entire alternate UI mode incomplete, intentionally disabled by default | `NEXT_PUBLIC_ENABLE_TALLY_UI` (frontend), `features/*/tally/*` |
| AI assistant | Multiple stub/null/heuristic branches; useful but experimental behavior | `ai/service/impl/AiActionServiceImpl.java`, `AiLocalAnswerServiceImpl.java`, `ListedCompanyService.java` |
| E-way Bill integration | Disabled by default; provider/client have sandbox/stub branches and credential test paths | `factory1.eway.enabled=false`, `ewaybill/provider/*`, `ewaybill/client/*` |
| OCR bill import | External-provider-dependent, fragile, untested | `billing/controller/PurchaseBillOcrController.java`, `features/billing/components/AutoPurchaseBillImportDialog.tsx` |
| Whitelabel partner provisioning | Uses a synthetic placeholder org id for `PARTNER_ADMIN` users — an explicit architectural shortcut called out in code comments | `whitelabel/service/impl/WhitelabelAdminServiceImpl.java` |
| Plan-change request | Sends an email; no real approval/state-machine workflow | `organization/service/impl/OrganizationSettingsServiceImpl.java` (`plan-change-request`) |
| Several service impls | `return null` edge branches worth auditing for silent failure modes | `accounting`, `billing`, `inventory`, `saasadmin`, `whitelabel` service impls |
| Inventory "delete" | UI action actually disables the item, not a true delete — potentially confusing labeling | `features/inventory` `handleDelete` |
| Attendance capture app | Referenced separate local checkout is not present in this environment; backend public capture APIs exist, but no offline-capable capture app could be audited locally | `/api/public/attendance-capture`, `/api/attendance/device-event` |

## 🔴 Biggest gaps

1. **Zero automated tests on the main frontend checkout** — no
   `*.test.*` / `*.spec.*` files found. Highest risk given the large/dense
   Accounting, Production, Billing, Payroll, and Leave pages.
2. **Uneven backend test coverage** — `billing`, `inventory`, `customer`,
   `supplier`, `product`, `ewaybill`, `importexport`, `notification`,
   `registration`, and most legacy employee CRUD paths have no direct tests.
   Payroll is no longer "no tests": the new PF/TDS calculators and statutory
   payroll wiring are covered, but the older salary-calculator paths still need
   broader direct coverage.
3. **No offline support** for the attendance capture surface — relevant given
   factory-floor connectivity is often unreliable; no service worker,
   IndexedDB queue, or retry/backoff logic found in the available checkout.
4. **No rate limiting or API versioning** across most APIs; payslip public
   token access and sandbox signup have bespoke rate limiters, but auth/public
   signup/SaaS-admin mutation paths do not share a platform-wide limiter.
5. **Hardcoded dev/local secrets remain in backend config files**; they appear
   intended for local/dev use, but should be cleaned up before wider team
   access.
6. **`.env.example` (frontend) is incomplete** — missing
   `NEXT_PUBLIC_API_BASE_URL` and the `NEXT_PUBLIC_FACTORY1_*_DOWNLOAD_URL`
   variables referenced by native download links.
7. **No audit logging** for sensitive admin actions outside production and
   payslip token access — partner creation, org approval/termination, pricing
   changes, and feature-gate overrides do not yet have the production-style
   immutable audit trail.
8. **No manager/reporting-line concept** exists in `User`/`Employee`; production
   deadline breach emails can notify only the order's `responsibleUserId`, not
   an overdue assignee's manager.

## 💡 Candidate next features / improvements

- Add the missing frontend payslip-template admin, secure-share action, and
  public token viewer to match the backend payslip APIs.
- Add PDF rendering/generation for payslips if statutory/compliance workflows
  require downloadable PDF documents rather than HTML/JPG snapshots.
- Replace SMS/WhatsApp no-op payslip stubs with a real provider abstraction once
  a vendor is chosen.
- Replace the whitelabel synthetic-org-id hack with a partner-user identity
  model that does not rely on unconstrained `users.organization_id`.
- Build a real plan-change/upgrade workflow (state machine + SaaS-owner review
  queue) instead of a one-way email notification.
- Add shared rate limiting + structured audit logging on auth, public signup,
  sandbox, and SaaS-admin mutation endpoints.
- Stand up baseline frontend tests, even smoke/integration tests for the
  highest-risk pages first: Accounting, Billing, Production, Payroll, Leave.
- Backfill backend tests for untested modules, prioritizing financial and
  stock-affecting logic: billing, inventory, product/BOM, e-way, legacy payroll
  salary calculators.
- Decide the long-term fate of Tally UI mode: finish it, or remove dead code
  paths if deprioritized.
- Add a lightweight offline queue to the attendance-capture surface for
  factory-floor resilience.
