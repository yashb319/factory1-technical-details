# Feature Status Snapshot

_Snapshot from a full-system audit. Update this file whenever a significant
gap is closed or a new one is discovered — treat it as a living technical-debt
register, not a one-time report._

## ✅ Solid & working end-to-end

- Core ERP: employees, attendance, leave, payroll, inventory, billing,
  accounting, production (most mature — phase 1+2, kanban, quality,
  notifications), customers, suppliers — full CRUD + dashboards + multi-tenant
  scoping on both backend and frontend.
- Platform layer: white-labeling, configurable SaaS pricing (plans + add-ons),
  partner onboarding + auto-provisioning, feature gating, sandbox trial
  signup — all wired end-to-end frontend ↔ backend and validated
  (tests/build/lint passing at time of implementation).
- Production tracking overhaul: third-party vendor outsourcing (new `Vendor`
  entity, mirrors Supplier), per-step deadlines with once-only breach-email
  notification, a full immutable per-order audit trail
  (`production_order_audit_log`, separate from the notification outbox),
  Jira-style drag-and-drop kanban (`@dnd-kit`, fixed/workflow-step view
  toggle, no horizontal scroll), inline partial-completion on kanban cards,
  and an org-configurable employee self-service "My Orders" view. See
  [`production-kanban-vendor-overhaul.md`](../sequence-diagrams/production-kanban-vendor-overhaul.md).

## 🟡 Half-cooked / rough edges

| Area | Issue | Where |
|---|---|---|
| Tally UI | Entire alternate UI mode incomplete, intentionally disabled by default | `NEXT_PUBLIC_ENABLE_TALLY_UI` (frontend), `features/*/tally/*` |
| AI assistant | Multiple `return null` stub branches; heuristic/experimental behavior | `ai/service/impl/AiActionServiceImpl.java`, `AiLocalAnswerServiceImpl.java`, `ListedCompanyService.java` |
| E-way Bill integration | Disabled by default; provider client has stub branches | `factory1.eway.enabled=false` in `application.properties`; `SandboxCoEwayBillProvider` |
| OCR bill import | External-provider-dependent, fragile, untested | `billing/controller/PurchaseBillOcrController.java` |
| Whitelabel partner provisioning | Uses a "synthetic placeholder org id" for `PARTNER_ADMIN` users — an explicit architectural shortcut, called out in code comments | `whitelabel/service/impl/WhitelabelAdminServiceImpl.java:209-215` |
| Plan-change-request | Just sends an email; no real workflow/approval backend | `organization/service/impl/OrganizationSettingsServiceImpl.java` (`plan-change-request`) |
| Several service impls | `return null` edge branches worth auditing for silent failure modes | `accounting/service/impl/AccountingReportServiceImpl.java`, `billing/service/impl/BillingServiceImpl.java`, `inventory/service/impl/InventoryServiceImpl.java`, `saasadmin/service/impl/SaasAdminServiceImpl.java`, `whitelabel/service/impl/WhitelabelPartnerServiceImpl.java` |
| Inventory "delete" | UI action actually disables the item, not a true delete — potentially confusing labeling | `features/inventory` `handleDelete` |
| Billing API surface | Several endpoints (`checkBillNumberAvailability`, `getEwayBillDetails`, `updateEwayPartB`, `cancelEwayBill`, OCR/template endpoints) may be unused by any visible UI — candidates for a dead-code pass | `features/billing/api/billingApi.ts` |

## 🔴 Biggest gaps

1. **Zero automated tests on the frontend** — no Jest/Vitest/Playwright setup
   at all. Highest risk given how large/dense the Accounting, Production,
   Billing, Payroll, and Leave pages are.
2. **Uneven backend test coverage** — `billing`, `inventory`, `payroll`,
   `customer`, `supplier`, `employee`, `ewaybill`, `importexport`,
   `notification`, `product`, `registration` have **no tests**.
3. **No offline support** for the attendance capture surface — relevant given
   factory-floor connectivity is often unreliable; no service worker,
   IndexedDB queue, or retry/backoff logic found.
4. **No rate limiting or API versioning** anywhere; hardcoded dev secrets
   present in `application.yml` / `application.properties` (local/dev only,
   but worth a hygiene pass before wider team access).
5. **`.env.example` (frontend) is incomplete** — missing
   `NEXT_PUBLIC_API_BASE_URL` (a hard dependency) and the four
   `NEXT_PUBLIC_FACTORY1_*_DOWNLOAD_URL` variables actually referenced in code.
6. **No audit logging** for sensitive admin actions outside production
   (partner creation, org approval/termination, pricing changes, feature-gate
   overrides) — production now has a full audit trail
   (`production_order_audit_log`); this pattern hasn't been extended to other
   modules yet.
7. **No manager/reporting-line concept** exists anywhere in `User`/`Employee`
   — surfaced concretely by the production deadline-breach feature, which can
   only notify the order's designated `responsibleUserId`, not an overdue
   assignee's manager. Worth deciding whether to introduce a reporting
   hierarchy if per-manager escalation becomes a real need.

## 💡 Candidate next features / improvements

- Replace the whitelabel synthetic-org-id hack with a proper partner-user
  identity model that doesn't rely on an unconstrained `organization_id`.
- Build a real plan-change/upgrade **workflow** (state machine + SaaS-owner
  review queue) instead of a one-way email notification.
- Add rate limiting + structured audit logging (following the new production
  audit-log pattern) on auth, public signup/sandbox, and SaaS-admin mutation
  endpoints.
- Stand up baseline test scaffolding on the frontend (even smoke/integration
  tests for the highest-risk pages: Accounting, Billing, Production) before
  the codebase grows further.
- Backfill backend unit tests for the untested modules, prioritizing
  financial/inventory-affecting logic (billing, payroll, inventory) first.
- Decide the long-term fate of the Tally UI mode: finish it, or remove the
  dead code paths if it's been deprioritized.
- Add a lightweight offline queue to the attendance-capture surface for
  factory-floor network resilience.
