# Data, vocabulary and configuration

> Native mini-apps (Dart on device). `okta-miniapp@50e394e`, `okta-app@e49423b`,
> `okta-web@52e48ee`, `okta-partners@608abc3`. Read 2026-08-20.
> Companion files: [`README.md`](./README.md) · [`flow.md`](./flow.md) ·
> [`contracts.md`](./contracts.md) · [`edges.md`](./edges.md) · [`working-notes.md`](./working-notes.md)

Nothing on the native path is stored in a mini-app-specific database table. The
data is: source on a partner's disk, a manifest column, a JSON payload in
flight, bytecode in a device cache, and a consent file. Each is owned by exactly
one repo.

---

## Where each piece lives

| Piece | Owner | Location | Lifetime |
|---|---|---|---|
| Dart source | partner repo | `okta_app/native/<entry>/lib/**.dart` | git |
| Surface declaration | okta-partners | `mobile_config` JSON on `partner_module_versions`, + `mobile_entry` per audience | per version |
| Installed manifest | okta-web | `modules` row, read via `Module::decodedManifest()` | per install, rewritten every install |
| Source bundle | — | in flight only; `no-store, private` | one request |
| Compiled bytecode | okta-app | app-support dir, `okta_miniapp_cache/` | until version or runtime signature moves |
| Consent decisions | okta-app | a JSON file per (user, slug) | until app uninstall |

**Device capabilities travel in the payload, not on the catalog card.** The
consumer is the runtime's capability gate, which exists only once the payload is
running; the card only decides whether to draw a tile. `[confirmed]`
`okta-web/app/Services/MobileAppCatalog/BundleMiniappSource.php:38-45`,
`okta-web/app/Http/Controllers/App/MiniappPayloadController.php:33-36`

## The bundle

`okta-miniapp/lib/src/bundle/okta_mini_app_bundle.dart:65`

| Field | Wire key | Default | Source |
|---|---|---|---|
| `slug` | `slug` | — | **injected client-side from the launch** |
| `package` | `package` | `mini_app` | partner's `pubspec.yaml` `name:` |
| `files` | `files` | — | every `.dart` under the entry's `lib/`, keyed relative to `lib/` |
| `entryFile` | `entry_file` | `main.dart` | manifest entry, minus the `okta_app/native/<entry>/lib/` prefix |
| `entryFunction` | `entry_function` | `main` | hard-coded server-side |
| `minContract` | `min_contract` | `1` | manifest |
| `capabilities` | `capabilities` | `[]` | manifest, normalised twice |
| `payloadVersion` | `payload_version` | — | **injected client-side from the launch** |

Limits enforced at assembly — `BundleMiniappSource.php:32-36`:
**256 KB** per file, **2 MB** per bundle, **128** files.

### Multiple files, and the import form

The whole `lib/` tree is walked recursively (`BundleMiniappSource.php:80-82`) and
handed to the compiler as **one** Dart package (`packages: {package: files}` —
`okta_mini_app_bundle.dart:170-174`). So your files import each other by
`package:` plus the package name from your `pubspec.yaml`:

```dart
// lib/main.dart          (package: 'demo_app')
import 'package:demo_app/widget.dart';          // a sibling file
import 'package:demo_app/widgets/card.dart';    // lib/widgets/card.dart
```

Live tested example: `okta-miniapp/test/okta_mini_app_bundle_test.dart:12-30`.
Nested folders work — the path relative to `lib/` is kept as the key
(`BundleMiniappSource.php:99`).

**Relative imports** (`import 'widgets/card.dart'`) are not exercised anywhere in
this repo — stick to `package:`. `[assumption]`

The entry file need not be `main.dart`: the regex allows a nested tail, so
`…/lib/src/app.dart` is a valid entry. What is fixed is the **function** name —
`entry_function` is hard-coded server-side (`BundleMiniappSource.php:116`).

## The four injected libraries

`okta-miniapp/lib/src/injected_sources.dart:41-46`. This map is the single
definition of the injected surface: `OktaHostPlugin` registers exactly these,
and `oktaMiniAppRuntimeSignature` hashes exactly these. Two lists could
disagree; one cannot. `[confirmed]` `:35-39`

