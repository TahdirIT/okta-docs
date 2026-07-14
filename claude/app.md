# okta-app — the client

A **Flutter** cross-platform client (Dart; pubspec name `okta`, described as the
*companion to okta-web*). It is a thin consumer of [`okta-web`](./web.md): the
user logs in, picks an active Tenant + role, and sees the **catalog of
applications that Tenant has installed**, launching each in a WebView. Guardians
and students additionally get **cross-tenant portals** with their own app
catalog. It talks only to `okta-web`'s `/api/mobile/*` API and has **no contact
with [`okta-partners`](./partners.md)**.

Related: [architecture.md](./architecture.md#dual-surface) (the client surface) ·
[installed-apps.md](./installed-apps.md#the-client-surface)

---

## Stack & supported platforms

From `okta-app/pubspec.yaml`:

- **State**: `flutter_riverpod`. **Routing**: `go_router`. **HTTP**: `dio`.
  **Secure storage**: `flutter_secure_storage`. **i18n**: `intl` +
  `flutter_localizations` (Arabic default + RTL, English; font *IBM Plex Sans
  Arabic*).
- **WebView** (the heart of the client): `webview_flutter` (iOS/Android),
  `webview_windows` (Windows, WebView2), `desktop_webview_window` (macOS/Linux
  fallback — opens a separate native window). Plus `permission_handler` (camera/
  mic just-in-time), `nfc_manager` (native NFC bridged into the page — currently
  Android-only; the iOS entitlement is disabled), and `mobile_scanner` (QR).
- **Push**: `firebase_core` + `firebase_messaging` (see *Notifications & push*).

Native shells **present in the repo**: `android/`, `ios/`, `windows/`
(`macos/`, `linux/`, and `web/` shells are not in-repo, though the desktop
WebView packages are wired for them).

```
okta-app/lib/
├── main.dart, app.dart            # ProviderScope + MaterialApp.router
├── router/app_router.dart         # go_router + auth-aware redirect
├── core/
│   ├── api/                       # api_client.dart (Dio), api_provider.dart
│   ├── storage/secure_store.dart  # token, base URL, active context
│   ├── settings/                  # locale, theme, server environment
│   ├── push/push_config.dart      # compile-time Firebase defines
│   ├── theme/ l10n/ widgets/
└── features/                      # feature-first: application/ data/ presentation/
    ├── auth/                      # login, QR sandbox login, tenant/role selection, splash bootstrap
    ├── app_catalog/               # installed-app cards + the WebView host
    ├── portal_app_catalog/        # cross-tenant portal cards (student/guardian)
    ├── portal/                    # student & guardian portal screens
    ├── home/                      # dashboard hosting the catalog section
    ├── notifications/             # in-app feed + push registration
    └── settings/
```

---

## How it talks to okta-web

`core/api/api_client.dart` configures Dio with `baseUrl` from settings and
request interceptors that inject:

- `Authorization: Bearer <token>` (read from secure storage);
- `X-App-Platform` (`ios`/`android`/OS name);
- `X-App-Version` (compile-time `--dart-define=APP_VERSION`; absent = server
  skips the min-version 426 gate);
- `Accept-Language`.

Server environments (`ServerEnvironment` in `core/settings/app_settings.dart`):
**production** `https://getokta.io` (default), **development**
`https://dev.getokta.io`, **local** `http://127.0.0.1:8000` (debug builds only —
compiled out of release to satisfy Play cleartext rules), and **custom**
(HTTPS-only). Dev/local/custom are developer affordances in Settings.

Endpoints consumed (all on `okta-web`):

| Endpoint | Purpose |
|---|---|
| `POST /api/mobile/auth/login` | identifier + password → `{token, user}` (Sanctum). Also the target of QR sandbox login (posted to the scanned host — the client does **not** call a separate `/auth/qr-login`). |
| `GET  /api/mobile/auth/me` | validate session on cold start |
| `GET/POST /api/mobile/auth/context` | list / select `{scope, tenant_id, role_ids}` |
| `GET  /api/mobile/app-catalog` | **installed-app cards** for the active `(tenant, role)` |
| `POST /api/mobile/app-catalog/{slug}/launch` | resolve launch URL (signed embedded URL or external URL + optional JWT) |
| `GET  /api/mobile/app-catalog/portal?portal=student\|guardian` | **portal cards** (cross-tenant, dependent audiences) |
| `POST /api/mobile/app-catalog/portal/{slug}/launch` | launch a portal card for one of the user's tenants |
| `POST /api/mobile/auth/logout` | best-effort logout |
| `POST /api/mobile/notification-tokens` | register this device's FCM token `{token, platform, app_version?, locale?}` |
| `POST /api/mobile/notification-tokens/revoke` | unregister on logout (called **before** the Sanctum token is destroyed) |
| `GET  /api/mobile/notifications` | in-app feed, paginated (`filter=all\|unread\|read`) |
| `GET  /api/mobile/notifications/unread-count` | badge count for the Home bell |
| `POST /api/mobile/notifications/{id}/read` · `/read-all` | mark read (single / all) |

> The client passes `tenant_id`/`role_id` as query parameters on the catalog
> `GET`. See the matching server section in [web.md](./web.md#2-mobile-client-api).

## Notifications & push

`lib/features/notifications/` renders the in-app feed at `/notifications`
(unread accents, optimistic mark-read, mark-all, pagination) and the Home
AppBar shows a bell + live unread badge. Push is an **optional capability**:

- Firebase options resolve from compile-time defines
  (`--dart-define=FIREBASE_{API_KEY,APP_ID,SENDER_ID,PROJECT_ID}`, see
  `lib/core/push/push_config.dart`), falling back to a generated
  `lib/firebase_options.dart`; with neither present the push layer is a silent
  no-op and the feed still works. iOS ships `GoogleService-Info.plist`
  registered in the Xcode project, with APNs entitlements + background mode
  wired in the AppDelegate.
- On login the FCM token is registered into `okta-web`'s **central device
  registry** (`notification_device_tokens`); it re-registers on token
  rotation and is revoked on logout. Installed apps never see device
  tokens — push fan-out happens host-side (see
  [web.md](./web.md#partner-notifications)).
- Foreground pushes render as an **in-app banner** (`OktaBanner`) — there is no
  OS-notification plugin; background/terminated pushes use the OS tray. Tapping
  a push (foreground, background, or cold start) opens `/notifications`.

---

## Auth & session flow

1. **Splash** → `AuthController.bootstrap()`: read token from secure storage; if
   present, `GET /auth/me` to validate; read saved active context.
2. **Login** (`/auth/login`): multi-tab identifier (username / email / phone /
   national id) + password → token stored under `okta.token`. A **QR sandbox
   login** (`/auth/qr-login`, `mobile_scanner`) reads
   `{t:"sandbox_login", base_url, identifier…}` from a QR, logs in against that
   sandbox host, and switches the app to it — a development convenience.
3. **Context** (`/auth/context`, `/auth/role`): pick `scope` (`tenant` or
   `system`), a Tenant, and (if multi-role) an **active role**; stored under
   `okta.context`.
4. **Home** (`/home`): greeting + tenant/role strip + the **app catalog
   section**. Guardian/student users also get the portal routes
   (`/portal/guardian`, `/portal/student`) showing the greeting + **portal app
   cards** (`/api/mobile/app-catalog/portal`). The client does **not** fetch a
   separate cross-tenant "portal data" feed — `features/portal` has no data
   layer beyond the card catalog.

`go_router`'s `redirect` enforces this state machine (`unknown → splash`,
`unauthenticated → login`, `needsContext → context`, `authenticated → home`).
Riverpod wires it together: changing the active context invalidates
`appCatalogProvider`, which refetches the catalog.

---

## Rendering a Tenant's installed applications

`features/app_catalog/` (and its portal sibling `features/portal_app_catalog/`)
is the **client surface** of the dual-surface model:

- `data/app_catalog_models.dart` — `AppCatalogCard { slug, displayName, iconUrl,
  mode ('embedded'|'external'), entry, allowedOrigins, requiredScope,
  passRoleClaim }` and a `LaunchPayload` sealed type (`EmbeddedLaunch` =
  short-lived signed URL on `/app/{slug}`; `ExternalLaunch` = partner URL +
  allowed-origins fence + optional JWT in the URL fragment
  `#okta_role_token=…`).
- `application/app_catalog_provider.dart` — `appCatalogProvider` (a
  `FutureProvider<List<AppCatalogCard>>`) fetches the catalog for the active
  `(tenant, role)`; auto-refetches when context changes.
- `presentation/app_catalog_section.dart` — renders the cards on Home;
  the portal catalogs render on the portal screens.
- `presentation/webview_screen.dart` — the launch target (below).

The client does **not** parse manifests or know about `okta-partners`. It renders
exactly the cards `okta-web` returns from each installed module's `mobile` block
(already filtered by platform / role / scope server-side; audience resolution —
which entry a given role or portal gets — is also server-side).

---

## The WebView host

`webview_screen.dart` picks an implementation per platform at runtime:
`webview_flutter` (iOS/Android), `webview_windows` (Windows), or
`desktop_webview_window` (macOS/Linux, separate window). Shared behavior:

- **Origin allow-list**: navigation is constrained to `allowedOrigins` (+ the
  initial URL) as URL prefixes — the security fence for external pages.
- **`OktaBridge` JS channel** (mobile/web): the hosted page posts JSON messages;
  the app answers. Used to bridge native capabilities the WebView can't reach:
  - **NFC** — native tag read (`nfc_manager`), UID posted back into the page
    (Android; iOS NFC is currently disabled at the entitlement level);
  - **Camera/mic** — just-in-time `permission_handler` prompts when the page asks
    (e.g. a barcode scanner).
- The app injects its **theme** into the page and a **path-aware back button**
  walks up hash/path segments before popping the screen.

For **embedded** cards the WebView loads the signed `/app/{slug}` URL on
`okta-web`; for **external** cards it loads the partner URL with the role JWT in
the fragment (when `passRoleClaim` is set). Either way, the page authenticates to
`okta-web`'s mobile API with the token minted for the session.

---

## Settings screen & design system

- **Settings** (`lib/features/settings/`) exposes exactly three controls:
  **theme** (light / dark / system), **locale** (ar / en), and **server
  environment** (production / development / local, + a custom HTTPS URL field).
  There is **no account management** in Settings (no profile edit, no
  account-deletion — the app has none; see the platform doc set for the web-only
  admin delete tooling).
- **Design system** (`lib/core/widgets/`, `lib/core/theme/`): a shared widget kit
  (`OktaBanner`, `OktaCard`, `OktaHero`, …) + theme tokens and motion, used by
  every screen. Arabic-default, RTL, font *IBM Plex Sans Arabic*.
