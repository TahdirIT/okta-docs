# Architecture

How the `okta` product fits together as one integrated system: the repositories,
the data/control flow between them, the **dual-surface** model for installed
applications, and the sandbox → production progression.

Read alongside [repos.md](./repos.md) (responsibilities & boundaries) and
[glossary.md](./glossary.md) (terms).

---

## The integrated product view

`okta` is a single educational platform, not a collection of independent
products. The split into repositories is a deployment/ownership boundary, not a
product boundary:

- **`okta-web`** is the core: it owns the domain data and is the source of truth.
  Everything else orbits it.
- **`okta-partners`** is how capabilities are added to the core: applications are
  authored there and *installed through it* into `okta-web`.
- **`okta-app`** is how Tenants consume the core on mobile/desktop: it lists and
  launches the applications a Tenant has installed (plus the cross-tenant
  student/guardian portals).
- **`okta-docs`** is how the whole thing is documented and standardized.
- **Installed-application repos** (one per app, each in its own partner repo)
  carry the applications themselves.

An **installed application** is the thread that ties the three runtime repos
together: authored/published in `okta-partners`, installed into `okta-web`, and
surfaced in `okta-app`.

---

## Workspace structure tree

Top-level layout of all repositories in the workspace and their key
subdirectories (generated from the filesystem; vendored/build dirs omitted).

```
.
├── okta-web/                      # Platform / core — Laravel 13, Livewire 4, PostgreSQL
│   ├── app/
│   │   ├── Http/                  # Controllers (Api/Apps, Api/Mobile, Api/Partners), Middleware
│   │   ├── Services/
│   │   │   ├── PartnerApi/        # in-process partner-facing API (Education/, Payments/, Notifications/, ...)
│   │   │   ├── PartnerScopes/     # Catalog/, Tokens/, Database/, Guard/, Webhooks/
│   │   │   ├── MobileAuth/ MobileAppCatalog/   # the okta-app client backend
│   │   │   └── Modules/           # sandbox provisioning + module source pull
│   │   ├── Modules/               # Core (ManifestValidator, Install/Uninstall) + Activators
│   │   ├── Models/, Observers/, Pulse/, Support/
│   ├── Modules/                   # created at runtime — installed (embedded) apps are pulled here
│   ├── routes/                    # web.php, api.php, apps.php, app.php, account.php, webhooks.php, finance.php, ...
│   ├── config/                    # partners.php, okta.php, multitenancy, modules, ...
│   ├── database/                  # migrations (landlord + tenant), seeders
│   ├── scripts/partner-policy/    # Scanner + UiScanner + PHPStan rule (source of truth for the boilerplate copy)
│   └── tests/
│
├── okta-partners/                 # Installation / deployment mechanism — Laravel 13, PostgreSQL, JWT
│   ├── app/
│   │   ├── Services/              # OktaWebService, BridgeSettings, GitHubAppService,
│   │   │                          #   ModuleLifecycleService, PartnerScopes/Catalog/*,
│   │   │                          #   PartnerModules/{Payments,Notifications,Ci}/, AppStorePricing/
│   │   ├── Models/                # PartnerModule, PartnerModuleVersion, PartnerAvailableScope,
│   │   │                          #   PartnerModulePricing, PartnerPayout*, UserMembership
│   │   ├── Livewire/              # Admin/{Modules,Sandbox,Settings,AppStore}/, Partner/Modules/*
│   │   ├── Http/                  # Controllers/Api (OktaWebWebhookController, CiPreviewInstall), Middleware
│   │   └── Enums/                 # IntegrationType, ModuleStatus, PaymentMethod, AppStore/*
│   ├── resources/partner-boilerplate/   # the skeleton pushed to every new application repo
│   ├── routes/                    # web.php, partner.php, admin.php, api.php, webhooks.php
│   └── tests/
│
├── okta-app/                      # Client — Flutter / Dart
│   ├── lib/
│   │   ├── main.dart, app.dart    # entry + root MaterialApp.router
│   │   ├── router/                # go_router (splash, auth, home, portals, notifications, settings)
│   │   ├── core/                  # api/ (Dio), storage/ (secure), settings/, push/, theme/, l10n/, widgets/
│   │   └── features/
│   │       ├── auth/              # login, QR sandbox login, tenant/role selection, session bootstrap
│   │       ├── app_catalog/       # installed-app cards + WebView host (the mobile surface)
│   │       ├── portal_app_catalog/# cross-tenant portal cards (student/guardian)
│   │       ├── portal/            # student & guardian portal screens
│   │       ├── home/              # dashboard hosting the catalog
│   │       ├── notifications/     # in-app feed + FCM push
│   │       └── settings/
│   ├── android/  ios/  windows/   # native platform shells present in-repo
│   ├── assets/                    # brand + fonts
│   └── pubspec.yaml
│
├── okta-docs/                     # Documentation hub (this repo)
│   ├── CLAUDE.md                  # entry point + map
│   └── docs/                      # all documentation
│       ├── reference/             # this cross-repo reference layer
│       ├── tech-standards/        # consumed by both Laravel repos
│       └── services/ · apps/ · roles-and-entities/ · …
│
└── <installed-app repos>/         # one per app, each in its own partner repo
    #   each: manifest.json, module.json, app/, mobile/, scripts/partner-policy/, ...
    #   scaffolded from okta-partners/resources/partner-boilerplate/
    #   embedded apps are installed (source-pulled) into okta-web/Modules/
```

