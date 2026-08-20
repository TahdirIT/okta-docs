# The golden path — partner keystroke to rendered widget

> Native mini-apps (Dart on device). `okta-miniapp@50e394e`, `okta-app@e49423b`,
> `okta-web@52e48ee`, `okta-partners@608abc3`. Read 2026-08-20.
> Companion files: [`README.md`](./README.md) · [`contracts.md`](./contracts.md) ·
> [`data.md`](./data.md) · [`edges.md`](./edges.md) · [`working-notes.md`](./working-notes.md)

The anchor: **a partner writes `Widget main()` in a `.dart` file, and it renders
as a card inside `okta-app`.** Every hop below is on that one path. Forks are
collected in §Side branches rather than followed.

> **The anchor is a traced case, not a structural constraint.** A mini-app is not
> one file: the **whole `lib/` tree** is bundled (128 files max — see Hop 7), and
> your files import each other as `package:<your_package_name>/…`. `entryFile`
> only names which file holds `main`.

---

## Hop 0 — The partner repo is scaffolded

`okta-partners` provisions a new partner repository and copies a skeleton into
it verbatim, including dot-directories.

- `okta-partners/app/Services/GitHubAppService.php` → `pushBoilerplate()`
  copies `resources/partner-boilerplate/**`.
- The native package lands at
  `resources/partner-boilerplate/okta_app/native/main/` — `lib/main.dart`,
  `pubspec.yaml`, `analysis_options.yaml`, `tool/validate.dart`, `README.md`.
- Template placeholders are substituted in
  `okta-partners/app/Services/GitHubAppService.php:386`, where
  `__MINIAPP_REF__` ← `config('okta-web.miniapp.runtime_ref')`
  (`okta-partners/config/okta-web.php:107`, env `OKTA_MINIAPP_RUNTIME_REF`).

The scaffolded `pubspec.yaml` pins `okta_miniapp` **by git commit ref**, not by
version range — deliberately, because the partner must compile against the exact
runtime the shipped `okta-app` binary carries. `[confirmed]`
`okta-partners/resources/partner-boilerplate/okta_app/native/main/pubspec.yaml:18-22`

> ⚠️ That pin is currently stale. See `working-notes.md` §Finding.

## Hop 1 — The partner writes Dart

`okta_app/native/<entry>/lib/main.dart`, exporting a **top-level `Widget main()`**.
`<entry>` names a standalone Dart package inside the partner repo; one slug may
ship several. `[confirmed]`
`okta-web/app/Services/MobileAppCatalog/BundleMiniappSource.php:20-27`

The only door out of the sandbox is the `Okta.*` static facade from
`package:okta_host`. That package is **not a real pub package** — it is injected
as source by the runtime at compile time, which is why `flutter analyze` reports
it as undefined and why that is expected rather than a misconfiguration.
`[confirmed]` `okta-partners/resources/partner-boilerplate/okta_app/native/main/README.md`
(“Develop & validate” section)

## Hop 2 — The partner validates locally (the CI gate)

```bash
cd okta_app/native/main
flutter pub get
flutter test tool/validate.dart
```

`tool/validate.dart` reads every `.dart` under `lib/` into an
`OktaMiniAppBundle` and runs `OktaMiniAppValidator().validate(bundle)`.
`[confirmed]` `okta-partners/resources/partner-boilerplate/okta_app/native/main/tool/validate.dart:44-58`

The validator builds the **same** compiler the device builds —
`flutterEvalPlugin` + `OktaHostPlugin.compileOnly()` — and reports success or
the compiler's own error text, without emitting an artifact. `[confirmed]`
`okta-miniapp/lib/src/validation/okta_mini_app_validator.dart:44-50`

**This is the load-bearing guarantee of the whole native path:** because the
same `okta_miniapp` commit performs the CI compile and the device compile,
“compiles in CI” deterministically implies “compiles on device”. `[confirmed]`
`okta-miniapp/lib/src/validation/okta_mini_app_validator.dart:31-36`

It is a *compile* gate only. Three whole classes of failure pass it clean —
silently-dropped named arguments, declared-but-unwired classes, and boxing
faults that only fire at runtime. See `working-notes.md` §Traps.

## Hop 3 — The partner declares the surface

On `okta-partners`, per **version** (there is no module-level value), the
partner sets: `mobile_support` on, `mode: native`, `runtime: dart`, the entry
path per audience, `min_contract`, and any device capabilities with a written
justification each.

Three write paths reach the same fields and must stay consistent — the Livewire
version editor, the manifest importer, and the hosted-MCP tool:

- `okta-partners/app/Livewire/Partner/Modules/VersionEditor.php`
- `okta-partners/app/Services/Partners/Manifest/PersistImportedSurfaces.php`
- `okta-partners/app/Services/PartnerMcp/Server/Tools/ControlTools.php:114`
  (`set_mobile_surface`) and `:1404` onward.

The entry-path shape is defined once, in
`okta-partners/app/Services/Partners/Manifest/MobileEntryRules.php:36`:

```
PATTERN_NATIVE = #^okta_app/native/[A-Za-z0-9_-]+/lib/[A-Za-z0-9._/-]+\.dart$#
```

`fits()` also rejects any entry containing `..` (`:83-86`) and bounds the length
to 255 (`:39`).

`min_contract` and `capabilities` are **native-only**: switching the mode to
anything else nulls both, in the editor and in the MCP tool alike. `[confirmed]`
`okta-partners/app/Services/PartnerMcp/Server/Tools/ControlTools.php:1569-1577`

## Hop 4 — Publish: okta-web validates the manifest

`okta-web/app/Modules/Core/ManifestValidator.php`:

| Rule | Line |
|---|---|
| `mobile.mode` ∈ `webview,external,native` | `:446`, `:487` |
| `mobile.minContract` integer ≥ 1 | `:453`, `:490` |
| Native entry must match `okta_app/native/<entry>/lib/**.dart`, no `..` | `:817-821` |
| Device capabilities → `validateDeviceCapabilities()` | `:856`, `:912` |
| Capabilities require **some** native surface | `:923-941` |
| Capabilities require `minContract >= DeviceCapability::GATE_CONTRACT` (13) | `:949-958` |
| Student-profile tab entry is also native-only, `minContract >= 17` | `:1000`, `:1024-1041`, `:1120-1125` |

The capability floor exists because the gate itself did not exist before
contract 13 — a device below the floor would run the mini-app on a host that
ignores the declared consent entirely. `[confirmed]` `:956-958`

## Hop 5 — Install: the manifest lands on the tenant's instance

`PartnerSandboxModuleInstaller` writes the manifest onto the local `modules`
row on every install, and the launch endpoint reads **that** row rather than the
partners-catalog mirror — so it tracks the partner's latest config without
depending on a sync. `[confirmed]`
`okta-web/app/Http/Controllers/Api/Mobile/MobileAppCatalogController.php:157-165`

## Hop 6 — The device asks for a launch

`okta-app` calls, with a Sanctum bearer token:

```
POST /api/mobile/app-catalog/{slug}/launch      { tenant_id, role_id }
POST /api/mobile/app-catalog/portal/{slug}/launch  { tenant_id, portal }
```

`okta-web/routes/api.php:119` and `:126`, both behind
`['auth:sanctum', 'mobile.min-version']` (`:117`).

The controller:
1. loads the active `Module` by slug, 404s unless `mobile.supported === true`
   (`:162-174`);
2. sets the spatie permissions team id to the tenant **before** hydrating the
   role, so permissions read in the right tenant scope (`:178-179`);
3. picks the audience for that role via `NormalizeMobileAudiences::pickForRole`
   (`:185`) — or `pickForPortal` on the portal route (`:125`);
4. hands `IssueWebviewLaunch` a resolved block carrying that audience's
   mode + entry (`:190-206`).

`IssueWebviewLaunch` branches on mode at
`okta-web/app/Services/MobileAppCatalog/IssueWebviewLaunch.php:51`; the native
branch (`:173-200`) returns a **short-lived signed URL** to the payload route
plus `payload_version`, derived from the module's `updated_at` timestamp
(`:200`).

> **This is why a sandbox loop needs no version bump.** A sandbox install
> re-resolves the branch's commit, so `github_commit_sha` on the `modules` row
> changes, `$module->update()` is dirty, and `updated_at` moves — which moves
> `payload_version` and misses the device cache. Reinstalling *without* pushing
> leaves every column identical, nothing is dirty, and the timestamp stays put.
> `[confirmed]` `PartnerSandboxModuleInstaller.php:77-87`,
> `okta-partners/app/Services/PartnerModules/Sandbox/InstallVersionOnSandbox.php:140-148`

The two launch paths differ in a way worth naming: the portal launch takes
`{tenant_id, portal}` and **no role**, and the tenant comes from the card rather
than the session — a portal aggregates schools, so the session has no tenant to
read. `[confirmed]`
`okta-app/lib/features/miniapps/bridge/okta_dart_miniapp_host.dart:117-122`

## Hop 7 — The device downloads the source bundle

```
GET /app/{slug}/miniapp-payload
```

