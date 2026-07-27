# okta-partners — installation / deployment mechanism

A **separate** Laravel 13 + Livewire 4 application (JWT via `firebase/php-jwt`)
where installable applications are authored, versioned, reviewed, published, and
installed *through the bridge* into [`okta-web`](./web.md). It shares **no code**
with `okta-web`; everything crosses the authenticated HTTP bridge. It has no
contact with [`okta-app`](./app.md).

> The repo ships its own `okta-partners/CLAUDE.md` with UI/standards detail. This
> file focuses on the deployment mechanism and the dev → prod (sandbox →
> production) flow.

Related: [deployment.md](./deployment.md) · [installed-apps.md](./installed-apps.md)

---

## Structure (key paths)

```
okta-partners/
├── app/
│   ├── Services/
│   │   ├── OktaWebService.php            # HTTP client to okta-web (publish/install/sync/ping)
│   │   ├── BridgeSettings.php            # live prod vs sandbox URLs + tokens (DB→config→env)
│   │   ├── GitHubAppService.php          # provision partner repos (pushBoilerplate)
│   │   ├── ModuleLifecycleService.php    # version create + publish state machine
│   │   └── PartnerScopes/Catalog/        # SyncFromOktaWeb, GetGroupedCatalog
│   ├── Models/                           # PartnerModule, PartnerModuleVersion, PartnerApplication,
│   │                                      #   PartnerAvailableScope
│   ├── Enums/                            # IntegrationType, ModuleStatus
│   ├── Livewire/                         # Admin/Settings/OktaWebBridge, Partner/Modules/*
│   └── Http/Controllers/Api/             # OktaWebWebhookController, AppStoreCatalogController
├── resources/partner-boilerplate/        # the skeleton pushed to each new application repo
└── routes/                               # partner.php, api.php, webhooks.php
```

---

## The application data model

- **`PartnerModule`** — the application: `slug`, `display_name(_en)`,
  `description(_en)`, `category`, `integration_type` (enum), `scopes` (JSON,
  canonical form) + legacy `permissions` (JSON), pricing, `status`,
  `okta_web_module_id` (set after first publish), and a `manifest` snapshot.
- **`PartnerModuleVersion`** — a concrete version: `version` (semver), `status`,
  per-version overrides (description, icon, scopes, webhook config for canary
  releases), database fields (`requires_database`, `schema_definition`,
  `database_schema`), mobile fields, and GitHub pinning (`github_branch`,
  `commit_sha`, `pinned_at`).
- **`PartnerAvailableScope`** — the **local mirror** of `okta-web`'s scope
  catalog (`key`, `feature`, `resource`, `action`, labels, `is_dangerous`,
  `is_active`, `sort_order`, `last_synced_at`). Partners never invent scopes here.

### Integration types (`App\Enums\IntegrationType`)

- **`embedded`** — ships as code into `okta-web`; has in-tenant UI; needs a
  GitHub repo + boilerplate (`requiresRepository()`).
- **`external`** — hosted by the partner; needs `webhook_url` + `webhook_secret`
  + `webhook_events` (`requiresWebhookEndpoint()`); optional `redirect_urls`.
