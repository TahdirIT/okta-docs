# Reference — cross-repo map + supporting reference

This section holds the **cross-repo reference** for the integrated `okta` product
(how the repositories fit together) plus supporting reference material. For
per-service and per-domain product docs see [`../`](../README.md); for the
standards the code repos must obey see
[`../tech-standards/`](../tech-standards/README.md).

## Cross-repo reference (the integrated product)

| File | Covers |
|---|---|
| [`architecture.md`](./architecture.md) | The whole picture: workspace tree, dependency map, dual surface, dev→prod. |
| [`repos.md`](./repos.md) | Each repository's responsibility and boundaries. |
| [`web.md`](./web.md) | `okta-web` — services, install mechanics, environments, the surfaces other repos consume. |
| [`partners.md`](./partners.md) | `okta-partners` — authoring, versioning, review/publish, the bridge into `okta-web`. |
| [`app.md`](./app.md) | `okta-app` — the Flutter client that lists/launches a Tenant's apps + student/guardian portals. |
| [`installed-apps.md`](./installed-apps.md) | The installed-app contract (manifest, isolation, dual surface) + the three workspace app repos. |
| [`deployment.md`](./deployment.md) | From authoring to appearing on both surfaces; the dev→prod (sandbox→production) flow. |
| [`glossary.md`](./glossary.md) | Terms (Tenant, scope, installed app, audience, environment, …). |

## Supporting reference

| File | Covers |
|---|---|
| [`packages.md`](./packages.md) | الحزم البارزة لكل منصّة (web / partners / app) ودورها — بلا إصدارات؛ المصدر الفعلي للقائمة والإصدارات هو مانيفست كل مشروع. |
| [`store-listings/`](./store-listings/store-listing.md) | نصوص متجر التطبيقات (App Store / Google Play) لتطبيق `okta-app`. |

> The repo-root map is [`../../CLAUDE.md`](../../CLAUDE.md) — start there; its
> "Read this when…" table routes into this reference and the rest of `docs/`.
