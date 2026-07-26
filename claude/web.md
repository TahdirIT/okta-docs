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
│   │   ├── Api/Apps/            # external-app runtime controllers (WhoAmI, Students, ...)
│   │   ├── Api/Mobile/          # the okta-app client API (MobileAuth, MobileContext, MobileAppCatalog)
│   │   └── Api/Partners/        # bridge endpoints for okta-partners (ScopeCatalogController)
│   ├── Http/Middleware/         # AuthenticateAppInstallation, EnsureAppScope, EnsureIdempotentRequest, ...
│   ├── Services/
│   │   ├── PartnerApi/          # the in-process partner-facing API surface
│   │   └── PartnerScopes/       # Catalog, Tokens, Database, Guard, Webhooks
│   ├── Modules/Core/            # InstallModule, UninstallModule, ManifestValidator
│   ├── Modules/Activators/      # DatabaseActivator
│   └── Support/PartnerScopes/   # AppContextManager (singleton), ModuleContext (DTO)
├── Modules/                     # embedded installed applications (nwidart modules)
├── routes/                      # web.php, api.php, apps.php, app.php, webhooks.php, finance.php
├── config/partners.php          # partner-runtime feature flags + limits
└── scripts/partner-policy/      # Scanner.php + PHPStan rule (source of truth)
```

---

## Services exposed by okta-web

okta-web follows the platform [feature-services standard](../docs/tech-standards/laravel-feature-services-structure.md):
one use-case per `app/Services/<Feature>/<Resource>/<Function>.php` with a single
`__invoke()`. The slices relevant to the rest of the product:

### `App\Services\PartnerApi\*` — the partner-facing API

The **only** sanctioned way an installed application reads/writes Tenant data.
Organized by domain, returning DTOs (not Eloquent models):

- `Education/Students/` — `ListStudents`, `GetStudent`, `CreateStudent`,
  `UpdateStudent`; likewise `Grades/`, `Sections/`, `Subjects/`, `Terms/`,
  `AcademicYears/`.
- `Employees/Directory/` — employee CRUD.
- `Notifications/` — `SendNotification`, `DispatchNotification`,
  `GetNotificationCapabilities`, and `Providers/` (ConnectWhatsApp, Http,
  Embedded, Hybrid).
- `Reports/Builder/` — `ListReports`, `RunReport`.
- `Dto/` — `StudentDto`, `EmployeeDto`, `ReportDto`, … (the stable shapes apps
  consume).
- `OpenApi/` — `GenerateOpenApiSpec`, `GeneratePostmanCollection`.

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

1. is gated by `app.installation` (`AuthenticateAppInstallation`) — verifies a
   Bearer **installation token** (or session + `X-Module-Slug` for embedded),
   resolves the installation, and builds a `ModuleContext` in the
   `AppContextManager` singleton;
2. declares `app.scope:<feature>.<resource>.<action>` (`EnsureAppScope`, → 403
   `X-Missing-Scope` on failure);
3. delegates to an `App\Services\PartnerApi\*` service — the controller never
   touches Eloquent directly;
4. write routes add `idempotent` (`EnsureIdempotentRequest`, honors
   `Idempotency-Key` for 24h).

Also rate-limited (`throttle:partner-app`, default 600/min) and audited
(`LogPartnerApiCall`). Example shape:

```php
Route::get('/education/students', [StudentsController::class, 'index'])
    ->middleware('app.scope:education.students.read')
    ->name('education.students.index');
