# Odoo vs Factory1 gap analysis

_Prepared 17 September 2026 from the current Factory1 technical documentation and
public Odoo product/developer references. This is a capability comparison, not a
recommendation to reproduce Odoo's UI, copy, code, or commercial packaging._

## Scope and comparison lens

Odoo is an integrated, modular business suite. Its official product surface spans
CRM, Sales, POS, Subscriptions, Website/eCommerce, Accounting, Inventory,
Manufacturing, Project, Timesheets, Helpdesk, Marketing Automation, Events,
Surveys, Employees, Time Off, Appraisals, Fleet, Documents and other apps. The
open-source repository is organized as installable addons, while Odoo Studio and
developer APIs provide configuration and extension paths. Useful public references
for this comparison are the [Odoo app catalogue](https://www.odoo.com/page/all-apps),
[user documentation](https://www.odoo.com/documentation/19.0/), [developer
documentation](https://www.odoo.com/documentation/19.0/developer.html), and the
[open-source repository](https://github.com/odoo/odoo).

Factory1 is narrower by design: a multi-tenant SaaS ERP for Indian manufacturing
factories. It already combines HR, attendance, leave, payroll, payslips,
inventory, production, billing/accounting, GST/e-way hooks, AI insights and a
white-label partner platform. The comparison therefore asks: which Odoo
capabilities are needed to win the target customer, which are platform
foundations, and which should remain out of scope?

## Executive summary

**Where Factory1 is already strong**

- Factory-floor production is more opinionated than a generic suite: versioned
  workflows and BOMs, workstations, assignments to employees or vendors,
  quantity-based execution, quality/material sections, deadlines, Kanban and an
  immutable production audit trail.
- Indian payroll foundations are first-class: PF/TDS rule snapshots, PAN/UAN/PF
  profiles, tax-regime declarations, statutory breakdowns and shareable payslips.
- A SaaS operator can run multiple tenant organizations with feature gates,
  plans/add-ons/offers, sandbox trials, partner provisioning and tenant
  white-label branding. This is a strategic platform capability, not merely an
  ERP module.
- The stack is designed for a focused deployment model: a Spring Boot API,
  PostgreSQL, Next.js web UI, Electron/Capacitor packaging and a separate
  attendance-capture surface.

**Where Odoo is materially ahead**

Odoo offers a much wider business lifecycle around the factory: lead-to-quote
sales, website/eCommerce, POS, subscriptions, project/service delivery,
helpdesk, marketing, documents, timesheets and a mature customization ecosystem.
It also has deeper multi-company/inter-company accounting, localization breadth,
workflow automation, reporting/spreadsheets, integrations and an installable app
extension model. Factory1 should close the customer-facing and platform gaps that
block adoption, while avoiding a low-differentiation attempt to become every
business app at once.

## Capability gap table

Severity means the likely impact on Factory1's Indian manufacturing/SaaS
positioning, not the number of Odoo screens that are absent.

| Odoo capability | Factory1 current state | Gap severity | Why it matters | Recommended next step |
|---|---|---:|---|---|
| CRM: leads, opportunities, activities and pipeline | Customers exist; no documented lead/opportunity/activity pipeline | High | Sales teams cannot manage demand before an order reaches production | Add a lightweight CRM with lead source, owner, stages, next activity and conversion to customer/quotation |
| Sales quotations and sales orders | Billing/customer CRUD exists; no documented quote-to-order workflow | High | Manufacturing needs a commercial commitment and promised delivery date before scheduling | Add quotations, approval, revisions, taxes/discounts, order confirmation and production-order handoff |
| Website builder and eCommerce | Public pricing, registration and white-label routes; no product catalogue/cart/checkout | Medium | Direct selling and tenant storefronts are important for smaller manufacturers | Start with a hosted product catalogue and enquiry/order capture; defer a general website builder |
| Point of Sale | No POS module documented | Low/Medium | Relevant to retail/showroom customers, but not every factory | Validate demand; integrate a focused counter-sale flow before a full restaurant/retail POS |
| Subscriptions and recurring invoicing | SaaS plans/add-ons and renewal notifications exist; tenant subscription billing is not a complete invoice/collection lifecycle | High | Recurring SaaS revenue and service contracts need reliable renewals, proration and payment state | Create a subscription ledger/state machine, payment-provider abstraction and dunning events |
| Manufacturing planning (MRP) | Strong execution, BOM, material posting, workstations, quality and vendor assignments | Medium | Odoo is deeper in planning and replenishment; execution strength is a differentiator to preserve | Add demand-based MRP, capacity/calendar planning, finite scheduling and subcontracting cost visibility incrementally |
| Inventory, warehouses and logistics | Inventory CRUD/movements, product BOMs and production postings; no documented multi-warehouse/routes/picking/lot-serial depth | High | Stock accuracy and traceability are prerequisites for larger factories | Add warehouses/locations, receipts/deliveries, reservations, transfers, lot/serial and reorder rules |
| Purchasing and vendor lifecycle | Supplier and production-vendor CRUD/insights; no RFQ/PO/receipt/three-way match flow | High | Material availability and vendor cost are upstream of production margin | Add RFQs, purchase orders, receipts, vendor bills and supplier performance linked to BOM demand |
| Accounting and invoicing | Ledgers, vouchers, GST summary, trial balance, P&L, balance sheet, aging and billing/e-way hooks | Medium | Core coverage exists, but Odoo has broader controls and accounting integrations | Harden posting/period close, bank reconciliation, recurring entries, attachments, approvals and audit history |
| Indian localization and compliance | PF/TDS payroll, GST suggestions/reports and e-way provider hooks; e-way disabled by default | Medium | India-first trust depends on dependable statutory workflows, not just fields | Prioritize GSTIN validation, GSTR workflows/reconciliation, e-invoice/e-way production provider and exportable audit packs |
| HR employee lifecycle | Employees, invitations, statutory profiles, attendance, leave, payroll and payslips | Medium | Odoo adds recruitment, appraisals, expenses, fleet and richer employee self-service | Add manager/reporting lines, onboarding/offboarding checklist, documents and employee portal before broad HR apps |
| Project, timesheets and field/service work | Production assignments and employee “My Assignments”; no general project/timesheet/helpdesk domain | High | Installation, job work and maintenance often surround factory orders | Build a shared work-item primitive, then project tasks, timesheets and customer-visible status |
| Helpdesk and customer service | No helpdesk/ticket workflow documented | Medium | Retention requires handling quality claims, delivery issues and support requests | Add tickets, SLA, assignment, email intake and link tickets to customer/order/product |
| Marketing, events and surveys | SaaS-admin marketing sends and static help/legal content; no campaign automation/events/surveys | Low/Medium | Useful for demand generation and onboarding, but not a core factory wedge | Expose integration/webhook events and basic lifecycle campaigns; defer a full marketing suite |
| Documents and approvals | Payslip templates/shares and import/export jobs; no general document repository/approval engine | High | Bills, drawings, certificates and compliance evidence need controlled access and history | Add attachments/document metadata, approval requests, versioning, retention and entity links |
| Reporting, BI and spreadsheets | Dashboard/trends, accounting reports and AI drilldowns; no ad-hoc pivot/spreadsheet model documented | Medium | Managers need cross-module margin, delivery and shop-floor decisions | Establish a metric layer, saved reports, exports and role-based dashboards before spreadsheet parity |
| Automation and workflow rules | Scheduled jobs and feature gates; notification plumbing; no user-configurable rule builder | High | Odoo’s automation reduces manual handoffs and increases extensibility | Add event/outbox conventions, scheduled actions, approval rules and safe no-code triggers |
| Studio/customization | Feature flags, configurable payslip templates, plans and branding; no user-defined fields/views/apps | High | Each factory has different forms, approval paths and terminology | Provide scoped custom fields, required fields, statuses and saved views first; introduce a governed “Factory1 Studio” later |
| Integrations and APIs | REST/JSON API, GST/e-way hooks, OCR edges, attendance capture, email; no public versioned API or connector catalogue | High | Integrations determine implementation cost and ecosystem reach | Publish API v1, webhooks, import templates, idempotency rules and prioritized Tally/ERP/banking/e-commerce connectors |
| Multi-company and inter-company | Shared-database tenant isolation; one organization context, partner/white-label administration | High | Groups with multiple plants/legal entities need consolidated and segregated views | Model companies/plants/branches separately from tenant, then inter-company stock/invoice and consolidated reporting |
| Access control and audit | JWT/roles/feature gates; production and payslip audit trails; several sensitive admin actions lack immutable audit | High | Financial and compliance trust requires “who changed what and why” | Create a common permission matrix and append-only audit service for finance, pricing, access, approvals and master data |
| Offline/mobile/device workflows | Capacitor/Electron packaging and separate attendance capture; no offline queue/service worker documented | High | Shop floors and attendance points have unreliable connectivity | Add signed local queue, replay/idempotency, conflict policy and device health telemetry to attendance first |
| Ecosystem and implementation tooling | Three focused repos and internal docs; no installable module marketplace or partner developer kit | Medium | Odoo’s ecosystem compounds coverage and reduces bespoke engineering | Define extension points, SDK/examples, tenant-safe events and a partner certification path before a marketplace |

## Prioritized roadmap

### Immediate: landing, positioning and UX polish

1. Position Factory1 as **India-first manufacturing operations plus payroll**, not
   “another general ERP.” Show the production lifecycle, statutory payroll,
   factory-floor resilience and partner/white-label model above generic feature
   counts.
2. Build a guided demo path: sample factory, quote/intake → BOM → production
   execution → stock posting → invoice → payroll/payslip. The sandbox already
   provides the right foundation.
3. Simplify the dense Accounting, Billing, Production and Leave surfaces with
   role-based navigation, progressive disclosure, saved filters, bulk actions,
   keyboard-friendly tables and clear empty/error states.
4. Close trust gaps visible to prospects: PDF payslips, real SMS/WhatsApp
   delivery, production e-way/e-invoice provider readiness, complete environment
   configuration, and a documented API/security posture.
5. Add smoke/integration coverage for the highest-risk frontend journeys and
   backend financial/stock modules before adding more surface area.

### Short term: product features that unlock adoption

- CRM-lite → quotations → sales orders → production handoff.
- Purchasing: RFQ/PO, receipts, vendor bills and material-demand linkage.
- Warehouse foundations: locations, receipts/deliveries, reservations,
  lot/serial traceability and reorder rules.
- A common documents/attachments/approvals service for invoices, drawings,
  quality certificates and HR records.
- Manager/reporting lines, employee onboarding/offboarding and a practical
  employee portal.
- Customer tickets and SLA basics, linked to customer/order/product records.
- Offline attendance capture with an idempotent replay queue.
- Subscription lifecycle for Factory1 tenants: trial, active, grace, past due,
  cancellation, add-ons, invoices and payment webhooks.

### Medium term: platform and extensibility

- Versioned public API, webhooks, import/export contracts, API keys and
  tenant-scoped rate limits.
- A shared domain-event/outbox framework so notifications, integrations and
  automations do not live as one-off scheduled code.
- Governed customization: custom fields, statuses, validations, saved views,
  approval rules and report definitions per tenant/role.
- Metric definitions and a reporting layer spanning order margin, OEE-like
  production measures, inventory turns, attendance and payroll cost.
- Company/plant hierarchy with branch-level access and consolidated reporting.
- Connector priorities: Tally import/export and reconciliation, GST/e-invoice/
  e-way providers, banking/payment rails, Shopify/WooCommerce where customer
  demand proves it.
- Extension SDK and partner documentation; keep custom code isolated from core
  schema and enforce tenant isolation in every extension.

### Long term: selective enterprise parity

- Capacity-aware MRP, demand planning, maintenance, subcontracting and
  production costing.
- Inter-company procurement/stock/invoicing and multi-currency/consolidation.
- Full customer portal, service projects, timesheets and field operations.
- Workflow/approval designer and a marketplace only after the extension contract
  is stable.
- Broader localization packs via a rules/versioning model, starting with India
  and adding countries only with a committed distribution channel.
- Optional POS, eCommerce and marketing apps as composable add-ons rather than
  default complexity in every tenant.

## India/manufacturing/SaaS strategy

Factory1 should **differentiate instead of cloning Odoo**:

- Make India compliance operational: GSTIN-aware masters, e-invoice/e-way
  reliability, PF/TDS explainability, statutory snapshots, regional payroll
  calendars and exportable evidence. Compliance must be auditable and
  configurable by financial year.
- Treat factory reality as a product advantage: offline-first capture,
  low-bandwidth screens, QR/device intake, vendor job-work handoffs, quality
  rejections, material shortages, deadline escalation and shift-aware
  execution.
- Preserve a Tally-friendly path. Use clean import/export and reconciliation
  rather than forcing every prospect to replace an entrenched accounting system
  on day one; finish or retire the currently disabled Tally UI deliberately.
- Package capabilities by outcome: “Factory operations,” “People and payroll,”
  “Finance and GST,” and “Growth” are easier to buy than dozens of technical
  feature flags. Keep feature gates internally, but expose simple plans.
- Use the SaaS/partner layer as distribution: reseller onboarding, white-label
  domains, sandbox conversion, tenant health, billing and support should be
  safer and more observable than an ad-hoc implementation.
- Avoid a generic website builder, social marketing suite or restaurant POS until
  evidence shows they improve factory acquisition or retention. Integrate with
  those ecosystems where that is cheaper than owning them.

## Landing-page content inventory

These are original, concise labels and blurbs for an Odoo-style module grid. They
are intentionally outcome-focused and can be mapped to the current frontend
navigation and future roadmap.

| Module name | Feature blurb |
|---|---|
| Factory Dashboard | See orders, stock, attendance, payroll and business signals in one operating view. |
| Production Control | Move every job from planned work to completed output with quantities, deadlines and accountability. |
| Workflows & BOMs | Standardize how each product is made with versioned recipes, steps and material requirements. |
| Shop-Floor Assignments | Give work to employees or outside vendors and track progress without losing the handoff history. |
| Quality & Traceability | Record checks, accepted/rejected output, material use and an audit trail for every production order. |
| Inventory & Stock Movements | Maintain reliable item balances, movements, imports and production postings across the factory. |
| Products & Materials | Keep product masters and BOM definitions ready for purchasing and production. |
| Suppliers & Job-Work Vendors | Manage supply partners, outsourced operations and performance context in one place. |
| Customers & Billing | Maintain customer records, bills and invoice workflows connected to the work being delivered. |
| Accounting & GST | Track ledgers, vouchers, GST summaries, aging, trial balance, profit and loss, and balance sheet views. |
| E-Invoice & E-Way Ready | Connect compliant transport and invoice processes when the chosen provider is enabled. |
| Employees | Keep employee records, invitations, statutory profiles and self-service access organized. |
| Attendance Capture | Collect attendance from a dedicated device-friendly surface built for factory entry points. |
| Leave & Holidays | Manage leave types, balances, approvals, calendars and holiday rules without spreadsheets. |
| Payroll & Payslips | Run payroll with PF/TDS calculations, explainable deductions, templates and secure sharing. |
| CRM & Quotations | Turn enquiries into qualified opportunities, clear quotes and production-ready orders. |
| Procurement | Convert material demand into RFQs, purchase orders, receipts and vendor bills. |
| Customer Helpdesk | Route quality, delivery and service questions with owners, priorities and response targets. |
| Reports & AI Insights | Explore operational trends and ask focused questions about the business data you already own. |
| SaaS Admin | Operate plans, add-ons, trials, feature access, tenant health and platform workflows. |
| Partner & White-Label | Let trusted partners onboard factories under their own brand and domain. |
| Integrations | Connect Tally, GST providers, payments, devices and other systems through controlled imports, APIs and webhooks. |
| Factory1 Extensions | Add tenant-specific fields, approvals, views and automation without forking the product. |

