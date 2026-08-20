# Working notes — traps, modification points, open questions

> Native mini-apps (Dart on device). `okta-miniapp@50e394e`, `okta-app@e49423b`,
> `okta-web@52e48ee`, `okta-partners@608abc3`. Read 2026-08-20.
> Companion files: [`README.md`](./README.md) · [`flow.md`](./flow.md) ·
> [`contracts.md`](./contracts.md) · [`data.md`](./data.md) · [`edges.md`](./edges.md)

---

## Finding: the scaffold pin has drifted again

**`[confirmed]`, and worth acting on.**

Every new partner repo is scaffolded with a `pubspec.yaml` pinning `okta_miniapp`
by commit ref, substituted from
`okta-partners/config/okta-web.php:107`:

```php
'runtime_ref' => env('OKTA_MINIAPP_RUNTIME_REF', '26d0f6cd3ec03ad48fabe43ad4d64e26158ee1b2'),
```

Verified against `okta-miniapp` history on this branch:

| | commit | date | `oktaHostContractVersion` |
|---|---|---|---|
| Pinned default | `26d0f6c` | 2026-08-16 | **13** |
| okta-miniapp HEAD | `50e394e` | 2026-08-19 | **17** |

`26d0f6c` is an ancestor of HEAD; **13 commits** separate them, spanning contracts
14, 15, 16 and 17 — identity-as-scalars, remote images and the camera drain,
glass, and student-profile tabs.

**Why it matters.** A partner scaffolded today compiles against contract 13. Any
mini-app using `Okta.locale()`, `Okta.remoteImage()`, `Okta.cameraScanStart()`,
`okta_glass` or `Okta.studentHashid()` fails `flutter test tool/validate.dart`
locally with a symbol the partner did not write and cannot fix. The CI gate's
guarantee — "compiles in CI implies compiles on device" — inverts: it now
rejects code the device would run.

This is the exact failure `okta-partners/CLAUDE.md` records as already having
happened once ("بقي القالب على commit سابق للعقد 8 عبر ثلاثة إصدارات عقد" — the
template stayed on a pre-contract-8 commit across three contract releases). The
config-not-literal fix was applied; the value still went stale.

**Caveats before acting:**
- This is the **committed default**. Production may set `OKTA_MINIAPP_RUNTIME_REF`
  to something current — no `.env` was readable here. `[assumption]`
- The correct value is not automatically HEAD. It must match the `okta_miniapp`
  commit the **shipped okta-app binary** was built against, since `okta_kit` and
  friends live in that binary (`contracts.md` §B5). Pinning ahead of the shipped
  app is the same bug pointing the other way.

**Fix**: one line in `okta-partners/config/okta-web.php:107`, plus re-running
`pushBoilerplate()` for existing partner repos (a platform-team job, separate PR).

---

## Traps — what compiles and still fails

Every item below is a crash that reached a device. Sources:
`okta-miniapp/CLAUDE.md`, the boilerplate README, and the tool headers.

### Trap 1 · A class in an injected library must be *reached* to exist

> **A class DECLARED in an injected library may only be mentioned by a method the
> mini-app actually CALLS.**

`dart_eval` resolves such a class only when the mini-app pulls it in. For an app
that never calls the method, the name does not exist — and the failure lands on
the injected library, naming a file the partner did not write.

Three wrong diagnoses preceded the right one, and each looked confirmed **because
the compiler stops at the first error**, so every fix revealed the next class in
line: (1) "a bare return type is the broken construct" — dropping the annotation
moved the error into the body; (2) "it compiles because it also references
`OktaCapability`" — a guess fitting one data point; (3) "async bodies are safe" —
checkable, and still wrong: fixing `context()` immediately produced
`Unknown type OktaUploadResult` in an **async** signature.

So `context`, `uploadFile`, `location` and `preciseLocation` return plain maps.
`[confirmed]` `okta-miniapp/lib/src/host/okta_host_source.dart:14-34`

**Do**: read maps with `['status']` / `['latitude']`; prefer the scalars
`Okta.locale()/tenantId()/roleId()/isDark()`.

