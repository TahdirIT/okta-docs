# The installed-application model

This is the **general contract** any application must follow to be installable via
[`okta-partners`](./partners.md) and to surface in **both**
[`okta-web`](./web.md) (platform) and [`okta-app`](./app.md) (client) — the
[dual surface](./architecture.md#dual-surface).

The model below is reverse-engineered from a real installed application in the
workspace and from the boilerplate `okta-partners` pushes to every new
application repo. Where something is likely specific to one application rather
than a general requirement, it is marked
`> TODO: confirm (may be specific to the example, not general)`.

> Throughout, an application is referred to generically as **an installable
> application** / **a Tenant's installed application**. Placeholders:
> `<module-name>` (StudlyCase, e.g. `ExampleApp`), `<module-slug>` (kebab-case,
> e.g. `example-app`), `<module_lower>` (snake_case), `<MODULE_UPPER>`
> (UPPER_SNAKE). These mirror the boilerplate's substitution tokens.

---

## 1. Identity: two descriptors at the repo root

An installable application is a Laravel **module package**. It carries two
distinct descriptor files at its root:

### `manifest.json` — the platform contract (source of truth for the platform)

The descriptor published to the platform; `okta-web`'s
`App\Modules\Core\ManifestValidator` validates it on publish/install. General
shape:

```json
{
  "moduleId": "<module-slug>",
  "displayName": "…",
  "displayNameEn": "…",
  "version": "1.0.0",
  "category": "…",
  "integrationType": "embedded",          // embedded | external | notification
  "description": "…",
  "icon": "https://…",
  "developer": { "tenantSlug": "…", "name": "…" },

  "scopes": [
    { "key": "education.students.read",  "required": true,  "reason": "" },
    { "key": "education.students.write", "required": true,  "reason": "" }
  ],

  "menu": { "route": "<module-slug>.dashboard" },   // the platform (okta-web) surface

  "mobile": {                                        // the client (okta-app) surface
    "supported": true,
    "mode": "webview",                              // webview | native | external
    "entry": "okta_app/webview/screens/dashboard.blade.php",   // native: okta_app/native/main/lib/main.dart
    "passRoleClaim": true,
    "allowedPlatforms": ["ios", "android", "windows", "linux"],
    "allowedRoles": ["tenant-admin"]
  },

  "notifications": [
    {
      "key": "<module-slug>.<resource>.<event>",
      "display_name": { "ar": "…", "en": "…" },
      "variables": { "student_id": "string", "event_at": "datetime" },
      "default_channels": ["in_app", "whatsapp", "push"],
      "severity": "info",
      "is_active": true
    }
  ],

  "database": {
    "requiresDatabase": true,
    "schema": "m_<module_lower>",
    "migrations": [
      { "version": "2024_01_01_000001",
        "name": "create_<module_lower>_<table>_table",
        "path": "database/migrations/2024_01_01_000001_create_<module_lower>_<table>_table.php" }
    ]
  }
}
```

**Validation rules that are general requirements** (from `ManifestValidator`):

- `moduleId` kebab-case; `version` semver `MAJOR.MINOR.PATCH`; `displayName`,
  `category`, `description` required.
- `scopes[].key` must be canonical `<feature>.<resource>.<action>`, **at least
  one `required:true`**, and **every key must already exist in the platform scope
  catalog** (`partner_scopes`, `is_active=true`). You cannot grant a scope the
  platform doesn't publish — add new scopes in `okta-web` first (see §5).
- Partners get only `read` / `write` actions — never `delete`.
- `integrationType` coherence: `external` ⇒ requires an `external.webhookUrl`
  (HTTPS) + `webhookEvents[]`; `embedded` ⇒ must **not** carry an `external`
  block; `notification` ⇒ requires its `notification` block.
- `database` block: `migrations[].version` is a timestamp and each migration has
  `path` XOR `sql_up`.

**`integrationType` / `scopes` set, channels, table names** in the example above
are illustrative — the specific scopes (`education.*`), notification keys, and
schema name belong to one application, not the general model.

### `module.json` — the loader descriptor (`nwidart/laravel-modules`)

Tells the host's module system how to boot the package. General shape:

```json
{
  "name": "<module-name>",
  "alias": "<module-slug>",
  "description": "…",
  "keywords": [],
  "priority": 0,
  "providers": ["Modules\\<module-name>\\app\\Providers\\<module-name>ServiceProvider"],
  "aliases": {},
  "files": [],
  "requires": []
}
```

`manifest.json` is the **platform/marketplace** contract (scopes, surfaces,
notifications, DB). `module.json` is the **runtime loader** that points the host
at the service provider. Both are required for an embedded application.

---

## 2. Package structure (general)

```
<application repo>/
├── manifest.json                  # platform contract (§1)
├── module.json                    # nwidart loader descriptor (§1)
├── composer.json                  # type: laravel-module; PSR-4 Modules\<module-name>\
├── phpstan.neon                   # includes the partner-policy PHPStan rule (§6)
├── phpunit.xml
├── .env.example                   # module-prefixed vars only (<MODULE_UPPER>_*)
├── app/
│   ├── Providers/<module-name>ServiceProvider.php   # registers everything (§3)
│   ├── Providers/RouteServiceProvider.php
│   ├── Http/{Controllers,Middleware,Requests}/
│   ├── Livewire/                  # in-tenant UI (platform surface)
│   ├── Models/                    # module-owned models only
│   ├── Services/
│   │   └── PartnerApi/            # thin wrappers that call the host (§4)
│   ├── Jobs/ Events/ Listeners/ Console/Commands/ Support/
├── config/
│   ├── config.php                 # merged under config('<module-slug>')
│   └── database.php               # the module's dedicated DB connection
├── database/migrations/           # module-owned tables only
├── lang/{ar,en}/
├── okta_app/                       # EVERYTHING the okta-app client renders (§ client)
│   ├── webview/screens/<entry>.blade.php    # mode: webview — the WebView entry
│   └── native/<entry>/             # mode: native — one Dart package per entry
│       ├── pubspec.yaml
│       └── lib/main.dart
├── resources/{views,assets,lang}/
├── routes/{web.php,api.php}        # web = platform UI; api = mobile/machine API
├── scripts/partner-policy/         # Scanner.php + check.php + phpstan/ (§6)
└── .github/workflows/partner-module-policy.yml   # CI gate (§6)
```

`composer.json` essentials (general):

```json
{
  "type": "laravel-module",
  "autoload": { "psr-4": { "Modules\\<module-name>\\": "app/" } },
  "extra": { "laravel": { "providers": [
    "Modules\\<module-name>\\app\\Providers\\<module-name>ServiceProvider"
  ] } }
}
```

---

## 3. Service provider responsibilities (general)

The single primary provider (declared in `composer.json` → `extra.laravel.providers`
and `module.json` → `providers`) wires the application into the host. Observed
boot responsibilities that generalize:

- Merge the module's dedicated DB connection
  (`config(['database.connections.<module-slug>' => require .../config/database.php])`).
- `mergeConfigFrom(.../config/config.php, '<module-slug>')`.
- Register the `RouteServiceProvider` (loads `routes/web.php` + `routes/api.php`).
- Register host-provided middleware aliases the routes need (e.g. a mobile-context
  middleware that reads the Tenant from a header/signed param).
- Define rate limiters for the mobile API.
- Load translations + views under the `<module-slug>` namespace.
- Register migrations from `database/migrations/`.
- Register Livewire components, event listeners, console commands, and scheduled
  jobs.

> The exact set of components/events/commands is application-specific; the
> *mechanism* (one provider that registers routes + migrations + views + i18n +
> events) is the general requirement.

---

## 4. How an installed application reaches the platform

An installed application **must not** touch host internals. Two sanctioned paths,
by integration type:

### Embedded — in-process Partner API

Embedded code runs inside `okta-web`, so it calls
`App\Services\PartnerApi\*` classes **directly** (returning DTOs), guarding with
`class_exists()` so a missing host service degrades gracefully rather than
fatally:

```php
// app/Services/PartnerApi/Students/GetStudent.php  (a thin module-side wrapper)
$platform = \App\Services\PartnerApi\Education\Students\GetStudent::class;
if (! class_exists($platform)) { return ['data' => []]; }   // graceful degradation
return ['data' => app($platform)($studentId)->toArray()];
```

Cross-cutting actions (parent messaging, push, in-app) are delegated to the host
too — e.g. a queued job calls the host's `DispatchNotification` service rather
than talking to WhatsApp/FCM directly. The notification *types* it may send must
be declared in `manifest.json → notifications`.

### External — HTTP runtime API + signed webhooks

External apps call `okta-web`'s `/api/apps/*` runtime
([web.md](./web.md#1-external-app-runtime-api--routesappsphp--apiapps)) with their
Bearer **installation token**; each call is scope-gated. They receive domain
events as **signed outbound webhooks** to their declared `external.webhookUrl`
(HMAC-SHA256 over `<timestamp>.<body>`, with a freshness window — receivers must
validate both). The set of events is declared in `external.webhookEvents`.

### Module-owned data

The application owns its own tables (module-prefixed, on its dedicated
connection/schema). They reference host entities by **opaque string IDs**
(e.g. ULIDs) with **no foreign keys** to platform tables (cross-database). The
number and shape of owned tables is application-specific.

---

## 5. Scopes: declare in the manifest, register in okta-web

- The catalog of grantable scopes is owned by `okta-web`
  (`App\Services\PartnerScopes\Catalog\RegisterScope`) and **mirrored** by
  `okta-partners`. A manifest can only reference scopes that already exist there.
- To add a new capability: register the scope in `okta-web` first, let it sync to
  `okta-partners`, then select it for the application. (See
  [partners.md](./partners.md#scope-catalog-sync-mirror-never-invent).)
- In the `okta-partners` UI a partner picks a **per-resource access level**: none
  / read / read+write. Selecting read+write grants both `…​.read` and `…​.write`.

---

## 6. Isolation enforcement (general requirement)

Every installable application repo ships static-analysis gates copied from
`okta-web` (the source of truth):

- **`scripts/partner-policy/Scanner.php`** — regex scanner (run via
  `scripts/partner-policy/check.php`) that **bans**: references to `App\Models\*`;
  references to `App\Services\*` **except** `App\Services\PartnerApi\*`; raw
  `DB::…` against platform-owned tables (`users`, `tenants`, `tenant_*`, `roles`,
  `permissions`, `partner_*`, …); `new PDO`; reading platform env vars
  (`DB_*`, `APP_*`, `OKTA_*`, …); reading platform config namespaces
  (`database.*`, `auth.*`, `partners.*`, …); and casting opaque ULID references to
  int. An audited line may opt out with a
  `// partner-policy:allow=<rule-id> reason=…` comment. <a id="policy-scanner"></a>
- **`scripts/partner-policy/phpstan/PartnerInternalAccessRule.php`** — an
  AST-based PHPStan rule enforcing the same boundaries more precisely, configured
  with the module's namespace + env/config prefixes (`phpstan.neon`).
- **`.github/workflows/partner-module-policy.yml`** — CI that runs the scanner on
  every PR/push and blocks the merge on any violation.

Module-prefixed access is always allowed (its own models, its own
`<module-slug>` config, its own `<MODULE_UPPER>_*` env, its own DB connection).

At runtime the platform reinforces this: sensitive host models use the
`BlocksPartnerDirectAccess` trait, throwing if reached from a partner context.

---

## 7. The platform surface (okta-web)

- Declared by `manifest.json → menu.route`.
- An embedded application mounts `routes/web.php` under its module prefix
  (e.g. `/<module-slug>/…`) using host gating middleware
  (`module.access:<module-slug>` plus context/tenant middleware). It does **not**
  add `app.scope:*` on these web routes — the host's `module.access` already gates
  installation.
- The UI is built from host UI components/Livewire and appears in the web
  sidebar for users of that Tenant who hold the relevant permissions.

---

<a id="the-client-surface"></a>
## 8. The client surface (okta-app)

Declared by `manifest.json → mobile`. The platform turns this block into the
catalog card `okta-app` renders (see
[app.md](./app.md#rendering-a-tenants-installed-applications)):

- `supported` — if false, the application is hidden from the mobile catalog.
- `mode` — `webview` (the platform serves the screen) or `external` (the partner
  hosts it).
- `entry` — for `webview`, a Blade path under the app's `okta_app/webview/` directory
  (e.g. `okta_app/webview/screens/<entry>.blade.php`).
- `allowedPlatforms` — filters cards by `X-App-Platform` (empty = all).
- `allowedRoles` — filters by the user's active role (empty = no filter).
- `passRoleClaim` — if true, an external launch receives a signed role JWT.

How discovery + render works:

1. `okta-app` calls `GET /api/mobile/app-catalog` for the active `(tenant, role)`.
2. `okta-web` (`GetMobileCatalogForUser`) reads each installed module's `mobile`
   block, filters by platform + role + `requiredScope`, and returns cards.
3. Launching a **webview** card → `okta-web` issues a short-lived **signed**
   URL to `/app/<module-slug>`, which renders the application's `entry` Blade.
   That page typically **mints a host token server-side** and hands it to its JS,
   so the in-WebView SPA calls the application's own `/api/<module-slug>/*`
   endpoints. Launching an **external** card → the partner URL (+ role JWT).

> The in-WebView SPA details (hash routing, token minting, `okta-app://close`
> bridge) are how one example implements its mobile entry; the **requirement** is
> only that `mobile.entry` renders a page the client WebView can host and that all
> data access still flows through the Partner API.

---

## 9. Checklist — making an application installable

1. Scaffold the repo from the `okta-partners` boilerplate (gets the structure,
   policy scanner, and CI for free).
2. Set identity: `composer.json` (`Modules\<module-name>\`), `module.json`,
   `manifest.json`.
3. Choose `integrationType` and declare `scopes` (must exist in the platform
   catalog), `menu` (platform surface), and `mobile` (client surface).
4. Implement UI/logic; reach host data **only** via the Partner API; own your
   tables on your dedicated connection.
5. Declare notifications + database migrations in the manifest.
6. Pass the policy scanner + PHPStan rule + CI.
7. Version, submit, review, **test on sandbox**, then **publish to production**
   (see [deployment.md](./deployment.md)).
