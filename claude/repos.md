# Repository map

`okta` is **one product** split across four core repositories (plus the
documentation hub you are reading). This file describes each repository's
responsibility, its boundaries, and how they depend on one another. For the
system picture and diagrams, read [architecture.md](./architecture.md).

> A fifth kind of repository — an **installable application** — is not a fixed
> repo: each application a Tenant installs lives in its own partner repository,
> scaffolded from `okta-partners`. Three such repos live in this workspace —
> `okta-smart-timetable`, `okta-exams`, and `okta-hdor` — and the general shape
> is documented in [installed-apps.md](./installed-apps.md).

---

## The core repositories

| Repo | One-sentence responsibility | Stack | Deep reference |
|---|---|---|---|
| `okta-web` | The platform/core: owns domain data, exposes services, and hosts installed applications as code; runs in sandbox + production. | PHP 8.2 / Laravel 12, Livewire 4, PostgreSQL, `nwidart/laravel-modules` | [web.md](./web.md) |
| `okta-partners` | The installation/deployment mechanism: author, version, review, publish, and install applications through the bridge into `okta-web`. | PHP 8.3 / Laravel 13, Livewire 4, `firebase/php-jwt` | [partners.md](./partners.md) |
| `okta-app` | The client: a Flutter cross-platform app that lists and launches, per Tenant + role, the applications a Tenant has installed (plus student/guardian portals). | Flutter / Dart, Riverpod, go_router, Dio, webview | [app.md](./app.md) |
| `okta-docs` | The documentation hub and session entry point (this repo). | Markdown | [`../CLAUDE.md`](../CLAUDE.md) |

### Installed-application repos in this workspace

| Repo | Application | Type |
|---|---|---|
| `okta-smart-timetable` | «اوكتا الجدول الذكي» — AI timetable generation + daily class operations | embedded |
| `okta-exams` | «اوكتا الاختبارات النهائية» — final-exam committees, seating, observers, reports | embedded (paid) |
| `okta-hdor` | «اوكتا حضور» — student attendance, leave requests, parent notifications | embedded |

Each follows the [installed-application contract](./installed-apps.md): scaffolded
from the `okta-partners` boilerplate, gated by the partner-policy scanners, and
installed (source-pulled) into `okta-web/Modules/` per deployment.

---

## `okta-web` — platform / core

- **Owns**: the domain (users, Tenants, students, employees, sections, terms,
  academic years, notifications, reports, finance, …), authentication, RBAC,
  multitenancy, and the partner-runtime machinery.
- **Hosts**: embedded installable applications as code under `Modules/`
  (`nwidart/laravel-modules`) — pulled in at install time (the repo ships no
  modules), activated via the DB-backed `DatabaseActivator`.
- **Exposes** (the surfaces other repos consume):
  - `App\Services\*` feature services — including the partner-facing
    `App\Services\PartnerApi\*` slice (in-process API for embedded apps:
    education, employees, notifications, payments, reports, AI, app settings).
  - `routes/apps.php` — the HTTP **Partner-app runtime API** (`/api/apps/*`) for
    external apps, Bearer-token + scope gated.
  - `routes/api.php` — the **mobile API** (`/api/mobile/*`) the `okta-app` client
    calls: auth/context, the installed-app **catalog** + launch, the
    student/guardian **portals** + portal catalog, and notifications.
  - `routes/app.php` — signed **WebView** routes (`/app/{slug}`) that render an
    embedded app's mobile screen.
  - The **bridge endpoints** consumed by `okta-partners`
    (`/api/partners/*`: scope/countries/account-types catalogs, publish, sandbox
    ensure/install/reset, app-store resync).
- **Boundary**: it is the **source of truth** for the scope catalog and for all
  Tenant domain data. Installed-application code must reach that data only through
  `App\Services\PartnerApi\*` — enforced by the policy scanner.

Details: [web.md](./web.md).

---

## `okta-partners` — installation / deployment mechanism

- **Owns**: the partner-application registry (`PartnerModule`,
  `PartnerModuleVersion`, `PartnerApplication`), the authoring/versioning UI, the
  review/publish lifecycle, GitHub repo scaffolding (via the platform GitHub
  App), monetization (pricing matrix, payouts, discount codes), and the
  boilerplate that defines an installable application's skeleton.
- **Mirrors** `okta-web`'s scope catalog locally (`partner_available_scopes`) via
  hash-aware sync — it never invents scopes (likewise the countries and
  account-types catalogs).
- **Drives deployment**: builds a manifest (`PartnerModuleVersion::buildManifest()`),
  publishes it to `okta-web`, and can install a version into a partner **sandbox**
  for testing before **production**.
- **Boundary**: a *separate* application — it shares **no code** with `okta-web`;
  all communication is over the authenticated HTTP **bridge** (`OktaWebService` +
  `BridgeSettings`). It does **not** talk to `okta-app` at all.

Details: [partners.md](./partners.md).

---

## `okta-app` — the client

- **Owns**: the cross-platform end-user client. Login + active-context selection,
  a per-Tenant **catalog** of installed-app cards, cross-tenant
  **student/guardian portals** with their own app cards, an in-app
  **notifications** feed with FCM push, and a platform-specific WebView host
  that launches embedded screens (served by `okta-web`) and external partner
  pages.
- **Depends only on `okta-web`**: every network call targets `okta-web`'s
  `/api/mobile/*` endpoints (auth, context, app-catalog, portals, launch,
  notifications). It has **no dependency on `okta-partners`** and does not read
  manifests directly — it consumes the catalog `okta-web` computes from
  installed modules' `mobile` blocks.
- **Boundary**: a thin client. It holds a Bearer token + base URL in secure
  storage and renders whatever cards `okta-web` returns for the active
  `(tenant, role)` or portal.

Details: [app.md](./app.md).

---

## `okta-docs` — documentation hub

- **Owns**: product documentation, the engineering/design **tech-standards**
  (`docs/tech-standards/`) that `okta-web` and `okta-partners` must obey, and this
  `claude/` reference layer.
- **No application code.**
- **Boundary**: changing a standard here can break the consumer repos — flag such
  changes as breaking (see the editing rules in [`../CLAUDE.md`](../CLAUDE.md)).

---

## Dependency direction (who depends on whom)

```
                 okta-docs  (standards consumed by both Laravel repos)
                    ▲   ▲
                    │   │
        okta-web ───┘   └─── okta-partners
            ▲   ▲                  │
            │   │                  │  HTTP bridge (publish / install / sync)
            │   └──────────────────┘  + pushes boilerplate to partner GitHub repos
            │
            │  HTTP /api/mobile/*          okta-smart-timetable / okta-exams / okta-hdor
            │                                  (installed-app repos; embedded code is
        okta-app                                source-pulled into okta-web/Modules/)
```

- **Runtime**: `okta-partners → okta-web` (bridge); `okta-app → okta-web`
  (mobile API). There is **no** `okta-app ↔ okta-partners` link.
- **Build/scaffold**: `okta-partners` pushes boilerplate to a new
  installable-application repo; an **embedded** application's code is then
  installed as a module *inside* `okta-web` (pulled at the pinned commit).
- **Standards**: both Laravel repos consume `okta-docs/docs/tech-standards/`.

See the full dependency map and workspace tree in
[architecture.md](./architecture.md).