| Import | Kind | Why it is that kind |
|---|---|---|
| `package:okta_kit/okta_kit.dart` | **source** | Ordinary Dart. Stays a real file (`lib/shared/okta_kit.dart`, 1567 lines) so the analyzer checks it and the host imports the same code natively. Mirrored verbatim into `lib/src/shared/okta_kit_source.g.dart` by `tool/generate_kit_source.dart`; `test/okta_kit_test.dart` fails the build on drift. |
| `package:okta_host/okta_host.dart` | **bridge facade** | Calls `$hostApiCall` and friends — symbols that exist only as bridge declarations inside the sandbox. It can never be a real file the host compiles, so it lives as a string by necessity. |
| `package:okta_motion/okta_motion.dart` | **bridge facade** | The sandbox has **no animation primitive at all** — no `Tween`, no `Curves`, no `TickerProvider`. Motion runs natively; the mini-app only describes the target state. |
| `package:okta_glass/okta_glass.dart` | **bridge facade** | No `BackdropFilter`, no `ImageFilter`; `BoxDecoration` drops `gradient` silently and refuses `Border` outright. The mini-app asks the host for the finished sheet — which also lets Okta restyle the material without any published mini-app being rebuilt. |

**source vs bridge is a security boundary, not a style choice:**

| | source (`addSource`) | bridge (`registerBridgeFunc`) |
|---|---|---|
| Runs | interpreted, in the sandbox | natively, in the host |
| Privileges | **none** | **all** — you gate it yourself |
| Constraint | must stay in the dart_eval subset | any Dart |
| For | models, pure logic, formatting, presentational widgets | tokens, network, camera, navigation |

Default to source. A bridge function is a new door out of the sandbox that you
alone are guarding.

### `okta_kit` public entry points

`okta-miniapp/lib/shared/okta_kit.dart` — `OktaJson` (`:59`), `OktaText`
(`:188`), `OktaPalette` (`:239`), `OktaAppBar` (`:357`), `OktaIcons` (`:619`),
`OktaTabs` (`:690`), `OktaMetrics` (`:1050`).

**Every class here is self-contained, and that is not a style preference** — a
class in an injected library may not reference *any other class in the same
library*, so `OktaList` carries its own copy of an icon tile that `OktaSurface`
also has. The analyzer cannot see this: natively the reference is valid Dart.
`test/okta_kit_test.dart` compiles a mini-app against every public entry point
for exactly this reason. `[confirmed]` `okta-miniapp/CLAUDE.md` §"One class in an
injected library cannot see another"

## The `Okta` facade — the full capability surface

`okta-miniapp/lib/src/host/okta_host_source.dart:74`. Grouped by the contract
that introduced them; the line number is the declaration.

| Group | Members |
|---|---|
| Identity (scalars — **prefer these**) | `locale():117` `tenantId():123` `roleId():129` `isDark():135` `studentHashid():151` `portal():165` |
| Identity (map) | `context():203` — returns a **decoded map**, read with `['locale']` |
| Contract | `contract():76` |
| API | `api():214` `get():233` `getQuery():237` `post():242` |
| Scanning | `scanBarcode():247` `scanNfc():250` |
| Files | `uploadFile():257` `openDocument():411` |
| UI | `toast():266` `appIcon():354` `remoteImage():569` `cameraPreview():649` |
| Storage (c5) | `storeGet():274` `storePut():279` `storeDelete():284` `storeKeys():291` |
| Sound (c5) | `playSound():306` |
| Location (c5/c6) | `location():318` `preciseLocation():338` |
| Navigation (c9) | `close():375` |
| Tool keys (c12) | `toolKeyStart():430` `toolKeyStop():439` `toolKeyReads():474` `toolKeyChannels():488` |
| Capabilities (c13) | `hasCapability():517` `requestCapability():534` + eleven name constants `:660-702` |
| Camera drain (c15) | `cameraScanStart():600` `cameraScanStop():610` `cameraScanReads():631` |

`context`, `uploadFile`, `location` and `preciseLocation` all return **plain
decoded maps** and name no class. `OktaApiResponse` is the one class still
mentioned, and that is a deliberate **bet, not a proof**: every mini-app reaches
its data through `Okta.get`, so `api()` is always called. `[confirmed]`
`okta_host_source.dart:14-34`. The rule behind it is in `working-notes.md`
§Trap 1.

## The capability vocabulary — three copies, no compiler between them

Eleven names, and they must match across three repos:

| Repo | File | Shape |
|---|---|---|
| **okta-miniapp** | `lib/src/host/okta_capabilities.dart` | `OktaCapabilities` static strings, `all:75`, `isKnown():94` |
| okta-web | `app/Enums/DeviceCapability.php:38-48` | backed enum |
| okta-partners | `app/Enums/DeviceCapability.php:30-40` | backed enum |

```
camera  microphone  nfc  bluetooth  usb  local_network
location_coarse  location_precise  files_read  documents_open  storage
```

