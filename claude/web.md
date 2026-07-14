# okta-web — platform / core

The core of `okta`: a Laravel 12 + Livewire 4 **modular monolith** (PostgreSQL,
`spatie/laravel-multitenancy`, `spatie/laravel-permission`,
`nwidart/laravel-modules`). It owns the domain data, exposes the services every
other repo consumes, hosts embedded installed applications as code, and runs in
**sandbox** and **production** environments.

> The repo ships its own detailed `CLAUDE.md` at `okta-web/CLAUDE.md` — consult
> it for module-by-module specifics and the Finance subsystem. This file covers
> what matters for the integrated product: services, the install machinery,
> environments, and the surfaces other repos call.

Related: [partners.md](./partners.md) · [app.md](./app.md) ·
[installed-apps.md](./installed-apps.md) · [deployment.md](./deployment.md)

---

## Structure (key paths)

```
okta-web/
├── app/
│   ├── Http/Controllers/
│   │   ├── Api/Apps/            # external-app runtime controllers (WhoAmI, Students, Payments, ...)
│   │   ├── Api/Mobile/          # the okta-app client API (MobileAuth, MobileContext, MobileAppCatalog, MobilePortal)
│   │   └── Api/Partners/        # bridge endpoints for okta-partners (ScopeCatalog, Sandbox, Modules)
│   ├── Http/Middleware/         # AuthenticateAppInstallation, EnsureAppScope, EnsureIdempotentRequest, ...
│   ├── Services/
│   │   ├── PartnerApi/          # the in-process partner-facing API surface
│   │   ├── PartnerScopes/       # Catalog, Tokens, Database, Guard, Webhooks
│   │   ├── MobileAuth/          # Context, Portal, Sandbox, Sessions, Tokens
│   │   ├── MobileAppCatalog/    # GetMobileCatalogForUser, GetPortalCatalogForUser, IssueWebviewLaunch, ...
│   │   └── Modules/             # SandboxTenantProvisioner, PartnerSandboxModuleInstaller, ModuleSourcePuller
│   ├── Modules/Core/            # ManifestValidator + Services/{InstallModule,UninstallModule}
│   ├── Modules/Activators/      # DatabaseActivator
│   └── Support/PartnerScopes/   # AppContextManager (singleton), ModuleContext (DTO)
├── Modules/                     # created at runtime — embedded installed applications are pulled
│                                #   into it on install; the repo itself ships no modules
├── routes/                      # web.php, api.php, apps.php, app.php, account.php, webhooks.php,
│                                #   finance.php, ai.php, crm.php, payment.php, ...
├── config/partners.php          # partner-runtime feature flags + limits
└── scripts/partner-policy/      # Scanner.php + UiScanner.php + PHPStan rule (source of truth)
```

---

## Services exposed by okta-web

okta-web follows the platform [feature-services standard](../docs/tech-standards/laravel-feature-services-structure.md):
one use-case per `app/Services/<Feature>/<Resource>/<Function>.php` with a single
`__invoke()`. The slices relevant to the rest of the product:

### `App\Services\PartnerApi\*` — the partner-facing API

The **only** sanctioned way an installed application reads/writes Tenant data.
Organized by domain, returning DTOs (not Eloquent models):

- `Education/` — `Students/`, `Guardians/`, `Subjects/`, `Grades/`, `Sections/`,
  `Terms/`, `AcademicYears/` (list/get/create/update use-cases per resource).
- `Employees/Directory/` — employee directory read/write.
- `Notifications/` — `SendNotification`, `DispatchNotification`,
  `GetNotificationCapabilities`, `ResolveAudience`, and `Providers/`
  (ConnectWhatsApp, Http, Embedded, Hybrid).
- `Payments/` — the unified payment runtime: `ListAvailableMethods`,
  `CreateCharge`, `GetCharge`, `ListCharges`, `RefundCharge`,
  `UpdateChargeStatus`, plus `Providers/{Http,Embedded,Hybrid}PaymentProvider`
  and `ResolveTenantPaymentProvider` (details in `okta-web/CLAUDE.md`).