### Trap 2 · A value that arrived from outside is not an interpreter value

Calling a bridged method on one dies — and **the message points at the argument,
not the receiver**, which is why it was misread for so long:

```
type 'String' is not a subtype of type '$Value?' in type cast
#0 _CastListBase.[]
#1 $String._startsWith
```

`args[0]` is the literal `'ar'`; nothing is wrong with it. The *receiver* is the
wrong shape, so the compiler took the dynamic-dispatch path and pushed the
argument unboxed. Any bridged method does it — `startsWith`, `toLowerCase`,
`trim`, `contains`, `substring`, and `toString()`, which is easy to forget is
bridged at all.

Two live sources: a value read out of a decoded map (`map['locale'] as String` —
the cast is **not** a guard, a host-derived String *is* a String), and a
`_text()`-style JSON reader that returns what it found.

**Fix at the mint, not the call sites:**

```dart
final dynamic value = map['locale'];
return '$value';          // not `map['locale'] as String`
```

Interpolation makes the interpreter allocate the string. For bools use
`== true`, not `as bool`.

It hid because three apps independently rediscovered the `'$x'` workaround at
their own call sites and each wrote it up as local folklore. `[confirmed]`
`okta_host_source.dart:87-116`

### Trap 3 · A missing map key produces a value you cannot even inspect

`dart_eval`'s `Map` bridge returns a **raw Dart null** for a missing key, not the
runtime's boxed `$null`. And `IsType.run` opens with
`runtime.frame[_objectOffset] as $Value`.

So `value is String` on a missing key does not return false — it **throws**:
`type 'Null' is not a subtype of type '$Value'`. And `== null` is worse than
useless: `CheckEq` falls through to a plain Dart `==`, so `rawNull == null` is
**false** and the guard silently passes it to the next method call, which dies
instead.

There is no defensive way to look at such a value. Never create one:

```dart
if (!source.containsKey(key)) return null;   // a source-level null IS a boxed $null
return source[key];
```

`OktaJson.at` does this (`okta-miniapp/lib/shared/okta_kit.dart:67`). Any
hand-rolled reader must too. `[confirmed]`

### Trap 4 · Named arguments are matched leniently and typed strictly

Read from `dart_eval`'s `helpers/argument_list.dart`. The compiler walks the
parameters **the bridge declares** and looks each up in your call. Three
consequences, biting in opposite directions:

1. **An argument the bridge does not declare is never read** — no error, no
   warning, silently discarded. `flutter_eval` keeps unsupported parameters as
   *commented-out* `BridgeParameter` lines, so `BoxDecoration` looks like it takes
   `borderRadius`, `gradient`, `image` and `shape` and takes **none** of them.
2. **`optional: false` is not enforced for named parameters** — a
   declared-but-omitted one arrives `null`. This is the real mechanism behind the
   `Expanded`/`flex` rule.
3. **A declared parameter you do supply is type-checked strictly**, against the
   bridge's own `$extends` graph, which knows nothing of Dart `implements`:
   `$Border` declares no supertype, so `Border` is **not** a `BoxBorder` and
   `BoxDecoration(border: Border.all(…))` fails to compile.

So `BoxDecoration` is effectively `color` + `boxShadow`. Round with
`ClipRRect` + `ColoredBox`; draw a hairline as `Container(height: 1.0, color: …)`.

### Trap 5 · Declared, but not wired

A class can be declared to the compiler and never registered with the runtime. It
compiles perfectly, passes `tool/validate.dart`, then throws while building:

```
UnimplementedError: Tried to invoke a nonexistent external function
```

In flutter_eval 0.8.2 that is true of **`InkWell`, `InkResponse` and `SafeArea`**.
A compile check cannot catch it, and neither can reading the bridged class list —
which is exactly how an `InkWell` tab bar shipped and crashed on the first device
that ran it.

**The same trap exists on METHODS, and the audit does not cover those.**
`List.sort()` is the live example: its runtime implementation reads
`args[0] as EvalCallable` (not `args[0]?`), so `list.sort()` with no comparator
compiles and then throws. Always pass the comparator, or don't sort.

