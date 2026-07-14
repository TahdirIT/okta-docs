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
│   │   ├── OktaWebService.php            # HTTP client to okta-web (publish/sandbox/sync/ping/...)
│   │   ├── BridgeSettings.php            # live prod vs sandbox URLs + tokens (DB→config→env)
│   │   ├── GitHubAppService.php          # GitHub App integration (pushBoilerplate, pin SHAs, sync files)
│   │   ├── ModuleLifecycleService.php    # version create + review/publish state machine
│   │   ├── PartnerScopes/Catalog/        # SyncFromOktaWeb, GetGroupedCatalog
│   │   ├── PartnerModules/               # Payments/BuildPaymentBlock, Notifications/BuildNotificationBlock, Ci/
│   │   ├── AppStorePricing/              # Pricing/, Payouts/, DiscountCodes/
│   │   └── AdminAccess/                  # Memberships/, Accounts/ (linked accounts), impersonation
│   ├── Models/                           # PartnerModule, PartnerModuleVersion, PartnerApplication,
│   │                                     #   PartnerAvailableScope, PartnerModulePricing, PartnerPayout*, UserMembership
│   ├── Enums/                            # IntegrationType, ModuleStatus, PaymentMethod, PaymentDelivery,
│   │                                     #   NotificationChannel/Delivery/Severity, AppStore/{BillingCycle,PayoutStatus,...}
│   ├── Livewire/                         # Admin/{Modules,Sandbox,Settings,AppStore,Access}/, Partner/{Modules,Payouts,Webhooks,...}
│   └── Http/Controllers/Api/             # OktaWebWebhookController, AppStoreCatalogController, CiPreviewInstallController
├── resources/partner-boilerplate/        # the skeleton pushed to each new application repo
└── routes/                               # web.php, partner.php, admin.php, api.php, webhooks.php
```

Partner-facing routes live under the `dashboard` prefix (`routes/partner.php`):
overview, team (+invites), api-keys, settings, docs (developer guide + prompt
generator), **modules** (create/edit, webhooks history, analytics, errors,
dev-panel per environment, database view per environment), payouts, earnings
ledger, and the GitHub connect flow. Admin review pages live in
`routes/admin.php` (`EnsureSuperAdmin`).

---

## The application data model

- **`PartnerModule`** — the application: `slug`, `display_name(_en)`,
  `description(_en)`, `category`, `integration_type` (enum), `scopes` (JSON,
  canonical form) + legacy `permissions` (JSON), `status`, `okta_web_module_id`
  (set after first publish), and a `manifest` snapshot.
- **`PartnerModuleVersion`** — a concrete version: `version` (semver), `status`,
  per-version overrides (description, icon, scopes, webhook config for canary
  releases), database fields (`requires_database`, `schema_definition`,
  `database_schema`), **mobile fields** (`mobile_support`, `mobile_config`
  JSON, and the canonical per-account-type `audiences` column), and GitHub
  pinning (`github_branch`, `commit_sha`, `pinned_at`).
- **`PartnerAvailableScope`** — the **local mirror** of `okta-web`'s scope
  catalog (`key`, `feature`, `resource`, `action`, labels, `is_dangerous`,
  `is_active`, `sort_order`, `last_synced_at`). Partners never invent scopes here.
- **Monetization** — `PartnerModulePricing` (one row per
  country × tenant-type × billing cycle; `BillingCycle` =
  `monthly | yearly | one_time | lifetime`), `PartnerModuleDiscountCode`,
  and the payout ledger (`PartnerPayout`, `PartnerPayoutWithdrawal`,
  `PartnerPayoutClawback` with their `AppStore` status enums). Partner UI:
  `PricingFormModal`/`PricingMatrix`, `PayoutsDashboard`,
  `WithdrawalRequestModal`, `Earnings/EarningsLedger`.

### Integration types (`App\Enums\IntegrationType`)

- **`embedded`** — ships as code into `okta-web`; has in-tenant UI; needs a
  GitHub repo + boilerplate (`requiresRepository()`).
- **`external`** — hosted by the partner; needs `webhook_url` + `webhook_secret`
  + `webhook_events` (`requiresWebhookEndpoint()`); optional `redirect_urls`.
- **`notification`** — a notification provider declaring channels + delivery
  (api/embedded/hybrid); its manifest `notification` block is built by
  `PartnerModules\Notifications\BuildNotificationBlock` and synced to okta-web
  via `OktaWebService::syncNotificationCatalog()` (fans out to prod + sandbox).
- **`payment`** — a payment gateway/provider (e.g. BNPL, cards) installed once
  per Tenant and consumed by other apps through okta-web's unified payment
  contract; its manifest `payment` block (methods, delivery `api|embedded|hybrid`,
  capabilities, custom methods) is built by
  `PartnerModules\Payments\BuildPaymentBlock`.

### Building the manifest

`PartnerModuleVersion::buildManifest()` (inheriting unset fields from the parent
`PartnerModule`) produces the platform manifest okta-web validates and stores. It
emits `moduleId`, `displayName`, `version`, `category`, `integrationType`,
`description`, `scopes`, `rbac_permissions`, `menu`/`sidebar`, `features`,
pricing, a `database` block, a `mobile` block (`buildMobileBlock()` →
`{supported, mode, entry, passRoleClaim, allowedOrigins, allowedPlatforms,
allowedRoles, requiredScope}` plus per-account-type `audiences[]`), the
`notification`/`payment` blocks per integration type, and an `external` block
**only** for external apps (the webhook *secret* is never put in the manifest —
okta-web fetches it over the bridge at install). The manifest contract is
documented in [installed-apps.md](./installed-apps.md).

### Mobile activation (client surface)

The client surface is opt-in per version: `mobile_support` +
`mobile_config` + `audiences` on `partner_module_versions` feed
`buildMobileBlock()`. Audiences map account types to entries — tenant-scoped
roles (e.g. `tenant-admin` as `primary`) or the cross-tenant portals
(`student`/`guardian` as `dependent`) — so one app can ship different screens
per audience (see [installed-apps.md](./installed-apps.md#the-client-surface)).

---

## The bridge to okta-web

### `BridgeSettings` — environment selection

Resolution order is **DB (PlatformSetting) → config → env**, so tokens/URLs can
be rotated from the admin UI (`Admin/Settings/OktaWebBridge`) without
redeploying. It distinguishes the two environments:

| Concern | Production | Sandbox |
|---|---|---|
| Base URL | `apiUrl()` (`okta_web.api.base_url`) | `sandboxApiUrl()` (`okta_web.sandbox.base_url`) |
| Bearer token | `outboundToken()` (`okta_web.api.token`) | `sandboxOutboundToken()` (`okta_web.sandbox.token`) |

If sandbox values aren't explicitly set, the sandbox getters **fall back to
production** (back-compat). `isSandboxIsolated()` returns true only when **both**
sandbox URL and token are configured — guarding against test traffic leaking
into production. A shared `webhookSecret()` (`okta_web.webhook.secret`) verifies
inbound webhooks.

### `OktaWebService` — the HTTP client

- `ping()` / `pingSandbox()` — health checks per environment;
  `probeScopeCatalog()` — catalog hash for drift detection.
- `publishModule()` — POST a version's manifest + git source
  (`repo`/`ref`/`commit_sha`/download token) + changelog to
  `okta-web` (`POST /partners/modules/publish`); `updateModule()` /
  `getModuleStatus()` / `probeModuleStatus()` for updates + deployment checks.
- `ensureSandbox()` / `installOnSandbox()` / `resetSandbox()` — create, install
  into, and wipe a partner's sandbox tenant; `installOnSandbox()` is async and
  **polls** an install-status URL with backoff.
- Partner-account plumbing: `validateCredential()`, `notifyNewApiKey()`,
  `revokeApiKey()`, `syncPartnerStatus()`, `getTenant()`, `getUser()`.
- Catalog/ops fan-out: `resyncAppStoreCatalog()`, `syncNotificationCatalog()`
  (prod + sandbox), `getDesignTokens()`, `getProducts()`,
  `listWebhookDeliveries()` / `replayWebhookDelivery()`.

---

## The dev → prod (sandbox → production) flow

The two environments are **sandbox** (development/testing) and **production**.
The lifecycle moves a version from authored draft to live-in-production:

1. **Author** — `ModuleCreate` (pick `integration_type` first), then a
   `PartnerModuleVersion`. Scopes are chosen via the
   `ManagesModuleScopes` trait against the mirrored catalog (per-resource access:
   none / read / read+write — never delete).
2. **Connect GitHub (embedded)** — the partner installs the platform's GitHub
   App on their account/org and picks a repo (`GitHubAppController`
   install/callback); `GitHubAppService::pushBoilerplate()` then wipes the repo
   and pushes the boilerplate skeleton with placeholders substituted.
   `syncModuleMetadataToBranch()` keeps `manifest.json`/`module.json` on the
   branch in step with the DB, and `syncManagedFiles()` refreshes the managed
   allowlist (`scripts/partner-policy/**`, the policy CI workflow, `CLAUDE.md`).
3. **Submit → Review → Approve** — the `ModuleStatus` state machine
   (`Draft → Submitted → InReview → Approved`, with `ChangesRequested`/`Rejected`
   loops; `Rejected`/`ChangesRequested → Submitted` to resubmit). `isEditable()`
   is **`Draft`, `ChangesRequested`, and `Rejected`** (a rejected version stays
   editable to fix and re-submit); only `Approved` is publishable. Platform
   admins drive this from `Admin/Modules/ModuleReviewIndex` (queue) and
   `Admin/Modules/ModuleShow` (per-app), with every transition logged to
   `PartnerModuleReview`.
4. **Test on sandbox** — `installOnSandbox()` installs the version into the
   partner's sandbox tenant in `okta-web` for end-to-end testing (async + poll);
   partner UI `InstallSandboxWizard`, admin UI `Admin/Sandbox/SandboxConsole`.
   CI can also trigger a preview install via `POST /api/ci/preview-install`.
5. **Publish to production** — `ModuleLifecycleService::publish()` builds the
   manifest, pins the commit SHA (`resolveCommitSha`), calls
   `OktaWebService::publishModule()`, marks prior versions `Deprecated`, stores
   `okta_web_module_id`, and triggers an app-store catalog resync. Status →
   `Published` (or `Beta`).

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
the picker UI. The same mirror pattern covers okta-web's **countries** and
**account-types** catalogs (`/api/partners/{countries,account-types}/catalog`)
and the core-ULID table list (`partners.core_ulid_tables.changed` →
`SyncCoreUlidTables`).

---

## Webhooks (inbound from okta-web)

One unified endpoint: `POST /webhooks/okta-web`
(`OktaWebWebhookController`), guarded by `VerifyOktaWebWebhook`:

- HMAC-SHA256 over `<timestamp>.<rawBody>` (header `X-Okta-Web-Signature`);
- ±300s freshness on `X-Okta-Web-Timestamp`;
- replay protection (nonce cached ~900s → 409 on reuse);
- one legacy body-only signature accepted for a single deploy window.

Handled events: `ping`, `tenant.updated`, `credential.revoked`,
`module.published` / `module.publish.failed`, `partner_scopes.catalog.changed`
(→ `SyncFromOktaWeb` force), `partners.core_ulid_tables.changed`, and the
app-store sale events `app_store.sale.recorded` / `app_store.sale.refunded`
(which feed the payout ledger). Add new events as a `case` in the controller —
do **not** create a second webhook route.

---

## Access model: memberships, linked accounts, impersonation

Three separate mechanisms let one person hold several hats:

- **Memberships** (`user_memberships`) — one login, multiple contexts
  (platform-admin and/or one row per partner tenant). Switching
  (`membership.switch/{membership}` → `SwitchActiveMembership`) is
  session-only and applied per-request in memory by the
  `ResolveActiveMembership` middleware — zero DB writes.
- **Linked accounts** (`users.identity_group_id`) — separately-owned identities
  grouped together; `account.switch/{user}` performs a real re-login into the
  sibling account (`AdminAccess/Accounts/SwitchToLinkedAccount`).
- **Impersonation** — admin support path (`StartImpersonation` /
  `StopImpersonation`), independent of both.

---

## Boilerplate (the application skeleton)

`resources/partner-boilerplate/` is copied verbatim (including dot-dirs) into
every new application repo by `pushBoilerplate()`, with placeholders substituted
(`__MODULE_NAME__`, `__MODULE_SLUG__`, `__MODULE_LOWER__`, `__MODULE_UPPER__`,
`__MODULE_HASHID__`, and the PHPStan `{{MODULE_NAMESPACE}}` / `{{MODULE_ENV_PREFIX}}`
/ `{{MODULE_CONFIG_PREFIX}}` / `{{MODULE_PATHS}}`). It contains `manifest.json`
(regenerated from `buildManifest()`), `module.json`, an `app/` skeleton, a
`mobile/` entry, `routes/`, `lang/`, `config/`, `database/`, a `CLAUDE.md` (AI
ground rules shipped to every partner repo), `scripts/partner-policy/` (Scanner +
UiScanner + PHPStan rule, mirrored from okta-web), and the
`.github/workflows/partner-module-policy.yml` CI gate. It lives under
`resources/` (not `storage/`) so it ships in the release artifact. When the
policy scanner changes in okta-web it must be re-copied here — okta-web's scanner
is the source of truth; `syncManagedFiles()` re-pushes the managed set to
existing partner repos. The resulting structure is the
[installed-application contract](./installed-apps.md).
