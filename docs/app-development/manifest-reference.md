# Manifest reference (the blocks that drive your surfaces)

`manifest.json` (at your repo root) is the platform contract for your application.
This page is a quick reference focused on the blocks you touch while developing the
two surfaces. The platform validates it (`App\Modules\Core\ManifestValidator` in
okta-web) on publish/install. For the complete contract and package structure, see
[`../../claude/installed-apps.md`](../../claude/installed-apps.md).

> `manifest.json` ≠ `module.json`. `manifest.json` is the **platform** descriptor
> (this page). `module.json` is the **nwidart loader** descriptor (name, alias,
> service provider) — you rarely edit it.

---

## Skeleton

```json
{
  "moduleId": "example-app",
  "displayName": "Example App",
  "displayNameEn": "Example App",
  "version": "1.0.0",
  "category": "communication",
  "integrationType": "embedded",
  "description": "…",
  "icon": "https://…",
  "developer": { "tenantSlug": "…", "name": "…" },

  "scopes": [
    { "key": "education.students.read",  "required": true,  "reason": "" },
    { "key": "education.students.write", "required": false, "reason": "" }
  ],

  "menu":   { "route": "example-app.dashboard" },

  "mobile": {
    "supported": true,
    "mode": "embedded",
    "entry": "mobile/screens/dashboard.blade.php",
    "allowedPlatforms": ["ios", "android", "windows", "linux"],
    "allowedRoles": ["tenant-admin"],
    "requiredScope": "education.students.read",
    "passRoleClaim": true,
    "audiences": [
      { "key": "admin",    "kind": "primary",   "roles": ["tenant-admin"],
        "entry": "mobile/screens/admin.blade.php" },
      { "key": "student",  "kind": "dependent", "portal": "student",
        "entry": "mobile/screens/student.blade.php" },
      { "key": "guardian", "kind": "dependent", "portal": "guardian",
        "entry": "mobile/screens/guardian.blade.php" }
    ]
  },

  "rbac_permissions": { "tenant-admin": ["example_app.dashboard.view"] },

  "notifications": [
    {
      "key": "example-app.resource.event",
      "display_name": { "ar": "…", "en": "…" },
      "variables": { "student_id": "string", "event_at": "datetime" },
      "default_channels": ["in_app", "whatsapp", "push"],
      "severity": "info",
      "is_active": true
    }
  ],

  "database": {
    "requiresDatabase": true,
    "schema": "m_example_app",
    "migrations": [
      { "version": "2024_01_01_000001",
        "name": "create_example_app_things_table",
        "path": "database/migrations/2024_01_01_000001_create_example_app_things_table.php" }
    ]
  }
}
```

---

## Which block drives what

| Block | Drives | Used by | More |
|---|---|---|---|
| `moduleId`, `version`, `category`, `displayName*`, `icon`, `screenshots`, `developer` | Identity & marketplace listing | both | — |
| `integrationType` | `embedded` / `external` / `notification` / `payment` — the whole integration shape | both | [`./data-access-and-security.md`](./data-access-and-security.md) |
| `scopes[]` | What Tenant data you may read/write | both | [`./data-access-and-security.md`](./data-access-and-security.md) |
| **`menu`** | The **okta-web sidebar entry** + landing route (supports per-account-type `menu.audiences[]`) | platform surface | [`./web-surface.md`](./web-surface.md) |
| **`mobile`** | The **okta-app catalog card(s)** + how they launch; `audiences[]` maps each account type (tenant roles or student/guardian portal) to its own entry | client surface | [`./app-surface.md`](./app-surface.md) |
| `rbac_permissions` | Permissions created on install + granted to roles | platform surface | [`./web-surface.md`](./web-surface.md#6-permissions-rbac) |
| `notifications[]` | The notification types your app may emit (via the host) | both | [`./data-access-and-security.md`](./data-access-and-security.md) |
| `database` | Declares your owned schema + migrations the platform tracks/runs | both | [`./data-access-and-security.md`](./data-access-and-security.md#owning-data) |
| `external` (external apps only) | `webhookUrl` + `webhookEvents[]` (+ `redirectUrls[]`) | external | [`../../claude/web.md`](../../claude/web.md) |
| `pricing` (paid apps) | Billing cycles + prices per country/tenant type (managed from the okta-partners pricing matrix) | marketplace | [`../../claude/partners.md`](../../claude/partners.md) |
| `aiSupport` / `aiMode` | Opts the app into the platform AI agent (`ai/agent/*` runtime; sandbox auto-approves) | both | [`../../claude/web.md`](../../claude/web.md) |
| `notification` (notification providers) / `payment` (payment providers) | Provider-type contracts: channels + delivery / methods + delivery + capabilities | provider apps | [`../../claude/partners.md`](../../claude/partners.md) |

---

## Validation rules you'll hit

- `moduleId` kebab-case; `version` semver `MAJOR.MINOR.PATCH`; `displayName`,
  `category`, `description` required.
- `scopes[].key` must be canonical `<feature>.<resource>.<action>`, **at least one
  `required: true`**, and **every key must already exist in the platform scope
  catalog** (you can't request a scope the platform doesn't publish). Partners get
  `read`/`write` only — never `delete`.
- `integrationType` coherence: `embedded` must **not** carry an `external` block;
  `external` **requires** `external.webhookUrl` (HTTPS) + `webhookEvents[]`;
  `notification` requires its `notification` block; `payment` requires its
  `payment` block (methods, delivery, capabilities — custom methods are
  cross-checked bidirectionally).
- `mobile`: when `mode: embedded`, `entry` must be a path under `mobile/`
  (re-checked at render time — must `realpath` inside `<module>/mobile/`, no `..`).
- `mobile.audiences[]`: each audience declares `kind` (`primary` | `dependent`)
  and **either** tenant `roles[]` **or** a `portal` (`student` | `guardian`) —
  never both; each needs a resolvable `mode` + `entry` (block-level values act
  as defaults).
- `database.migrations[].version` is a timestamp; each migration has `path` XOR
  `sql_up`.

---

## Scopes: add new ones the right way

You can only request scopes that exist in the platform catalog. To use a brand-new
capability:

1. It must first be **registered in okta-web** (`PartnerScopes\Catalog\RegisterScope`).
2. okta-partners **mirrors** the catalog (hash-aware sync).
3. Then it appears in the scope picker and you can add it to your manifest.

See [`../../claude/partners.md`](../../claude/partners.md#scope-catalog-sync-mirror-never-invent).