**`State.mounted` is not bridged** — an async gap cannot ask whether it is still
on screen. Guard with your own flag.

### Trap 6 · Boxing faults — four roads to the same cast error

All produce `type '$X' is not a subtype of type 'X' in type cast`, and whether any
single site dies appears to turn on **register allocation** — so "this one works"
is not evidence.

| Road | Bad | Good |
|---|---|---|
| Primitive through an interpreted parameter into a bridged constructor | `IconData _icon(int c)` | build from literals at the construction site, or a zero-arg method (`OktaIcons`) |
| Reading a **bool out of a list** | `final dot = dots[i];` | `final dot = i < dots.length && dots[i] == true;` |
| `setState` with an **arrow body** | `setState(() => _busy = true)` | `setState(() { _busy = true; })` |
| Loop counter **initialised from a parameter** | `for (int i = from; i <= to; i++)` | literal bounds, one method per range |

The list-index mechanism is **confirmed against dart_eval's compiler**:
`IndexedReference.getValue` returns `Variable.alloc(ctx, listElementType)` whose
`boxed` flag is false for `List<bool>`, while the runtime hands back a `$bool`;
the later `boxIfNeeded` emits `BoxBool` on something already boxed. The comparison
works because `_compileShortCircuit` ends with `.copyWith(boxed: true)`, so
`boxIfNeeded` returns early.

Convert **every** `setState` arrow body, not only the ones that crash.

### Trap 7 · A nested ternary loses its value

```
type 'Null' is not a subtype of type 'String' in type cast
13525: CopyValue (L24 <-- L24)     ← written to slot 24
13527: BoxInt (L23)  <<< EXCEPTION ← read from slot 23, never written
```

The compiler allocates the result slot for the outer conditional and the inner one
writes to a different slot. **Nesting is the trigger; where the value was going is
irrelevant** — an argument *or* a plain local. A single ternary is fine.

The fix is not to hoist into a local, because Trap 6 still applies to a primitive
feeding a bridged constructor. **Branch, and repeat the call:**

```dart
if (selected) return OktaMotion.box(-1.0, -1.0, 0xFF6D428F, 999.0, 180, child);
if (_dark)    return OktaMotion.box(-1.0, -1.0, 0xFF1F2A37, 999.0, 180, child);
return              OktaMotion.box(-1.0, -1.0, 0xFFF3F4F6, 999.0, 180, child);
```

Widgets hoist to a local fine — build the child once. Only the primitive is pinned.

### Trap 8 · Map-typed receivers, tear-offs, nested loops

From the boilerplate README (the partner-facing list):

- **Index JSON on a `dynamic` receiver, never a `Map`-typed one.** An `is Map`
  guard *promotes* it and springs the trap: dart_eval binds raw `Map` to an
  arbitrary bridged instantiation and rejects your String key at compile time with
  `Cannot use variable of type String as index to map of type <int, Color>`. The
  `<int, Color>` never comes from your data — don't go hunting for it. Guard on
  the **result** (`v is List`), never the receiver. `is List` promotion is fine;
  only `Map` is affected.
- **Callbacks must be closure literals**, not tear-offs: `onPressed: () => _f()`,
  never `onPressed: _f` ("Cannot box Function").
- **No `cond ? null : closure`** in a callback slot.
- **No nested loops** — dart_eval 0.8.5 throws a `RangeError` at compile time when
  a loop body, directly *or through a method it calls*, contains another loop.
  Flatten first, ideally server-side.
- **Buttons: `ElevatedButton` / `TextButton` only**; no `style` parameter, so
  buttons cannot be branded.
- **`Wrap`, `Divider`, `SingleChildScrollView`, `TextAlign`, `Material`,
  `CircularProgressIndicator`, `Colors`, `TabBar*` are absent.** No animation
  primitives at all. `ThemeData.colorScheme` is absent, so a mini-app **cannot**
  read the host palette — which is why `OktaPalette` carries its own values.
