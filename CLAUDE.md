# CLAUDE.md — okta documentation hub

`okta` is a single, integrated educational platform — not five separate products.
A core platform (`okta-web`) owns the domain data and exposes services; a partner
platform (`okta-partners`) is the mechanism through which applications are
authored and **installed into** the core; a Flutter client (`okta-app`) shows each
Tenant the applications it has installed; and this repository (`okta-docs`) is the
documentation hub and standards source. Applications added to the product have a
**dual surface** — the same installed application appears both inside `okta-web`
and inside `okta-app` — and they progress through environments **dev → prod**
(sandbox → production).

This file is the **map**. Depth lives in [`claude/`](./claude/). Start here, then
open the reference file the table points you to.

---

## Repository map

| Repo | Responsibility (one sentence) | Deep reference |
|---|---|---|
| **okta-web** | The platform/core: owns domain data, exposes services, and hosts installed applications as code; runs in sandbox + production. | [`claude/web.md`](./claude/web.md) |
| **okta-partners** | The installation/deployment mechanism: author, version, review, publish, and install applications through the bridge into `okta-web`. | [`claude/partners.md`](./claude/partners.md) |
| **okta-app** | The client: a Flutter app that lists and launches, per Tenant + role, the applications a Tenant has installed. | [`claude/app.md`](./claude/app.md) |
| **okta-docs** | This repo: documentation hub + engineering/design standards. | [`docs/`](./docs/README.md) · [`docs/tech-standards/`](./docs/tech-standards/README.md) |

> Applications that Tenants install are **not** listed as a repository: each lives
> in its own partner repo (scaffolded by `okta-partners`) and is represented here
> by the general **installed-application model** —
> [`claude/installed-apps.md`](./claude/installed-apps.md).

---

## How the pieces connect (compact)

- **Source of truth**: `okta-web` owns all Tenant domain data and the scope
  catalog. Everything orbits it.
- **Runtime edges**: `okta-partners → okta-web` over an authenticated HTTP
  **bridge** (publish/install/sync) + signed webhooks back; `okta-app → okta-web`
  over the mobile API (`/api/mobile/*`). There is **no** `okta-app ↔ okta-partners`
  link.
- **Dual surface**: one install lights up both the **platform** surface
  (`okta-web` web UI, from the manifest's `menu` block) and the **client** surface
  (`okta-app` catalog, from the manifest's `mobile` block).
- **dev → prod**: applications are tested in `okta-web` **sandbox**, then
  **published to production** — selected from `okta-partners` via `BridgeSettings`.
- **Isolation**: an installed application reaches Tenant data only through the
  Partner API (in-process for embedded, scope-gated HTTP for external); direct
  access to platform internals is blocked by a static policy scanner + runtime
  guards.

Diagrams, the workspace tree, and the dependency map:
[`claude/architecture.md`](./claude/architecture.md).

---

## Read this when…

| Your question | Open |
|---|---|
| How does the whole product fit together? Workspace tree, dependency map, dual surface. | [`claude/architecture.md`](./claude/architecture.md) |
| What is each repo responsible for, and where are the boundaries? | [`claude/repos.md`](./claude/repos.md) |
| What services does the core expose? How do installs work? Environments? | [`claude/web.md`](./claude/web.md) |
| How are applications authored/published/installed? The dev→prod (sandbox→prod) flow? | [`claude/partners.md`](./claude/partners.md) · [`claude/deployment.md`](./claude/deployment.md) |
| How does the mobile/desktop client list and launch a Tenant's apps? | [`claude/app.md`](./claude/app.md) |
| What contract must an application follow to be installable and dual-surfaced? | [`claude/installed-apps.md`](./claude/installed-apps.md) |
| **Build** an installed app's pages — where/how they're developed & appear in `okta-web` and `okta-app` (developer how-to). | [`docs/app-development/`](./docs/app-development/README.md) |
| End-to-end: from publish to appearing in both `okta-web` and `okta-app`. | [`claude/deployment.md`](./claude/deployment.md) |
| What does a term mean (Tenant, scope, installed app, environment, …)? | [`claude/glossary.md`](./claude/glossary.md) |
| Product vision, Tenant types, end-users (Arabic). | [`docs/README.md`](./docs/README.md) · [`docs/tenants.md`](./docs/tenants.md) · [`docs/end-users.md`](./docs/end-users.md) |
| Engineering/design standards both Laravel repos obey. | [`docs/tech-standards/README.md`](./docs/tech-standards/README.md) |

---

## About this repository (editing rules)

`okta-docs` holds documentation only — no application code. Two layers:

- [`docs/`](./docs/README.md) — product docs + the **tech-standards** consumed by
  `okta-web` and `okta-partners`. Primary language is **Arabic**; keep the
  existing style and RTL-friendly Markdown. Changing a standard can **break** the
  consumer repos — flag such changes as breaking and note a migration path.
- [`claude/`](./claude/) — this English reference layer describing the integrated
  product. Use **relative links**; verify each one resolves.

Conventions: don't create doc files at the repo root (put them under `docs/` or
`claude/`); treat `tmp/` as scratch (never a reference); when adding a
tech-standard, update [`docs/tech-standards/README.md`](./docs/tech-standards/README.md).

Each code repo also ships its own `CLAUDE.md` (`okta-web/CLAUDE.md`,
`okta-partners/CLAUDE.md`) with repo-local detail — consult those for in-repo
specifics; use *this* hub for the cross-repo, whole-product picture.
