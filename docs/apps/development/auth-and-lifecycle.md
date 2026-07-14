# Auth, identity & lifecycle

The part that confuses developers most: **there are three different ways your app
is authenticated**, one per entry path. Getting them straight makes the rest of the
system obvious. This page also sketches the install/uninstall lifecycle and the
sandbox → production progression.

Companion: [`./web-surface.md`](./web-surface.md) · [`./app-surface.md`](./app-surface.md) ·
[`../../../claude/deployment.md`](../../../claude/deployment.md).

---

## Three identity contexts

| Entry path | How the request is authenticated | What identifies your app | Scope/gate |
|---|---|---|---|
| **Platform UI** (embedded) — `GET /<slug>/…` | The logged-in okta-web **session** + active Tenant (set by `context`/`active-tenant-context`). | `module.access:<slug>` confirms the Tenant installed your app. | User **permissions** (`can()`); in-process Partner API enforces **scopes**. |
| **External HTTP runtime** — `/api/apps/*` | Your **installation token** (`Authorization: Bearer …`), resolved by `AuthenticateAppInstallation` into a `ModuleContext`. | The token is bound to `(tenant, module, installation)` + granted scopes. | `app.scope:<key>` per route (403 + `X-Missing-Scope`). |
| **Client WebView** (embedded) — `GET /app/{slug}` then your `/api/<slug>/*` | A **signed launch URL** (the signature *is* the session) → your entry page **mints a Sanctum token** for the in-WebView code → that token authenticates calls to your own API. | The signed URL carries `u`/`t`/`r`; the minted token is yours. | `EnsureAppWebview` re-checks `requiredScope`/`allowedRoles`; your own context middleware sets the Tenant. |

```
PLATFORM UI                 EXTERNAL HTTP                 CLIENT WEBVIEW
───────────                 ─────────────                 ──────────────
session + active tenant     Bearer installation token     signed /app/{slug} URL
        │                            │                              │  (signature = session)
module.access:<slug>        AuthenticateAppInstallation     EnsureAppWebview (re-auth + re-scope)
        │                            │                              │
in-process PartnerApi       app.scope:<key> →               render entry blade →
+ can() permissions         PartnerApi service              mint Sanctum token → your /api/<slug>/*
                                                            (Bearer + X-Okta-Tenant-Id → Tenant::makeCurrent())
```

### Why each exists

- **Platform UI** rides the existing okta-web login; you don't manage tokens, just
  permissions + `module.access`. (Do **not** put `app.scope` on these routes — it's
  for the HTTP runtime and rejects UI requests with `no_app_context`.)
- **External HTTP** is for apps hosted off-platform: a long-lived, scope-bound
  Bearer credential (default 90-day TTL) issued at install.
- **Client WebView** can't carry the web session into a WebView and your API group
  is stateless, so okta-web hands out a **short-lived signed URL**; your entry page
  then mints a longer-lived API token for the session and passes the Tenant via the
  `X-Okta-Tenant-Id` header to your own context middleware.

---

## The minted WebView token (client surface)

Generated **server-side** in your `mobile/` entry Blade, not by the client:

- revoke older tokens of the same name, then `createToken('<slug>:webview', ['*'],
  now()->addHours(8))` — long enough for a working session, short enough to expire;
- handed to JS in the `BOOT` object and sent as `Authorization: Bearer …` on every
  call to your `/api/<slug>/*`;
- paired with `X-Okta-Tenant-Id` so your module's context middleware verifies
  membership and calls `Tenant::makeCurrent()`.

Details + code: [`./app-surface.md`](./app-surface.md#2-develop-the-entry-page-embedded-mode).

---

## Install / uninstall (what happens to your app)

When a Tenant installs your app, okta-web's `InstallModule`:

1. validates your manifest;
2. records the installation (+ a free subscription);
3. **issues the installation token** bound to your granted scopes;
4. for **external** apps, syncs webhook subscriptions from `external.webhookEvents`;
5. provisions your isolated DB role/schema (if DB isolation is on) and runs your
   `database.migrations`;
6. creates + grants your `rbac_permissions`; syncs cross-module access;
7. fires `ModuleInstalled`.

Uninstall reverses it: revokes tokens, deactivates webhook subscriptions (kept for
audit), revokes permissions, drops your schema, fires `ModuleUninstalled`. Your app
disappears from **both** surfaces.

Full steps: [`../../../claude/web.md`](../../../claude/web.md#how-applications-get-installed-into-okta-web).

---

## Environments: sandbox → production (dev → prod)

okta-web runs as **sandbox** (development/testing) and **production**. Your version
is **installed into sandbox first** (from okta-partners, via the bridge) so you can
exercise both surfaces end-to-end, then **published to production**. Sandbox installs
auto-approve platform-AI access so you can test without manual approval.

The authoring → review → sandbox → publish flow lives in
[`../../../claude/deployment.md`](../../../claude/deployment.md) and
[`../../../claude/partners.md`](../../../claude/partners.md). For local development against
a host checkout, see [`./web-surface.md`](./web-surface.md#7-local-development).