- **`Text` bridges neither `maxLines` nor `overflow`** — a too-wide label wraps
  mid-word. Use `FittedBox(fit: BoxFit.scaleDown, …)` inside a bounded width.
- **Every `Expanded` needs an explicit `flex:`**; `Flexible` is not bridged.
- **There IS a clock**: `DateTime`, `Duration` and `Future.delayed` are bridged and
  wired. Only `Timer` is absent — so no *periodic* callback, but delays and real
  event timestamps work. `dart:convert` (`jsonEncode`/`jsonDecode`) is bridged.

### The three gates, and what each cannot catch

| Gate | Catches | Blind to |
|---|---|---|
| `flutter analyze` | ordinary Dart errors | everything sandbox-specific; also **false-flags `Okta`/`package:okta_host` as undefined** — expected |
| `flutter test tool/validate.dart` | compile errors, against the real runtime | silently-dropped arguments (Trap 4.1), unwired classes (Trap 5), all boxing faults (Traps 6–7) |
| `dart run tool/audit_bridge_usage.dart <main.dart>` | class bridged? parameter accepted? type satisfies? **constructor wired?** | **methods** (`List.sort`), boxing faults; heuristic, not a parser — false-positives on capitalised words in strings |

`okta-miniapp/tool/audit_bridge_usage.dart:1-36`. Treat a clean audit run as
**necessary rather than sufficient** (`:35`).

---

## Modification points

### A · Add a capability to the vocabulary

Ordered; skipping any one produces a name that publishes and is never granted.

1. `okta-miniapp/lib/src/host/okta_capabilities.dart` — add the constant **and**
   add it to `all` (`:75-89`). *This is the only copy the gate reads.*
2. `okta-miniapp/lib/src/host/okta_host_delegate.dart` — add the delegate method.
3. `okta-miniapp/lib/src/host/okta_host_plugin.dart` — register the bridge func in
   `configureForRuntime` (`:79-141`) **wrapped in `_gated(...)`** (`:191-212`),
   choosing a refusal value from the vocabulary in `edges.md` §The refusal
   vocabulary. Wiring it here is what exposes it; gating it here is not optional.
4. `okta-miniapp/lib/src/host/okta_host_source.dart` — add the `Okta.*` facade
   method **and** a name constant (`:660-702`). Obey Trap 1: name no class.
5. **Bump `oktaHostContractVersion`** — `okta_host_delegate.dart:228`.
6. `okta-web/app/Enums/DeviceCapability.php` — add the enum case (`:38-48`).
   Note this enum has only the shared `GATE_CONTRACT` (`:60`), no per-case floor.
7. `okta-partners/app/Enums/DeviceCapability.php` — add the case (`:30-40`) **and
   its per-case floor** in `minContract()` (`:70-77`). The floor is kept per-case
   here, and only here, "so a capability added in, say, contract 15 does not
   silently claim to work on 13" — a `match` with no default, so a new case that
   is not added there is a PHP error rather than a silent `GATE_CONTRACT`.
8. `okta-web/lang/{ar,en}/partner_apps.php` — the platform's own fixed description
   (`capability_descriptions`), which the partner cannot write.
9. `okta-app/lib/features/miniapps/consent/mini_app_capability_labels.dart` — the
   label and description shown on the prompt. This is a **second copy of the
   names**: the file cannot import `package:okta_miniapp` (see `edges.md` §4), so
   a name missing here still gates correctly but prompts with its **raw key** —
   which looks like a bug on the one screen where someone is deciding something.
10. `okta-app/lib/features/miniapps/bridge/okta_host_delegate.dart` — implement it,
    including the OS-level permission request.
11. Tests: `okta-miniapp/test/okta_host_contract*_test.dart`,
    `okta-app/test/mini_app_consent_test.dart` (the vocabulary-coverage group at
    `:109-137` fails automatically if step 9 is skipped),
    `okta-web/tests/Feature/Store/DeviceCapabilitiesManifestTest.php`,
    `okta-partners/tests/Feature/Modules/DeviceCapabilitiesManifestTest.php`.
