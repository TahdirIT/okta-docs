# Dependent accounts & per-audience entries

One installed app can reach more than one kind of account from a **single
install**, each with its own entry screen. Two ideas:

- **Primary audience** — tenant-scoped roles (the subscriber/installer's staff:
  `tenant-admin`, `teacher`, …). Seen inside the **tenant scope** (a chosen Tenant
  + role).
- **Dependent audience** — a **general-scope portal** (`student` / `guardian`).
  When a Tenant installs the app, it surfaces automatically to that Tenant's
  students/guardians **cross-tenant** (in their general scope, without choosing a
  Tenant). Dependents don't install or subscribe — visibility is **inherited from
  the Tenant's install**.

An installed app declares a set of **audiences**, each = *(who → which entry)*.
"Who" is either **tenant-scoped roles** or a **general portal**; two audiences may
share one entry. The same audiences concept drives **both** surfaces — the web
launcher (`menu.audiences[]`, see [`./web-surface.md`](./web-surface.md)) and the
mobile catalog (`mobile.audiences[]`, below).

---

## 1. Authoring (okta-partners)

Audiences live in a first-class `audiences` JSON column on
`partner_module_versions` (migration
`2026_05_22_000000_add_audiences_to_partner_module_versions`). Each entry:

```
{ key, kind: primary|dependent, target: role|portal,
  role,          // when target = role   (e.g. tenant-admin, teacher)
  portal,        // when target = portal  (student | guardian)
  web_route,     // this audience's okta-web entry
  mobile_entry } // this audience's okta-app entry
```

- Authored in `App\Livewire\Partner\Modules\VersionEditor` via **two repeatable
  lists** — one per surface tab (Okta-App → `mobile_entry`, okta-web →
  `web_route`) — sharing `partials/surface-audiences-editor.blade.php`. On save
  `buildAudiencesData()` merges both lists into the single `audiences` column,
  keyed by account type (blank-entry rows dropped).
- **Account-type vocabulary** (which roles/portals exist) is the
  `partner_account_type_catalog` table, synced from okta-web
  (`GET /api/partners/account-types/catalog` via
  `SyncAccountTypeCatalogFromOktaWeb`), with `config/account-types.php` as the
  fallback (`tenant-admin`/`teacher`/`administrator` = role; `student`/`guardian`
  = portal). The two portals are `student` and `guardian`.
- `PartnerModuleVersion::buildMobileBlock()` emits `mobile.audiences[]` and
  `buildManifest()` emits `menu.audiences[]`. Both keep mirroring the **primary**
  audience at the block's top level, so an older okta-web still reads a single-block
  manifest during a partial deploy.

---

## 2. Manifest shape & validation (okta-web)

`mobile.audiences[]` in the published manifest:

```json
"mobile": {
  "supported": true,
  "audiences": [
    { "key": "staff", "kind": "primary",
      "roles": ["tenant-admin", "teacher"],
      "mode": "embedded", "entry": "mobile/screens/staff.blade.php",
      "requiredScope": "education.students.read",
      "allowedPlatforms": ["ios", "android"] },
    { "key": "guardian", "kind": "dependent", "portal": "guardian",
      "mode": "embedded", "entry": "mobile/screens/family.blade.php" },
    { "key": "student", "kind": "dependent", "portal": "student",
      "mode": "embedded", "entry": "mobile/screens/family.blade.php" }
  ]
}
```

Validated by `App\Modules\Core\ManifestValidator::validate()`:

- `kind ∈ {primary, dependent}`; `portal ∈ {student, guardian}`;
  `mode ∈ {embedded, external}`; `key` ≤ 64.
- **roles[] XOR portal** — each audience targets either `roles[]` **or** a
  `portal`, never both or neither (enforced in `after()`, only when
  `mobile.supported === true`).
- Each audience must resolve a `mode` + `entry` (own or inherited from the block);
  an embedded `entry` is fenced under `mobile/` with no `..`.
- The **legacy single block** (`mobile.entry` + `mobile.allowedRoles[]`, no
  `audiences`) stays valid — it normalizes to one `primary`/roles audience.

---

## 3. Consumption (okta-web)

`App\Services\MobileAppCatalog\NormalizeMobileAudiences` is the shared normalizer:
legacy → one primary/roles audience; `pickForRoleKeys()` (tenant scope) and
`pickForPortal()` (general scope). It forces `required_scope = null` and
`roles = []` on portal audiences.

### 3.1 Tenant-scope catalog

`GetMobileCatalogForUser(user, tenantId, roleId)` lists the Tenant's active
installs and, for each, picks the audience whose `roles[]` match the **active role
name plus employee category** (`TenantEmployee.type` — so `teacher` /
`administrator` resolve), emitting **that audience's** `entry`/`mode`/
`requiredScope`/`allowedRoles`; then post-filters by `required_scope` (permission)
and `allowed_roles`.

- `GET /api/mobile/app-catalog?tenant_id&role_id` → `MobileAppCatalogController::index`
  (both params required).

### 3.2 Portal (general-scope) catalog — the cross-tenant one

`GetPortalCatalogForUser(user, portal)`:

1. finds the user's **active** `TenantStudent`/`TenantGuardian` links across
   **all** Tenants (`released_at` + `deleted_at` null);
2. for each Tenant behind those links, reads active installs whose manifest
   declares a **dependent audience with `portal == $portal`** (`mobile.supported
   === true`, via `pickForPortal`);
