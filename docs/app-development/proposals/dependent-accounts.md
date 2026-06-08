# Design spec — Dependent accounts & per-audience entries

> **Status: PROPOSED (design only — not yet implemented).** This document
> specifies a change to the installed-application model. It describes the target
> design and the gap from today's code; nothing here is live yet. The "how it
> works today" guides under [`../`](../README.md) remain the source of truth for
> current behaviour. Statements about current code cite real files; design
> decisions still open are marked `> TODO: confirm`.
>
> Written generically (no specific app named), per the docs convention.

---

## ملخّص بالعربية (للمراجعة قبل التنفيذ)

- يبقى **الحساب الأساسي (Primary)** هو المشترِك/المثبِّت للتطبيق كما هو الآن.
- نضيف **حسابات تابعة (Dependent)**: عند تثبيت المالك/المعلم للتطبيق، يظهر تلقائياً
  لأولياء الأمور والطلاب — **ويُحدَّد ذلك في partners** ضمن إعداد التطبيق.
- **كل نوع حساب (أساسي/تابع) له `entry` خاص** يُعرَّف في partners، **ويجوز أن
  يتشارك نوعان نفس الـ `entry`**.
- **الإضافة الأخيرة (المحورية):** ولي الأمر/الطالب يرى التطبيق **في النطاق العام
  (general/portal) لا في نطاق الجهة (tenant)** — أي بصفته `guardian`/`student`
  بشكل عام عبر كل الجهات المرتبطة به، **دون اختيار جهة**.
- **الخبر الجيّد:** سطح الـ `general` scope مع بورتالات student/guardian **موجود
  بالفعل** في الكود (طبقة السياق + بيانات البورتال)، لكن **كتالوج التطبيقات لا
  يَخدمه بعد**. لذا العمل = ربط مفهوم "الفئات/الجماهير (audiences)" الجديد بهذا
  السطح الموجود، لا بناؤه من الصفر.
- **أهم قرار مفتوح:** عند فتح ولي الأمر تطبيقاً تابعاً في النطاق العام (وقد يكون
  لديه أبناء في أكثر من مدرسة) — كيف نحدّد سياق الجهة؟ (تجميع داخل التطبيق /
  اختيار المدرسة أولاً / لكل طالب). يحتاج قرارك (القسم 6).

---

## 1. The requirement

1. **Primary account** — the subscriber/installer (a Tenant: school owner / teacher).
   Unchanged from today.
2. **Dependent accounts** — when the primary installs the app, it surfaces
   automatically to **guardians and students of that school**, configured in
   `okta-partners`.
3. **Per-audience entry** — each account type (primary or dependent) has its **own
   `entry`**, configured in partners; **two types may share one `entry`**.
4. **General-scope surfacing (the new addition)** — a guardian/student sees a
   dependent app in their **general (cross-tenant) scope as `guardian`/`student`**,
   **not** bound to a selected Tenant. They don't "subscribe" — they inherit
   visibility from the school's install.

Generalised model: an installed app declares a set of **audiences**, each =
*(who → which entry + options)*, where "who" is either a set of **tenant-scoped
roles** (e.g. `tenant-admin`, `teacher` → seen in tenant scope) or a **general
portal** (`student` / `guardian` → seen cross-tenant in the general scope).

---

## 2. Current state (grounded in code)

### 2.1 One entry + one role list per app (no audiences)
- Partners stores per-version mobile config in `partner_module_versions.mobile_support`
  + `mobile_config` (JSON). Keys: `mode`, `entry`, `allowed_roles`,
  `allowed_platforms`, `allowed_origins`, `required_scope`, `pass_role_claim`
  (`okta-partners/.../2026_05_19_000000_add_mobile_support_to_partner_module_versions.php`).
- `PartnerModuleVersion::buildMobileBlock()` emits **one** block
  `{ supported, mode, entry, passRoleClaim, allowedRoles?, allowedPlatforms?, requiredScope?, allowedOrigins? }`.
  → **One `entry` and one `allowed_roles` list for the whole app.** Multiple roles
  can be listed, but they all share the **same** screen; any per-role branching is
  the partner's own code inside that single entry.

### 2.2 The app catalog is tenant-scoped only
- `GET /api/mobile/app-catalog` requires `tenant_id` + `role_id`
  (`MobileAppCatalogController::index`).