12. Ship a new **okta-app build** — the injected libraries live in that binary
    (`contracts.md` §B5). Republishing mini-apps does nothing.
13. Move `OKTA_MINIAPP_RUNTIME_REF` (`okta-partners/config/okta-web.php:107`) to the
    commit that build used.

> **Steps 6–7 have no compiler between them and step 1.** Three copies of eleven
> names, three repos. Step 9 *is* mechanically checked —
> `okta-app/test/mini_app_consent_test.dart:109-137` asserts the labels and
> `OktaCapabilities` cover each other exactly, in both directions. **The two PHP
> enums are not.** `okta-partners` has a test asserting its enum matches the
> manifest schema and the template, but as its own `CLAUDE.md` says:
> *"نجاحه ليس دليلاً على أن الجهاز يعرف الاسم"* — passing it is not evidence the
> device knows the name. A name in the PHP enums but not in `OktaCapabilities`
> publishes cleanly and is then never granted.

### B · Add a symbol to an injected library (e.g. a method on `OktaTabs`)

1. Edit `okta-miniapp/lib/shared/okta_kit.dart`. Obey the **self-containment
   rule** — no reference to any other class in the same library.
2. `dart run tool/generate_kit_source.dart` and **commit the `.g.dart`**.
   `test/okta_kit_test.dart` fails the build on drift.
3. Add a compile case to `test/okta_kit_test.dart` — it compiles a mini-app
   against every public entry point precisely because the analyzer is blind here.
4. **Bump `oktaHostContractVersion`.** A new method is as breaking as a new
   library; this is the `OktaTabs.bottomBar` incident (`contracts.md` §B5).
5. Steps 12–13 of §A.

`oktaMiniAppRuntimeSignature` moves on its own (it digests the sources) and will
correctly bust the cache — but it **does not gate admission**. Only the bump does.

### C · Add a new injected library

Register it in `okta-miniapp/lib/src/injected_sources.dart:41-46` **and nowhere
else** — that one map is both what the plugin compiles in and what the cache key
hashes, so shipping and cache-busting cannot drift. Then bump the contract, and
steps 12–13 of §A.

### D · Change the native entry-path shape

Three places, two repos (see `contracts.md` §B3): `MobileEntryRules.php:36`,
`ManifestValidator.php:818`, `BundleMiniappSource.php:50`. The third also serves
modules stored before the publish gate existed, so it cannot simply defer.

### E · Ship a new okta-app build

Always `--no-tree-shake-icons`, on **every** release target (`data.md` §flags).

---

## Open questions

Each is answerable in about a sentence by someone who already knows.

1. **Is `OKTA_MINIAPP_RUNTIME_REF` set in production**, and to what? The committed
   default pins contract 13 against a HEAD of 17. (§Finding)
2. **Which `okta_miniapp` commit is the currently-shipped okta-app built against?**
   That, not HEAD, is the correct scaffold pin.
3. **Is there any automated check that the two PHP `DeviceCapability` enums agree
   with `OktaCapabilities`?** The okta-app ↔ okta-miniapp half **is** covered:
   `okta-app/test/mini_app_consent_test.dart:109-137` asserts bidirectionally that
   every `OktaCapabilities.all` name has a label, and that no label names something
   `OktaCapabilities.isKnown` rejects. Nothing equivalent was found spanning the
   Dart vocabulary and the two PHP enums. (§A steps 6–7)
4. **`okta-miniapp/tool/validate.dart:11` defaults `OKTA_MINIAPP_DIR` to
   `miniapp_dart`** — the pre-`okta_app/native/` name. Dead default, or still used
   by something? The boilerplate's own copy hardcodes `lib/`, so partners are
   unaffected. `[confirmed]` as drift, unknown as to impact.
5. ~~**Are cache writes atomic against a mid-write kill?**~~ **Answered, and
   fixed.** They were not: `writeAsBytes` wrote straight to the target, so a
   kill mid-write left a truncated `.evc` that `read()` returns happily and
   `Runtime(ByteData)` then dies on, every launch, until the partner publishes.
   `write()` now stages to `<target>.writing` and renames — a rename either
   happened or did not. Same pass also fixed `read()` returning its future from
   inside a `try` (`return f` rather than `return await f`), which let an I/O
   error escape the method its doc comment promised would never throw.