> An installable application's own structure (`manifest.json`, `module.json`,
> `app/`, `mobile/`, `scripts/partner-policy/`, …) is documented in
> [installed-apps.md](./installed-apps.md).

---

## Build / runtime dependency map

```
            ┌─────────────────────────────────────────────────────────┐
            │                       okta-docs                          │
            │  docs/tech-standards/  (feature-services, permissions,   │
            │  design) — consumed as standards by both Laravel repos    │
            └───────────────▲───────────────────────▲─────────────────┘
                            │ standards              │ standards
                            │                        │
   ┌────────────────────────┴────┐      ┌────────────┴───────────────────────┐
   │          okta-web            │◄─────┤            okta-partners            │
   │   (platform / core; source   │ HTTP │  (authoring + deployment mechanism) │
   │    of truth for domain data  │bridge│                                     │
   │    and the scope catalog)    │      │  • publishes manifests → okta-web   │
   │                              │─────►│  • mirrors scope catalog (hash sync)│
   │  • /api/partners/...         │ wbhk │  • installs versions into sandbox   │
   │  • /api/apps/...  (external) │      │  • pushes boilerplate to partner    │
   │  • /api/mobile/... (client)  │      │    GitHub repos                     │
   │  • Modules/ (embedded apps)  │      └─────────────────┬───────────────────┘
   └───────▲──────────────────────┘                        │ git push boilerplate + managed files
           │ HTTP /api/mobile/*                            ▼
           │                                      ┌──────────────────────────┐
   ┌───────┴────────┐                             │ installed-application     │
   │    okta-app    │                             │ repos (one per app)       │
   │ (Flutter client)│                            │ • embedded ⇒ code pulled   │
   └────────────────┘                             │   into okta-web/Modules    │
                                                  │ • external ⇒ hosted by     │
                                                  │   partner, HTTP+webhooks   │
                                                  └──────────────────────────┘
```

Key edges (all code-grounded):

- **`okta-partners → okta-web`** (HTTP bridge): `OktaWebService` calls, addressed
  via `BridgeSettings::apiUrl()`/`sandboxApiUrl()` with `outboundToken()`;
  endpoints on `okta-web` are guarded by `partner.api` middleware. Carries
  publish, install-on-sandbox, status, and catalog traffic.
- **`okta-web → okta-partners`** (signed webhooks): `POST /webhooks/okta-web`
  (scope-catalog changes, publish results, app-store sales), verified by
  `VerifyOktaWebWebhook` (HMAC envelope + replay protection).
- **`okta-app → okta-web`** (mobile API): Dio client with a Bearer token; calls
  `/api/mobile/auth/*`, `/api/mobile/app-catalog` (+ the portal card catalog
  `/app-catalog/portal`), and the per-app launch endpoints. No other repo is
  contacted.