```

`GET /api/apps/whoami` returns the active `{module_slug, tenant_id,
installation_id, scopes}`. The whole surface is feature-flagged by
`partners.app_runtime_enabled`.

<a id="2-mobile-client-api"></a>
### 2. Mobile client API — `routes/api.php` (`/api/mobile/*`)

What `okta-app` calls (see [app.md](./app.md)):

- `POST /api/mobile/auth/login` — identifier + password → Bearer token + tenant/
  role list (`throttle:10,1`).
- `POST /api/mobile/auth/context` — select `{scope, tenant_id, role_ids}`.
- `GET /api/mobile/app-catalog` — the **installed-app catalog** for the active
  `(tenant_id, role_id)`. Returns `{ modules: [ {slug, display_name, mode,
  entry, allowed_platforms, allowed_roles, required_scope, pass_role_claim,
  allowed_origins, icon} ], _debug }`. Built by
  `App\Services\MobileAppCatalog\GetMobileCatalogForUser` from each installed
  module's manifest `mobile` block, filtered by platform / role / scope.
  Version-gated by `EnforceMobileMinVersion` (HTTP 426 if the client is too old)
  and silenceable via a `mobile.app_catalog.kill_switch` platform setting.
- `POST /api/mobile/app-catalog/{slug}/launch` — returns either a short-lived
  **signed** URL to `/app/{slug}` (embedded) or the partner URL + optional role
  JWT (external).
- <a id="partner-notifications"></a>**Notifications surface** (plain
  `auth:sanctum`, no min-version gate):
  `POST /api/mobile/notification-tokens` (+ `/revoke`) maintains the central
  push-device registry (`notification_device_tokens` — one row per FCM token,
  reassigned on login, pruned when FCM reports `UNREGISTERED`);
  `GET /api/mobile/notifications` (+ `/unread-count`,
  `POST …/{id}/read`, `POST …/read-all`) reads the standard `notifications`
  table through `App\Services\Notifications\Center\*` — the same services the
  web `/notifications-center` page uses. Push delivery is FCM **HTTP v1**
  (`App\Services\Notifications\Push\*`): the service-account JSON is stored
  encrypted in platform settings (platform-delivery page), minted into a
  cached OAuth token, and sent per device by
  `SendPushNotificationJob` — wired as the `push` arm of the partner
  `DispatchNotification` fan-out. Unconfigured platforms record
  `skipped_no_transport`. Dynamic audiences from partner dispatch payloads
  (`recipient: {type: parent_of_student | school_admin | host_user, id}`)
  resolve to concrete users via
  `App\Services\PartnerApi\Notifications\ResolveAudience`, falling back to
  the tenant-configured recipients.

> The catalog route is registered as `GET`; the client sends `tenant_id`/
> `role_id` as query parameters. `> TODO: confirm` — one server-side comment
> describes a `POST` body; treat `GET` (the route definition) as authoritative.

### 3. Embedded WebView — `routes/app.php` (`/app/{slug}`)

`GET /app/{slug}` (middleware `web`, `signed`, `app.webview`, `throttle:30,1`)
renders the embedded module's `mobile.entry` Blade file from its `mobile/`
directory, re-confirming that the active role holds the required scope, with
`Cache-Control: no-store` and `Content-Security-Policy: frame-ancestors 'none'`.
This is the page the `okta-app` WebView loads for embedded cards.

### 3b. Native mini-app payload — `MiniappPayloadController`

For `mobile.mode: native` cards, okta-app fetches a signed payload URL instead of
a WebView URL. `MiniappPayloadController` calls `BundleMiniappSource`, which reads
every `.dart` file under the module's `miniapp_dart/lib/` (realpath-fenced, size
caps) into one **source bundle** (`{package, entry_file, entry_function,
min_contract, files}`) and returns it with `Cache-Control: no-store`. okta-app
compiles that source **on the device** (via `okta_miniapp`/`dart_eval`) and renders
it natively — no WebView. The legacy schema/JSON bundler (`BundleMiniappPayload`)
and the `miniapp/` runtime have been removed.

### 4. Scope-catalog bridge — `routes/api.php` (`/api/partners/*`)

Consumed by `okta-partners` (guarded by `partner.api` shared Bearer token):

- `GET /api/partners/permissions/catalog` → `{hash, count, data[], generated_at}`
  with an `ETag` (`App\Http\Controllers\Api\Partners\ScopeCatalogController` →
  `GetCatalog`).
- `GET /api/partners/permissions/catalog/hash` → `{hash, generated_at}` for cheap
  drift detection.

Plus the publish and sandbox-install endpoints `okta-partners` posts to (see
[deployment.md](./deployment.md)).

---

## How applications get installed into okta-web

`App\Modules\Core\InstallModule::execute()` (idempotent; reactivates if
previously uninstalled). Steps:

1. Load the module row + decode its `manifest`; **validate** with
   `ManifestValidator`.
2. Reject if already `active`; enforce that all `required` scopes are approved.
3. Upsert `tenant_module_installations` (status `active`) + a `free`
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
- `integrationType` ∈ `embedded | external | notification` with cross-field
  checks: `external` requires an `external.webhookUrl` (HTTPS) + `webhookEvents`;
  `embedded` must not carry an `external` block; `notification` requires its
  `notification` block (channels + delivery).
- `database` block: `requiresDatabase`, optional `schema`/`prefix`,
  `migrations[].version` timestamp + (`path` XOR `sql_up`).
- `mobile` block shape (consumed for the client surface — see
  [installed-apps.md](./installed-apps.md)).

The full manifest contract is in [installed-apps.md](./installed-apps.md).

---

## Modules (`nwidart/laravel-modules`)

Embedded applications live under `Modules/<ModuleName>/` with a `module.json`
descriptor and a service provider. Activation state is stored in the DB, not the
filesystem:

- **`DatabaseActivator`** (`MODULES_ACTIVATOR=database`) keeps enable/disable
  state in the landlord `module_statuses` table, so `git pull` on production
  cannot reset what an admin toggled. `MODULES_ACTIVATOR=file` restores the
  legacy `modules_statuses.json` behavior.
- First migration from file → database:
  `php artisan migrate && php artisan modules:import-statuses-from-json`.

---

## Environments: sandbox & production

okta-web is operated as **sandbox** (development/testing) and **production** —
the dev → prod progression in [deployment.md](./deployment.md). Distinctions seen
in config/`.env`:

- **Module databases**: `MODULES_DB_NAME` (production) vs
  `MODULES_SANDBOX_DB_NAME` (sandbox tenants), each with optional host/port/cred
  overrides.
- **Sandbox tenants**: `is_sandbox=true` on a Tenant; instance-level sandbox via
  `APP_ENV=sandbox`. Sandbox installs auto-approve platform-AI access
  (`SandboxAutoApprover`) so partners can test without manual approval.
- **Partner-runtime config** (`config/partners.php`):
  `PARTNER_APP_RUNTIME_ENABLED`, `PARTNER_APP_RATE_LIMIT_PER_MINUTE` (600),
  `PARTNER_INSTALLATION_TOKEN_TTL_DAYS` (90), `PARTNER_DB_ISOLATION_ENABLED`,
  `PARTNER_NOTIFICATIONS_ENABLED`.
- **Bridge auth**: `OKTA_PARTNERS_API_TOKEN` (shared Bearer for `/api/partners/*`),
  plus outbound webhook URL/secret to notify `okta-partners`.

The production base host observed in client config is `https://getokta.io`
(`okta-app` default base URL). `> TODO: confirm` the exact sandbox/production
hostnames for the bridge (the partners side resolves them dynamically through
`BridgeSettings`).

---

## Key models

`PartnerInstallationToken` (bearer credential), `PartnerInstallDbCredential`
(isolated role/schema), `PartnerScope` (catalog mirror),
`TenantModuleInstallation` / `TenantModuleSubscription`,
`PartnerWebhookSubscription` / `PartnerWebhookDelivery`, `Module` (the registry
row holding the validated `manifest`). The request-scoped `ModuleContext` (DTO) +
`AppContextManager` (singleton) carry tenant + module + installation + granted
scopes through a request.
