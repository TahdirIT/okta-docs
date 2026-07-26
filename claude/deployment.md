# Deployment — from authoring to both surfaces, dev → prod

End-to-end: how an application moves from being authored in
[`okta-partners`](./partners.md), through installation into
[`okta-web`](./web.md), to appearing on **both** the platform surface
(`okta-web` web UI) and the client surface ([`okta-app`](./app.md)) — and how the
**sandbox → production** (dev → prod) progression works.

Companion files: [installed-apps.md](./installed-apps.md) (the contract) ·
[architecture.md](./architecture.md) (the system view).

---

## The two environments

`okta-web` runs as two environments — **sandbox** (development / testing) and
**production**. This is the dev → prod progression. The two are selected from the
`okta-partners` side by `BridgeSettings`:

| | Sandbox (dev/test) | Production |
|---|---|---|
| Bridge base URL | `BridgeSettings::sandboxApiUrl()` | `BridgeSettings::apiUrl()` |
| Bridge token | `sandboxOutboundToken()` | `outboundToken()` |
| okta-web tenants | `is_sandbox=true`, `MODULES_SANDBOX_DB_NAME` | normal tenants, `MODULES_DB_NAME` |
| AI approval | auto-approved on install (`SandboxAutoApprover`) | manual approval |

If sandbox bridge settings aren't explicitly configured they **fall back to
production**; `BridgeSettings::isSandboxIsolated()` is true only when both a
sandbox URL and token are set, which prevents test traffic from leaking into
production.

> Terminology bridge: the product overview calls this "dev → prod". In code the
> environments are named **sandbox** and **production**. `> TODO: confirm` whether
> any third/staging environment exists beyond these two.

---

## End-to-end flow

```
 author in okta-partners ─▶ publish manifest ─▶ okta-web validates & registers
        │                                              │
        │ (embedded) push boilerplate to GitHub repo   │
        ▼                                              ▼
   build app code                          install into a Tenant (InstallModule)
                                                       │
                          ┌────────────────────────────┴───────────────────────┐
                          ▼                                                      ▼
              PLATFORM surface (okta-web)                          CLIENT surface (okta-app)
        embedded module UI in the web sidebar                 card in /api/mobile/app-catalog,
        at manifest `menu.route`                              launched via WebView (/app/{slug}
                                                              embedded, or partner URL external)
```

### 1. Author (okta-partners)

Create a `PartnerModule` (pick `integrationType`) and a `PartnerModuleVersion`;
select scopes from the **mirrored** catalog (`partner_available_scopes`). For
embedded apps, link GitHub — `GitHubAppService::pushBoilerplate()` scaffolds the
repo and `syncModuleMetadataToBranch()` keeps `manifest.json` on the branch in
sync with the DB. See [installed-apps.md](./installed-apps.md) for what gets
built.

### 2. Lifecycle review (okta-partners)

`App\Enums\ModuleStatus` state machine:

```
Draft ─▶ Submitted ─▶ InReview ─▶ Approved ─▶ Beta / Published
  ▲          │            │
  └ ChangesRequested ◀────┘   (and Rejected ─▶ back to Draft)
        Published ─▶ Suspended / Deprecated
```

Only `Draft` and `ChangesRequested` are editable.

### 3. Test on sandbox (okta-partners → okta-web sandbox)

`OktaWebService::ensureSandbox()` then `installOnSandbox()` installs the version
into the partner's **sandbox** tenant in `okta-web`. The call is **async** —
okta-partners polls an install-status URL with backoff until done. The partner
exercises both surfaces against sandbox before going live.

### 4. Publish to production (okta-partners → okta-web)

`ModuleLifecycleService::publish()`:

1. promote draft migrations; adopt AI/notification declarations from the repo;
2. `PartnerModuleVersion::buildManifest()`;
3. prepare git source (pin `commit_sha`);
4. `OktaWebService::publishModule()` → POST to `okta-web` with manifest + source;
5. okta-web runs `ManifestValidator::validate()` and records the `Module` row
   (storing the validated `manifest`);
6. mark prior versions `Deprecated`; store `okta_web_module_id`; set status
   `Published` (or `Beta`); trigger an app-store catalog resync.

okta-web confirms results back to okta-partners via the signed
`POST /webhooks/okta-web` (`module.published` / `module.publish.failed`).

### 5. Install into a Tenant (okta-web)

When a Tenant installs the published application, `InstallModule::execute()`
([web.md](./web.md#how-applications-get-installed-into-okta-web)) issues an
installation token, grants scopes + RBAC, optionally provisions an isolated DB
schema, runs the module's migrations, and (for external apps) syncs webhook
subscriptions. This single install is what lights up **both** surfaces.

### 6. Platform surface goes live (okta-web)

The embedded module's UI appears in the web sidebar at `manifest.menu.route`,
gated by `module.access:<slug>` for users of that Tenant with the right
permissions.

### 7. Client surface goes live (okta-app)

`okta-app` calls `GET /api/mobile/app-catalog` for the active `(tenant, role)`;
`GetMobileCatalogForUser` builds a card from the module's `mobile` block
(filtered by platform/role/scope). Launching it:

- **embedded** → a short-lived **signed** URL to `okta-web`'s `/app/{slug}`,
  which renders the module's `mobile.entry` Blade inside the WebView;
- **native** → a signed payload URL; `BundleMiniappSource` returns the module's
  `miniapp_dart/lib/**.dart` as a source bundle, which okta-app compiles **on the
  device** (cached per published version) and renders natively — no WebView;
- **external** → the partner-hosted URL, with a signed role JWT if
  `passRoleClaim` is set.

The dev → prod progression repeats per environment: steps 3–7 run first against
**sandbox**, then against **production**.

---

## Runtime sync & ongoing operations

- **Scope catalog**: `okta-web` is the source of truth; `okta-partners` mirrors it
  via hash-aware `SyncFromOktaWeb` (cron `partners:sync-scope-catalog`, the
  `partner_scopes.catalog.changed` webhook, or `--force`).
- **Webhooks in** (web → partners): all events land on the unified
  `POST /webhooks/okta-web`, verified by `VerifyOktaWebWebhook` (HMAC envelope +
  freshness + replay).
- **Webhooks out** (web → external apps): `DispatchEvent` → queued
  `DeliverPartnerWebhook` jobs sign and deliver to each subscriber's
  `webhookUrl`, with retry/backoff and a recovery scheduler.
- **Uninstall**: `UninstallModule` revokes installation tokens, deactivates
  webhook subscriptions (kept for audit), revokes RBAC + cross-module access, and
  drops the isolated schema — removing the application from both surfaces.

---

## Where to look (code anchors)

| Concern | Repo · path |
|---|---|
| Environment selection (prod/sandbox) | `okta-partners` · `app/Services/BridgeSettings.php` |
| Publish / sandbox install client | `okta-partners` · `app/Services/OktaWebService.php` |
| Lifecycle / publish orchestration | `okta-partners` · `app/Services/ModuleLifecycleService.php` |
| Repo scaffolding | `okta-partners` · `app/Services/GitHubAppService.php` |
| Manifest validation | `okta-web` · `app/Modules/Core/ManifestValidator.php` |
| Install / uninstall | `okta-web` · `app/Modules/Core/{InstallModule,UninstallModule}.php` |
| Mobile catalog (client surface) | `okta-web` · `app/Services/MobileAppCatalog/GetMobileCatalogForUser.php` |
| Embedded WebView render | `okta-web` · `routes/app.php` + `WebviewController` |
| Catalog consumption (client) | `okta-app` · `lib/features/app_catalog/` |
