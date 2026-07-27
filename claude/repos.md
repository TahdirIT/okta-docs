# Repository map

`okta` is **one product** split across four repositories (plus the documentation
hub you are reading). This file describes each repository's responsibility, its
boundaries, and how they depend on one another. For the system picture and
diagrams, read [architecture.md](./architecture.md).

> A fifth kind of repository — an **installable application** — also exists, but
> it is not a fixed repo: each application a Tenant installs lives in its own
> partner repository, scaffolded from `okta-partners`. Its general shape is
> documented separately in [installed-apps.md](./installed-apps.md).

---

## The four repositories

| Repo | One-sentence responsibility | Stack | Deep reference |
|---|---|---|---|
| `okta-web` | The platform/core: owns domain data, exposes services, and hosts installed applications as code; runs in sandbox + production. | PHP 8.2 / Laravel 12, Livewire 4, PostgreSQL, `nwidart/laravel-modules` | [web.md](./web.md) |
| `okta-partners` | The installation/deployment mechanism: author, version, review, publish, and install applications through the bridge into `okta-web`. | PHP 8.3 / Laravel 13, Livewire 4, `firebase/php-jwt` | [partners.md](./partners.md) |
| `okta-app` | The client: a Flutter cross-platform app that lists and launches, per Tenant + role, the applications a Tenant has installed. | Flutter / Dart, Riverpod, go_router, Dio, webview | [app.md](./app.md) |
| `okta-docs` | The documentation hub and session entry point (this repo). | Markdown | [`../CLAUDE.md`](../CLAUDE.md) |

---

## `okta-web` — platform / core

- **Owns**: the domain (users, Tenants, students, employees, sections, terms,
  academic years, notifications, reports, finance, …), authentication, RBAC,
  multitenancy, and the partner-runtime machinery.
- **Hosts**: embedded installable applications as code under `Modules/`
  (`nwidart/laravel-modules`), activated via the DB-backed `DatabaseActivator`.
- **Exposes** (the surfaces other repos consume):
  - `App\Services\*` feature services — including the partner-facing
    `App\Services\PartnerApi\*` slice (in-process API for embedded apps).
  - `routes/apps.php` — the HTTP **Partner-app runtime API** (`/api/apps/*`) for
    external apps, Bearer-token + scope gated.
  - `routes/api.php` — the **mobile API** (`/api/mobile/*`) the `okta-app` client
    calls, including the installed-app **catalog** endpoint.
  - `routes/app.php` — signed **WebView** routes (`/app/{slug}`) that render an
    embedded app's mobile screen.
  - The **scope catalog bridge** (`/api/partners/permissions/catalog[/hash]`) and
    sandbox install/publish endpoints consumed by `okta-partners`.
- **Boundary**: it is the **source of truth** for the scope catalog and for all
  Tenant domain data. Installed-application code must reach that data only through
  `App\Services\PartnerApi\*` — enforced by the policy scanner.

Details: [web.md](./web.md).

---

## `okta-partners` — installation / deployment mechanism

- **Owns**: the partner-application registry (`PartnerModule`,
  `PartnerModuleVersion`, `PartnerApplication`), the authoring/versioning UI, the
  review/publish lifecycle, GitHub repo provisioning, and the boilerplate that
  defines an installable application's skeleton.
- **Mirrors** `okta-web`'s scope catalog locally (`partner_available_scopes`) via
  hash-aware sync — it never invents scopes.
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
  a per-Tenant **catalog** of installed-app cards, and a platform-specific WebView
  host that launches webview-mode screens (served by `okta-web`) and external partner
  pages.
- **Depends only on `okta-web`**: every network call targets `okta-web`'s
  `/api/mobile/*` endpoints (auth, context, app-catalog, launch). It has **no
  dependency on `okta-partners`** and does not read manifests directly — it
  consumes the catalog `okta-web` computes from installed modules' `mobile`
  blocks.
- **Boundary**: a thin client. It holds a Bearer token + base URL in secure
  storage and renders whatever cards `okta-web` returns for the active
  `(tenant, role)`.

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
            │   └──────────────────┘  + provisions partner GitHub repos
            │
            │  HTTP /api/mobile/*
            │
        okta-app
```

- **Runtime**: `okta-partners → okta-web` (bridge); `okta-app → okta-web`
  (mobile API). There is **no** `okta-app ↔ okta-partners` link.
- **Build/scaffold**: `okta-partners` pushes boilerplate to a new
  installable-application repo; an **embedded** application's built code is then
  installed as a module *inside* `okta-web`.
- **Standards**: both Laravel repos consume `okta-docs/docs/tech-standards/`.

See the full dependency map and workspace tree in
[architecture.md](./architecture.md).