- `GetMobileCatalogForUser(user, tenantId, roleId)`:
  resolves the active `TenantUser` link, lists `tenant_module_installations` for
  **that one tenant**, reads each module's `manifest['mobile']`, emits **one card
  per app**, then filters by `required_scope` (role permission) and `allowed_roles`
  (role name). → Strictly tenant-scoped; needs a chosen tenant + role.
- Launch: `IssueWebviewLaunch` signs `/app/{slug}?u&t&r` for embedded, returns the
  single `entry`; `EnsureAppWebview` + `WebviewController` serve it.

### 2.3 The general (portal) scope ALREADY EXISTS — but carries no apps
This is the key finding. The platform already models cross-tenant student/guardian
portals:

- **Context selection** accepts three scopes (`MobileContextController::select`):
  `scope ∈ {tenant, system, general}`. For `general`, `role_ids` are **not** used
  and a `portal ∈ {student, guardian}` is required.
- **`BuildMobileContext::buildGeneralContext()`** returns a context with
  `tenant_id = null`, no role, `active_role_key = portal`, gated on an **active link
  existing** (`TenantStudent`/`TenantGuardian` with `released_at` + `deleted_at`
  null). It calls `Tenant::forgetCurrent()` — there is deliberately **no tenant and
  no spatie role/team** in this scope.
- **`ListAvailableContexts`** returns `has_student_access` / `has_guardian_access`
  flags so the client shows the portal entries.
- **Portal data surface** (cross-tenant): `GET /api/mobile/portal/student`,
  `GET /api/mobile/portal/guardian` (`MobilePortalController` →
  `GetStudentPortal`/`GetGuardianPortal`) return the user's children/schools grouped
  across **all** tenants — **no tenant scope applied**. Plus
  `POST /api/mobile/portal/students/{hashid}/profile-launch` → a signed
  `/portal/students/{hashid}` web page (`portal.webview`).
- Web equivalents exist too: `/my-school` (`student.portal`) and the guardian
  children page.

**The gap:** these portal endpoints return only **native data** (children, school,
profiles). They carry **no installed-app cards**. And the app catalog
(`GetMobileCatalogForUser`) can't run in this scope because it demands a
`tenant_id` + `role_id`, while the general scope has neither.

### 2.4 Summary of the gap
| Needed | Today |
|---|---|
| Multiple audiences per app (primary + dependent) | One flat `allowed_roles` + one `entry` |
| Per-audience `entry` (sharable) | Single `entry` for all |
| Dependent apps shown to guardian/student in **general** scope | General/portal scope exists but **serves no apps**; catalog is tenant-only |

---

## 3. Target design

### 3.1 Audiences on the manifest
Replace the single mobile block with a list of **audiences**. Each audience targets
either tenant-scoped roles **or** a general portal, and names its own entry:

```json
"mobile": {
  "supported": true,
  "audiences": [
    {
      "key": "staff",
      "kind": "primary",
      "roles": ["tenant-admin", "teacher"],     // tenant-scope audience
      "mode": "embedded",
      "entry": "mobile/screens/staff.blade.php",
      "requiredScope": "education.students.read",
      "allowedPlatforms": ["ios", "android"]
    },
    {
      "key": "guardian",
      "kind": "dependent",
      "portal": "guardian",                      // general-scope audience
      "mode": "embedded",
      "entry": "mobile/screens/family.blade.php"
    },
    {
      "key": "student",
      "kind": "dependent",
      "portal": "student",                       // shares the SAME entry as guardian
      "mode": "embedded",
      "entry": "mobile/screens/family.blade.php"
    }
  ]
}
```

Rules:
- An audience has **either** `roles[]` (tenant scope) **or** `portal`
  (`student`/`guardian`, general scope) — not both.
- `kind`: `primary` | `dependent` (informational + drives defaults/labels;
  precedence rules in §6).
- Two audiences may reference the **same** `entry` (the sharing requirement — see
  `guardian` + `student` above).
- **Backward compatibility:** a version with today's single-block shape
  (`entry` + `allowedRoles`) is read as **one `primary` audience**
  `{ kind: primary, roles: allowedRoles, entry, … }`. No published app breaks.

