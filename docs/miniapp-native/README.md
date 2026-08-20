# Native mini-apps (Dart on the device) — technical notes

> **Scope**: how a partner authors a mini-app in real Dart, and how that source
> reaches a device, compiles there, and renders inside `okta-app`.
> **Commits read**: `okta-miniapp@50e394e`, `okta-app@e49423b`, `okta-web@52e48ee`,
> `okta-partners@608abc3`, `okta-docs@9f95c93` — all on branch
> `claude/okta-miniapp-native-dev-jbztqi`, read 2026-08-20.
> **Depth**: standard. What was skipped is listed at the bottom of `edges.md`.

## The one-paragraph model

A native mini-app is **ordinary Dart whose entrypoint returns a Widget**
(`Widget main() => ...`). It is never compiled by the partner and never shipped
as a binary. The partner's `.dart` files are read off disk by `okta-web`,
assembled into a JSON **source bundle**, and handed to `okta-app` over a
short-lived signed URL. `okta-app` compiles that source **on the device** using
`dart_eval` + `flutter_eval` (wrapped by the `okta_miniapp` package), caches the
resulting bytecode, and renders the widget. The mini-app runs in a sandbox with
**no network, no filesystem, no plugins** — every capability it has is a bridge
function the host registered, reached through the injected `package:okta_host`
facade.

Two consequences drive almost everything else in these notes:

1. **What is reviewed is exactly what runs.** No compiled artifact ever crosses
   the network, in either direction.
2. **The sandbox is much smaller than Flutter.** `flutter_eval` hand-bridges 185
   classes and a thin slice of each one's parameters. Anything outside that
   slice fails either at device-compile time or — worse — silently at runtime.
   `working-notes.md` is largely a catalogue of those failures.

## Repo map for this feature

| Repo | Role on the native path | Key path |
|---|---|---|
| **okta-miniapp** | The runtime itself: the compiler wrapper, the sandbox bridge, the injected libraries, the capability gate, the contract number. | `lib/src/` |
| **okta-app** | The Flutter host. Fetches the bundle, compiles, caches, renders, implements the delegate, owns device consent. | `lib/features/miniapps/` |
| **okta-web** | Assembles + serves the source bundle; validates the manifest at publish; issues the signed launch URL. | `app/Services/MobileAppCatalog/` |
| **okta-partners** | Where the partner declares the surface (mode `native`, entry path, `minContract`, capabilities) and gets scaffolded a working package. | `resources/partner-boilerplate/okta_app/native/` |
| **okta-docs** | These notes. No code. | `docs/miniapp-native/` |

## How to read these notes

| File | Holds |
|---|---|
| [`flow.md`](./flow.md) | The golden path, hop by hop, from partner keystroke to rendered widget. Start here. |
| [`contracts.md`](./contracts.md) | Every repo-to-repo boundary: exact routes, payload shapes, where names drift. The most consulted file. |
| [`data.md`](./data.md) | The bundle shape, the injected libraries, the capability vocabulary, cache keys, config and flags. |
| [`edges.md`](./edges.md) | Failure paths, the two authorization axes, consent, versioning, what was skipped. |
| [`working-notes.md`](./working-notes.md) | The sandbox traps that actually bite, modification points, open questions, anchor index. |

**Confidence tags** appear inline: `[confirmed]` = the code that does it was
read; `[likely]` = inferred from one side of a boundary or from naming;
`[assumption]` = a gap being filled, and every one has a matching entry in
Open Questions in `working-notes.md`.

## Read this first if you are…

- **Onboarding** → `flow.md`, then `contracts.md`.
- **About to write a native mini-app** → `working-notes.md` §Traps, then
  `data.md` §The injected libraries. The traps are not style advice; each one
  is a crash that reached a device.
- **About to extend the host contract** → `working-notes.md` §Modification
  points, which lists the bump-everywhere checklist and the three places the
  capability vocabulary is duplicated with no compiler between them.

## One finding worth surfacing immediately

`okta-partners` pins the runtime every new partner repo is scaffolded against.
Its committed default is **13 commits and 4 contract versions behind**
`okta-miniapp` HEAD — see `working-notes.md` §Finding: the scaffold pin has
drifted again. This is the exact failure `okta-partners/CLAUDE.md` records as
having already happened once.
