# The client surface — screens inside okta-app

How you develop the screen that appears as a **card** in the **okta-app** Flutter
client and runs inside its WebView. This is the second half of the
[dual surface](./README.md); it's driven entirely by your manifest's `mobile`
block plus one server-rendered entry page.

There are two modes:

- **embedded** — your screen is a Blade page **served by okta-web** at
  `/app/{slug}` and loaded into the client's WebView. (Most apps. The bulk of this
  page.)
- **external** — your screen is **hosted by you**; the client opens your URL
  directly. (See the last section.)

---

## What "appearing in okta-app" means

okta-app never reads your manifest or talks to okta-partners. It calls okta-web's
mobile API, gets a list of **cards** (one per installed app that opted into mobile),
renders them on the home screen, and when the user taps a card it asks okta-web for
a **launch URL** and opens it in a platform-specific WebView.

```
okta-app (Flutter)                              okta-web
──────────────────                              ─────────
GET /api/mobile/app-catalog ──────────────────▶ build cards from each installed
   ?tenant_id&role_id                            module's manifest.mobile block
   ◀───────────────────────────── { modules:[ {slug, display_name, mode,
                                      entry, allowed_platforms, allowed_roles,
                                      required_scope, pass_role_claim, icon} ] }
user taps a card
POST /api/mobile/app-catalog/{slug}/launch ────▶ embedded → temporarySignedRoute()
   {tenant_id, role_id}                          external → your URL (+role JWT)
   ◀──────── LaunchPayload (embedded signed URL | external URL)
open in WebView
  embedded → GET /app/{slug}?u&t&r&expires&sig ─▶ EnsureAppWebview + WebviewController
                                                  renders mobile/screens/<entry>.blade.php
```

> The catalog request is a `GET` with `tenant_id`/`role_id` as **query params**
> (confirmed from the client: `AppCatalogRepository.list(...)`). The launch is a
> `POST` with a JSON body.

---

## 1. Declare the card — manifest `mobile` block

This block is the entire contract for the client surface:

```json
{
  "mobile": {
    "supported": true,                 // false → no card at all
    "mode": "embedded",                // "embedded" | "external"
    "entry": "mobile/screens/dashboard.blade.php",   // embedded only: page okta-web renders
    "allowedPlatforms": ["ios", "android", "windows", "linux"],  // [] = all
    "allowedRoles": ["tenant-admin"],  // [] = no role filter
    "requiredScope": "education.students.read",       // empty = no scope gate
    "passRoleClaim": true,             // external: include a signed role JWT

    "audiences": [                     // optional: one card/entry per account type
      { "key": "admin",    "kind": "primary",   "roles": ["tenant-admin"],
        "entry": "mobile/screens/admin.blade.php" },
      { "key": "guardian", "kind": "dependent", "portal": "guardian",
        "entry": "mobile/screens/guardian.blade.php" }
    ]
  }
}
```

okta-web filters cards before returning them: by **platform** (`X-App-Platform`
header), by the active **role** (`allowedRoles`), and by **scope** (`requiredScope`
must be held by the active role). A `mobile.app_catalog.kill_switch` platform
setting and a minimum-app-version gate (`HTTP 426`) can also suppress cards.

Field reference: [`./manifest-reference.md`](./manifest-reference.md).

### Audiences — different screens per account type

Without `audiences[]`, the whole app has **one** entry and one role list. With
`audiences[]`, each account type gets its own screen (two audiences may share
one entry):

- **`kind: primary` + `roles[]`** — tenant-scoped audiences (e.g. the school
  admin, a teacher/observer). Their cards appear in the normal per-tenant
  catalog above.
- **`kind: dependent` + `portal: student|guardian`** — cross-tenant audiences.
  When the school installs your app, it automatically surfaces to its students/
  guardians in their **general-scope portals** — no Tenant selection. okta-app
  fetches these via `GET /api/mobile/app-catalog/portal?portal=student|guardian`
  and launches via `POST /api/mobile/app-catalog/portal/{slug}/launch`
  (`GetPortalCatalogForUser` server-side). Block-level fields act as defaults
  for every audience.

`okta-exams` is a live example: admin/observer tenant audiences plus
student/guardian portal audiences, each with its own `mobile/screens/*.blade.php`.

---

## 2. Develop the entry page (embedded mode)

