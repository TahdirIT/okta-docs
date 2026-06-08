# Glossary

Key terms used across the `okta` product and these reference files. Terms are
grounded in the actual code of the workspace repositories; where a meaning is
inferred rather than explicit in code, it is marked `> TODO: confirm`.

> Language note: the product's primary domain language is Arabic. These English
> terms map onto the Arabic vocabulary used in [`../docs/`](../docs/README.md)
> (e.g. «الكيانات التعليمية» = *educational entities* = Tenants).

---

## Product & repositories

- **okta** — The single, integrated educational platform product. Not five
  separate products: one product split across the repositories below. See
  [architecture.md](./architecture.md).
- **okta-web** — The platform / core. A Laravel 12 + Livewire 4 modular
  monolith that owns the domain data, exposes services, and hosts installed
  applications as code. Runs in two environments (sandbox and production). See
  [web.md](./web.md).
- **okta-partners** — The installation / deployment mechanism. A separate
  Laravel 13 app where applications are authored, versioned, reviewed, and
  published; from here they are installed *through* the bridge into `okta-web`.
  See [partners.md](./partners.md).
- **okta-app** — The client. A Flutter cross-platform app (iOS / Android /
  Windows present in-repo) that shows, per Tenant + role, the applications a
  Tenant has installed. See [app.md](./app.md).
- **okta-docs** — This repository. The documentation hub and entry point for any
  new session. See the root [`../CLAUDE.md`](../CLAUDE.md).

---

## Tenancy & users

- **Tenant** — An educational entity that subscribes to the platform: individual
  teacher, school, complex, college, university, institute, academy, or
  educational company. The unit of data isolation
  (`spatie/laravel-multitenancy`). Full list: [`../docs/tenants.md`](../docs/tenants.md).
- **Landlord** — The shared/central database and tables that are not
  tenant-scoped (e.g. `module_statuses`, the partner scope catalog, finance).
- **End-user / role** — A person acting inside a Tenant with a role
  (account admin, administrative staff, teacher, student, guardian, guardian
  delegate). See [`../docs/end-users.md`](../docs/end-users.md). The mobile
  client resolves an **active role** per session.
- **Active context** — The `(scope, tenant_id, active_role_id)` selection a user
  makes after login in `okta-app` (`scope` = `tenant` or `system`). Drives which
  installed-app cards are shown.

---

## Services & data access

