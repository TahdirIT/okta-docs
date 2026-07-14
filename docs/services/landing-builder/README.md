# Landing Page Builder

> **توجيه (عربي):** هذا توثيق **منشئ صفحة الهبوط** — محرّر مرئي يمنح كل جهة موقعاً
> عاماً على نطاق فرعي تلقائي أو نطاق مخصّص مُوثَّق، من مكتبة بلوكات قابلة للتحرير.
> يُحفَظ هذا المجلد **بالإنجليزية** عمداً لأنه موجَّه للمطوّرين (كما
> [`apps/development/`](../../apps/development/README.md))؛ بقية توثيق المنتج عربي.

Visual landing-page editor for okta-web tenants. Each tenant gets a public
website rendered on either an auto-subdomain (`{slug}.{platform-base-domain}`)
or a verified custom domain, built from a library of block components and
edited inline in a three-column visual editor.

## Contents

1. [Architecture](./architecture.md) — data model, request flow, tenant resolution
2. [Admin guide](./admin-guide.md) — how tenant admins edit and publish
3. [Domain setup](./domain-setup.md) — adding a custom domain, DNS, TLS
4. [Block development](./block-development.md) — adding a new block to the library
5. [Super admin guide](./super-admin-guide.md) — advanced blocks and overrides
6. [API reference](./api-reference.md) — routes, events, interfaces

## Stack

- **Laravel 12** + **Livewire 4** + **Tailwind v4**
- **Alpine.js** + **SortableJS** for drag & drop
- **spatie/laravel-multitenancy** (single-DB) with a composite tenant finder
- **spatie/laravel-permission** (dot-style names, scope = `system`/`tenant`)
- No Flux components inside the Landing Builder — the rest of the admin shell
  keeps using Flux as before.

## Tables

| Table | Purpose |
|-------|---------|
| `tenant_landing_pages` | Single row per tenant. `draft_blocks` / `published_blocks` as JSON. |
| `tenant_domains` | Both auto-subdomains and verified custom domains. |
| `tenant_brand_settings` | Logos, color palette, fonts. |
| `platform_settings` | Holds `landing.base_domain` globally. |

## Permissions (scope = tenant unless noted)

- `landing.edit` — edit draft blocks & theme
- `landing.publish` — publish / unpublish
- `landing.manage_domains` — add/verify/remove custom domains
- `landing.advanced_blocks` — **system scope**, gates Custom HTML / Script / Embed

## Quick links

- Editor route: `GET /settings/landing-page`
- Domains route: `GET /settings/domains`
- Caddy ask endpoint: `GET /_caddy/ask?domain=<host>`
- Public landing: `GET /` on the tenant host
