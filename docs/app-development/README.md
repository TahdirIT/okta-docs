# Building installed applications — developer guide

A hands-on reference for **okta developers** building an application that installs
into the platform and shows up for Tenants. It focuses on the thing you'll spend
most of your time on: **developing the pages/screens and getting them to appear**
on the two places an installed app lives —

- the **platform surface** inside **okta-web** (the web admin UI), and
- the **client surface** inside **okta-app** (the Flutter mobile/desktop client).

> Scope & audience. This is the *how-to-build-the-UI* guide. For the cross-product
> map (how the repos relate), the full installable-application contract, and the
> dev→prod publish flow, see the reference layer:
> [`../../claude/installed-apps.md`](../../claude/installed-apps.md),
> [`../../claude/architecture.md`](../../claude/architecture.md),
> [`../../claude/deployment.md`](../../claude/deployment.md).
>
> Examples use **generic placeholders** — a slug `example-app`, StudlyCase
> `ExampleApp`, namespace `Modules\ExampleApp`, env prefix `EXAMPLE_APP_`. Swap in
> your own. They're patterns derived from a real installed app, not one app's docs.

---

## The one idea to hold onto: one app, two surfaces

You build **one** application (one repo, one manifest). A single install lights up
**both** surfaces, each driven by a different block of your `manifest.json`:

```
                         your installed application
                                    │
        ┌───────────────────────────┴────────────────────────────┐
        ▼                                                          ▼
  PLATFORM surface (okta-web)                       CLIENT surface (okta-app)
  ───────────────────────────                       ──────────────────────────
  Full-page Livewire/Blade screens                  A catalog "card" the user taps,
  rendered inside the okta-web shell;               which opens your mobile screen
  a sidebar entry points at them.                   inside a WebView.

  Driven by manifest `menu` block.                  Driven by manifest `mobile` block.
  Built with the okta-web design system.            Built as a server-rendered page
  Routes in your routes/web.php.                    your okta-web serves at /app/{slug}.
  See ./web-surface.md                              See ./app-surface.md
```

The same install, the same granted scopes, and the same data-access rules (the
[Partner API](./data-access-and-security.md)) back both surfaces.

---

## Where your code lives

You scaffold from the partner boilerplate (see
[`../../claude/partners.md`](../../claude/partners.md)) and get this layout. The
two files that drive the surfaces are highlighted:

```
example-app/
├── manifest.json                 ← declares BOTH surfaces (menu + mobile blocks)   ★
├── module.json                   ← module loader descriptor (nwidart)
├── composer.json                 ← Modules\ExampleApp\ PSR-4 + provider
├── app/
│   ├── Providers/ExampleAppServiceProvider.php   ← registers routes, Livewire, views
│   ├── Http/{Controllers,Middleware}/
│   ├── Livewire/                 ← platform-surface page components (okta-web)       ★
│   └── Services/PartnerApi/      ← thin wrappers that call the host for data
├── resources/views/             ← Blade for the platform surface
├── mobile/screens/<entry>.blade.php   ← client-surface entry rendered in the WebView ★
├── routes/{web.php,api.php}      ← web = platform UI; api = your mobile/machine API
├── config/{config.php,database.php}
├── database/migrations/         ← your own tables only
└── scripts/partner-policy/      ← the isolation gate (don't edit; CI runs it)
```

---

## Read this when…

| You want to… | Open |
|---|---|
| Build pages that appear in the **okta-web** sidebar/admin UI | [`./web-surface.md`](./web-surface.md) |
| Build the screen that appears as a **card in okta-app** and runs in its WebView | [`./app-surface.md`](./app-surface.md) |
| Know exactly which manifest blocks drive what | [`./manifest-reference.md`](./manifest-reference.md) |
| Read/write Tenant data the sanctioned way, and stay inside the isolation rules | [`./data-access-and-security.md`](./data-access-and-security.md) |
| Understand auth/identity across the surfaces and the install lifecycle | [`./auth-and-lifecycle.md`](./auth-and-lifecycle.md) |
| See the whole product picture / publish flow | [`../../claude/`](../../claude/) |

---

## Hard rules (don't skip)

1. **Never touch host internals.** Reach Tenant data only through the
   [Partner API](./data-access-and-security.md). Importing `App\Models\*` or
   non-`PartnerApi` `App\Services\*`, raw `DB` on platform tables, platform `env`/
   `config` → the policy scanner fails your CI.
2. **Use the platform design system** on the okta-web surface. Raw `<button>`/
   `<input>`/`<table>` or hard-coded palettes (`gray-*`, `indigo-*`) get a UI PR
   rejected — see [`../tech-standards/design-standards.md`](../tech-standards/design-standards.md).
3. **Scopes are `read`/`write` only**, and every scope you request must already
   exist in the platform catalog. See [`./manifest-reference.md`](./manifest-reference.md).
4. **Own your data.** Your tables live on your dedicated connection/schema, are
   prefixed, and reference host entities by opaque string IDs (no foreign keys to
   platform tables).