### 3.2 okta-partners (the configuration place)
- Extend `mobile_config` to hold `audiences[]`; `buildMobileBlock()` emits
  `mobile.audiences[]` (and keeps emitting the legacy single keys when there's
  exactly one tenant-role audience, for older okta-web).
- `VersionEditor` mobile tab → a repeatable list of audiences; each row picks
  *target* (roles **or** portal) + entry + options; validation (entry under
  `mobile/`, mode/entry minimum, roles-xor-portal).
- **Role vocabulary** for the picker: partners needs to know the platform roles
  (`tenant-admin`, `teacher`, …) and the two portals. `> TODO: confirm` whether to
  ship a fixed enum or sync a small **roles catalog** from okta-web (analogous to
  the scope catalog).

### 3.3 okta-web (consumption)
- **`ManifestValidator`**: accept `mobile.audiences[]` (validate `kind`,
  roles-xor-portal, `entry` under `mobile/`, `mode`); keep the legacy single block
  valid.
- **Tenant-scope catalog** (`GetMobileCatalogForUser`): instead of one card per
  app, select the **audience whose `roles` match the active role** and emit the card
  with **that audience's `entry`** (+ its `requiredScope`/`allowedPlatforms`). No
  matching audience → no card (same as today's role filter).
- **NEW general-scope catalog** (the heart of the addition): a new resolver, e.g.
  `App\Services\MobileAppCatalog\GetPortalAppCatalogForUser(User, string $portal)`
  that:
  1. finds the user's **active** `TenantStudent`/`TenantGuardian` links across **all**
     tenants (the same gate the portal already uses);
  2. for the tenants behind those links, finds installed+active apps whose manifest
     declares a **dependent audience with `portal == $portal`**;
  3. emits one **deduplicated** card per such app, using that audience's `entry`.
  - Surfaced either by **extending the portal payload** (`GetGuardianPortal`/
    `GetStudentPortal` gain an `apps: [...]` array) or a **new endpoint**
    `GET /api/mobile/portal/{portal}/app-catalog`. `> TODO: confirm` which.
- **Launch in general scope**: a portal launch variant (e.g.
  `IssuePortalWebviewLaunch` + a `portal.app.webview` middleware) that signs a URL
  carrying `u` + `portal` (+ a resolved tenant/student per §6), since there is no
  single `t`/`r` to put in today's `/app/{slug}?u&t&r`.
- **Gating in general scope is by active link, NOT by role/permission.** The portal
  user has no spatie role/team (`tenant_id = null`), so `requiredScope`/`allowedRoles`
  filters from §3.1 **do not apply** to portal audiences. Visibility = (active
  student/guardian link) × (a tenant behind it installed the app) × (the app's
  manifest declares a dependent audience for this portal). `> TODO: confirm`.

### 3.4 okta-app (client)
- The student/guardian **portal screens already exist**; add an "apps" section that
  renders the dependent-app cards from §3.3 and opens them with the **same WebView
  host** used for tenant-scope cards. Minimal new wiring; no new auth flow (the
  general context + portals are already there).

### 3.5 Web surface (okta-web) — phase 2 (optional)
Mirror dependent apps on the web portal pages (`/my-school`, guardian children
page) so guardians/students see them on the website too. `> TODO: confirm` whether
this is in scope now or a follow-up (the request emphasises "الموقع أو التطبيق").

---

## 4. Security & isolation

- **General-scope gating** reuses the existing, proven guard: an **active
  `TenantStudent`/`TenantGuardian` link** (`released_at` + `deleted_at` null). No
  role/permission check (there is no role in this scope).
- **Reaching tenant data**: an installed app still reads/writes Tenant data only via
  the [Partner API](../data-access-and-security.md), under a **per-tenant**
  installation token. Because a guardian spans tenants, opening a dependent app must
  resolve to a concrete tenant context under the hood (see §6.1). The per-user
  WebView token is minted as today; the app calls its own `/api/<slug>/*` and passes
  the resolved tenant via the existing context header.
- **No new install records for dependents.** Visibility derives from the **primary's
  tenant install** + the manifest's dependent audience — dependents don't install or
  subscribe. (Confirm in §6.)

---

## 5. Backward compatibility

- Old single-block manifests → one `primary` audience; tenant-scope catalog behaves
  exactly as today.
- Apps with **no** dependent/portal audience → never appear in the general scope
  (no behaviour change).
- The legacy `allowed_roles`/`entry` keys keep being emitted for single-audience
  apps so an older okta-web still works during a partial deploy.

---

## 6. Open decisions (need your call before coding)

**New, general-scope-specific:**

6.1 **Tenant resolution when a dependent app is opened in general scope** (a
guardian may have children in several schools, possibly only some installed the
app). Options:
   - (a) **App aggregates internally** — one entry, the app lists the guardian's
     children/schools (via Partner API) and handles multi-tenant itself.
   - (b) **Pick-school/child first** — the card expands to choose a school/child,
     then launches in that tenant context.
   - (c) **Per-child cards** — one card per child-in-an-installed-school.
   *(Recommend (a) for simplicity; (b) if data must be strictly one-tenant-at-a-time.)*

6.2 **Surfacing point**: extend the portal payload (`/portal/{student,guardian}`
with an `apps[]`) vs a dedicated `GET /api/mobile/portal/{portal}/app-catalog`.
*(Recommend a dedicated endpoint — keeps portal data and app catalog separable.)*

6.3 **Confirm portal audiences are gated by active link only** (no `requiredScope`
for portal audiences), and that an app reaches data through the relevant tenant's
installation token resolved per §6.1.

6.4 **Web surface now or phase 2?** (§3.5)

**Carried over from the audiences discussion:**

6.5 Role vocabulary in partners: fixed enum vs synced roles catalog (§3.2).
6.6 Precedence when a user matches multiple audiences (primary-first / first-match /
multiple cards).
6.7 Billing stays on the **primary** only; dependents are free-by-inheritance — and
any seat limits? (assumed none).
6.8 Dependent identity reuses the tenant install (no separate token/DB schema per
dependent) — confirm.

---

## 7. Implementation plan (after sign-off — not started)

Phased, backward-compatible, on branch `claude/sleepy-volta-TgIyL`:

1. **Manifest + validator** (okta-web `ManifestValidator`) — accept `audiences[]`;
   keep legacy. + docs.
2. **Partners config + UI** (`mobile_config.audiences[]`, `buildMobileBlock()`,
   `VersionEditor`) + tests; (roles catalog if chosen).
3. **Tenant-scope catalog** — resolve per-role audience entry
   (`GetMobileCatalogForUser`, `IssueWebviewLaunch`, `EnsureAppWebview`,
   `WebviewController`) + tests.
4. **General-scope catalog** — `GetPortalAppCatalogForUser`, portal launch variant,
   surface in portal payload/endpoint + tests (the core of this addition).
5. **okta-app** — render app cards in the student/guardian portal screens.
6. **Web surface** (if §6.4 = now).
7. **Docs** — promote this spec into the dev guide + update
   [`../manifest-reference.md`](../manifest-reference.md),
   [`../app-surface.md`](../app-surface.md),
   [`../../../claude/installed-apps.md`](../../../claude/installed-apps.md).

---

## 8. Code anchors (current behaviour referenced above)

| Concern | okta-web / okta-partners path |
|---|---|
| Mobile block builder (single entry today) | `okta-partners` · `app/Models/PartnerModuleVersion.php::buildMobileBlock()` |
| Mobile config storage | `okta-partners` · `database/migrations/2026_05_19_000000_add_mobile_support_to_partner_module_versions.php` |
| Tenant-scope catalog | `okta-web` · `app/Services/MobileAppCatalog/GetMobileCatalogForUser.php` |
| Catalog endpoint (tenant_id+role_id) | `okta-web` · `app/Http/Controllers/Api/Mobile/MobileAppCatalogController.php` |
| Launch / webview serve | `okta-web` · `app/Services/MobileAppCatalog/IssueWebviewLaunch.php`, `app/Http/Controllers/App/WebviewController.php` |
| **General scope context** | `okta-web` · `app/Http/Controllers/Api/Mobile/MobileContextController.php`, `app/Services/MobileAuth/Context/BuildMobileContext.php` |
| **Portal data (cross-tenant)** | `okta-web` · `app/Http/Controllers/Api/Mobile/MobilePortalController.php`, `app/Services/MobileAuth/Portal/{GetStudentPortal,GetGuardianPortal}.php`, `app/Services/MobileAuth/Context/ListAvailableContexts.php` |
| Portal routes | `okta-web` · `routes/api.php` (`mobile/portal/*`, `mobile/app-catalog`) |
