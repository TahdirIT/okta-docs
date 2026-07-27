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
    "mode": "webview",
    "entry": "mobile/screens/dashboard.blade.php",
    "minContract": 1,
    "allowedPlatforms": ["ios", "android", "windows", "linux"],
    "allowedRoles": ["tenant-admin"],
    "requiredScope": "education.students.read",
    "passRoleClaim": true
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
| `moduleId`, `version`, `category`, `displayName*`, `icon`, `developer` | Identity & marketplace listing | both | — |
| `integrationType` | `embedded` / `external` / `notification` — the whole integration shape | both | [`./data-access-and-security.md`](./data-access-and-security.md) |
| `scopes[]` | What Tenant data you may read/write | both | [`./data-access-and-security.md`](./data-access-and-security.md) |
| **`menu`** | The **okta-web sidebar entry** + landing route | platform surface | [`./web-surface.md`](./web-surface.md) |
| **`mobile`** | The **okta-app catalog card** + how it launches | client surface | [`./app-surface.md`](./app-surface.md) |
| `rbac_permissions` | Permissions created on install + granted to roles | platform surface | [`./web-surface.md`](./web-surface.md#6-permissions-rbac) |
| `notifications[]` | The notification types your app may emit (via the host) | both | [`./data-access-and-security.md`](./data-access-and-security.md) |
| `database` | Declares your owned schema + migrations the platform tracks/runs | both | [`./data-access-and-security.md`](./data-access-and-security.md#owning-data) |
| `external` (external apps only) | `webhookUrl` + `webhookEvents[]` (+ `redirectUrls[]`) | external | [`../../claude/web.md`](../../claude/web.md) |

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
  `notification` requires its `notification` block.
- `mobile`: `mode` is `webview` | `native` | `external`. When `mode: webview`,
  `entry` must be a path under `mobile/`; when `mode: native`, `entry` must be a
  `.dart` file under `miniapp_dart/lib/` (source-on-device — the schema/JSON
  `miniapp/` runtime has been removed) and `minContract` declares the minimum host
  contract. Both are re-checked at render/serve time (`realpath` inside the module,
  no `..`).
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