- **Service** — A single use-case class under
  `app/Services/<Feature>/<Resource>/<Function>.php` exposing one `__invoke()`
  (the platform's [feature-services standard](../docs/tech-standards/laravel-feature-services-structure.md)).
  `okta-web` exposes a dedicated, partner-facing slice of these under
  `App\Services\PartnerApi\*`.
- **Partner API** — The *only* sanctioned surface through which an installed
  application may read/write Tenant domain data. Two forms:
  - In-process (`App\Services\PartnerApi\*` classes) — used by **embedded** apps
    shipped as code inside `okta-web`.
  - HTTP (`/api/apps/*`, Bearer installation token) — the runtime API used by
    **external** apps.
- **Scope** — A granted permission key in the canonical form
  `<feature>.<resource>.<action>` (e.g. `education.students.read`). Partners only
  ever receive `read` and `write`; never `delete`. The canonical catalog lives in
  `okta-web` (`App\Services\PartnerScopes\Catalog\RegisterScope`) and is mirrored
  by `okta-partners`. See [installed-apps.md](./installed-apps.md).
- **Permission** — A `spatie/laravel-permission` RBAC grant on a `User × Role`.
  Distinct from a Scope (which gates an *app's* access) — see the
  [permissions-naming standard](../docs/tech-standards/permissions-naming.md).

---

## Installed applications

- **Installable application / installed application** — A packaged Laravel module
  that a Tenant installs through `okta-partners` into `okta-web`. Identified by a
  `moduleId` (kebab-case slug). The general contract is documented in
  [installed-apps.md](./installed-apps.md).
- **Module** — The implementation form of an installable application:
  an `nwidart/laravel-modules` package (`module.json` + service provider) under
  `okta-web`'s `Modules/` directory.
- **Manifest** (`manifest.json`) — The platform-facing descriptor an installable
  application publishes: `moduleId`, `version`, `integrationType`, `scopes`,
  `mobile` block, `notifications`, `database` block, `menu`. Validated by
  `okta-web`'s `App\Modules\Core\ManifestValidator`.
- **Module descriptor** (`module.json`) — The `nwidart/laravel-modules` loader
  file (name, alias, service providers). Distinct from `manifest.json`.
- **Integration type** — `embedded`, `external`, or `notification`
  (`App\Enums\IntegrationType` in `okta-partners`):
  - **embedded** — Ships as code inside `okta-web`; renders in-tenant UI;
    calls `App\Services\PartnerApi\*` in-process; requires a GitHub repo +
    boilerplate; subject to the policy scanner.
  - **external** — Hosted by the partner; consumes the HTTP API and receives
    signed webhooks; declares `external.webhookUrl` + `webhookEvents`.
  - **notification** — A notification provider (channels such as WhatsApp / SMS /
    push). `> TODO: confirm` the full provisioning contract — see
    [partners.md](./partners.md).
- **Dual surface** — The defining property of an installed application: it
  appears in **both** the platform (`okta-web` web UI, via `menu.route`) **and**
  the client (`okta-app` mobile catalog, via the `mobile` manifest block). See
  [architecture.md](./architecture.md#dual-surface).
- **Installation token** — A Bearer credential (`PartnerInstallationToken`) bound
  to `(tenant, module, installation)` and a set of granted scopes; issued by
  `okta-web` at install time. Default TTL 90 days.
- **Installation DB credentials** — A dedicated PostgreSQL role + schema issued
  per installation when DB isolation is enabled, so module-owned tables stay
  isolated (`PartnerInstallDbCredential`).
- **Policy scanner** — The regex (`scripts/partner-policy/Scanner.php`) and
  PHPStan (`PartnerInternalAccessRule`) static-analysis tools that forbid an
  installable application from touching `okta-web` internals
  (`App\Models\*`, non-`PartnerApi` `App\Services\*`, platform tables/env/config).

---

## Environments & bridge

- **Environment** — `okta-web` runs as **sandbox** (development / testing) and
  **production**. This is the dev → prod progression referenced in the product
  overview. Selected on the `okta-partners` side by `BridgeSettings`
  (`apiUrl()`/`outboundToken()` vs `sandboxApiUrl()`/`sandboxOutboundToken()`).
  See [deployment.md](./deployment.md).
- **Sandbox** — A per-partner or instance-level test environment in `okta-web`
  (`is_sandbox` Tenants, `MODULES_SANDBOX_DB_NAME`, `APP_ENV=sandbox`). Module
  installs here auto-approve platform-AI access.
- **Bridge** — The authenticated HTTP link between `okta-partners` and `okta-web`
  (`OktaWebService` + `BridgeSettings` on the partners side; the
  `partner.api`-guarded endpoints on the web side). Carries publish, sync,
  install, and catalog traffic.
- **Catalog hash / drift detection** — A SHA-256 hash of the scope catalog
  (`GET /api/partners/permissions/catalog/hash`) used by `okta-partners` to skip
  a full re-sync when nothing changed.
- **Webhook (inbound to partners)** — Signed events `okta-web` sends to
  `okta-partners` at `POST /webhooks/okta-web` (HMAC-SHA256 over
  `<timestamp>.<body>`, ±5-min freshness, replay protection).
- **Webhook (outbound to external apps)** — Signed events `okta-web` delivers to
  an external app's `webhookUrl` (`DispatchEvent` → `DeliverPartnerWebhook`).

---

## Module lifecycle states

`App\Enums\ModuleStatus` (okta-partners): `Draft → Submitted → InReview →
Approved → Beta/Published` (plus `ChangesRequested`, `Suspended`, `Deprecated`,
`Rejected`). Only `Draft` and `ChangesRequested` are editable. See
[deployment.md](./deployment.md).
