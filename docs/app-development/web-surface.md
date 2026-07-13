# The platform surface — pages inside okta-web

How you develop pages for an **embedded** application and make them appear in the
**okta-web** web admin UI. (External apps don't render here — they live entirely
on the client surface + their own host; see [`./app-surface.md`](./app-surface.md).)

Prereq mental model: [`./README.md`](./README.md). Data access:
[`./data-access-and-security.md`](./data-access-and-security.md).

---

## What "appearing in okta-web" means

Your application ships **as code** into okta-web (under `Modules/ExampleApp/`).
Once a Tenant installs it:

1. A **sidebar entry** appears for that Tenant, pointing at your landing route.
   This is declared by your manifest `menu` block.
2. Clicking it loads one of **your pages** — a full-page Livewire component (or a
   controller returning a Blade view) — rendered **inside the okta-web shell**
   (its top bar, sidebar, theme, RTL, and auth/tenant context all wrap your page).

So "developing a platform page" = writing a normal Laravel/Livewire page in your
module, mounting it on a module-prefixed route, registering it, and pointing the
manifest `menu` at it.

```
sidebar item (manifest.menu.route)
        │  user clicks
        ▼
GET /example-app/                       ← your routes/web.php
  middleware: context, active-tenant-context, tenant.scope, module.access:example-app
        │
        ▼
Livewire full-page component (App\Livewire\...)  → renders Blade using okta-web design system
        │  needs data?
        ▼
App\Services\PartnerApi\*  (in-process)  → DTOs
```

---

## 1. Routes — `routes/web.php`

Mount your pages under a module prefix and the platform's gating middleware. Note
what is **not** here (see the box below):

```php
use Illuminate\Support\Facades\Route;
use Modules\ExampleApp\app\Livewire\Dashboard\DashboardPage;
use Modules\ExampleApp\app\Livewire\Students\StudentList;

Route::prefix('example-app')
    ->name('example-app.')
    ->middleware(['context', 'active-tenant-context', 'tenant.scope', 'module.access:example-app'])
    ->group(function () {
        Route::get('/', DashboardPage::class)->name('dashboard');        // landing route
        Route::get('students', StudentList::class)->name('students');
        // controller actions are fine too:
        // Route::post('reports', [ReportController::class, 'store'])->name('reports.store');
    });
```

The middleware stack does the heavy lifting for you:

| Middleware | What it gives you |
|---|---|
| `context` + `active-tenant-context` | The authenticated user and the **active Tenant** (multitenancy) are already resolved. |
| `tenant.scope` | RBAC permissions are scoped to the active Tenant. |
| `module.access:example-app` | Rejects the request unless **this Tenant has your app installed**. |

> ⚠️ **Do not add `app.scope:*` on these web routes.** That middleware is for the
> external HTTP runtime (inbound scoped Bearer tokens) and will reject in-tenant UI
> requests with `no_app_context`. On the platform surface, `module.access` is the
> gate; scope enforcement happens where you call the Partner API. See
> [`./auth-and-lifecycle.md`](./auth-and-lifecycle.md).

`routes/web.php` is loaded by your module's `RouteServiceProvider` (registered from
the main service provider's `register()`), typically inside a `web` middleware group.

---

## 2. Pages are full-page Livewire components

The platform surface is **Livewire-first** (Livewire 4). A page is a component
mapped directly onto a route:

```php
namespace Modules\ExampleApp\app\Livewire\Dashboard;

use Livewire\Component;
use Modules\ExampleApp\app\Services\PartnerApi\Students\GetStudents;

class DashboardPage extends Component
{
    public array $stats = [];

    public function mount(): void
    {
        // Authorize against a tenant-scoped permission you declared in the manifest:
        abort_unless(auth()->user()->can('example_app.dashboard.view'), 403);

        // Read host data ONLY through your PartnerApi wrappers (never App\Models\*):
        $this->stats = app(GetStudents::class)(['per_page' => 1])['meta'] ?? [];
    }

    public function render()
    {
        // Namespaced view: <slug>::... (registered via loadViewsFrom)
        return view('example-app::livewire.dashboard.dashboard-page');
    }
}
```

Its Blade view lives at `resources/views/livewire/dashboard/dashboard-page.blade.php`
and is referenced as `example-app::livewire.dashboard.dashboard-page`.

---

## 3. Wire it up in the service provider

Your module's main provider registers everything Laravel/Livewire needs. The parts
that matter for the platform surface:

```php
public function boot(\Illuminate\Routing\Router $router): void
{
    // Views/translations under the <slug>:: namespace
    $this->loadViewsFrom(module_path($this->name, 'resources/views'), $this->alias);        // example-app::
    $this->loadTranslationsFrom(module_path($this->name, 'resources/lang'), $this->alias);
    $this->loadMigrationsFrom(module_path($this->name, 'database/migrations'));

    // Register every full-page (and embedded) Livewire component you route to
    \Livewire\Livewire::component('example-app.dashboard', \Modules\ExampleApp\app\Livewire\Dashboard\DashboardPage::class);
    \Livewire\Livewire::component('example-app.student-list', \Modules\ExampleApp\app\Livewire\Students\StudentList::class);
}
```

Routes are loaded by the `RouteServiceProvider` the main provider registers in
`register()`. (Config + your dedicated DB connection are merged in `register()` too
— see [`./data-access-and-security.md`](./data-access-and-security.md).)

---

## 4. The sidebar entry — manifest `menu`

The sidebar item is declared in `manifest.json`. At minimum it names the route the
item links to:

```json
{
  "menu": { "route": "example-app.dashboard" }
}
```

Name your landing route to match (`->name('dashboard')` under the `example-app.`
group → `example-app.dashboard`). The platform reads installed apps' `menu`
blocks at runtime (`App\Livewire\AppsMenu` in okta-web) to render the entry for
Tenants that have the app installed. Resolution order for the launch route:
`menu.route` → `sidebar.route` → `<slug>.dashboard`. A `menu.audiences[]` array
can point different account types (tenant `roles[]` or a student/guardian
`portal`) at different routes — mirroring the mobile `audiences` concept. See
[`./manifest-reference.md`](./manifest-reference.md).

---

## 5. Build the UI with the platform design system (mandatory)

Because your pages render **inside okta-web**, you have its whole Blade
design-system component library available — and you are **required** to use it.
This is enforced on UI PRs.

- Use platform components instead of raw HTML:
  `<x-button>`, `<x-input-field>`, `<x-select>`, `<x-textarea>`, `<x-table>` +
  `<x-table.tr/th/td>`, `<x-badge>`, `<x-checkbox>`/`<x-switch>`, `<x-date-picker>`,
  `<x-file-upload>`, `<x-alert>`, `<x-card>`, `<x-tabs>`, `<x-avatar>`, `<x-spinner>`,
  modals via `wire-elements/modal` + `<x-modal-card>`.
- Use the platform **CSS theme tokens** (`var(--color-primary-*)`,
  `var(--color-neutral-*)`) — **not** fixed Tailwind palettes like `gray-*`,
  `zinc-*`, `slate-*`, `indigo-*` (they break theme + brand color + dark mode).
- No `max-w-*` on in-page components (the layout owns width).
- Review **dark mode** and **RTL** (the platform is Arabic-first).
- Follow the card pattern (`rounded-2xl + border + bg-white/dark + p-6`).

Full, binding rules: [`../tech-standards/design-standards.md`](../tech-standards/design-standards.md).
(okta-web also keeps a stricter in-repo `docs/design.md` with the complete
component list — read it before building UI.)

---

## 6. Permissions (RBAC)

Declare the permissions your pages check in the manifest `rbac_permissions` block;
on install the platform creates them (tenant-scoped) and grants them to the roles
you map. Then gate pages and actions with normal `can()` / `abort_unless()`:

```php
abort_unless(auth()->user()->can('example_app.dashboard.view'), 403);
```

Permission names follow the platform standard `<feature>.<resource>.<action>`
(lowercase, snake_case parts, verbs `view`/`create`/`update`/`delete`/`activate`,
no wildcards) — see [`../tech-standards/permissions-naming.md`](../tech-standards/permissions-naming.md).

> Don't confuse the three gates: **permission** (user×role, above), **scope**
> (your app's data access, [`./data-access-and-security.md`](./data-access-and-security.md)),
> and **install** (`module.access`). A sensitive action often needs all three.

---

## 7. Local development

Develop the module against a host checkout (it isn't a standalone app):

1. Place the module under the host's `Modules/ExampleApp/` (or symlink it).
2. `php artisan module:enable ExampleApp` and run host + module migrations
   (`php artisan module:migrate ExampleApp`).
3. `php artisan serve` + `npm run dev`; log in to a Tenant that has the app
   installed and open `/example-app/`.
4. Keep `php artisan queue:work` running so your events/jobs/notifications fire.

Build/lint with the host's tooling (`composer run lint`, `php artisan test`).

---

## Checklist — a platform page

- [ ] Route in `routes/web.php` under your `slug` prefix with `module.access:<slug>`
      (and **no** `app.scope`).
- [ ] Full-page Livewire component registered via `Livewire::component(...)`.
- [ ] View namespaced `<slug>::...`, built from platform components + theme tokens.
- [ ] `menu.route` in the manifest points at your landing route's name.
- [ ] Data read via `App\Services\PartnerApi\*` only.
- [ ] Permission checks via `can()`; permissions declared in `rbac_permissions`.
- [ ] Dark mode + RTL reviewed.