**okta-miniapp's copy is the only one that is actually checked.** The gate on
the device reads it; the other two only decide what may be *declared*. A name
that exists in the PHP enums but not in `OktaCapabilities` publishes cleanly and
is then **never granted** — the gate fails closed. `[confirmed]`
`okta-miniapp/lib/src/host/okta_capabilities.dart:94`,
`okta-partners/CLAUDE.md` §`mobile.capabilities`

`GATE_CONTRACT = 13` is declared in both PHP enums (`okta-web:60`,
`okta-partners:50`) and is the minimum `minContract` for declaring any
capability.

`microphone` is **reserved**: declarable, reviewable and consentable, but no
bridge symbol exists behind it. `[confirmed]`
`okta_capabilities.dart:22-31`

The **ungated** set is written down explicitly, with a reason each, in
`oktaUngatedCapabilities` (`okta_capabilities.dart:133-145`) — because "we forgot
to gate it" and "we decided it needs no gate" look identical in code six months
later.

## The cache key

`okta-miniapp/lib/src/cache/okta_mini_app_file_cache.dart`

```
<baseDir>/<slug>__<entry>/<payloadVersion>__<runtimeSignature>.evc
```

Directory: `_slugDir():64-66`. File: `_file():68-71`.

Four inputs, each answering a distinct question:

| Input | Moves when | Without it |
|---|---|---|
| `slug` | different app | apps collide |
| `entry` | different package under one slug | **two entries of one slug collide** — they share slug, `payloadVersion` and signature, so whichever launched first won and the other replayed *its* bytecode (`:8-21`) |
| `payloadVersion` | partner publishes | a new release replays old bytecode |
| `runtimeSignature` | any injected source changes | a host fix never reaches devices (`:27-32`) |

`write()` prunes every other file in the directory — which is how an old
`payloadVersion` is cleaned up, and is why the directory is per-`(slug, entry)`
rather than per-slug (`:57-62`).

### `oktaMiniAppRuntimeSignature`

`okta-miniapp/lib/src/runtime_info.dart:28` =
`oktaMiniAppToolchainSignature` (`:8`, currently
`dart_eval-0.8.5+flutter_eval-0.8.2`) + `inj-<digest of every injected source>`.

The digest is not decoration. `payloadVersion` is the *partner's* version and
never moves when the host changes, so a signature that ignored content would
leave devices running bytecode compiled against old injected libraries —
silently, until each partner happened to publish. `[confirmed]` `:13-21`

Deliberately **not** cryptographic: it is a cache key, not a trust boundary, and
the two modular hashes stay under 2^53 at every step so web and native agree
(`:22-27`).

## Configuration and flags

| Setting | Where | Value read | Note |
|---|---|---|---|
| `OKTA_MINIAPP_RUNTIME_REF` | `okta-partners/config/okta-web.php:107` | committed default `26d0f6c…` | Substituted as `__MINIAPP_REF__` into every scaffolded partner repo. **Stale — see `working-notes.md`.** |
| Flutter SDK bound | `okta-miniapp/pubspec.yaml` | `>=3.19.0 <3.39.0` | Load-bearing, not tidiness — see below |
| `--no-tree-shake-icons` | every release build of okta-app | required | `okta-app/BUILDING.md:65-71` |
| `mobile.min-version` floor | okta-web Mobile App Catalog settings page | — | 426 for old clients, `routes/api.php:113-115` |

### The Flutter upper bound

`flutter_eval` bridges the SDK by hand-writing `class $X implements X` for 185
Flutter classes, forwarding every member. `implements` demands the whole
interface, so a member Flutter *adds* upstream is a compile error in a package
nobody edited (`Container.isAntiAlias` is the live example). There is nothing to
upgrade to: 0.8.2 is the latest on pub.dev. The bound turns "several hundred
errors deep inside an iOS archive" into one line from `pub get`. `[confirmed]`
`okta-miniapp/pubspec.yaml:8-27`

It is repeated verbatim in `okta-app/pubspec.yaml` and every mini-app's
`pubspec.yaml`, because `okta_miniapp` is consumed as a git dependency **pinned
by commit ref** — a constraint added in one place does not reach those
resolutions until somebody moves the ref.

### `--no-tree-shake-icons`

Required on **every** release target — apk, appbundle, ipa, web. Flutter subsets
the icon font by scanning const `IconData` at build time and cannot see inside
code compiled on the device, so tree-shaking strips precisely the glyphs
mini-apps ask for and they render as empty boxes — silently.

You meet the loud half first: `flutter_eval` constructs `IconData` non-const in
its own source, which trips the const-finder and **fails the build**. Keep the
flag for the silent half regardless. `[confirmed]`
`okta-app/lib/features/miniapps/bridge/okta_dart_miniapp_host.dart:8-19`,
`okta-app/BUILDING.md:65-81`
