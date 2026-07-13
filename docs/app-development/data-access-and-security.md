# Data access & security

The single most important rule for an installed application: **you reach Tenant
data only through the Partner API, and you never touch platform internals.** This
page covers how to read/write host data, how to own your own data, the scope model,
and the static + runtime guards that enforce all of it.

Companion: [`./manifest-reference.md`](./manifest-reference.md) ·
[`../../claude/installed-apps.md`](../../claude/installed-apps.md).

---

## The boundary

```
        your app code
            │
            │  ALLOWED                              FORBIDDEN (policy scanner fails CI)
            ▼                                       ─────────────────────────────────
  App\Services\PartnerApi\*   (host data)           App\Models\*  (host Eloquent)
  your own Modules\ExampleApp\* models              App\Services\* (non-PartnerApi)
  your own DB connection/schema                     DB::table('users'|'tenants'|…)
  config('example-app.*'), env('EXAMPLE_APP_*')     env('DB_*'|'APP_*'|'OKTA_*'), config('database.*'|…)
```

The platform is the source of truth for all Tenant domain data; you read/write it
through a stable, scoped API and keep **your** data in **your** schema.

---

## Reading/writing host data — the Partner API

Two forms, depending on your `integrationType`:

### Embedded — in-process

Your code runs inside okta-web, so you call `App\Services\PartnerApi\*` classes
directly and get back **DTOs** (never Eloquent models). Wrap each call in a thin
service under your own `app/Services/PartnerApi/` and guard with `class_exists()`
so a host that lacks a given service degrades gracefully instead of fataling:

```php
namespace Modules\ExampleApp\app\Services\PartnerApi\Students;

class GetStudents
{
    /** Scope: education.students.read */
    public function __invoke(array $filters = []): array
    {
        $platform = \App\Services\PartnerApi\Education\Students\ListStudents::class;
        if (! class_exists($platform)) {
            return ['data' => [], 'meta' => []];   // host service unavailable
        }
        $result = app($platform)(
            page: $filters['page'] ?? 1,
            perPage: $filters['per_page'] ?? 25,
        );
        return ['data' => $result->data->map->toArray()->all()];
    }
}
```

Cross-cutting actions (parent messaging, push, in-app) are delegated to the host
too — e.g. a queued job calls the host's `DispatchNotification` service. You never
talk to WhatsApp/FCM/etc. directly, and the notification *types* you may send must
be declared in the manifest `notifications` block.

### External — HTTP runtime + signed webhooks

If you're hosted externally, you call okta-web's `/api/apps/*` runtime with your
**installation Bearer token**; each call is scope-gated by `app.scope:<key>`. You
receive domain changes as **signed outbound webhooks** to your `external.webhookUrl`
(HMAC-SHA256 over `<timestamp>.<body>` — validate signature + freshness). See
[`../../claude/web.md`](../../claude/web.md) and [`./auth-and-lifecycle.md`](./auth-and-lifecycle.md).

---

## Scopes

- A **scope** is a granted capability key, canonical form
  `<feature>.<resource>.<action>` (e.g. `education.students.read`).
- Partners only ever get **`read`** and **`write`** — never `delete`.
- A scope must exist in the platform catalog before you can request it (see
  [`./manifest-reference.md`](./manifest-reference.md#scopes-add-new-ones-the-right-way)).
- On the surfaces:
  - **platform (embedded)**: the in-process Partner API service enforces the scope;
    your page also gates with tenant-scoped **permissions** (`can()`).
  - **external HTTP**: the `app.scope:<key>` middleware rejects with `403` +
    `X-Missing-Scope`.
  - **client card**: `mobile.requiredScope` decides whether the active role even
    sees the card.

> Three different gates, don't conflate them: **permission** (user×role),
> **scope** (app×data), **install** (`module.access`, tenant×app). See
> [`./auth-and-lifecycle.md`](./auth-and-lifecycle.md).

---

## Owning data

Your application owns its own tables — and only those.

- Declare a dedicated DB connection in `config/database.php` and merge it in your
  provider's `register()`:
  ```php
  config(['database.connections.example-app' => require module_path($this->name, 'config/database.php')]);
  ```
  When DB isolation is enabled, the platform provisions a dedicated PostgreSQL
  **role + schema** per installation; your connection binds to it.
- **Prefix** every table (`example_app_*`) and create them on your connection:
  ```php
  Schema::connection('example-app')->create('example_app_things', function (Blueprint $t) {
      $t->id();
      $t->string('student_id', 64)->index();   // opaque host reference (ULID/string)
      // ... no foreign keys to platform tables (cross-database)
      $t->timestamps();
  });
  ```
- Reference host entities by **opaque string IDs** (ULIDs). Never cast a ULID to
  int (the scanner forbids it), never add an FK to a platform table.
- List your migrations in the manifest `database.migrations[]` so the platform
  tracks and runs them on install.

---

## The isolation guards (what fails your build)

Every app repo ships static analysis (copied from okta-web; **don't edit it** — CI
runs it on every PR/push):

- **`scripts/partner-policy/Scanner.php`** (regex) and the **PHPStan**
  `PartnerInternalAccessRule` (AST) both **ban**:
  - importing/using `App\Models\*`;
  - importing/using `App\Services\*` **except** `App\Services\PartnerApi\*`;
  - raw `DB::…` against platform tables (`users`, `tenants`, `tenant_*`, `roles`,
    `permissions`, `partner_*`, …) and `new PDO`;
  - reading platform env (`DB_*`, `APP_*`, `OKTA_*`, …) or platform config
    (`database.*`, `auth.*`, `partners.*`, …);
  - casting ULID references to int.
  - Your own module-prefixed models/connection/`config('example-app.*')`/
    `env('EXAMPLE_APP_*')` are always allowed.
- An audited exception can opt a single line out:
  `// partner-policy:allow=<rule-id> reason=…`.
- **CI**: `.github/workflows/partner-module-policy.yml` blocks the merge on any
  violation.

**Runtime backstop:** sensitive host models use a `BlocksPartnerDirectAccess` trait
that throws if reached from a partner context — so even if something slipped past
static analysis, it fails at runtime.

Full enforcement details: [`../../claude/installed-apps.md`](../../claude/installed-apps.md#6-isolation-enforcement-general-requirement).
