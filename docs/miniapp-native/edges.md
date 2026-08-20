# Edges — failures, authorization, consent, versioning

> Native mini-apps (Dart on device). `okta-miniapp@50e394e`, `okta-app@e49423b`,
> `okta-web@52e48ee`, `okta-partners@608abc3`. Read 2026-08-20.
> Companion files: [`README.md`](./README.md) · [`flow.md`](./flow.md) ·
> [`contracts.md`](./contracts.md) · [`data.md`](./data.md) · [`working-notes.md`](./working-notes.md)

---

## 1 · Failure paths

### The four typed exceptions

`okta-miniapp/lib/src/errors.dart` — one base class so the host catches a single
stable type, with the low-level `dart_eval` cause retained for logging (`:12-17`).

| Type | Line | Raised when | Host presents |
|---|---|---|---|
| `OktaMiniAppCompileException` | `:57` | source fails to compile to EVC | error state (+ diagnostics on non-prod) |
| `OktaMiniAppLoadException` | `:63` | EVC fails to load into a runtime | error state |
| `OktaMiniAppRuntimeException` | `:70` | entrypoint throws, or returns a non-`Widget` | error state |
| `OktaMiniAppContractException` | `:78` | `minContract > hostContract` | **"update the app"**, never a generic failure (`:75-77`) |

### The contract gate has a documented gap, and a second route past it

The declared route is `min_contract`, checked **before anything compiles**
(`okta_mini_app_bundle.dart:154-163`). But a deployment older than the point
where okta-web started emitting `min_contract` sends nothing, the bundle defaults
to `1`, and the gate never fires — so the partner's app dies with a raw compiler
message instead of an actionable screen.

The undeclared route closes it: a compile error containing
`Cannot find import 'package:okta_` is treated as "this host is too old", because
a mini-app can only import `package:okta_*` libraries the **host** injects, so a
missing one is never the partner's mistake. `[confirmed]`
`okta-app/lib/features/miniapps/bridge/okta_dart_miniapp_host.dart:192-209`

Both routes converge on the same `updateApp` builder
(`okta_dart_miniapp_host.dart:275-277`).

### Retry must evict the cache, or it does nothing

A failure out of the compiled program means the **cached bytecode** is what
failed. Re-fetching alone hits the same cache entry and fails identically —
"which is why retry could never recover from a bad compile, only look like it did
nothing". So the error path evicts first, then invalidates the bundle provider:

```dart
unawaited(cache.evict(slug).whenComplete(() {
  if (context.mounted) ref.invalidate(oktaDartBundleProvider(target));
}));
```

`[confirmed]` `okta_dart_miniapp_host.dart:281-295`. The `context.mounted` guard
is load-bearing — `ref` is dead once the element is gone.

### A render crash does not take the host down

Flutter has **no per-subtree error boundary**: when `build()` throws, the
framework substitutes the single global `ErrorWidget.builder`, which has no
`BuildContext`. So `okta-miniapp` inserts an inherited marker
(`OktaMiniAppErrorScope`) and a probe widget that looks up its own position in
the tree to answer "did this failure happen inside a mini-app?". `[confirmed]`
`okta-miniapp/lib/src/mini_app_error_boundary.dart:24-48`

Two consequences:

- A mini-app crash reaches the host's presenter **only while
  `installOktaMiniAppErrorWidget()` has run**; outside a scope, whatever
  `ErrorWidget.builder` the app already had still applies (`:22-23`).
- `reload` discards the runtime and loads again, rather than rebuilding in
  place — a build failure leaves the tree poisoned at that spot (`:36-39`).

Failing to start (`errorBuilder`) and crashing after start (`renderErrorBuilder`)
are deliberately distinct states in okta-app
(`okta_dart_miniapp_host.dart:236-247`). Without the second, the framework paints
its own red screen inside the mini-app's area — a wall of yellow-on-red stack
trace with no way back.

### Diagnostics are gated on server environment, not build mode

The compiler message *is* the diagnosis, so hiding it leaves both the partner and
the platform with nothing. But no tenant should see a stack trace. The gate is
`settings.serverEnvironment.showsDiagnostics` — **not** `kReleaseMode` —
because a release build pointed at dev is exactly what a partner developer tests
from, and it used to show them nothing. `[confirmed]`
`okta-app/lib/features/miniapps/miniapp_screen.dart:112-119`

### Every failure carries a reference the user can read out

`_errorRef()` derives a stable reference from the fault rather than generating a
random one, so the same fault on two devices yields the same reference and a
recurring problem is recognisable. It carries `hostContract` **and**
`runtimeSignature`, because the contract alone does not identify a build.
Reporting is hung off the same call that produces the reference, so a surface
that displays a code always files it — safe from `build` because `reportOnce` is
idempotent per reference. `[confirmed]` `miniapp_screen.dart:71-85`

### The refusal vocabulary — refusals are values, never throws