3. emits one card **per (Tenant, app)** — carrying that audience's `entry` plus the
   card's own **`tenant_id` + `tenant_name`** — sorted by Tenant then app.

A card is therefore **per-school**: the same app appears once for each school the
user is linked to (cards are **not** deduped by slug — the concrete Tenant on the
card is what the launch needs).

- `GET /api/mobile/app-catalog/portal?portal=student|guardian` →
  `MobileAppCatalogController::portal`.

### 3.3 Launch

- Tenant: `POST /api/mobile/app-catalog/{slug}/launch` `{tenant_id, role_id}`.
- Portal: `POST /api/mobile/app-catalog/portal/{slug}/launch` `{tenant_id, portal}`
  (**no role**) → `launchPortal`. The client sends the card's own `tenant_id` — the
  concrete school is **picked from the card**, not aggregated inside the app.
- `IssueWebviewLaunch::portal(user, tenantId, portal, slug, mobileBlock)` signs an
  `app.webview.show` URL carrying `u` (user) + `t` (tenant) + `p` (portal), and
  **no** role id (external mode returns the partner URL instead).
- Middleware `app.webview` (`EnsureAppWebview`) branches to `handlePortal()` when
  `p` is present: re-verifies the active student/guardian link on that Tenant,
  calls `setPermissionsTeamId($tenantId)` so the app's Partner-API reads resolve
  per-Tenant, and passes `portal`/`tenant_id`/`role=null` to
  `WebviewController::show`, which renders the sandboxed `mobile/` entry
  (`Cache-Control: no-store`, `frame-ancestors 'none'`).

### 3.4 Gating & isolation

- **Portal visibility is gated by the active link only** — no `requiredScope`, no
  spatie role/team (the general scope has `tenant_id = null` and no role).
  `NormalizeMobileAudiences` nulls `required_scope` for portal audiences,
  `GetPortalCatalogForUser` filters purely on link presence, and
  `EnsureAppWebview::handlePortal()` re-checks the link at launch.
- **Reaching Tenant data** is unchanged: the app reads/writes only via the
  [Partner API](./data-access-and-security.md) under the resolved Tenant's
  installation token (`setPermissionsTeamId` sets that Tenant). There is **no new
  install record** for dependents — visibility derives from the Tenant's install
  plus the manifest's dependent audience.

---

## 4. Client (okta-app)

- Feature `lib/features/portal_app_catalog/` — `PortalAppCatalogRepository`,
  `PortalAppCard` (carries `tenantId`/`tenantName`), `portalAppCatalogProvider`
  (keyed by portal), `PortalAppCatalogSection` (groups cards by Tenant when >1).
- Calls `GET /api/mobile/app-catalog/portal?portal=…` and
  `POST …/portal/{slug}/launch {tenant_id, portal}`.
- Cards render under `PortalScreen` (`StudentPortalScreen`/`GuardianPortalScreen`,
  routes `/portal/student`, `/portal/guardian`) and open in the **same**
  `AppWebviewScreen` host as tenant-scope cards.
- The portal identity (`student`/`guardian`) is a route param + the signed-in
  `auth.user` gate — the client's `SessionContext.scope` is `tenant|system` only,
  so the portal catalog rides on the signed-in person, not a separate client
  "general" context or auth flow.

---

## 5. Backward compatibility

- Old single-block manifests (`entry` + `allowedRoles`) → one `primary` audience;
  the tenant-scope catalog behaves exactly as before.
- An app with **no** dependent/portal audience never appears in the portal catalog.
- okta-partners keeps mirroring the primary's `entry`/`allowedRoles` at the top of
  the `mobile`/`menu` blocks so an older okta-web still works during a partial
  deploy.

---

## 6. Code map

| Concern | Path |
|---|---|
| Audiences storage (authoring) | okta-partners · `partner_module_versions.audiences` (migration `2026_05_22_000000_add_audiences_to_partner_module_versions.php`) |
| Manifest emission | okta-partners · `PartnerModuleVersion::buildMobileBlock()` / `buildManifest()` |
| Audiences editor | okta-partners · `App\Livewire\Partner\Modules\VersionEditor` + `partials/surface-audiences-editor.blade.php` |
| Account-type vocabulary | okta-partners · `partner_account_type_catalog` ← okta-web `GET /api/partners/account-types/catalog` |
| Manifest validation | okta-web · `App\Modules\Core\ManifestValidator::validate()` |
| Audience normalization | okta-web · `App\Services\MobileAppCatalog\NormalizeMobileAudiences` |
| Tenant-scope catalog | okta-web · `GetMobileCatalogForUser` + `MobileAppCatalogController::index` |
| Portal catalog | okta-web · `GetPortalCatalogForUser` + `MobileAppCatalogController::portal` |
| Launch / webview | okta-web · `IssueWebviewLaunch::portal()`, `EnsureAppWebview`, `WebviewController` |
| Client feature | okta-app · `lib/features/portal_app_catalog/`; host `lib/features/app_catalog/presentation/webview_screen.dart` |

See also [`./manifest-reference.md`](./manifest-reference.md) (all manifest blocks),
[`./app-surface.md`](./app-surface.md) (the mobile WebView surface), and
[`./web-surface.md`](./web-surface.md) (the web `menu.audiences[]` sibling).
