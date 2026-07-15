# CLAUDE.md — okta documentation hub

`okta` is a single, integrated educational platform — not several separate
products. A core platform (`okta-web`) owns the domain data and exposes
services; a partner platform (`okta-partners`) is the mechanism through which
applications are authored and **installed into** the core; a Flutter client
(`okta-app`) shows each Tenant the applications it has installed; each
installable application lives in its **own repo** (three are in this
workspace); and this repository (`okta-docs`) is the documentation hub and
standards source. Applications added to the product have a **dual surface** —
the same installed application appears both inside `okta-web` and inside
`okta-app` — and they progress through environments **dev → prod**
(sandbox → production).

This file is the **map**. Depth lives in [`docs/reference/`](./docs/reference/). Start here, then
open the reference file the table points you to.

---

## Repository map

| Repo | Responsibility (one sentence) | Deep reference |
|---|---|---|
| **okta-web** | The platform/core: owns domain data, exposes services, and hosts installed applications as code; runs in sandbox + production. | [`docs/reference/web.md`](./docs/reference/web.md) |
| **okta-partners** | The installation/deployment mechanism: author, version, review, publish, and install applications through the bridge into `okta-web`. | [`docs/reference/partners.md`](./docs/reference/partners.md) |
| **okta-app** | The client: a Flutter app that lists and launches, per Tenant + role, the applications a Tenant has installed (plus student/guardian portals). | [`docs/reference/app.md`](./docs/reference/app.md) |
| **okta-docs** | This repo: documentation hub + engineering/design standards. | [`docs/`](./docs/README.md) · [`docs/tech-standards/`](./docs/tech-standards/README.md) |
| **okta-smart-timetable** · **okta-exams** · **okta-hdor** | Installed-application repos (one per app): AI timetable, final-exam committees, and student attendance — all follow the installed-application contract. | [`docs/reference/installed-apps.md`](./docs/reference/installed-apps.md) |

> Installable applications are not part of the core platform repos: each lives
> in its own partner repo (scaffolded by `okta-partners`) and follows the
> general **installed-application model** —
> [`docs/reference/installed-apps.md`](./docs/reference/installed-apps.md).

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
  (`okta-app` catalog, from the manifest's `mobile` block — with optional
  per-audience entries; guardian/student **portal** cards surface cross-tenant).
- **dev → prod**: applications are tested in `okta-web` **sandbox**, then
  **published to production** — selected from `okta-partners` via `BridgeSettings`.
- **Isolation**: an installed application reaches Tenant data only through the
  Partner API (in-process for embedded, scope-gated HTTP for external); direct
  access to platform internals is blocked by a static policy scanner + runtime
  guards.

Diagrams, the workspace tree, and the dependency map:
[`docs/reference/architecture.md`](./docs/reference/architecture.md).

---

## Read this when…

| Your question | Open |
|---|---|
| How does the whole product fit together? Workspace tree, dependency map, dual surface. | [`docs/reference/architecture.md`](./docs/reference/architecture.md) |
| What is each repo responsible for, and where are the boundaries? | [`docs/reference/repos.md`](./docs/reference/repos.md) |
| What services does the core expose? How do installs work? Environments? | [`docs/reference/web.md`](./docs/reference/web.md) |
| How are applications authored/published/installed? The dev→prod (sandbox→prod) flow? | [`docs/reference/partners.md`](./docs/reference/partners.md) · [`docs/reference/deployment.md`](./docs/reference/deployment.md) |
| How does the mobile/desktop client list and launch a Tenant's apps? | [`docs/reference/app.md`](./docs/reference/app.md) |
| What contract must an application follow to be installable and dual-surfaced? What do the three workspace app repos look like? | [`docs/reference/installed-apps.md`](./docs/reference/installed-apps.md) |
| **Build** an installed app's pages — where/how they're developed & appear in `okta-web` and `okta-app` (developer how-to). | [`docs/apps/development/`](./docs/apps/development/README.md) |
| End-to-end: from publish to appearing in both `okta-web` and `okta-app`. | [`docs/reference/deployment.md`](./docs/reference/deployment.md) |
| What does a term mean (Tenant, scope, installed app, audience, environment, …)? | [`docs/reference/glossary.md`](./docs/reference/glossary.md) |
| Product vision (Arabic). | [`docs/README.md`](./docs/README.md) |
| Entity types (Tenants), end-user roles, and the relationships between them. | [`docs/roles-and-entities/README.md`](./docs/roles-and-entities/README.md) |
| Roles ↔ account-type map (followed → follower, containment, derivation) as a graph. | [`docs/roles-and-entities/role-hierarchy.md`](./docs/roles-and-entities/role-hierarchy.md) |
| Engineering/design standards both Laravel repos obey. | [`docs/tech-standards/README.md`](./docs/tech-standards/README.md) |

---

## About this repository (editing rules)

`okta-docs` holds documentation only — no application code. Everything lives under
[`docs/`](./docs/README.md), in two kinds of content:

- **product docs + tech-standards** — the per-service and per-domain documentation
  plus the engineering/design **tech-standards** (`docs/tech-standards/`) consumed
  by `okta-web` and `okta-partners`. Primary language is **Arabic**; keep the
  existing style and RTL-friendly Markdown. Changing a standard can **break** the
  consumer repos — flag such changes as breaking and note a migration path.
- [`docs/reference/`](./docs/reference/) — the **English cross-repo reference layer**
  (the integrated-product map: architecture, repos, web, partners, app,
  installed-apps, deployment, glossary). Use **relative links**; verify each resolves.

Conventions: don't create doc files at the repo root (put them under `docs/`);
don't commit scratch/draft trees (drafts belong in a branch or
outside the repo); when adding a tech-standard, update
[`docs/tech-standards/README.md`](./docs/tech-standards/README.md); when adding a
service doc, list it in [`docs/services/README.md`](./docs/services/README.md)
and follow the standard per-service layout documented there.

Each code repo also ships its own `CLAUDE.md` (`okta-web/CLAUDE.md`,
`okta-partners/CLAUDE.md`, and one per installed-app repo) with repo-local
detail — consult those for in-repo specifics; use *this* hub for the cross-repo,
whole-product picture.