- `Reports/Builder/` — `ListReports`, `RunReport`.
- `AppSettings/` — `GetAppSetting`/`GetAppSettings` (read the app's own
  developer-panel data store at runtime).
- `DeveloperPanel/` — cross-tenant aggregates for the partner dev panel
  (installs count, charges summary, notification stats) + `Settings/`.
- `Bridge/` — `ResolveUlid`, `ResolveTenantSupportedCountry` (opaque-ID helpers).
- `Health/GetModulesHealth`, `OpenApi/` (`GenerateOpenApiSpec`,
  `GeneratePostmanCollection`), `Dto/` (the stable shapes apps consume).

**Embedded apps** call these classes **in-process** (with a `class_exists` guard
so a missing host service degrades gracefully). **External apps** reach the same
capabilities over HTTP via `routes/apps.php` (below).

### `App\Services\PartnerScopes\*` — scopes, tokens, isolation

- `Catalog/` — `RegisterScope` (the **canonical** scope registry), `GetCatalog`,
  `GetCatalogHash`, and `Events/PartnerScopeCatalogChanged`. This is what
  `okta-partners` mirrors.
- `Tokens/` — `IssueInstallationToken`, `Resolve…FromToken`, `Resolve…ForUser`
  (embedded, session-based), `Rotate…`, `Revoke…`.
- `Database/` — `IssueInstallDbCredentials` (provisions a PostgreSQL role +
  schema), `RegisterInstallConnection`, `RevokeInstallDbCredentials`.
- `Guard/AppPermissionGuard` — `ensure($scope)` / `allows($scope)` against the
  active `ModuleContext`.
- `Webhooks/` — `DispatchEvent` (fan-out to external subscribers) and
  `SyncSubscriptions`.

---

## Surfaces other repos call

<a id="1-external-app-runtime-api--routesappsphp--apiapps"></a>
### 1. External-app runtime API — `routes/apps.php` (`/api/apps/*`)

Used by **external** integrations. Every route:

1. is gated by `auth.app` (`AuthenticateAppInstallation`) — verifies a Bearer
   **installation token** (or session + module slug for embedded), resolves the
   installation, and builds a `ModuleContext` in the `AppContextManager`
   singleton;
2. declares `app.scope:<feature>.<resource>.<action>` (`EnsureAppScope`, → 403
   on failure);
3. delegates to an `App\Services\PartnerApi\*` service — the controller never
   touches Eloquent directly;
4. write routes add `idempotent` (`EnsureIdempotentRequest`, honors
   `Idempotency-Key` for 24h).

Also rate-limited (`throttle:partner-app`, default 600/min) and audited
(`LogPartnerApiCall`).

Route groups and their scopes:

| Group | Scopes used |
|---|---|
| `whoami` | — (auth only) |
| `education/{students,guardians,subjects,grades,sections}` | `education.<resource>.read` / `.write` |
| `education/{academic-years,terms}` | `education.academic_years.read`, `education.terms.read` |
| `employees/directory` | `employees.directory.read` / `.write` |
| `reports/builder` | `reports.builder.read` |
| `notifications/{send,capabilities,dispatch}` | `notifications.providers.send`, `notifications.dispatch.send` |
| `payments/{methods,charges,refunds,status}` | `payments.methods.read`, `payments.charges.{create,read,update}`, `payments.refunds.create` |
| `ai/agent/{stream,summarize}` | — (gated by `EnsureAiPlatformApproved`) |
| `partner-bridge/resolve-numeric` | — (opaque-ID bridge) |