`okta-web/routes/app.php:45`, behind
`['web', 'signed', 'app.webview', 'throttle:30,1']` (`:33`).

The signature **is** the authorization; `app.webview` has already rehydrated the
user and re-confirmed the active role holds the card's required scope, so
`MiniappPayloadController` only assembles. `[confirmed]`
`okta-web/app/Http/Controllers/App/MiniappPayloadController.php:12-22`

It refuses anything whose resolved mode is not `native` (`:29-31`), then calls
`BundleMiniappSource` (`:36-41`), and returns the JSON with
`Cache-Control: no-store, private` (`:44-45`).

### `BundleMiniappSource` — what actually gets read

`okta-web/app/Services/MobileAppCatalog/BundleMiniappSource.php`

1. Re-validates the entry against the native pattern and rejects `..` (`:49-55`)
   — a second enforcement, independent of the publish gate.
2. Resolves `Modules/<StudlySlug>/okta_app/native/<entry>/lib` and `realpath`s it
   (`:60-70`).
3. Walks it recursively, taking `.dart` files only, and **fences every file**:
   any path that does not `realpath` back under that `lib/` is skipped, which is
   what stops a symlink escaping (`:86-90`).
4. Enforces three limits — 256 KB per file, 2 MB per bundle, 128 files
   (`:32-36`, `:91-97`, `:101-104`).
5. Requires the named entry file to be present (`:109-111`).
6. Reads the Dart package name out of the package's own `pubspec.yaml`, falling
   back to `mini_app` (`:174-188`).
7. Normalises declared capabilities, dropping unknown names and blank reasons
   (`:127-168`).

**Sibling entries are not bundled.** Capturing `<entry>` in step 1 is what keeps
one package's bundle from carrying another's source. `[confirmed]` `:26-29`

## Hop 8 — The device compiles and renders

`okta-app/lib/features/miniapps/bridge/okta_dart_miniapp_host.dart`

- `oktaMiniAppCacheProvider` (`:64-71`) opens a persistent bytecode cache under
  the app-support directory.
- `oktaDartBundleProvider` (`:74-95`) is **family-keyed by target**, so a portal
  card and a role card for the same slug never share a fetch.
- `OktaDartMiniApp.build` (`:250-296`) resolves cache then bundle, then calls
  `OktaMiniApp.fromBundle(...)` with
  `plugins: [OktaHostPlugin(ref.read(oktaHostDelegateProvider(target)))]`
  (`:263-265`).

Inside `okta-miniapp`:

- `OktaMiniAppBundle.requireSupportedBy` throws `OktaMiniAppContractException`
  when `minContract > hostContract` — **before anything compiles**
  (`okta-miniapp/lib/src/bundle/okta_mini_app_bundle.dart:154-163`).
- The compiler adds `flutterEvalPlugin` plus `OktaHostPlugin`, which injects the
  four libraries in `oktaMiniAppInjectedSources`
  (`okta-miniapp/lib/src/injected_sources.dart:41-46`).
- `OktaHostPlugin.configureForRuntime` registers every bridge function
  (`okta-miniapp/lib/src/host/okta_host_plugin.dart:79-144`), and chains
  `OktaMotionPlugin` and `OktaGlassPlugin` (`:142-143`).
- The program runs; `runtime.executeLib(lib, 'main')` returns a `$Value` whose
  `.$value` is the root `Widget`.

Bytecode is cached under `(slug, entry, payloadVersion, runtimeSignature)` —
see `data.md` §The cache key.

---

## Side branches (noted, not followed)

| Fork | Where | Note |
|---|---|---|
| `mode: webview` | `IssueWebviewLaunch.php:51` | Files under `okta_app/webview/`, rendered in a sandboxed WebView. Cannot declare device capabilities. |
| `mode: external` | same | An https page the partner hosts. Same capability exclusion. |
| Portal launch | `MobileAppCatalogController::launchPortal:104` | Student/guardian; no role, tenant from the card. Traced only far enough to name the difference. |
| Student-profile tab | `ManifestValidator.php:995-1055` | Contract 17. A mini-app mounted on a student; `Okta.studentHashid()` is non-empty only here. |
| `OktaMotion` / `OktaGlass` | `okta-miniapp/lib/src/motion/`, `lib/src/glass/` | Host-rendered animation and material, because the sandbox has neither. Bridged the same way as `okta_host`. |
| Tool-key readers | `okta-app/lib/features/toolkeys/` | Contract 12. Poll-drain shape; see `edges.md`. |
| Compile cache eviction on render failure | `okta_dart_miniapp_host.dart:281-295` | Covered in `edges.md`. |
