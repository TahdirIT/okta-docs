# Core packages in okta-web

A quick reference to the main packages `okta-web` is built on. The real source of
truth is always `okta-web/composer.json` and `okta-web/package.json` — update this
file when they change.

## PHP packages (composer)

- `spatie/laravel-multitenancy` — multitenancy (Tenants): resolving the current
  tenant, and tenant-aware queries/queues/commands.
- `spatie/laravel-permission` — roles & permissions (RBAC) at the database level,
  with guard support and Blade directives.
- `nwidart/laravel-modules` — organizes the app into modules (a modular monolith);
  it's the mechanism that hosts embedded installed apps as code under `Modules/`.
- `mhmiton/laravel-modules-livewire` — wires Livewire components to modules.
- `livewire/livewire` + `wire-elements/modal` — interactive UIs and modals.
- `laravel/fortify` — authentication and two-factor.
- `laravel/sanctum` — tokens (the mobile app login and the `/api/mobile/*` APIs).
- `spatie/laravel-activitylog` — activity log.
- `laravel/pulse` — monitoring (with custom cards for partner activity).
- `maatwebsite/excel`, `barryvdh/laravel-dompdf`, `mpdf/mpdf` — export, import, and
  PDF printing.
- `firebase/php-jwt` — used in `okta-partners` to sign JWTs (not in okta-web
  itself; okta-web's signatures are hand-rolled HS256).

## JavaScript packages (`package.json`)

### Dependencies

- `@tailwindcss/vite` — Tailwind CSS v4 integration with Vite.
- `autoprefixer` — adds browser prefixes to CSS rules automatically.
- `axios` — browser HTTP client.
- `concurrently` — runs several dev commands together (`composer run dev`).
- `driver.js` — in-app product tours.
- `flatpickr` — date/time picker.
- `laravel-vite-plugin` — Laravel's official Vite integration.
- `sortablejs` — drag-and-drop (list reordering).
- `vite` — build tool and dev server.

### Optional dependencies

- `@rollup/rollup-linux-x64-gnu`, `@tailwindcss/oxide-linux-x64-gnu`,
  `lightningcss-linux-x64-gnu` — native binaries that speed up builds on Linux x64.

### Dev dependencies

- `tailwindcss` — the core CSS framework for the UI.