There is no error shape for "you may not". Each capability refuses in the shape
it already defines, so a mini-app written before the gate existed degrades exactly
as it does on a device that simply lacks the hardware:

| Call | Refusal |
|---|---|
| `scanBarcode` / `scanNfc` | `null` |
| `uploadFile` | `null` — same as a cancelled picker |
| `storePut` / `storeDelete` | `false` |
| `openDocument` | `false` |
| `toolKeyStart` | `false` |
| `location` / `preciseLocation` | a **negative accuracy** |
| `cameraPreview` (ungated) | an empty box until an allowed `cameraScanStart` |

`[confirmed]` `okta-miniapp/lib/src/host/okta_host_plugin.dart:214-254`,
`okta-miniapp/lib/src/host/okta_capabilities.dart:133-145`

The negative accuracy is deliberate: a failed fix that reported `(0, 0)` would
hand back a real coordinate that passes a naive "did I get coordinates?" test.
A caller who forgets to check `ok` still fails `preciseTo(...)`.

**`false` is an ordinary answer.** A mini-app that ignores these return values
looks to the user like a button that does nothing.

## 2 · Authorization — two axes that must not be collapsed

| | **scopes** | **capabilities** |
|---|---|---|
| Govern | DATA | DEVICE resources |
| Example | `education.students.read` | `camera`, `nfc` |
| Granted by | the tenant admin, at install | the person holding the phone, per device |
| Enforced by | okta-web, server-side (`app.scope:` middleware, `AppPermissionGuard`) | okta-miniapp, on the device (`_gated()`) |
| Revocable by | the tenant | the user, in settings |

A school happy to expose its roster has said nothing about turning on the camera,
and the reverse. `[confirmed]`
`okta-miniapp/lib/src/host/okta_capabilities.dart:9-16`

**okta-web does not enforce capabilities and cannot.** It owns only the
*declaration*: accepting it at publish, carrying it in the catalog, and showing
it to the tenant admin before install.

### The gate's placement is the design

`_gated()` wraps the dispatch inside the plugin —
`okta-miniapp/lib/src/host/okta_host_plugin.dart:191-212`:

```dart
bool granted;
try { granted = await delegate.hasCapability(capability); }
on Object { granted = false; }   // fail CLOSED
if (!granted) return denied();
return work();
```

A host **cannot forget to check**, because checking is not something the host
does. Adding a method to `OktaHostDelegate` does not expose a capability —
wiring it into `configureForRuntime` does, and wiring it there means gating it.

It fails closed everywhere: a `hasCapability` that throws is ungranted
(`:199-205`), and a name outside the vocabulary is rejected before it reaches the
delegate (`okta-app/.../okta_host_delegate.dart:628-630`). The alternative — a
broken permissions store reading as "everything allowed" — is the one failure
mode this must not have.

### Three sequential authorizations on the native path

1. **Tenant → app**, at install: scopes, plus acknowledgement that the app *may
   ask* for the declared capabilities.
2. **Role → card**, at launch: `app.webview` reconfirms the active role holds the
   card's required scope before the payload is assembled
   (`MiniappPayloadController.php:12-22`).
3. **Person → capability**, at first use: the consent sheet, on this device only.

## 3 · Consent — device-local, deliberately

`okta-app/lib/features/miniapps/consent/mini_app_consent_store.dart`

Not on the server, and the reasons are the design (`:1-21`):

- The tenant admin agreed the app may *ask*. What is stored is the answer of the
  person holding **this** phone. A teacher who allowed the camera on their own
  device has said nothing about the shared tablet at reception.
- Uninstalling okta-app takes the app-support directory with it, so a reinstall
  starts from no consent — the behaviour users expect from every other app, and
  free only because the store is local.

Files rather than a database or secure storage: read-all / write-one, no query,
and this holds **decisions, not secrets**. Knowing "this person allowed NFC" is
not itself a capability (`:17-21`).

### Three states, not a bool

`enum MiniAppConsent { unknown, granted, denied }` (`:30`). "Not asked yet" and
"asked and refused" must lead to different behaviour — an app that re-prompts
every time someone declines is one people learn to dismiss without reading,
which costs the prompts that matter.

### `hasCapability` must not prompt; `requestCapability` may

`okta-app/lib/features/miniapps/bridge/okta_host_delegate.dart`

- `hasCapability` (`:626`) is a pure question. A mini-app calls it before it draws
  a button, so a dialog fired from a build path would be an ambush (`:628-629`).
- `requestCapability` (`:647`) returns an already-recorded answer **without
  prompting**, in both directions (`:672-675`): granted means no nagging on every
  entry into a feature; denied means a person who said no once is not asked again
  by an app that simply keeps asking. They change it in settings, on their terms.
- **Both answers are written** (`:698-701`). Storing only the yes would re-prompt
  after every no.
- Undeclared or unknown capability → `false` before anything else (`:627-630`,
  `:648-650`). A payload that has not said what it asked for has not asked for
  anything.
