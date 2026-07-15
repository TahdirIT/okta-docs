# Core packages in okta-web

A quick reference to the main packages `okta-web` is built on, with the version
constraint each is required at. The real source of truth is always
`okta-web/composer.json` and `okta-web/package.json` — update this file when they
change. Runtime: **PHP `^8.3`**, **Laravel `^13.0`** (framework), **Livewire `^4.0`**,
PostgreSQL.

## PHP packages (composer)

### Framework & UI
- `laravel/framework` `^13.0` — the Laravel framework.
- `livewire/livewire` `^4.0` — interactive server-driven UI.
- `livewire/flux` `^2.9` — the Flux UI component kit.
- `wire-elements/modal` `^3.0` + `laravelcm/livewire-slide-overs` `^1.1` — modals & slide-overs.
- `mhmiton/laravel-modules-livewire` `^7.0` — wires Livewire components to modules.
- `laravel/tinker` `^3.0` — REPL.

### Multitenancy, RBAC & audit
- `spatie/laravel-multitenancy` `^4.1` — multitenancy (Tenants): current-tenant resolution + tenant-aware queries/queues/commands.
- `spatie/laravel-permission` `^6.24` — roles & permissions (RBAC) with guard support and Blade directives.
- `spatie/laravel-activitylog` `^4.11` — activity log.

### Modules (installed apps)
- `nwidart/laravel-modules` `^13.0` — organizes the app into modules (a modular monolith); hosts embedded installed apps as code under `Modules/`.

### Auth
- `laravel/fortify` `^1.30` — authentication and two-factor.
- `laravel/sanctum` `^4.0` — tokens (the mobile app login and the `/api/mobile/*` APIs).

### Monitoring & logs
- `laravel/pulse` `^1.7` — monitoring (with custom cards for partner activity).
- `opcodesio/log-viewer` `^3.24` — in-app log viewer.

### Export, PDF & data
- `maatwebsite/excel` `^3.1`, `barryvdh/laravel-dompdf` `^3.1`, `mpdf/mpdf` `^8.2`,
  `mpdf/qrcode` `^1.2` — export/import and PDF/QR printing.
- `vinkla/hashids` `^14.0` — opaque public IDs (hashids) on CRM/public URLs.
- `nnjeim/world` `^1.1` — countries/states/cities reference data.

### AI, platform SDK & icons
- `prism-php/prism` `^0.100` — the LLM/AI-agent runtime (`ai/agent/*`).
- `getokta/okta-connect-sdk` `^0.6` — the okta-whatsapp / Connect SDK.
- `afatmustafa/blade-hugeicons` `^2.0`, `outhebox/blade-flags` `^1.7` — Blade icon/flag sets.

### Dev / testing
- `pestphp/pest` `^4.3` (+ `pest-plugin-laravel`) — tests; `laravel/pint` `^1.24` — formatting;
  `laravel/boost` `^2.0`, `laravel/pail` `^1.2`, `laravel/sail` `^1.41`,
  `nunomaduro/collision` `^8.6`, `mockery/mockery` `^1.6`, `fakerphp/faker` `^1.23`.

> `firebase/php-jwt` is **not** an okta-web dependency; it is used in `okta-partners`
> (`^7.0`) to sign JWTs. okta-web's own signatures are hand-rolled HS256.

## JavaScript packages (`package.json`)

### Dependencies
- `@tailwindcss/vite` `^4.1` — Tailwind CSS v4 integration with Vite.
- `vite` `^7.0` + `laravel-vite-plugin` `^2.0` — build tool/dev server + Laravel integration.
- `axios` `^1.7` — browser HTTP client.
- `autoprefixer` `^10.4` — adds browser prefixes to CSS rules automatically.
- `concurrently` `^9.0` — runs several dev commands together (`composer run dev`).
- `driver.js` `^1.4` — in-app product tours.
- `flatpickr` `^4.6` — date/time picker.
- `sortablejs` `^1.15` — drag-and-drop (list reordering).

### Dev dependencies
- `tailwindcss` `^4.1` — the core CSS framework for the UI.

### Optional dependencies
- `@rollup/rollup-linux-x64-gnu`, `@tailwindcss/oxide-linux-x64-gnu`,
  `lightningcss-linux-x64-gnu` — native binaries that speed up builds on Linux x64.
