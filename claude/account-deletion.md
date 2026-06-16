# Account deletion (self-service)

How an **end user** deletes their own `okta` account, across the two surfaces a
user touches — the `okta-web` platform UI and the `okta-app` mobile client.
This is distinct from the **platform-admin** "delete a user" tool (an admin
removing *someone else's* account from `okta-web` settings); that one is an
immediate teardown described in `okta-web/CLAUDE.md`.

> Why it exists on mobile too: the Apple App Store and Google Play require any
> app that lets you create an account to also let you delete it **from inside
> the app**. So the same capability is dual-surfaced — web + app — over a shared
> backend.

Related: [web.md](./web.md) · [app.md](./app.md) · [glossary.md](./glossary.md)

---

## Model: grace period, not instant erase

A deletion is a **scheduled** action with a cancellation window, not an
immediate purge:

```
request ──▶ deactivated (grace window open) ──▶ [grace_days] ──▶ purge
                  │                                                  ▲
                  └────────────── cancel (sign in) ─────────────────┘
```

1. **Request** — the user confirms (see *Confirmation* below). The account is
   deactivated *now*: every **other** web session and mobile token is revoked
   (the current one is kept so the user can still cancel), and the account is
   fenced to a "deletion pending" surface.
2. **Grace window** — `accounts.deletion.grace_days` (a `PlatformSetting`,
   default **30**). During it the user can **cancel** by signing in, which fully
   restores access.
3. **Purge** — once the window elapses, a daily job runs the platform's
   thorough teardown (the same `DeleteUser` the admin tool uses: identifiers,
   tenant links, memberships and sessions are dropped, the user is soft-deleted;
   roles + activity log are retained for audit).

### Why `users.status` is **not** reused for the pending state

The auth guards (Fortify `authenticateUsing`, the mobile
`AuthenticateWithIdentifier`, and the `EnsureActiveUser` middleware) all treat
any non-`active` `users.status` as **suspended** and block sign-in. If we
flipped the status, the user could not sign in to **cancel**. So the pending
state lives in dedicated columns and `status` stays `active`:

| Column (`users`)          | Meaning                                       |
|---------------------------|-----------------------------------------------|
| `deletion_requested_at`   | when the user asked                           |
| `deletion_scheduled_at`   | when the purge runs (non-null ⇒ **pending**)  |
| `deletion_reason`         | optional free-text reason                     |
| `deletion_confirmed_via`  | `password` or `otp`                           |

`User::isPendingDeletion()` is simply `deletion_scheduled_at !== null`.

---

## Confirmation (password **or** OTP)

The request must be authenticated as the account owner:

- **Credential accounts** → re-enter the **current password**.
- **Passwordless accounts** (phone-only logins, which are OTP-based) → a
  one-time code is sent to the phone and verified. This reuses the platform OTP
  plumbing (`OtpService` / `OtpDispatcher`, channel chain
  WhatsApp → SMS → email).

The surface decides which to ask for from the deletion-status payload
(`confirmation: "password" | "otp"`), based on whether the user has a stored
credential.

---

## Eligibility — who is blocked

`CheckAccountDeletionEligibility` returns a list of blockers (empty ⇒ allowed):

- **Super admin** — the platform super admin can never self-delete.
- **Sole `tenant-admin` of a paying tenant** — if the user is the *only*
  `tenant-admin` of a Tenant whose subscription is `active`/`grace_period`,
  they must transfer ownership or cancel the subscription first. This mirrors
  the `DeleteTenant` guard so a paying org is never left ownerless. (If a
  co-admin exists, or the subscription is inactive, deletion is allowed.)

Blockers are shown on both surfaces; the confirm action is hidden/disabled
while any exist.

---

## okta-web (platform) surfaces & code

All under `okta-web`:

- **Services** — `app/Services/UserManagement/AccountDeletion/`:
  `CheckAccountDeletionEligibility`, `RequestAccountDeletion`,
  `CancelAccountDeletion`, `ProcessDueAccountDeletions`,
  `SendAccountDeletionOtp`, `VerifyAccountDeletionOtp`.
- **Web UI** — the account-settings "danger zone" (`pages/account/appearance`)
  opens the `App\Livewire\Account\DeleteAccountModal` (wire-elements modal).
  After a request the user lands on the **deletion-pending** page
  (`account/deletion-pending`, route `account.deletion-pending`) with *cancel*
  + *sign out*.
- **Fence** — `EnsureAccountNotPendingDeletion` (appended to the `web` group)
  redirects a pending user to that page for every route except the page itself,
  logout, and Livewire AJAX (so cancel works). For JSON it returns `423 Locked`.
- **Notification** — `AccountDeletionScheduledNotification` (mail + database)
  tells the user when the purge is scheduled and how to stop it.
- **Scheduler** — `accounts:process-deletions` runs daily (`routes/console.php`)
  → `ProcessDueAccountDeletions` → `DeleteUser` per due account.

## Mobile API (the bridge to okta-app)

`okta-web` exposes a Sanctum-authenticated surface at `/api/mobile/account`
(`MobileAccountController`). It is deliberately **not** behind the
`mobile.min-version` gate so even an outdated client can still delete:

| Method & path                                   | Purpose                              |
|-------------------------------------------------|--------------------------------------|
| `GET  /api/mobile/account/deletion`             | status: pending? scheduled_at, grace_days, `confirmation`, `can_delete`, `blockers[]` |
| `POST /api/mobile/account/deletion/send-otp`    | send the confirmation code (passwordless) |
| `POST /api/mobile/account/deletion/request`     | schedule deletion (`password` or `otp_code`, optional `reason`) |
| `POST /api/mobile/account/deletion/cancel`      | cancel a pending deletion            |

## okta-app (client) surfaces & code

All under `okta-app`, feature `lib/features/account/`:

- **Data** — `DeletionStatus` model + `AccountRepository` (status / send-otp /
  request / cancel) over the mobile API above.
- **State** — Riverpod `accountRepositoryProvider` +
  `accountDeletionStatusProvider` (autoDispose).
- **UI** — `DeleteAccountScreen`: the grace-period warning, password/OTP
  confirmation, blocker alerts, and — once scheduled — the pending state with
  *cancel* + *sign out* (sign-out clears the token and the router returns to
  login). Reached from a **Settings → Account → Delete account** tile; routed at
  `/account/delete` (available whenever signed in).

---

## Configuration

| Key (`PlatformSetting`)        | Default | Effect                          |
|--------------------------------|---------|---------------------------------|
| `accounts.deletion.grace_days` | `30`    | days between request and purge  |

---

## Quick reference

- Pending ⇔ `users.deletion_scheduled_at IS NOT NULL` (status stays `active`).
- Cancel is always possible during the window by signing in.
- The purge reuses `DeleteUser`, so it inherits the same audit guarantees.
- Confirmation = password, or OTP for passwordless phone accounts.
- Blocked for the super admin and the sole admin of a paying tenant.
