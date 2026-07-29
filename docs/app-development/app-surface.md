# The client surface — screens inside okta-app

How you develop the screen that appears as a **card** in the **okta-app** Flutter
client. This is the second half of the [dual surface](./README.md); it's driven
entirely by your manifest's `mobile` block.

There are three modes:

- **webview** — your screen is a Blade page **served by okta-web** at
  `/app/{slug}` and loaded into the client's WebView. (The bulk of this page.)
- **native** — your screen is **real Dart** under `okta_app/native/main/lib/`, shipped as
  source and **compiled on the device** (source-on-device), then rendered
  natively — **no WebView**. This is the successor to the retired schema/JSON
  mini-app runtime. (See "Native mode".)
- **external** — your screen is **hosted by you**; the client opens your URL
  directly. (See "External mode".)

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
POST /api/mobile/app-catalog/{slug}/launch ────▶ webview/native → temporarySignedRoute()
   {tenant_id, role_id}                          external → your URL (+role JWT)
   ◀──────── LaunchPayload (signed URL | external URL)
webview  → open in WebView
  GET /app/{slug}?u&t&r&expires&sig ────────────▶ EnsureAppWebview + WebviewController
                                                  renders okta_app/webview/screens/<entry>.blade.php
native   → fetch the signed source bundle, compile on device, render (no WebView)
  GET <signed payload URL> ─────────────────────▶ BundleMiniappSource
                                                  reads okta_app/native/<entry>/lib/**.dart
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
    "mode": "webview",                // "webview" | "native" | "external"
    "entry": "okta_app/webview/screens/dashboard.blade.php",   // webview: page okta-web renders.
                                       // native: okta_app/native/main/lib/main.dart
    "minContract": 1,                  // native only: minimum host contract
    "allowedPlatforms": ["ios", "android", "windows", "linux"],  // [] = all
    "allowedRoles": ["tenant-admin"],  // [] = no role filter
    "requiredScope": "education.students.read",       // empty = no scope gate
    "passRoleClaim": true              // external: include a signed role JWT
  }
}
```

okta-web filters cards before returning them: by **platform** (`X-App-Platform`
header), by the active **role** (`allowedRoles`), and by **scope** (`requiredScope`
must be held by the active role). A `mobile.app_catalog.kill_switch` platform
setting and a minimum-app-version gate (`HTTP 426`) can also suppress cards.

Field reference: [`./manifest-reference.md`](./manifest-reference.md).

---

## 2. Develop the entry page (webview mode)

Your entry is a **single server-rendered Blade page** at the `entry` path under
your module's `okta_app/webview/` directory. It is **not** a Livewire page and does **not**
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

## 4. How okta-web serves the webview screen (so you know the guarantees)

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

- requires `mode === 'webview'` (external cards never render here);
- resolves your installed module dir `Modules/<StudlySlug>/`;
- **sandboxes** the manifest `entry` to your `okta_app/webview/` directory — it must start
  with `okta_app/webview/`, contain no `..`, and `realpath` under `<module>/okta_app/webview/`, else 404;
- renders the Blade with view bag `{ user, role, slug }`;
- **injects platform chrome** (typography/base styles) right before `</head>` so
  every webview screen inherits the okta-web look;
- sets `Cache-Control: no-store`, `Content-Security-Policy: frame-ancestors 'none'`,
  `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`.

Takeaways for you: put your entry under `okta_app/webview/`, expect `$user`/`$role`/`$slug`
in the view bag, and don't rely on cookies or framing.

---

## 5. Native mode — a source-on-device Dart mini-app

When `mobile.mode` is `native`, your screen is **real Dart**, not a WebView page.
You author it under `okta_app/native/<entry>/lib/` in your app repo (entry
`okta_app/native/<entry>/lib/main.dart`, a `Widget main()`), pinned to the
platform's `okta_miniapp` runtime.

`<entry>` names a **standalone Dart package** — its own `pubspec.yaml`, its own
`lib/`. The folder is keyed by **entry, not by account type**: several audiences
may point at the same entry, and an app that wants one screen for everyone ships
a single `okta_app/native/main/`. Add a sibling folder only when an audience
genuinely needs a different app. A bundle carries exactly one package: siblings
are never bundled together, so they can't leak into each other.

The flow:

1. **Publish** — okta-web reads every `.dart` file under the named entry's `lib/`
   into one **signed source bundle** (`{package, entry_file, entry_function,
   min_contract, files}`). No compiled artifact ever leaves the repo, so what is
   reviewed is exactly what runs.
2. **On device** — okta-app downloads the bundle, **compiles it on the device**
   (once per published version, then caches the bytecode keyed by
   `(slug, payload_version, runtime signature)`), and renders the widget your
   `main()` returns — with the full platform identity, **no WebView**.

**Host contract** — the only way out of the sandbox is the static `Okta.*` facade
(`package:okta_host`, injected by the runtime): `Okta.context()` (tenant/role/
locale/dark), `Okta.get/getQuery/post` (allow-listed to `/api/<your-slug>/…` and
the scope-gated `/api/apps/*` partner API, tenant auth attached), `Okta.scanBarcode
/scanNfc`, `Okta.uploadFile`, `Okta.toast`. The mini-app gets no `dart_eval`
permissions — no direct network or filesystem.

**Runtime subset** — the device runtime is `dart_eval` + `flutter_eval`, a subset
of Dart/Flutter. A CI compile-check (`flutter test tool/validate.dart`) compiles
your code against the exact device runtime, so a broken widget fails the build,
not the user's phone. Declare `minContract` for host capabilities you depend on;
the app shows "update the app" instead of running a mini-app it is too old for.
The partner boilerplate's `okta_app/native/main/README.md` carries the supported-patterns
catalog (static `Okta.*` only, no `State.mounted`, closure-literal callbacks,
JSON indexed on a `dynamic` receiver — never a `Map`-typed one, an explicit
`flex:` on every `Expanded`/`Flexible`, `ElevatedButton`/`TextButton` only, no
`Wrap` / `AlignmentDirectional` / `Icons.*`, no nested loops, and `ClipRRect`
rather than `BoxDecoration` for rounding or borders).

Note the limit of that compile-check: it catches what the runtime **rejects**,
not what the runtime **discards**. A named argument the bridge does not declare
is dropped silently rather than refused, so a call can pass CI, pass the
analyzer, and still paint the wrong pixels — `BoxDecoration(borderRadius: …)`
is the standing example. Visual review on a device stays part of the gate.

> This replaces the earlier schema/JSON mini-app runtime (`miniapp/`, the
> `miniapp_kit` engine), which has been removed platform-wide.

---

## 6. External mode

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
  [`../../claude/web.md`](../../claude/web.md).

---

## Checklist — a client screen (webview)

- [ ] `mobile` block in the manifest: `supported`, `mode: webview`, `entry` under
      `okta_app/webview/`, plus `allowedPlatforms`/`allowedRoles`/`requiredScope` as needed.
- [ ] Entry Blade mints a Sanctum token, builds a `BOOT` object, renders a
      hash-routed SPA.
- [ ] All data calls go to **your** `/api/<slug>/*` with `Authorization: Bearer` +
      `X-Okta-Tenant-Id`; your context middleware calls `Tenant::makeCurrent()`.
- [ ] Native hardware via `OktaBridge` (guarded by `window.OktaBridge`).
- [ ] Theme from the URL fragment; `okta-app://close` to exit.
- [ ] Verified on each platform you listed in `allowedPlatforms`.