- **`notification`** — a notification provider declaring channels + delivery
  (api/embedded/hybrid). `> TODO: confirm` the end-to-end provisioning contract
  with `okta-web` (fields exist; the runtime handshake wasn't fully traceable).

### Building the manifest

`PartnerModuleVersion::buildManifest()` (inheriting unset fields from the parent
`PartnerModule`) produces the platform manifest okta-web validates and stores. It
emits `moduleId`, `displayName`, `version`, `category`, `integrationType`,
`description`, `scopes`, `rbac_permissions`, `menu`/`sidebar`, `features`,
pricing, a `database` block, a `mobile` block (`buildMobileBlock()` →
`{supported, mode, entry, allowedOrigins, allowedPlatforms, allowedRoles}`), and
an `external` block **only** for external apps (the webhook *secret* is never put
in the manifest — okta-web fetches it over the bridge at install). The manifest
contract is documented in [installed-apps.md](./installed-apps.md).

---

## The bridge to okta-web

### `BridgeSettings` — environment selection

Resolution order is **DB (PlatformSetting) → config → env**, so tokens/URLs can
be rotated from the admin UI without redeploying. It distinguishes the two
environments:

| Concern | Production | Sandbox |
|---|---|---|
| Base URL | `apiUrl()` (`okta_web.api.base_url`) | `sandboxApiUrl()` (`okta_web.sandbox.base_url`) |
| Bearer token | `outboundToken()` (`okta_web.api.token`) | `sandboxOutboundToken()` (`okta_web.sandbox.token`) |

If sandbox values aren't explicitly set, the sandbox getters **fall back to
production** (back-compat). `isSandboxIsolated()` returns true only when **both**
sandbox URL and token are configured — guarding against test traffic leaking
into production. A shared `webhookSecret()` verifies inbound webhooks.

### `OktaWebService` — the HTTP client

- `ping()` / `pingSandbox()` — health checks per environment.
- `probeScopeCatalog()` — fetch the catalog hash for drift detection.
- `publishModule()` — POST a version's manifest + git source
  (`repo`/`ref`/`commit_sha`/download token) + changelog to okta-web.
- `updateModule()` — PUT updates for an already-published module.
- `ensureSandbox()` / `installOnSandbox()` / `resetSandbox()` — create, install
  into, and wipe a partner's sandbox tenant; `installOnSandbox()` is async and
  **polls** an install-status URL with backoff.
- `probeModuleStatus()` / `resyncAppStoreCatalog()` / `syncNotificationCatalog()`
  — route to the right environment (some fan out to **both** prod + sandbox).

---

## The dev → prod (sandbox → production) flow

The two environments are **sandbox** (development/testing) and **production**.
The lifecycle moves a version from authored draft to live-in-production:

1. **Author** — `ModuleCreate` (pick `integration_type` first), then a
   `PartnerModuleVersion`. Scopes are chosen via the
   `ManagesModuleScopes` trait against the mirrored catalog (per-resource access:
   none / read / read+write — never delete).
2. **Scaffold (embedded)** — link GitHub; `GitHubAppService::pushBoilerplate()`
   creates the repo skeleton; `syncModuleMetadataToBranch()` keeps `manifest.json`
   on the branch in step with the DB.
3. **Submit → Review → Approve** — the `ModuleStatus` state machine
   (`Draft → Submitted → InReview → Approved`, with `ChangesRequested`/`Rejected`
   loops). Only `Draft`/`ChangesRequested` are editable.
4. **Test on sandbox** — `installOnSandbox()` installs the version into the
   partner's sandbox tenant in `okta-web` for end-to-end testing (async + poll).
5. **Publish to production** — `ModuleLifecycleService::publish()` builds the
   manifest, pins the commit SHA, calls `OktaWebService::publishModule()`, marks
   prior versions `Deprecated`, stores `okta_web_module_id`, and triggers an
   app-store catalog resync. Status → `Published` (or `Beta`).

Full end-to-end (including how it then reaches both surfaces) is in
[deployment.md](./deployment.md).

---

## Scope catalog sync (mirror, never invent)

`App\Services\PartnerScopes\Catalog\SyncFromOktaWeb`:

1. probe `GET {apiUrl}/partners/permissions/catalog/hash`; **bail if unchanged**;
2. on drift, pull the full catalog and upsert `partner_available_scopes`
   row-by-row;
3. deactivate scopes no longer upstream.

Triggers: cron `partners:sync-scope-catalog` (interval
`OKTA_WEB_SCOPE_CATALOG_SYNC_INTERVAL`, default 60 min), the inbound webhook
`partner_scopes.catalog.changed`, or manual `--force`.
`GetGroupedCatalog` reshapes the mirror into `feature → resource → actions` for
the picker UI.

---

## Webhooks (inbound from okta-web)

One unified endpoint: `POST /webhooks/okta-web`
(`OktaWebWebhookController`), guarded by `VerifyOktaWebWebhook`:

- HMAC-SHA256 over `<timestamp>.<rawBody>` (header `X-Okta-Web-Signature`);
- ±300s freshness on `X-Okta-Web-Timestamp`;
- replay protection (nonce cached ~900s → 409 on reuse);
- one legacy body-only signature accepted for a single deploy window.

Handled events include `partner_scopes.catalog.changed` (→ `SyncFromOktaWeb`
force), `partners.core_ulid_tables.changed`, `module.published` /
`module.publish.failed`, `credential.revoked`, and app-store sale/refund events.
Add new events as a `case` in the controller — do **not** create a second webhook
route.

---

## Boilerplate (the application skeleton)

`resources/partner-boilerplate/` is copied verbatim (including dot-dirs) into
every new application repo by `pushBoilerplate()`, with placeholders substituted
(`__MODULE_NAME__`, `__MODULE_SLUG__`, `__MODULE_LOWER__`, `__MODULE_UPPER__`,
`__MODULE_HASHID__`, and the PHPStan `{{MODULE_NAMESPACE}}` / `{{MODULE_ENV_PREFIX}}`
/ `{{MODULE_CONFIG_PREFIX}}` / `{{MODULE_PATHS}}`). It contains `manifest.json`,
`module.json`, an `app/` skeleton, a `okta_app/webview/` entry, `routes/`, `lang/`,
`scripts/partner-policy/` (Scanner + PHPStan rule, mirrored from okta-web), and
the `.github/workflows/partner-module-policy.yml` CI gate. It lives under
`resources/` (not `storage/`) so it ships in the release artifact. When the
policy scanner changes in okta-web it must be re-copied here — okta-web's scanner
is the source of truth. The resulting structure is the
[installed-application contract](./installed-apps.md).
