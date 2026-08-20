# Boundary contracts — what crosses, in what shape, under whose authority

> Native mini-apps (Dart on device). `okta-miniapp@50e394e`, `okta-app@e49423b`,
> `okta-web@52e48ee`, `okta-partners@608abc3`. Read 2026-08-20.
> Companion files: [`README.md`](./README.md) · [`flow.md`](./flow.md) ·
> [`data.md`](./data.md) · [`edges.md`](./edges.md) · [`working-notes.md`](./working-notes.md)

Five boundaries matter on the native path. Three are network hops; two are
in-process seams that behave like network hops because a version mismatch
across them fails on a device rather than in CI.

---

## B1 · okta-app → okta-web · launch

| | |
|---|---|
| **Route** | `POST /api/mobile/app-catalog/{slug}/launch` — `okta-web/routes/api.php:119` |
| **Portal twin** | `POST /api/mobile/app-catalog/portal/{slug}/launch` — `:126` |
| **Auth** | Sanctum bearer + `mobile.min-version` middleware — `:117` |
| **Request** | `{tenant_id:int, role_id:int}`; portal: `{tenant_id:int, portal:'student'\|'guardian'}` — `MobileAppCatalogController.php:152-155`, `:106-109` |
| **Response (native)** | `{mode:'native', url:<signed>, payload_version:<int>}` — `IssueWebviewLaunch.php:139-143`, `:196-200` |
| **Sync/async** | Synchronous |
| **Failure** | 404 when the module is inactive, `mobile.supported !== true`, or no audience matches the role — `:167-188`. 426 from `mobile.min-version` for a too-old client — `routes/api.php:113-115` |

**`payload_version` is derived from the module row's `updated_at` timestamp**
(`IssueWebviewLaunch.php:143`, `:200`), not from a partner-authored version
string. It is the cache-key input on the device. `[confirmed]`

## B2 · okta-app → okta-web · source bundle

| | |
|---|---|
| **Route** | `GET /app/{slug}/miniapp-payload` — `okta-web/routes/app.php:45` |
| **Auth** | The **URL signature is the authorization**. `['web','signed','app.webview','throttle:30,1']` — `routes/app.php:33`. `app.webview` rehydrates the user and reconfirms the role holds the card's required scope, so the controller only assembles — `MiniappPayloadController.php:12-22` |
| **Response** | JSON, `Cache-Control: no-store, private` — `:44-45` |
| **Failure** | 404 unless resolved mode is `native` (`:29-31`); 404 from `BundleMiniappSource` on a bad entry, a missing `lib/`, a missing entry file, or any limit breach |

### The payload shape, and a name drift worth knowing

Server emits — `BundleMiniappSource.php:114-121`:

```json
{ "package": "...", "entry_file": "main.dart", "entry_function": "main",
  "min_contract": 13, "capabilities": [{"key":"...","reason":"..."}],
  "files": { "main.dart": "<source>", "widgets/card.dart": "<source>" } }
```

Client parses — `okta-miniapp/lib/src/bundle/okta_mini_app_bundle.dart:176-191`.
Same snake_case keys, mapped onto camelCase Dart fields (`entry_file` →
`entryFile`, `min_contract` → `minContract`).

**The two fields the server does not send.** `slug` and `payload_version` are
written into the map **client-side, from the launch response**, immediately
before parsing — `okta-app/lib/features/miniapps/bridge/okta_dart_miniapp_host.dart:160-162`:

```dart
map['slug'] = slug;
map['payload_version'] = '${launch.payloadVersion}';
```

This is deliberate: both are cache-key inputs, and taking them from the signed
launch rather than the payload body makes the key authoritative regardless of
what the body says. `[confirmed]`
`okta_dart_miniapp_host.dart:97-100`

`payload_version` is an **int** on the wire from B1 and a **String** on the
bundle (`'${json['payload_version']}'`, `:189`). Interpolated, not cast — so a
change of server type would not throw here. `[confirmed]`

## B3 · okta-web publish gate ← okta-partners manifest

| | |
|---|---|
| **Shape** | The `mobile` block of the module manifest: `supported`, `mode`, `runtime`, `minContract`, `entry` (per audience), `capabilities[]` |
| **Producer** | `okta-partners` — `VersionEditor.php`, `PersistImportedSurfaces.php`, MCP `set_mobile_surface` (`ControlTools.php:114`) |
| **Consumer** | `okta-web/app/Modules/Core/ManifestValidator.php:446-490`, `:817-821`, `:912-958` |
| **Entry-shape authority** | `okta-partners/app/Services/Partners/Manifest/MobileEntryRules.php:36` — and re-implemented as an inline regex in `ManifestValidator.php:818` and again in `BundleMiniappSource.php:49` |

Three copies of the native entry regex exist, in two repos:

```
okta-partners  MobileEntryRules.php:36        ^okta_app/native/[A-Za-z0-9_-]+/lib/[A-Za-z0-9._/-]+\.dart$
okta-web       ManifestValidator.php:818      ^okta_app/native/[A-Za-z0-9_-]+/lib/
okta-web       BundleMiniappSource.php:50     ^okta_app/native/([A-Za-z0-9_-]+)/lib/
```