> **Payments are the embedded-only exception.** Although the `payments/*` routes
> live on this external-app surface, **consuming** them is restricted to
> **embedded** apps: `App\Services\PartnerApi\Payments\PaymentCallerGuard`
> `ensureEmbeddedConsumer()` throws `payment_consumption_embedded_only` (**403**)
> for any non-embedded caller (create/list/read/refund). Reason: an external app
> has no in-process `ChargeUpdated` path so it would be blind to the result. And
> `POST /payments/charges/{ref}/status` (`payments.charges.update`) is further
> restricted to the charge's **provider install** via `ensurePaymentProvider()`
> (`payment_status_update_provider_only`, 403) — it is not a generic tenant
> scope. The primary consumption path is in-process (embedded apps call
> `PartnerApi\Payments\*` directly). See `okta-web/CLAUDE.md` for the full
> unified-payment contract.

`GET /api/apps/whoami` returns the active `{module_slug, tenant_id,
installation_id, scopes}`. The whole surface is feature-flagged by
`partners.app_runtime_enabled`.

<a id="2-mobile-client-api"></a>
### 2. Mobile client API — `routes/api.php` (`/api/mobile/*`)

What `okta-app` calls (see [app.md](./app.md)). Four groups, all
`auth:sanctum` except login:

- **Auth** (`/api/mobile/auth/*`): `POST /login` (identifier + password →
  Sanctum token, `throttle:10,1`), `POST /qr-login` (sandbox one-scan login;
  404 outside sandbox), `GET /me`, `GET|POST /context` (list / select
  `{scope, tenant_id, role_ids}`), `POST /logout`.
- **App catalog** (gated by `mobile.min-version` — HTTP 426 if the client is
  too old; silenceable via a `mobile.app_catalog.kill_switch` platform
  setting):
  - `GET /api/mobile/app-catalog` — the **installed-app catalog** for the
    active `(tenant_id, role_id)` (sent as query parameters). Built by
    `App\Services\MobileAppCatalog\GetMobileCatalogForUser` from each installed
    module's manifest `mobile` block (including its per-audience `audiences[]`
    entries), filtered by platform / role / scope.
  - `POST /api/mobile/app-catalog/{slug}/launch` — returns either a
    short-lived **signed** URL to `/app/{slug}` (embedded) or the partner URL +
    optional role JWT (external) — `IssueWebviewLaunch` / `IssueExternalJwt`.
  - `GET /api/mobile/app-catalog/portal?portal=student|guardian` — the
    **cross-tenant portal catalog**: dependent-audience cards a guardian/
    student sees in the general scope (`GetPortalCatalogForUser`).
  - `POST /api/mobile/app-catalog/portal/{slug}/launch` — launch a portal card
    for one of the user's tenants.
- **Portals** (`/api/mobile/portal/*`): `GET /student`, `GET /guardian`
  (cross-tenant portal data) and `POST /students/{hashid}/profile-launch`.
- <a id="partner-notifications"></a>**Notifications** (plain `auth:sanctum`,
  no min-version gate):
  `POST /api/mobile/notification-tokens` (+ `/revoke`) maintains the central
  push-device registry (`notification_device_tokens` — one row per FCM token,
  reassigned on login, pruned when FCM reports `UNREGISTERED`);
  `GET /api/mobile/notifications` (+ `/unread-count`,
  `POST …/{id}/read`, `POST …/read-all`) reads the standard `notifications`
  table through `App\Services\Notifications\Center\*` — the same services the
  web `/notifications-center` page uses. Push delivery is FCM **HTTP v1**
  (`App\Services\Notifications\Push\*`): the service-account JSON is stored
  encrypted in platform settings, minted into a cached OAuth token, and sent
  per device by `SendPushNotificationJob` — wired as the `push` arm of the
  partner `DispatchNotification` fan-out. Unconfigured platforms record
  `skipped_no_transport`. Dynamic audiences from partner dispatch payloads
  (`recipient: {type: parent_of_student | school_admin | host_user, id}`)
  resolve to concrete users via
  `App\Services\PartnerApi\Notifications\ResolveAudience`, falling back to
  the tenant-configured recipients.

