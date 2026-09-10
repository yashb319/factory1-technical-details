# Factory1 — Technical Details

This repository is the **single source of technical truth** for the Factory1 SaaS
platform: system architecture, module responsibilities, data flow, and sequence
diagrams covering both the backend and frontend. It contains **no application
code** — only documentation, kept in sync with the real repos as the system evolves.

## Related repositories

| Repo | Role |
|---|---|
| [`factory1-backend`](https://github.com/yashb319/factory1-backend) | Spring Boot multi-tenant ERP API (Java) |
| [`factory1-frontend`](https://github.com/yashb319/factory1-frontend) | Next.js/React admin + public web app (main product UI, includes attendance management screens) |
| [`factory1-frontend-attendance-capture`](https://github.com/yashb319/factory1-frontend-attendance-capture) | Device/kiosk-facing attendance capture surface, hosted separately at `attendance-capture.factory1.in` |

## Contents

- [`docs/architecture/system-overview.md`](docs/architecture/system-overview.md) — high-level system architecture, tech stack, deployment topology
- [`docs/architecture/data-model.md`](docs/architecture/data-model.md) — core entity relationships and multi-tenancy model
- [`docs/architecture/module-map.md`](docs/architecture/module-map.md) — backend package ↔ frontend feature mapping
- [`docs/sequence-diagrams/`](docs/sequence-diagrams/) — key flows as Mermaid sequence diagrams:
  - `auth-and-signup.md`
  - `sandbox-trial-signup.md`
  - `partner-onboarding.md`
  - `whitelabel-branding-resolution.md`
  - `production-order-lifecycle.md`
  - `production-kanban-vendor-overhaul.md`
  - `configurable-pricing.md`
  - `feature-gating.md`
- [`docs/modules/`](docs/modules/) — per-domain-module reference (endpoints, status, known gaps)
- [`docs/status/feature-status.md`](docs/status/feature-status.md) — current build status across all modules (working / partial / missing), test coverage, and open technical debt

## How to view diagrams

All diagrams are written in [Mermaid](https://mermaid.js.org/) syntax inside fenced
code blocks (` ```mermaid `). They render natively on GitHub — just open the `.md`
file in the GitHub UI. No extra tooling required.

## Keeping this up to date

This repo is documentation-only and evolves alongside the three product repos.
When a structurally significant feature lands (new module, new cross-repo flow,
schema change to a core entity), update the relevant diagram/doc here as part of
that work.
