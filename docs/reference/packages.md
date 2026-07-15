# Packages by platform

An annotated map of the **notable** packages each platform uses and *why* — the
orientation a manifest can't give. It is deliberately **version-free**: the
authoritative, complete list **and the exact versions** always live in each
repo's manifest. Update this file only when a package's *role* changes, not for
version bumps.

| Platform | Source of truth (packages + versions) |
|---|---|
| okta-web | `okta-web/composer.json` + `okta-web/package.json` |
| okta-partners | `okta-partners/composer.json` + `okta-partners/package.json` |
| okta-app | `okta-app/pubspec.yaml` |

---

## okta-web (Laravel + Livewire, PostgreSQL)

**Framework & UI** — `laravel/framework`, `livewire/livewire` (+ `livewire/flux`,
`wire-elements/modal`, `laravelcm/livewire-slide-overs` for modals/slide-overs),
`mhmiton/laravel-modules-livewire` (Livewire ↔ modules), `laravel/tinker`.

**Modules (installed apps)** — `nwidart/laravel-modules`: the modular monolith
that hosts embedded installed apps as code under `Modules/`.

**Multitenancy, RBAC & audit** — `spatie/laravel-multitenancy` (tenant isolation),
`spatie/laravel-permission` (roles/permissions), `spatie/laravel-activitylog`.

**Auth** — `laravel/fortify` (auth + 2FA), `laravel/sanctum` (mobile/API tokens for
`/api/mobile/*`).

**Monitoring & logs** — `laravel/pulse` (with partner-activity cards),
`opcodesio/log-viewer`.

**Export, PDF & data** — `maatwebsite/excel`, `barryvdh/laravel-dompdf`,
`mpdf/mpdf`, `mpdf/qrcode` (export/print); `vinkla/hashids` (opaque public IDs);
`nnjeim/world` (countries/states/cities data).

**AI, SDK & icons** — `prism-php/prism` (the LLM/AI-agent runtime),
`getokta/okta-connect-sdk` (Connect/okta-whatsapp), `afatmustafa/blade-hugeicons`,
`outhebox/blade-flags`.

**Frontend build** — `vite` + `laravel-vite-plugin`, `@tailwindcss/vite` +
`tailwindcss`, `axios`, `autoprefixer`, `concurrently`, `driver.js` (product
tours), `flatpickr` (date picker), `sortablejs` (drag-reorder).

**Dev / testing** — `pestphp/pest` (+ `pest-plugin-laravel`), `laravel/pint`,
`laravel/boost`, `laravel/pail`, `laravel/sail`, `nunomaduro/collision`,
`mockery/mockery`, `fakerphp/faker`.

---

## okta-partners (Laravel + Livewire; standard skeleton, no modules)

**Framework & UI** — `laravel/framework`, `livewire/livewire`,
`wire-elements/modal`, `laravel/tinker`. (Does **not** use
`nwidart/laravel-modules` — a standard Laravel app, not a modular monolith.)

**Partner auth / JWT** — `firebase/php-jwt`: signs/verifies the JWTs partners use
against the API. (okta-web has no such dep — its signatures are hand-rolled HS256.)

**Multitenancy & RBAC** — `spatie/laravel-multitenancy`, `spatie/laravel-permission`.

**Monitoring & logs** — `laravel/pulse`, `opcodesio/log-viewer`.

**AI, SDK & icons** — `prism-php/prism`, `getokta/okta-connect-sdk`,
`afatmustafa/blade-hugeicons`.

**Frontend build** — `vite` + `laravel-vite-plugin`, `@tailwindcss/vite` +
`tailwindcss`, `axios`, `concurrently`.

---

## okta-app (Flutter / Dart client)

**Core** — `flutter`, `flutter_localizations`, `intl` (i18n), `cupertino_icons`.

**State & routing** — `flutter_riverpod` (state/providers), `go_router` (navigation).

**Networking & storage** — `dio` (HTTP client to `okta-web`'s `/api/mobile/*`),
`flutter_secure_storage` (Bearer token + context).

**WebView hosts** — `webview_flutter` (mobile), `webview_windows` +
`desktop_webview_window` (desktop): render installed apps' embedded screens.

**Device & capture** — `permission_handler`, `nfc_manager`, `mobile_scanner`
(device permissions + NFC/barcode capture, e.g. attendance check-in).

**Firebase & push** — `firebase_core`, `firebase_messaging` (FCM push).

**Dev** — `flutter_test`, `flutter_lints`.
