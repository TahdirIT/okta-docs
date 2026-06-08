# okta-app — the client

A **Flutter** cross-platform client (Dart). It is a thin consumer of
[`okta-web`](./web.md): the user logs in, picks an active Tenant + role, and sees
the **catalog of applications that Tenant has installed**, launching each in a
WebView. It talks only to `okta-web`'s `/api/mobile/*` API and has **no contact
with [`okta-partners`](./partners.md)**.

Related: [architecture.md](./architecture.md#dual-surface) (the client surface) ·
[installed-apps.md](./installed-apps.md#the-client-surface)

---

## Stack & supported platforms

From `okta-app/pubspec.yaml`:

- **State**: `flutter_riverpod`. **Routing**: `go_router`. **HTTP**: `dio`.
  **Secure storage**: `flutter_secure_storage`. **i18n**: `intl` +
  `flutter_localizations` (Arabic default, English; font *IBM Plex Sans Arabic*).
- **WebView** (the heart of the client): `webview_flutter` (iOS/Android),
  `webview_windows` (Windows, WebView2), `desktop_webview_window` (macOS/Linux
  fallback — opens a separate native window). Plus `permission_handler` (camera/
  mic just-in-time) and `nfc_manager` (native NFC bridged into the page).

Native shells **present in the repo**: `android/`, `ios/`, `windows/`. The
pubspec declares the product as a multi-platform companion; `macos/`, `linux/`,
and `web/` shells are **not** present in-repo, though the WebView packages for
desktop are wired for future use. `> TODO: confirm` whether macOS/Linux are
shipped targets or aspirational.

```
okta-app/lib/
├── main.dart, app.dart            # ProviderScope + MaterialApp.router
├── router/app_router.dart         # go_router + auth-aware redirect
├── core/
│   ├── api/                       # api_client.dart (Dio), api_provider.dart
│   ├── storage/secure_store.dart  # token, base URL, active context
│   ├── settings/                  # locale, theme, server environment
│   ├── theme/ l10n/ widgets/
└── features/
    ├── auth/                      # login, tenant/role selection, splash bootstrap
    ├── app_catalog/               # installed-app cards + the WebView host
    ├── home/                      # dashboard hosting the catalog section
    └── settings/
```

---

## How it talks to okta-web

`core/api/api_client.dart` configures Dio with `baseUrl` from settings (default
`ServerEnvironment.production` = `https://getokta.io`) and request interceptors
that inject:

- `Authorization: Bearer <token>` (read from secure storage);
- `X-App-Platform` (`ios`/`android`/OS name);
- `X-App-Version` (compile-time `--dart-define=APP_VERSION`; absent = server skips
  the version gate).

Endpoints consumed (all on `okta-web`):

| Endpoint | Purpose |
|---|---|
| `POST /api/mobile/auth/login` | identifier + password → `{token, user}` |
| `GET  /api/mobile/auth/me` | validate session on cold start |
| `GET/POST /api/mobile/auth/context` | list / select `{scope, tenant_id, role_ids}` |
| `GET  /api/mobile/app-catalog` | **installed-app cards** for the active `(tenant, role)` |
| `POST /api/mobile/app-catalog/{slug}/launch` | resolve launch URL (signed embedded URL or external URL + optional JWT) |
| `POST /api/mobile/auth/logout` | best-effort logout |

> The client passes `tenant_id`/`role_id` as query parameters on the catalog
> `GET`. See the matching server note in [web.md](./web.md#2-mobile-client-api).

---

## Auth & session flow

1. **Splash** → `AuthController.bootstrap()`: read token from secure storage; if
   present, `GET /auth/me` to validate; read saved active context.
2. **Login** (`/auth/login`): multi-tab identifier (username / email / phone /
   national id) + password → token stored under `okta.token`.
3. **Context** (`/auth/context`): pick `scope` (`tenant` or `system`), a Tenant,
   and (if multi-role) an **active role**; stored under `okta.context`.
4. **Home** (`/home`): greeting + tenant/role strip + the **app catalog
   section**.

`go_router`'s `redirect` enforces this state machine (`unknown → splash`,
`unauthenticated → login`, `needsContext → context`, `authenticated → home`).
Riverpod wires it together: changing the active context invalidates
`appCatalogProvider`, which refetches the catalog.

---

## Rendering a Tenant's installed applications

`features/app_catalog/` is the **client surface** of the dual-surface model:

- `data/app_catalog_models.dart` — `AppCatalogCard { slug, displayName, iconUrl,
  mode ('embedded'|'external'), entry, allowedOrigins, requiredScope,
  passRoleClaim }` and a `LaunchPayload` sealed type (`EmbeddedLaunch` =
  short-lived signed URL on `/app/{slug}`; `ExternalLaunch` = partner URL +
  allowed-origins fence + optional JWT in the URL fragment
  `#okta_role_token=…`).
- `application/app_catalog_provider.dart` — `appCatalogProvider` (a
  `FutureProvider<List<AppCatalogCard>>`) fetches the catalog for the active
  `(tenant, role)`; auto-refetches when context changes.
- `presentation/app_catalog_section.dart` — renders the cards on Home.
- `presentation/webview_screen.dart` — the launch target (below).

The client does **not** parse manifests or know about `okta-partners`. It renders
exactly the cards `okta-web` returns from each installed module's `mobile` block
(already filtered by platform / role / scope server-side).

---

## The WebView host

`webview_screen.dart` picks an implementation per platform at runtime:
`webview_flutter` (iOS/Android), `webview_windows` (Windows), or
`desktop_webview_window` (macOS/Linux, separate window). Shared behavior:

- **Origin allow-list**: navigation is constrained to `allowedOrigins` (+ the
  initial URL) as URL prefixes — the security fence for external pages.
- **`OktaBridge` JS channel** (mobile/web): the hosted page posts JSON messages;
  the app answers. Used to bridge native capabilities the WebView can't reach:
  - **NFC** — native tag read (`nfc_manager`), UID posted back into the page;
  - **Camera/mic** — just-in-time `permission_handler` prompts when the page asks
    (e.g. a barcode scanner).
- **Path-aware back button** walks up hash/path segments before popping the
  screen.

For **embedded** cards the WebView loads the signed `/app/{slug}` URL on
`okta-web`; for **external** cards it loads the partner URL with the role JWT in
the fragment (when `passRoleClaim` is set). Either way, the page authenticates to
`okta-web`'s mobile API with the token minted for the session.