- **`okta-partners → partner repo`** (GitHub App): `GitHubAppService::pushBoilerplate()`
  scaffolds the application repository the partner connected;
  `syncManagedFiles()` keeps the policy scanners/CI/CLAUDE.md in step.
- **partner repo → `okta-web`** (install): `ModuleSourcePuller` fetches the
  version's pinned commit into `okta-web/Modules/` at install time.
- **embedded app code → `okta-web`** (in-process): the module calls
  `App\Services\PartnerApi\*` directly; **never** touches platform internals
  (enforced by the policy scanner).

---

## Dual surface

The defining property of an installed application: **one installation, two
surfaces.**

```
                         Tenant installs an application
                                     │
              ┌──────────────────────┴───────────────────────┐
              ▼                                                ▼
   ── Platform surface (okta-web) ──            ── Client surface (okta-app) ──
   Embedded module renders in-tenant            okta-app calls /api/mobile/app-catalog,
   Livewire/Blade UI in the web sidebar         gets a card built from the module's
   at manifest `menu.route`, gated by           `mobile` manifest block (audience-resolved,
   `module.access:<slug>`.                       filtered by platform + role + scope), then
                                                 launches:
                                                  • embedded ⇒ signed WebView URL to
                                                    okta-web `/app/{slug}` rendering the
                                                    audience's `entry` blade
                                                  • external ⇒ partner URL (+ optional
                                                    role JWT if `passRoleClaim`)

   Dependent audiences (student/guardian) additionally surface in the
   cross-tenant portals via /api/mobile/app-catalog/portal — no Tenant selection.
```

Both surfaces are driven by the **same** published manifest and the **same**
installation/scopes. The platform surface uses the manifest's `menu` block; the
client surface uses the manifest's `mobile` block (with optional per-audience
entries). See [installed-apps.md](./installed-apps.md) for the manifest contract
and [deployment.md](./deployment.md) for how an install reaches both surfaces.

---

## Control flow: from authoring to both surfaces

1. **Author** an application in `okta-partners` (choose `integrationType`,
   select scopes from the mirrored catalog, create a version).
2. **Publish**: `ModuleLifecycleService::publish()` builds the manifest and calls
   `OktaWebService::publishModule()` → `okta-web` validates it
   (`ManifestValidator`) and records it.
3. **Install** into a Tenant: `okta-web`'s `InstallModule` issues an installation
   token, grants scopes/RBAC, (optionally) provisions an isolated DB schema,
   pulls the module source and runs its migrations, and — for external apps —
   syncs webhook subscriptions.
4. **Platform surface** lights up immediately: the embedded module's UI appears
   in the web sidebar.
5. **Client surface** lights up: `okta-app` requests `/api/mobile/app-catalog`
   for the active `(tenant, role)` (and the portal catalog for dependent
   audiences) and renders the cards; launching one opens the embedded WebView or
   the external page.

The same sequence runs first in **sandbox** (development/testing) and then in
**production** — the dev → prod progression detailed in
[deployment.md](./deployment.md).

---

## Data isolation & trust boundaries

- **Tenant isolation**: `spatie/laravel-multitenancy` scopes domain data per
  Tenant; landlord tables hold shared data (catalog, module statuses, finance).
- **Application isolation**: an installed application may read/write Tenant data
  *only* through `App\Services\PartnerApi\*` (in-process for embedded) or the
  scope-gated `/api/apps/*` HTTP API (for external). Direct access to
  `App\Models\*`, non-`PartnerApi` services, platform tables, env, and config is
  blocked statically by the [policy scanner](./installed-apps.md#policy-scanner)
  and at runtime by the `BlocksPartnerDirectAccess` trait on sensitive models.
- **Per-installation DB**: when DB isolation is enabled, each installation gets a
  dedicated PostgreSQL role + schema (`<slug>_<hashid>_<env>`); module-owned
  tables never join platform tables (cross-database, opaque ULID references).
- **Bridge trust**: partners↔web traffic is authenticated (shared Bearer +
  signed webhooks with replay protection); the scope catalog is one-directional
  (web is source of truth, partners mirrors).