### 3. Embedded WebView — `routes/app.php` (`/app/{slug}`)

`GET /app/{slug}` (middleware `web`, `signed`, `app.webview`, `throttle:30,1`)
renders the embedded module's `mobile` entry Blade file from its `mobile/`
directory (per-audience entries resolve to the audience the launch was issued
for), re-confirming that the active role holds the required scope, with
`Cache-Control: no-store` and `Content-Security-Policy: frame-ancestors 'none'`.
This is the page the `okta-app` WebView loads for embedded cards.

### 4. Bridge endpoints for okta-partners — `routes/api.php` (`/api/partners/*`)

Consumed by `okta-partners` (guarded by `partner.api` shared Bearer token):

- `GET /api/partners/permissions/catalog` → `{hash, count, data[], generated_at}`
  with an `ETag` (`ScopeCatalogController` → `GetCatalog`); plus
  `…/catalog/hash` for cheap drift detection.
- Sibling mirrored catalogs on the same pattern:
  `GET /api/partners/countries/catalog[/hash]` and
  `GET /api/partners/account-types/catalog[/hash]`.
- Publish/install machinery: `/api/partners/modules/*` (publish, status),
  `/api/partners/sandbox/{ensure,install,install-status,reset}`
  (`PartnerSandboxController`), `/api/partners/app-store/catalog/resync`, and
  `/api/partners/openapi.json`. See [deployment.md](./deployment.md).

### 5. Tenant store API — `routes/api.php` (`/api/store/*`, session auth)

The in-platform marketplace UI: `GET /modules`, `GET /modules/{slug}`,
`POST /modules/{slug}/install|uninstall`, `GET /modules/{slug}/status`,
`GET /installed`, plus installation-token management
(`show/rotate/revoke`). Container tenants can bulk-install for their children
via `BulkInstallModuleForChildren` (see `okta-web/CLAUDE.md`).

---

## How applications get installed into okta-web

`App\Modules\Core\Services\InstallModule::execute()` (idempotent; reactivates if
previously uninstalled). Steps:

1. Load the module row + decode its `manifest`; **validate** with
   `ManifestValidator`.
2. Reject if already `active`; enforce that all `required` scopes are approved.
3. Upsert `tenant_module_installations` (status `active`) + a
   `tenant_module_subscriptions` row (in a transaction).
4. **Issue an installation token** (`IssueInstallationToken`) bound to the
   granted scopes (default TTL 90 days).
5. For **external** apps: `SyncSubscriptions` materializes the manifest's
   `external.webhookEvents` into `partner_webhook_subscriptions`.
6. If DB isolation is enabled (`partners.db_isolation.enabled`):
   `IssueInstallDbCredentials` provisions a Postgres role + schema (hard-fails →
   rollback).
7. Run the module's migrations/seeders (first install only).
8. Grant `rbac_permissions` to roles; sync `cross_module_access`.
9. In sandbox, `SandboxAutoApprover` may auto-approve platform-AI access.
10. Dispatch the `ModuleInstalled` event.

The embedded module's **code** arrives on the host via
`App\Services\Modules\ModuleSourcePuller` (pulls the pinned commit from the
partner repo into `Modules/<StudlyName>/`) — the okta-web repo itself ships no
modules; `Modules/` is populated per deployment.

`UninstallModule::execute()` reverses this: mark `uninstalled`, cancel the
subscription, revoke RBAC + cross-module access, **revoke installation tokens**,
deactivate (not delete) webhook subscriptions, drop the isolated schema, clear
auth cache, dispatch `ModuleUninstalled`.

---

## Manifest validation

`App\Modules\Core\ManifestValidator::validate($manifest)` is the gate every
published/installed manifest passes. Highlights:

- `moduleId` kebab-case; `version` semver; `displayName`, `category`,
  `description` required.