Your entry is a **single server-rendered Blade page** at the `entry` path under
your module's `mobile/` directory. It is **not** a Livewire page and does **not**
share the okta-web shell — it's a standalone HTML document the WebView loads as its
main document. The established pattern:

1. **Mint a short-lived API token** for the in-WebView code to call your own API.
   The signed launch URL expires quickly (~10 min), but the user may keep the
   screen open for hours, so mint a longer-lived token here and revoke older ones
   of the same name:

   ```php
   @php
       // View bag from WebviewController: $user, $role, $slug
       $user->tokens()->where('name', 'example-app:webview')->delete();
       $apiToken = $user->createToken(
           name: 'example-app:webview',
           abilities: ['*'],
           expiresAt: now()->addHours(8),
       )->plainTextToken;

       $tenantId = (int) (request()->attributes->get('app.webview.tenant_id')
           ?: request()->query('t', 0));

       $boot = [
           'apiBase'  => '/api/example-app',          // your routes/api.php prefix
           'token'    => $apiToken,                    // Bearer for your API calls
           'tenantId' => $tenantId,
           'role'     => ['id' => $role?->id, 'name' => $role?->name],
           'locale'   => app()->getLocale(),
           'closeUrl' => 'okta-app://close',           // tell the shell to close the WebView
           'theme'    => in_array(request()->query('theme'), ['dark','light'], true)
                            ? request()->query('theme') : 'auto',
       ];
   @endphp
   ```

2. **Render a self-contained SPA** that hands `$boot` to JS and routes internally
   with the **URL hash** (path navigation isn't allowed — only `/app/{slug}` exists;
   the client's allow-list strips the fragment before matching, so you may navigate
   `#/students/42` freely):

   ```blade
   <!DOCTYPE html>
   <html lang="{{ str_replace('_','-', app()->getLocale()) }}"
         @if (app()->getLocale() === 'ar') dir="rtl" @endif>
   <head> <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover"> </head>
   <body>
     <main id="view"></main>
     <script>
       var BOOT = @json($boot, JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES);

       function api(path, opts) {
         opts = opts || {};
         var headers = Object.assign({ 'Accept': 'application/json' }, opts.headers || {});
         if (BOOT.token)    headers['Authorization']   = 'Bearer ' + BOOT.token;
         if (BOOT.tenantId) headers['X-Okta-Tenant-Id'] = String(BOOT.tenantId); // your context middleware reads this
         return fetch(BOOT.apiBase + path, { method: opts.method || 'GET', headers: headers, body: opts.body });
       }

       function render() { /* read location.hash, draw into #view, call api(...) */ }
       window.addEventListener('hashchange', render);
       render();
     </script>
   </body>
   </html>
   ```

3. **Call your own mobile API** (`/api/example-app/*`, defined in your
   `routes/api.php`) with that Bearer token + an `X-Okta-Tenant-Id` header. Your
   module's own context middleware reads the header, verifies membership, and calls
   `Tenant::makeCurrent()` server-side so your endpoints run in the right Tenant.

> Why a minted token and not a cookie? Your module's API group is mounted
> **stateless** (`auth:sanctum`), and the WebView's only authenticated entry was the
> signed URL. A Bearer token is the clean bridge from "rendered page" to "API calls".

### Theming & RTL

The client passes the theme in the URL **fragment** (`#theme=dark`), never a query
param — adding a query param would break the signed URL. Apply it before first
paint from `location.hash`, and keep the server `$boot['theme']` as a fallback.
Mirror the okta-web tokens/typography so the screen feels native (the platform also
injects base chrome before `</head>` — see §4).

### Closing the screen

Navigate to `okta-app://close` (provided as `BOOT.closeUrl`) when the user finishes;
the Flutter shell intercepts that scheme and pops the WebView.

---

## 3. The native bridge — `OktaBridge` (NFC, camera)

A WebView can't reach some native hardware directly. okta-app exposes a JavaScript
channel named **`OktaBridge`** so your page can ask the shell to do it. The protocol
(only present when running inside okta-app — guard with `if (window.OktaBridge)`):

**Capabilities** — the shell injects `window.OktaBridgeCaps` (e.g. `{ nfc: true }`)
once it has probed the device. Build hardware UI conditionally.

**NFC** — your page posts ops; the shell calls back into your page:

```js
if (window.OktaBridge) {
  // ask the shell to start/stop the native NFC reader
  OktaBridge.postMessage(JSON.stringify({ op: 'nfc.start' }));   // and { op: 'nfc.stop' }

  // the shell drives this callback you define:
  window.__oktaNfc = {
    onEvent: function (ev) {
      // ev.type: 'started' | 'stopped' | 'read' | 'error'
      if (ev.type === 'read' && ev.uid) {
        api('/mobile/checkin/nfc', { method: 'POST', body: JSON.stringify({ uid: ev.uid }) });
      }
    }
  };
}
```

**Camera / microphone** — just use a normal web camera flow (e.g. a barcode
scanner). When the page requests the camera, the shell raises the **OS permission
prompt just-in-time**; you don't call anything special. (Stop the radio/camera when
you navigate away — release on `hashchange` off the capture screen.)

> Which native capabilities exist beyond NFC + camera is shell-dependent and may
> grow. `> TODO: confirm` the full `OktaBridge` op list for your target shell
> version; NFC (`nfc.start`/`nfc.stop` + `__oktaNfc.onEvent`) and camera permission
> bridging are the verified ones.

---

## 4. How okta-web serves the embedded screen (so you know the guarantees)

You don't write this, but understanding it explains the view bag and the headers.

**Route** — `routes/app.php`: `GET /app/{slug}` with middleware
`['web', 'signed', 'app.webview', 'throttle:30,1']`. `signed` validates the
launch-URL signature (the signature *is* the session — no cookie).

**`app.webview` (`EnsureAppWebview`)** re-authenticates and re-authorizes at open
time (not just at launch time):

- reads `u`/`t`/`r` (user, tenant, role) from the signed query and hydrates the user;
- verifies the user still belongs to the Tenant and still holds the role;
- reads your **`mobile` block from the locally-stored manifest** and re-checks
  `requiredScope` and `allowedRoles` — so a role change inside the signature's
  validity window can't sneak a now-disallowed card open;
- stashes `app.webview.{role, tenant_id, mobile_block}` as request attributes.

**`WebviewController::show`** then:

- requires `mode === 'embedded'` (external cards never render here);
- resolves your installed module dir `Modules/<StudlySlug>/`;
- **sandboxes** the manifest `entry` to your `mobile/` directory — it must start
  with `mobile/`, contain no `..`, and `realpath` under `<module>/mobile/`, else 404;
- renders the Blade with view bag `{ user, role, slug }`;
- **injects platform chrome** (typography/base styles) right before `</head>` so
  every embedded screen inherits the okta-web look;
- sets `Cache-Control: no-store`, `Content-Security-Policy: frame-ancestors 'none'`,
  `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`.

Takeaways for you: put your entry under `mobile/`, expect `$user`/`$role`/`$slug`
in the view bag, and don't rely on cookies or framing.

---

## 5. External mode

If `integrationType` is `external` and `mobile.mode` is `external`, you host the
screen yourself:

- The launch endpoint returns **your URL** instead of an `/app/{slug}` URL.
- okta-app loads it in the WebView behind an **origin allow-list** (`allowedOrigins`
  — navigation is fenced to those prefixes).
- If `passRoleClaim: true`, the client appends a signed **role JWT** in the URL
  **fragment** (`#okta_role_token=…`); read `window.location.hash` to extract it and
  exchange/verify it on your backend.
- Your backend consumes the okta-web HTTP runtime (`/api/apps/*`) with your
  installation Bearer token + receives signed webhooks — see
  [`./data-access-and-security.md`](./data-access-and-security.md) and
  [`../../../claude/web.md`](../../../claude/web.md).

---

## Checklist — a client screen (embedded)

- [ ] `mobile` block in the manifest: `supported`, `mode: embedded`, `entry` under
      `mobile/`, plus `allowedPlatforms`/`allowedRoles`/`requiredScope` as needed.
- [ ] Entry Blade mints a Sanctum token, builds a `BOOT` object, renders a
      hash-routed SPA.
- [ ] All data calls go to **your** `/api/<slug>/*` with `Authorization: Bearer` +
      `X-Okta-Tenant-Id`; your context middleware calls `Tenant::makeCurrent()`.
- [ ] Native hardware via `OktaBridge` (guarded by `window.OktaBridge`).
- [ ] Theme from the URL fragment; `okta-app://close` to exit.
- [ ] Verified on each platform you listed in `allowedPlatforms`.