6. **Can two mini-apps compile concurrently**, and is on-device compilation
   CPU-bound enough to matter on low-end devices? Not traced.
7. **Does anything re-run `pushBoilerplate()` for existing partner repos**, or is
   every scaffolded repo frozen at its creation-time pin? `okta-partners/CLAUDE.md`
   calls it a platform-team job; no scheduler was found.
8. **`microphone` is reserved with no bridge symbol.** Is a capability landing, or
   should it be removed from the vocabulary? A partner declaring it today draws a
   "declared but unused" publish warning.
9. **Does `toolKeyStart()` really refuse only when none of its four channels is
   granted, with the host filtering channel by channel?** Stated in
   `okta-miniapp/CLAUDE.md` §contract 13 and consistent with the ungated list, but
   the per-channel filtering itself lives in `okta-app/lib/features/toolkeys/` and
   was not traced. (`edges.md` §Tool keys gate coarsely, tagged `[likely]`)
10. **Do relative imports work inside a mini-app package?** All of `lib/` is handed
    over as one package, and the only multi-file example
    (`okta-miniapp/test/okta_mini_app_bundle_test.dart:12-30`) uses `package:`
    exclusively. No Flutter SDK was available here to try
    `import 'widgets/card.dart'`. If they work it is worth documenting; if they
    don't, it is worth an explicit rule in the boilerplate README.
    `[assumption]` (`data.md` §Multiple files, and the import form)

---

## Anchor index

The files worth opening first, by question.

| Question | File |
|---|---|
| What can a mini-app call? | `okta-miniapp/lib/src/host/okta_host_source.dart` |
| What does the host implement? | `okta-miniapp/lib/src/host/okta_host_delegate.dart` (contract at `:228`) |
| How is it wired + gated? | `okta-miniapp/lib/src/host/okta_host_plugin.dart` (`:79`, `:191`) |
| What is injected? | `okta-miniapp/lib/src/injected_sources.dart` |
| Cache key | `okta-miniapp/lib/src/cache/okta_mini_app_file_cache.dart` (`:57-71`) |
| Cache signature | `okta-miniapp/lib/src/runtime_info.dart` |
| Capability names | `okta-miniapp/lib/src/host/okta_capabilities.dart` |
| Shared widgets/logic | `okta-miniapp/lib/shared/okta_kit.dart` |
| Wire format | `okta-miniapp/lib/src/bundle/okta_mini_app_bundle.dart` (`:176-202`) |
| CI gate | `okta-miniapp/lib/src/validation/okta_mini_app_validator.dart` |
| Device fetch/compile/render | `okta-app/lib/features/miniapps/bridge/okta_dart_miniapp_host.dart` |
| Device delegate + consent | `okta-app/lib/features/miniapps/bridge/okta_host_delegate.dart` (`:626`, `:647`) |
| Consent storage | `okta-app/lib/features/miniapps/consent/mini_app_consent_store.dart` |
| Import boundary | `okta-app/.github/workflows/miniapp-boundary.yml` |
| Bundle assembly | `okta-web/app/Services/MobileAppCatalog/BundleMiniappSource.php` |
| Payload serving | `okta-web/app/Http/Controllers/App/MiniappPayloadController.php` |
| Launch | `okta-web/app/Http/Controllers/Api/Mobile/MobileAppCatalogController.php` (`:104`, `:150`) |
| Publish rules | `okta-web/app/Modules/Core/ManifestValidator.php` (`:446`, `:817`, `:912`) |
| Entry-path shape | `okta-partners/app/Services/Partners/Manifest/MobileEntryRules.php` |
| Scaffold | `okta-partners/resources/partner-boilerplate/okta_app/native/main/` |
| Scaffold pin | `okta-partners/config/okta-web.php:107` |
| **Prose reference** | `okta-miniapp/CLAUDE.md` — the deepest written account of the traps |