The okta-web pair is a **prefix** check plus a separate `str_ends_with('.dart')`
and `str_contains('..')`, so they are equivalent in effect rather than
identical in text. `[confirmed]` `ManifestValidator.php:817-821`,
`BundleMiniappSource.php:49-55`. The third is not redundancy for its own sake —
`BundleMiniappSource` also serves modules stored before the publish gate existed.

## B4 · The sandbox seam · mini-app ↔ host

The only boundary the partner's code actually sees.

| | |
|---|---|
| **Facade** | `package:okta_host/okta_host.dart` → the `Okta` static class — `okta-miniapp/lib/src/host/okta_host_source.dart:74` |
| **Transport** | `registerBridgeFunc` bindings, all `$host*`-prefixed — `okta_host_plugin.dart:79-141` |
| **Wire format** | **JSON strings in both directions.** `_context` returns `$String(jsonEncode(...))` (`:152-154`); `_apiCall` decodes `args[0]` as JSON and re-encodes the response (`:156-172`) |
| **Async** | `$Future.wrap(...)` — `:171`, `:215` |
| **Direction** | **Sandbox → host only.** There is no registered path for the host to call in. |

Three consequences that shape every capability:

1. **A continuous reader cannot push.** `Okta.toolKeyReads()` is a *drain*: the
   host buffers, the mini-app comes and takes everything since the last call.
   This is forced by the one-directional binding, not chosen. `[confirmed]`
   `okta-miniapp/CLAUDE.md` §contract 12, and the absence of any host→sandbox
   registration in `okta_host_plugin.dart:79-144`.
2. **Nothing typed crosses.** An object built inside a mini-app is an
   interpreted value, not a native instance. Everything crossing is JSON.
3. **Refusals are ordinary return values, never throws.** See the table in
   `edges.md` §The refusal vocabulary.

## B5 · The version seam · okta-app binary ↔ partner package

Not a network hop, and the most expensive boundary to get wrong.

`okta_kit`, `okta_host`, `okta_glass` and `okta_motion` are **compiled into the
okta-app binary**, not into the partner's bundle
(`okta-miniapp/lib/src/injected_sources.dart:41-46`). So:

- A fix to an injected library reaches a user only when a **new okta-app build**
  is installed. Republishing the mini-app does nothing. `[confirmed]`
  `okta-miniapp/CLAUDE.md` §"A fix in an INJECTED library ships with okta-app"
- The partner compiles against a **pinned commit** of `okta_miniapp`
  (`pubspec.yaml:18-22`), which must match the runtime the shipped binary
  carries — otherwise CI's guarantee is void in the one direction that matters.

**Two numbers, two different jobs, routinely confused:**

| | `oktaHostContractVersion` | `oktaMiniAppRuntimeSignature` |
|---|---|---|
| Defined | `okta-miniapp/lib/src/host/okta_host_delegate.dart:228` (= **17**) | `okta-miniapp/lib/src/runtime_info.dart:28` |
| Is | An integer, hand-bumped | Toolchain string + digest of every injected source |
| Gates | **Admission.** `minContract <= hostContract` or the app refuses to run — `okta_mini_app_bundle.dart:154-163` | **Nothing.** It keys the compile cache only |
| Moves when | Someone bumps it | Any injected source byte changes |

`oktaMiniAppRuntimeSignature` **does not gate admission**, and relying on it to
is the documented way this has already gone wrong: `OktaTabs.bottomBar` landed
four days after contract 7 shipped without a bump, okta-hdor declared
`minContract: 7`, every contract-7 device passed the gate, and then died with
`CompileError: Cannot find static method OktaTabs.bottomBar`. `[confirmed]`
`okta-miniapp/lib/src/host/okta_host_delegate.dart:50-57`

Both numbers are re-exported by okta-app for error reports, because the contract
alone does not identify a build — two hosts can both say 17 while only one
carries a fix. `[confirmed]`
`okta_dart_miniapp_host.dart:47`, `:60`

---

## Where names drift across the seams

| Concept | okta-partners | okta-web | wire | okta-miniapp / okta-app |
|---|---|---|---|---|
| Contract floor | `min_contract` (`mobile_config`) | `mobile.minContract` (manifest), `min_contract` (payload) | `min_contract` | `minContract` |
| Entry path | `mobile_entry` (per audience) | `entry` (resolved block) | — (resolved server-side) | `entryFile` (basename-relative to `lib/`) |
| Package | — | `package` (read from partner `pubspec.yaml`) | `package` | `package` |
| Version | — | `payload_version` (from `updated_at`) | `payload_version` (int) | `payloadVersion` (**String**) |
| Capability | `capabilities[].{key,reason}` | same | same | `OktaDeclaredCapability{key,reason}` → gateway type `MiniAppDeclaredCapability` |

The last row crosses one extra seam **inside** okta-app: `OktaDeclaredCapability`
may not be named outside `lib/features/miniapps/bridge/`, so it is translated
once into a plain gateway type that the consent sheet, permissions screen and
catalog card all speak. `[confirmed]`
`okta_dart_miniapp_host.dart:168-176`. That confinement is enforced
mechanically — see `edges.md` §The import boundary.
