# Installed-app notifications — declare, dispatch, deliver, read the outcome

How a notification raised by an **installed application** (a check-in, a leave
decision, a day-closed summary) reaches a parent's phone or a school admin's
inbox — and how to find out when it did not. The path crosses every repo:
the application **declares** its catalog in `okta-partners`, **dispatches** from
its own code, `okta-web` **delivers** over the channels the school enabled,
and `okta-app` **receives** push and shows the in-app feed. `okta-hdor`
(attendance) is the worked example throughout.

> Why this page exists: the first embedded app to ship notifications followed
> the developer guide literally — a third `recipients:` argument, a comment
> saying `manager_id` "resolves to the recipient automatically", and a
> `partner_notify()` helper — none of which existed on the platform. It
> flattened the recipient into a `student_id` variable, every send arrived
> unaddressed, and every delivery row read `failed: No … recipients`. No parent
> got a message. The contract below is the one the platform actually enforces.

Related: [web.md](./web.md#partner-notifications) · [partners.md](./partners.md) ·
[app.md](./app.md#notifications--push) · [installed-apps.md](./installed-apps.md) ·
[glossary.md](./glossary.md)

---

## The model in one picture

```
 okta-partners                okta-web                                   okta-app
 ─────────────                ────────                                   ────────
 Notifications tab ──publish/─▶ SyncCatalogFromManifest ──▶ partner_app_notifications
 (catalog per version) bridge          │
                                       ▼ tenant installs the app (ModuleInstalled)
                              SeedTenantNotificationSettings ──▶ tenant_partner_notification_settings
                                       │   enabled=false · channels=default_channels · recipients=tenant admins
                                       ▼ school flips keys/channels on /settings/notifications
 the app's code ──────────────▶ DispatchNotification(key, payload)
 (in-process, or                       │ normalize · plan gate · setting · quiet hours · channel ∩
  POST /api/apps/notifications/        ▼
  dispatch for External apps)   one delivery row per channel  ──▶ channel jobs
                                       │                          in_app → `notifications` table ──▶ feed + badge
                                       │                          push   → FCM HTTP v1 ─────────────▶ device
                                       │                          whatsapp / sms / email / webhook_out
                                       ▼
                              partner_notification_deliveries  (status + error per attempt)
                                       │
                 ┌─────────────────────┼──────────────────────────────┐
                 ▼                     ▼                              ▼
   /settings/notifications     GET /api/apps/notifications/logs    bridge → MCP `notification_deliveries`
   delivery-log card (school)  (the app, notifications.logs.read)  (the developer, across every school)
```

Two facts decide everything else:

- **Who receives a message is said inside the payload** — `recipient: {type, id}`.
  Nothing else in the payload is an address.
- **Which channels carry it is decided by the school**, inside the ceiling the
  developer declared. The app never names a channel.

---

## 1. Declare — the catalog (okta-partners)

Every application version carries a **notifications catalog**: one entry per
event the app can raise. Partners edit it on the dashboard
(*Apps → app → Notifications*, or the `create_notification` /
`update_notification` MCP tools); the dashboard writes the
`manifest.json["notifications"]` block back to the version branch — the block
is never hand-edited.

| Entry field | Meaning |
|---|---|
| `key` | `<slug>.<resource>.<event>`, immutable after creation |
| `display_name` / `description` (`ar`/`en`) | what the school sees on its preferences page |
| `variables` | `{name → type}` — what `{{ placeholders }}` may reference |
| `default_template` | the developer's own wording; what the school starts from and what is sent until the school customises |
| `audience` | `guardian` / `student` / `staff` / `admin` — informational on the school page; **read by the scanner** (see below) |
| `default_channels` | a **ceiling**, not a default: the school narrows it and can never widen it |
| `severity` | `info` / `warning` / `critical` |
| `is_active` | inactive entries are left **out of the manifest** (`BuildManifestFragment`), so they deactivate on okta-web at the next sync |

`PartnerModuleNotification::toManifestEntry()` is the single writer of the
entry shape. The catalog is **frozen when the version is published** and
cloned forward into the next draft (`CloneNotificationsToNewVersion`).

**The boilerplate scanner** (`scripts/partner-policy/NotificationScanner.php`,
shipped into every partner repo and run by CI) compares declared keys with
dispatched keys:

| Rule | Level | Fires when |
|---|---|---|
| `notification-key-not-declared` | blocking | a key dispatched in code is not in `manifest.json` — the platform refuses it by name anyway |
| `notification-key-unused` | advisory | declared, never dispatched |
| `notification-helper-undefined` | advisory | `partner_notify(...)` — not a platform function; fails at runtime unless the module defines it |
| `notification-recipient-missing` | advisory | a `guardian`/`student` key dispatched with a **literal** payload that has no `recipient` (a `$payload` variable is not judged) |

---

## 2. Ship and mirror (okta-partners → okta-web)

The manifest block reaches okta-web on **publish** and on **every dashboard
edit** through the bridge
(`POST /api/partners/modules/{slug}/notifications/sync`,
`OktaWebService::syncNotificationCatalog()` — fans out to production and
sandbox). `SyncCatalogFromManifest` upserts `partner_app_notifications` for the
version and **soft-deactivates** keys that the incoming manifest no longer
carries. Tenant settings pointing at a deactivated key keep their row (see the
known gap below).

---

## 3. Install → the school's settings (okta-web)

When a school installs the app, `ModuleInstalled` triggers
`SeedNotificationSettingsOnModuleInstalled` → `SeedTenantNotificationSettings`,
which writes one `tenant_partner_notification_settings` row per active catalog
key: **disabled**, `channels = default_channels`, `recipients = the tenant-admin
roles`. The same seeder is used by the preferences page and by the dispatcher
(a declared key whose row is missing — an installation that predates the
version that added it — is seeded on first dispatch, disabled). A one-off
migration backfilled rows the page had created without defaults
(`2026_09_09_000001_backfill_partner_notification_settings_created_by_the_preferences_page`).

The school then works on `/settings/notifications`
(`App\Livewire\Settings\Notifications\PartnerAppNotificationsPage`): one tab per
installed app, a row per key with its audience, an enable switch, channel
switches (only channels the platform can actually carry **and** the developer
declared), a template editor (`NotificationTemplateModal`) and a test-send
button (`NotificationTestSendModal` — renders the template with typed dummy
values and really sends to a recipient the operator picks). `in_app` is always
offered; `push` only when the platform has an FCM service account configured
(`/settings/platform-delivery`); `whatsapp`/`sms`/`email` when a provider app
is installed (email also falls back to the platform mailer).

---

## 4. Dispatch — the app's side

Two entry points, one contract:

- **Embedded** apps call the service in-process:
  `app(\App\Services\PartnerApi\Notifications\DispatchNotification::class)($key, $payload)`.
  No scope is consulted; what is required is a **partner-app context** (below).
- **External** apps (or anyone with an installation token) call
  `POST /api/apps/notifications/dispatch` with `{key, payload}` behind the
  `notifications.dispatch.send` scope (`idempotent`). Answer: `202` with
  `delivery_ids` (one per channel), `422` naming an undeclared key.

### The payload

`NormalizeDispatchPayload` is the one shape every channel job reads. Reserved
keys: `recipient`, `variables`, `subject`, `body`, `url`, `locale`, `metadata`;
any other top-level key is moved under `variables` (the flat form the old guide
showed still works — as variables).

```php
[
    'recipient' => ['type' => 'parent_of_student', 'id' => $student->ulid],
    'variables' => ['student_name' => 'خالد', 'event_time' => '٧:٠٢ ص'],
    'subject'   => '…',                        // optional fallback title
    'body'      => '…',                        // optional fallback body
    'url'       => '/okta-hdor/students/…',   // optional, what the in-app card opens
    'locale'    => 'ar',                       // optional
    'metadata'  => ['record_id' => 9],         // optional, stored, never shown
]
```

| `recipient.type` | resolves to (`ResolveAudience`) |
|---|---|
| `parent_of_student` | the active guardians of the student whose ULID (or id) is `id` |
| `host_user` | the user whose id or ULID is `id` |
| `school_admin` | the school's configured recipients, else every `tenant-admin` |

**No `recipient`** → the recipients configured on the school's settings row
(its admins by default). Right for a broadcast to the office (a day-closed
summary); wrong for anything about one person — a `student_id` or `user_id`
beside the variables is a **template variable**, never an address, and such a
send reaches the office or nobody (`failed: No … recipients`).

### The context requirement

`DispatchNotification` refuses with «called outside a partner-app context»
unless a `ModuleContext` is active in `AppContextManager`. okta-web builds it
with `BootModuleContext`; on an embedded app's own web routes that is the
`module.context` middleware. Every other surface must build it itself —
**around** the work, because a middleware's effect lasts exactly as long as
the `$next` it is handed. okta-hdor needed four: web routes (`module.context`),
the mobile API (`BootPartnerAppContext`), queue jobs (`BootsSchoolTenant`) and
**Livewire actions** (`ActsInPartnerAppContext` — an action posts to
`/livewire/update`, an okta-web route with no `module.context`). Calling the
middleware with a closure that returns at once and dispatching afterwards
shipped twice and refused ~1,700 check-in notifications a day while every stack
trace showed the middleware present.

### Worked example — okta-hdor

`app/Jobs/SendParentNotificationJob.php` is the module's single call site
(dispatched synchronously from `NotifyOnAttendanceRecorded`,
`NotifyOnLeaveRequestCreated/Decided` and `AnnounceDayClosed`). It resolves the
title/body from the module's own templates, builds
`{recipient, variables, subject, body, locale, metadata}` and makes **one**
call per notification; the platform fans out. The ten keys and their
recipients are tabled in `okta-hdor/NOTIFICATIONS.md`.

---

## 5. Deliver — inside `DispatchNotification` (okta-web)

In order:

1. `NormalizeDispatchPayload`.
2. Plan gate — the school's plan lacks `partner_notifications` → one row
   `skipped_plan_feature`.
3. Setting lookup — key not in the active catalog → `ValidationException`
   naming the key (the `422` above); declared but no row → seed it disabled.
4. Row disabled → `skipped_disabled`. Quiet-hours window → `skipped_quiet`.
5. Channels = the school's ticked channels ∩ the developer's active
   `default_channels`; empty → `skipped_disabled`.
6. Per channel: a `queued` delivery row, then the channel job.

| Channel | Job | Transport |
|---|---|---|
| `in_app` | `WriteInAppNotificationJob` | Laravel `notifications` table (rendered `subject`/`body` beside the raw payload) — the feed on both surfaces |
| `push` | `SendPushNotificationJob` | FCM HTTP v1, per device from the central registry; `skipped_no_transport` without a service account, `failed` when the audience has no registered device |
| `whatsapp` | `SendWhatsAppNotificationJob` | the school's installed provider app, else the Connect bridge |
| `sms` | `SendSmsNotificationJob` | installed provider app only — no platform fallback on purpose |
| `email` | `SendEmailNotificationJob` | installed provider app, else the platform mailer |
| `webhook_out` | `SendWebhookOutNotificationJob` | HMAC-SHA256 signed POST (the outbound-webhook envelope) |

**The text** is chosen once for every channel by `RenderNotificationBody`:
the school's own wording → the developer's `default_template` → `payload.body`
→ a stand-in from the catalog description. Title: `payload.subject` → display
name → key.

Every attempt ends in exactly one of `queued`, `sent`, `failed`,
`skipped_disabled`, `skipped_quiet`, `skipped_plan_feature`,
`skipped_no_transport` (`App\Enums\PartnerNotificationDeliveryStatus`), with
the channel job's error text on the row.

---

## 6. Receive (okta-app)

The client never addresses anyone and never sees a device token of anyone
else. On login it registers its own FCM token into the central registry
(`POST /api/mobile/notification-tokens`, revoked on logout), reads the feed
(`GET /api/mobile/notifications`, unread count, mark read) and shows the bell
badge; a push opens the feed. Push is optional — without the Firebase
compile-time defines the layer is a no-op and the in-app feed still works.
Details in [app.md](./app.md#notifications--push).

---

## 7. Read the outcome

Three surfaces show the **same** delivery rows:

| Who | Where | What |
|---|---|---|
| The school | `/settings/notifications` → "Delivery log" card | last 7 days for the selected app: time, key, channel, status, error text |
| The app | `GET /api/apps/notifications/logs` (scope `notifications.logs.read`) | its own installation's rows; filters `channel`, `status`, `key`, `from`, `to`; `page`/`per_page` (max 100) → `{data, total, page, per_page, last_page}` — okta-hdor wraps it as `GetNotificationLogs` |
| The developer | MCP `notification_deliveries {slug, environment?, key?, days?}` over the bridge `GET /api/partners/modules/{slug}/notification-deliveries` | counts by status / channel / key across every school (≤ 30 days) plus the latest failures and skips with their error — no recipient, no school name |

Reading a status:

| You see | It means | Fix lives in |
|---|---|---|
| `failed` · `No … recipients` | payload had no `recipient` (or an unknown type) and the school configured nobody | the app — add `recipient: {type, id}` |
| `skipped_disabled` | the school has not enabled the key, or ticked no channel the developer allows | the school's page — nothing in code |
| `skipped_no_transport` | channel ticked, platform has no transport (push without FCM, sms without a provider app) | platform / school configuration |
| `skipped_plan_feature` | the school's plan excludes app notifications | subscription |
| `skipped_quiet` | quiet hours | nothing — it was deliberate |
| `422 … not declared in this app's catalog` | key missing from the installed version, or `is_active=false` and never configured | declare, publish, make sure the school installed that version |
| «outside a partner-app context» (exception, no row) | dispatched from a surface that never built the context | wrap the work in the context (see §4) |
| nothing at all | the call never ran — check `app_errors` / `recent_errors` over MCP | the app |

---

## Known gaps and decisions

- **`is_active=false` stops the spread, not the sends.** The key leaves the
  manifest, disappears from the school page, is not seeded for new schools and
  is refused for an installation that never configured it — but a school that
  enabled it earlier keeps receiving it as long as the app dispatches it
  (`DispatchNotification` reads the active catalog row only to intersect
  channels; a missing row means "all channels"). Stopping a feature means
  stopping the dispatch in code.
- **`notifications.dispatch.send` uses the verb `send`.** The partner scope
  picker and `sync_from_manifest` only express `read`/`write`, so the scope is
  not in the boilerplate seed catalog; External apps receive it through the
  live catalog, Embedded apps need no scope for the in-process call.
- **`partner_notify()` never existed.** The scanner and the discover tool still
  match it (so the key counts as used) and flag it.
- **Test-send goes where the operator points it**, not through the dispatch
  audience — it proves transport (FCM, a device), not payload addressing.
- **No result notification to the sender.** An app learns what happened only by
  reading the log surfaces above; there is no callback.

---

## Where to look (code anchors)

| Concern | Repo · path |
|---|---|
| Catalog entry shape (single writer) | `okta-partners` · `app/Models/PartnerModuleNotification.php::toManifestEntry()` |
| Manifest block / clone / push to okta-web | `okta-partners` · `app/Services/PartnerModules/Notifications/{BuildManifestFragment,CloneNotificationsToNewVersion,PushCatalogToOktaWeb}.php` |
| Declared-vs-used scanner | `okta-partners` · `resources/partner-boilerplate/scripts/partner-policy/NotificationScanner.php` |
| Developer-facing contract (AR/EN) | `okta-partners` · `docs/partner-platform/developer-guide{,.en}.md` → "Notifications catalog" |
| API reference entries | `okta-partners` · `app/Services/PartnerDocs/ApiReference/BuildApiReference.php` (`post_notifications_dispatch`, `get_notifications_logs`) |
| MCP delivery diagnostics | `okta-partners` · `app/Services/PartnerMcp/Server/Tools/MonitorTools.php` (`notification_deliveries`) + `app/Services/OktaWebService.php::listNotificationDeliveries()` |
| Catalog ingestion | `okta-web` · `app/Services/PartnerApi/Notifications/SyncCatalogFromManifest.php` |
| Seeding on install / on first dispatch / backfill | `okta-web` · `app/Listeners/PartnerNotifications/SeedNotificationSettingsOnModuleInstalled.php`, `app/Services/PartnerApi/Notifications/{SeedTenantNotificationSettings,BackfillTenantNotificationSettings}.php` |
| Dispatch orchestration | `okta-web` · `app/Services/PartnerApi/Notifications/{DispatchNotification,NormalizeDispatchPayload,ResolveAudience,ResolveRecipients,RenderNotificationBody}.php` (file map in that directory's `README.md`) |
| Channel jobs | `okta-web` · `app/Jobs/PartnerNotifications/*` |
| Runtime routes | `okta-web` · `routes/apps.php` (`POST /notifications/dispatch`, `GET /notifications/logs`) → `App\Http\Controllers\Api\Apps\NotificationsController` |
| Bridge aggregate | `okta-web` · `routes/api.php` (`GET /api/partners/modules/{slug}/notification-deliveries`) → `App\Services\Partners\Diagnostics\GetModuleNotificationDeliveries` |
| School page + delivery card + test send | `okta-web` · `app/Livewire/Settings/Notifications/{PartnerAppNotificationsPage,NotificationTemplateModal,NotificationTestSendModal}.php` |
| Push transport + device registry | `okta-web` · `app/Services/Notifications/Push/*`, `notification_device_tokens`, mobile routes in `routes/api.php` |
| Client feed + push | `okta-app` · `lib/features/notifications/`, `lib/core/push/push_config.dart` |
| Worked example | `okta-hdor` · `app/Jobs/SendParentNotificationJob.php`, `app/Support/Tenancy/{PartnerAppContext,BootsSchoolTenant,ActsInPartnerAppContext}.php`, `app/Http/Middleware/BootPartnerAppContext.php`, `app/Services/PartnerApi/Notifications/GetNotificationLogs.php`, `NOTIFICATIONS.md` |