- No signed-in user → `null` store → nothing can be granted, because an answer
  belonging to nobody would be inherited by whoever signs in next (`:609-613`).
- Nothing on screen to ask on → refuse rather than record a decision nobody made
  (`:676-680`).

Isolation is a **path, not a convention**: the store file is derived from
(user id, slug), so one mini-app physically cannot address another's answers
(`mini_app_consent_store.dart:36-39`).

The prompt shows three things together: the platform's own fixed description of
the capability, the **partner's** written justification, and the app name
(`okta_host_delegate.dart:686-693`). The justification travels in the payload
rather than being looked up, because a prompt whose justification failed to load
is a prompt nobody can answer. `[confirmed]`
`okta-miniapp/lib/src/bundle/okta_mini_app_bundle.dart:7-12`

### Tool keys gate coarsely, on purpose

One `toolKeyStart()` call opens NFC, Bluetooth, a wired reader and a LAN gate box.
It refuses only when **none** of the four is granted, and the **host** filters
channel by channel — it is the only side that knows which channels it has.
`toolKeyChannels()` reports what survived both filters, so the operator can be
shown *which* reader is being waited on rather than one undifferentiated spinner.
The keyboard-wedge channel is ungated: a reader presenting itself as a keyboard
exposes no device resource the user has not already handed over by typing.
`[likely]` — read from `okta-miniapp/CLAUDE.md` §contract 13 and the ungated list
at `okta_capabilities.dart:133-145`; the per-channel filtering itself lives in
`okta-app/lib/features/toolkeys/` and was not traced.

## 4 · The import boundary (mechanically enforced)

`okta-app/.github/workflows/miniapp-boundary.yml`

1. `package:okta_miniapp`, `package:dart_eval` and `package:flutter_eval` may be
   imported **only** inside `lib/features/miniapps/bridge/` (`:24-32`).
2. The removed schema runtime (`package:miniapp_kit`, `package:rfw`) must never
   reappear anywhere (`:34-40`).

Same philosophy as the partner-policy scanners on the Laravel repos. This is why
`MiniAppScreen` is the only file the rest of okta-app needs to know about, and
why `OktaDeclaredCapability` is translated into a gateway type at the boundary
(see `contracts.md` §name drift).

## 5 · Concurrency and ordering — partially traced

- The bundle provider is **family-keyed by target**, so a portal card and a role
  card for the same slug never share a fetch (`okta_dart_miniapp_host.dart:74-77`).
- The cache directory is per-`(slug, entry)` and `write()` prunes siblings, so two
  entries of one slug do not evict each other on alternate launches
  (`okta_mini_app_file_cache.dart:57-62`).
- Cache reads and writes never throw; any failure is treated as a miss
  (`:76-90`).
- The consent store caches decisions in memory because the gate is consulted on
  **every** gated call — a tool-key drain asks several times a second, so a disk
  read per call would make the gate the slowest thing in the app
  (`mini_app_consent_store.dart:41-44`).

**Not traced**: what happens if two mini-apps compile concurrently, whether cache
writes are atomic against a mid-write kill, and consent-store write races between
two sheets. `[assumption]` — see Open Questions.

## 6 · Versioning

- **Contract**: integer, hand-bumped, currently **17**
  (`okta-miniapp/lib/src/host/okta_host_delegate.dart:228`). Contracts referenced
  in these notes: 5 (storage/sound/location), 6 (precise location), 9 (`close`),
  11 (`openDocument`), 12 (tool keys), 13 (capability gate), 14 (identity as
  scalars), 15 (remote images + camera drain), 16 (glass), 17 (student-profile
  tabs).
- **A new symbol on an existing class is as breaking as a new library.** Bump on
  any change to the compile surface — see `contracts.md` §B5 for the incident
  that established this.
- **Backwards compatibility on the device**: a mini-app below the contract floor
  is refused before compiling; one above it runs unchanged, because refusals are
  values rather than new error shapes (§1).
- **A deliberate partner-visible break** was taken at contract 15:
  `Okta.context().locale` no longer resolves. It could not be preserved — the
  class-shaped `context()` failed to compile for any mini-app that did not name
  the class, so the apps this "broke" were apps that could not run. `[confirmed]`
  `okta-miniapp/lib/src/host/okta_host_source.dart:174-200`

---

## What this pass did NOT cover

Stated so the gap is visible rather than silent:

- `okta_motion` and `okta_glass` internals — named, not traced.
- The tool-key subsystem in `okta-app/lib/features/toolkeys/` beyond its shape.
- The student-profile-tab surface (contract 17) beyond its manifest rules.
- The portal launch path beyond how it differs from the role path.
- `NormalizeMobileAudiences` audience-matching logic.
- The webview and external modes, except where they exclude native features.
- Concurrency specifics listed in §5.
- Any load/perf characteristics of on-device compilation.