- `scopes[].key` must match `^[a-z][a-z0-9_]*(\.[a-z][a-z0-9_]*){2}$`
  (`<feature>.<resource>.<action>`), at least one `required:true`, **and every
  key must exist in `partner_scopes` with `is_active=true`** (no orphan grants).
- `integrationType` ∈ `embedded | external | notification | payment` with
  cross-field checks: `external` requires an `external.webhookUrl` (HTTPS) +
  `webhookEvents`; `embedded` must not carry an `external` block;
  `notification` requires its `notification` block (channels + delivery);
  `payment` requires its `payment` block (methods, delivery, capabilities,
  optional `custom_methods` with a bidirectional cross-check).
- `database` block: `requiresDatabase`, optional `schema`/`prefix`,
  `migrations[].version` timestamp + (`path` XOR `sql_up`).
- `mobile` block: block-level defaults (`supported`, `mode`, `entry`,
  `passRoleClaim`, `requiredScope`, `allowedRoles[]`, `allowedPlatforms[]`,
  `allowedOrigins[]`) plus multi-audience `audiences[]` — each
  `{key, kind: primary|dependent, roles[] XOR portal: student|guardian, mode,
  entry, …}`; embedded entries must live under `mobile/` with no `..`.
- Also validated: `pricing`, store metadata (icon/screenshots/keywords/…),
  `developer`, `rbac_permissions`, `aiSupport`/`aiMode`, and `developer_ui`
  (the partner dev-panel block).

> The `menu` block is **not** part of `ManifestValidator` — it is a runtime
> concept read by `App\Livewire\AppsMenu` (the okta-web **header apps menu**) to
> place the launcher tile: `menu.route` (falling back to `sidebar.route`, then
> `<slug>.dashboard`, `<slug>.index`, `/<slug>`, then `store.show`), with optional
> per-account-type `menu.audiences[]`.

The full manifest contract is in [installed-apps.md](./installed-apps.md).

---

## Modules (`nwidart/laravel-modules`)

Embedded applications live under `Modules/<ModuleName>/` with a `module.json`
descriptor and a service provider. **The okta-web repo ships no modules** — the
`Modules/` directory is created on the host when the first application is
installed (source pulled from the partner repo at the pinned commit).
Activation state is stored in the DB, not the filesystem:

- **`DatabaseActivator`** (`MODULES_ACTIVATOR=database`) keeps enable/disable
  state in the landlord `module_statuses` table, so `git pull` on production
  cannot reset what an admin toggled. `MODULES_ACTIVATOR=file` restores the
  legacy `modules_statuses.json` behavior.
- First migration from file → database:
  `php artisan migrate && php artisan modules:import-statuses-from-json`.

Three real installed-application repos live in this workspace —
`okta-smart-timetable`, `okta-exams`, `okta-hdor` — see
[installed-apps.md](./installed-apps.md#real-examples).

---

## Environments: sandbox & production

okta-web is operated as **sandbox** (development/testing) and **production** —
the dev → prod progression in [deployment.md](./deployment.md). Distinctions
seen in config/`.env`:

- **Instance flag**: `OKTA_IS_SANDBOX` (`config/okta.php → is_sandbox`),
  honored by `SandboxAutoApprover::isInstanceSandbox()` (which also accepts
  `APP_ENV=sandbox`).
- **Module databases**: `MODULES_DB_NAME` (production) vs
  `MODULES_SANDBOX_DB_NAME` (sandbox tenants), each with optional
  host/port/cred overrides. Installed apps' isolated DBs are named
  `<slug>_<hashid>_<sandbox|production>`.
- **Sandbox tenants**: `is_sandbox=true` on a Tenant; provisioned by
  `App\Services\Modules\SandboxTenantProvisioner` and installed into by
  `PartnerSandboxModuleInstaller` (driven from okta-partners through
  `/api/partners/sandbox/*`). Sandbox installs auto-approve platform-AI access
  (`SandboxAutoApprover`) so partners can test without manual approval.
- **Partner-runtime config** (`config/partners.php`):
  `PARTNER_APP_RUNTIME_ENABLED`, `PARTNER_APP_RATE_LIMIT_PER_MINUTE` (600),
  `PARTNER_INSTALLATION_TOKEN_TTL_DAYS` (90), `PARTNER_DB_ISOLATION_ENABLED`,
  `PARTNER_NOTIFICATIONS_ENABLED`.
- **Bridge auth**: `OKTA_PARTNERS_API_TOKEN` (shared Bearer for
  `/api/partners/*`), plus outbound webhook URL/secret to notify
  `okta-partners`.

Base hosts as configured in the `okta-app` client: production
`https://getokta.io`, development `https://dev.getokta.io` (the partners side
resolves its own bridge URLs dynamically through `BridgeSettings`).

---

## Key models

`PartnerInstallationToken` (bearer credential), `PartnerInstallDbCredential`
(isolated role/schema), `PartnerScope` (catalog mirror),
`TenantModuleInstallation` / `TenantModuleSubscription`,
`PartnerWebhookSubscription` / `PartnerWebhookDelivery`,
`PartnerSandboxCredential` / `SandboxModuleInstallation` (sandbox machinery),
`PartnerModuleDevSetting` (dev-panel data store), `PartnerPaymentCharge`
(unified payment runtime), and `Module` (the registry row holding the validated
`manifest`). The request-scoped `ModuleContext` (DTO) + `AppContextManager`
(singleton) carry tenant + module + installation + granted scopes through a
request.

---

## Partner-runtime operations (monitor, dev panel, Pulse)

Operational surfaces around the partner runtime (detail in `okta-web/CLAUDE.md`):

- **Payments Monitor** — unified monitoring of `partner_payment_charges`. Tenant
  page `GET /partner-apps/payment/charges` (`PaymentChargesIndex`) and
  landlord-wide `…/charges/all` (`PaymentChargesAllIndex`); shared trait
  `Concerns/MonitorsPaymentCharges` + `PaymentChargeDetail` modal (keyed by ULID
  `chargeRef` only). Attribution to service (consumer install) + provider install
  with no N+1 via `PartnerApi\Payments\ResolveInstallationApps`. Permissions
  `payments.charges.view` (tenant) / `payments.charges.view_all` (system).
- **Developer Panel (host runtime)** — a per-app dashboard the platform renders
  for the app's author. Route `GET /partner-dev/{moduleSlug}/panel`
  (`DeveloperPanelController`, gated only by `VerifyDevPanelToken` — no tenant
  context). JWT handoff from okta-partners (HS256, `aud=okta-web-devpanel`,
  `exp=iat+300`) writes a trusted session context (`Support/DevPanel/DevPanelContext`,
  `dev_panel.<slug>`, TTL 60m) so later Livewire actions re-authorize without the
  token. Editable per-app data store `partner_module_dev_settings`
  (`PartnerModuleDevSetting`, ≤100 keys/module) via
  `PartnerApi\DeveloperPanel\Settings\{List,Put,Delete}Setting`; read at runtime
  by `PartnerApi\AppSettings\GetAppSetting[s]` (per-module 60s cache invalidated
  on write). Manifest opt-in: `developer_ui` block.
- **Pulse cards** — `App\Pulse\Recorders\{PartnerApiCalls,PartnerWebhookDeliveries}`
  feed `<livewire:pulse.partner-api-activity />` and
  `<livewire:pulse.partner-webhook-outcomes />` (keyed by scope / terminal outcome).
- **Outbound webhook detail** — `Observers/PartnerEvents/*` turn domain changes
  into canonical events → `DispatchEvent` → queued `DeliverPartnerWebhook` (sign +
  send, backoff `30s → 6h` then `giving_up`; `PartnerWebhookDelivery` rows);
  recovery scheduler `partners:recover-webhook-deliveries` every minute.
