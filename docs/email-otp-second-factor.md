# Email OTP second factor — requirements

Status: agreed 2026-11-22. Applies to `myfitness-trainer-webapp`, both apps in `myfitness-mobileapp` (client, trainer), and `myfitness-api-backend`. Provider: Firebase project `myfitness-test`.

## Scope

- Opt-in second factor for accounts that sign in with email and password.
- Google sign-in is exempt: it bypasses the code.
- One account setting applies to all three surfaces.
- Clients have no webapp; enrollment and challenge must work entirely in the mobile client app.
- Admin disable is out of scope for these three apps. It is specified below for the future `myfitness-admin-webapp`.

## Decisions

1. Opt-in per account through a Security toggle in account settings in the webapp and both mobile apps. Enabling requires the current password plus one successfully entered code. The account email must already be verified.
2. The code is required on an unrecognized device, browser, or app install. A verified device is trusted for 30 days, then challenged again.
3. Code format: 6 numeric digits, 10-minute TTL, single-use, stored only as a hash. Resend cooldown 60 seconds, maximum 5 sends per hour per account.
4. Five wrong codes invalidate the challenge and lock further attempts for 15 minutes.
5. Backup codes: 10 single-use codes issued at enrollment, shown once, usable at sign-in in place of the email code. Regeneration requires a fresh code and invalidates the previous set.
6. Beyond sign-in, a code is also required to change the account email and to disable OTP. Password changes require the password only. Password reset is unchanged; a sign-in after reset still hits the OTP step if enabled.
7. Google sign-in bypasses the code even when the account also has a password.
8. Enabling OTP revokes refresh tokens on all other devices. Disabling OTP also revokes them.
9. Email language comes from the account preference (`users/{uid}.locale`), falling back to the locale of the app where sign-in started, then `en-GB`. All 12 locales are required.
10. The second factor is server-enforced. A verified claim gates Firestore/Storage rules and backend callables; a modified client cannot skip it.
11. Admin disable (identity-checked, reason required, audit-logged) is a requirement for `myfitness-admin-webapp`. The backend action ships with this work; no admin UI appears in the existing three apps.

## Assumed defaults

- Feature is off for every existing account; no migration or forced enrollment.
- Trust is recorded per app installation or browser, so the trainer and client apps on the same phone remember separately.
- The code email contains the code, its expiry, and a "if this was not you" warning. No links. Existing Resend sender and branding are reused.
- Behavior is identical for coaches and clients; no workspace-level policy or coach enforcement.
- Enabling in one app affects sign-in on all three surfaces.
- `users/{uid}.locale` stays the single account-level language source; both mobile apps already mirror their locale into it.

## Enforcement model

- Accounts with OTP enabled carry an `otp` claim. A session counts as verified only when the token's `otp_at` claim is at or after its `auth_time`; a new password sign-in advances `auth_time` and the old verification no longer satisfies it.
- After a successful code (or trusted-device) check, the backend stamps `otp_at` and the client force-refreshes its ID token.
- Security state (enabled flag, backup-code hashes, device trust, challenges) lives in server-only documents, never in `users/{uid}`, which clients can write.
- Trusted-device checks are a backend callable; the client cannot mark itself trusted.
- Changing the account email is routed through a backend action (Admin SDK) after an OTP step sent to the current email, replacing the client-side email update. Otherwise a stolen session could redirect future codes.
- Callables that mutate data require the same fresh-claim check as Firestore rules.

## Admin requirements — future `myfitness-admin-webapp`

This section is the functional spec for the admin app; it does not describe UI in the three existing apps.

- Look up an account by uid or email and view OTP status: enabled/disabled, enrollment date, email at enrollment, trusted-device count and last trust date.
- Disable OTP only after identity verification. Require a reason. Write an audit row with actor, target, action, reason, before/after state, and timestamp. The user must re-enroll afterwards.
- Revoke a single trusted device or all trusted devices.
- Never display, recover, or reset the user's codes or backup codes; regeneration always happens from the user's own verified session.
- Admin access restricted to the existing `ops_staff` claim.

## Repo impact summary

- `myfitness-api-backend`: code issuance, mail in 12 locales, challenges, device trust, backup codes, claims, rules, admin action, tests.
- `myfitness-trainer-webapp`: callable routes, post-sign-in gate, verify screen, Security settings card, email-change flow, 12-locale copy.
- `myfitness-mobileapp` (`packages/data`, `packages/design_system`, both apps): session OTP stage, code screen, Security settings, device id, 12-locale copy, contract DTOs.
- `myfitness-admin-webapp` (future): status lookup, disable and trust revocation with audit.

## Phase 0 result (2026-11-22)

Validated against the Auth and Firestore emulators with `myfitness-api-backend/tests/qa/otp-spike/claims-spike.mjs`:

- Password ID tokens carry a numeric `auth_time`, and Firestore rules can read it.
- Admin-set `otp` / `otp_at` claims appear after `getIdToken(true)`; `auth_time` does not change on refresh.
- The rule `otp_at >= auth_time` denied a stale claim, allowed the verified session, denied again after a new sign-in, and allowed after re-verification.
- Missing `auth_time`, non-integer `otp_at`, and absent/false `otp` are handled without rule errors.
- The `beforeUserSignedIn` fallback is not needed.

## Validation items before build

- Confirm the contract pipeline used to generate `packages/contract_dart` DTOs from the backend contract.
- The backend currently never reads `users/*`; add an Admin-SDK reader for the account locale.
- The webapp changes email through `updateEmail` in `src/lib/firebase/auth.ts:197`; that path is replaced, not just hidden.
- Mobile apps have no sign-up and no deep links; codes are typed in-app, so no messaging or link infrastructure is added.

## Phase 1 result — backend (2026-11-22)

Implemented in `myfitness-api-backend`:

- Contract: `OtpPurpose` (`signin` | `enroll` | `disable` | `backup_codes` | `email_change`), `OtpAppSurface` (`web` | `trainer` | `client`), and the seven callable DTOs; OTP limits live in `LIMITS` (code length/TTL, cooldown, hourly cap, attempts/lock, trust window, backup-code shape, recent-auth window, device-id bounds, admin reason cap).
- Callables: `getOtpStatus`, `startOtpChallenge`, `verifyOtpCode`, `verifyBackupCode`, `checkTrustedDevice`, `adminDisableOtp`, `adminRevokeTrustedDevices`.
- Rules: Firestore and Storage gate on `otpFresh()`; `userSecurity/**`, `otpChallenges/**`, `otpSendState/**` and `serverSecrets/**` are server-only. QA rules registry v0.8 adds OT-01..06.

Implementation deltas from the decisions above:

- The social-provider bypass is evaluated from the verified `firebase.sign_in_provider` claim (rules and the callable guard), not from a callable; a password-only attacker cannot self-attest.
- Enabling, disabling and changing email revoke refresh tokens on other devices and return `reauthRequired`; the client signs out and re-authenticates, skipping the code on a trusted device. Without this, the acting session would keep a token that no longer satisfies the gate.
- A stale session may still read its own `users/{uid}` document (locale for the OTP screen) but cannot write it; all workspace data stays gated.
- Email change is a generic `email_change` OTP purpose sent to the current address: the backend applies the new address with the Admin SDK, audits `account.emailChanged`, revokes sessions and returns `verificationRequired` for the new address. No separate request/complete DTOs were added.
- Enrollment requires a password provider, a verified email and authentication within the last five minutes.
- Backup codes are 10 single-use codes of 10 characters from an ambiguity-free alphabet, displayed grouped; regeneration replaces the whole set inside one transaction.
- Admin callables accept only the `ops_staff` claim and a reason of up to 500 characters; disable and revoke are audit-logged (`otp.adminDisabled`, `otp.trustedDevicesRevoked`).
- Local tooling: `npm run test:rules` runs CI mode via a `--ci` flag because Windows `cmd` does not support inline `VAR=1` prefixes; CI's `ci-precheck.mjs` still sets `QA_RULES_EXPECT_RUN=1`.

Verification (2026-11-22): lint and build clean; root unit tests 387 passed / 235 skipped (L2 suites need emulators); package tests 18 passed; rules suite against Firestore/Auth/Storage emulators total=89 executed=89 passed=89 failed=0 skipped=0 in CI mode (82 rules cases + 7 always-run, registry total 90 including 7 tracked-pending L2-callable entries).

## Phase 2 result — trainer webapp (2026-11-22)

Implemented in `myfitness-trainer-webapp` (branch `offset-feature-01`, not deployed):

- Transport: `src/app/api/auth/otp/route.ts` is the single BFF forwarder. It accepts a closed action set (`status` | `start` | `verify` | `backup-code` | `trusted`), pins `app: 'web'`, requires a bearer token, and returns machine failure codes only (`unauthorized`, `stale`, `invalid`, `expired`, `locked`, `email-in-use`, `unavailable`, `failed`). Upstream prose never reaches the browser.
- Client helpers: `src/lib/firebase/otp.ts` mirrors the OTP DTOs from contract 0.17.0 (published 2026-11-22; the webapp dependency stays pinned at 0.15.1 until a deliberate bump), plus the per-browser device id (`localStorage` `uppra-device-id`, 32 hex), the claims reader/gate predicate, the challenge cache (`sessionStorage` `uppra-otp-challenge:<purpose>`), and backup-code input normalisation. Switch to contract types once the pin moves.
- Auth adapter: `getOtpClaims`, `refreshIdToken` and `reauthenticate(password)`; reauthentication mints a fresh token so the enrollment callable's five-minute window sees the advanced `auth_time`.
- Gate: `AuthProvider` decodes claims locally; a fresh session costs no network. A stale token asks `checkTrustedDevice` once, then refreshes and clears, or sends the user to `/verify-otp`. `unavailable` releases the shell (rules stay the real gate). `AuthGate` redirects before the email-verification gate; `AppShell` holds protected content and navigation while the decision is pending.
- Screens: `/verify-otp` (`VerifyOtpPanel`) and the Settings "Security" card (`SecuritySettings`) with enable, disable, backup-code regeneration and email change; `OtpCodeForm` handles code entry, resend countdown and backup-code fallback. All copy is localized through new top-level `otp` and `security` namespaces in the 12 message catalogs.

Implementation deltas from the decisions above:

- Email change in the webapp no longer uses `updateEmail`; it runs through `email_change` + the backend Admin-SDK write. The client signs out afterwards because refresh tokens were revoked.
- An undeployed callable (bare HTML 404 from the functions host) maps to `unavailable`, not `expired`; only a callable-protocol 404 (traveling `NOT_FOUND` token) means a dead challenge. This keeps rollout order safe.
- The BFF route shares one forwarder for all actions instead of per-action routes.

Verification (2026-11-22): lint, `tsc --noEmit` and production build clean (build run in a throwaway tree copy so the running dev server kept `.next`); unit tests 482 passed across 63 files, including new suites for the OTP lib, the BFF route mapping, the auth adapter additions and the verify panel. Browser checks on `localhost:3000` (signed out): `/settings` and `/verify-otp` redirect to `/signin`; `POST /api/auth/otp` answers 401 without a bearer token and 503 `unavailable` against the `myfitness-test` deployment as it stood (the Phase 1 callables have since been deployed there, so the route now answers the callable protocol), never upstream prose. Authenticated flows (gate, enable/disable, backup codes, email change) were not exercised in the browser because no test credentials are available locally — they are covered by the unit suites and the backend rules/emulator tests.

## Phase 3 result — Flutter apps (2026-11-22)

Implemented in `myfitness-mobileapp` (both apps; not committed, not built to a device):

- `domain`: pure OTP vocabulary (`OtpFailure`/`OtpFailureKind`, `OtpChallengeInfo`, `OtpSecurityOverview`, `OtpSecurityPort`) so the design system renders typed failures only.
- `data`: contract-0.17.0 DTOs hand-mirrored (the melos contract pin is still 0.15.1; wire spellings `backup_codes`/`email_change` are pinned in one map), `FunctionsOtpService` (region `europe-west2`), the same per-action failure classification as the webapp BFF, `PrefsDeviceIdStore` (`SharedPreferences` `otp-device-id`, 32 hex, storage failure falls back to a session id), the claims freshness predicate, `OtpSignInGate` (send/verify/backup/sign-out) and `OtpSecurityActions` (the `OtpSecurityPort` adapter). `CoachSession` gained `idTokenClaims`, `refreshIdToken` and `reauthenticate`; the token refreshes after every successful verify or trust check.
- Boot: `CoachBootFlow` runs an OTP stage before workspace discovery — a fresh stamped session costs no network, a trusted device refreshes and continues, an untrusted device lands on `CoachBootStage.otpRequired`. `unavailable` releases the shell (rollout order: rules remain the server-side gate).
- Screens: `CoachOtpScreen` + shared `OtpCodeEntry` (masked email notice, 6-digit entry, backup-code fallback, wall-clock resend cooldown, typed error line, "sign in with a different account") and `SecuritySettingsPage` (enable with password re-auth, disable/replace behind destructive confirms, one-time backup-code display with copy, trusted-until line). Both apps push the Security page from Settings and re-boot after sign-out from the code screen.
- Copy: 48 new keys across the 12 locales in `mobile_extra_keys.dart` / `mobile_extra_european.dart` / `mobile_extra_nordic.dart` / `mobile_extra_asian.dart`, append-only; the parity test covers key sets, placeholders, uniqueness and the English-differs rule.

Implementation deltas from the decisions above:

- No mobile email change and no deep links on the code screen: the docs' web-only email-change decision stands, and mobile has no `/verify-otp` route to link into — sign-in lands on the code stage directly.
- The app's own locale tag rides each `startOtpChallenge` call; the backend still prefers the account locale.
- The five test `CoachSession` fakes carry the three new methods; widget tests drive every stage through ports and fakes (no Firebase).

Verification (2026-11-22): per-member `dart analyze --fatal-infos` clean and `flutter test` green — domain 14, data 135, design_system 41, client 18, trainer 7 (215 tests), including new suites for the OTP transport mapping, claims/freshness, both gates, the boot stage, the 12-locale parity and widget tests for the code entry, sign-in code screen and Security page. No device or emulator run and no Firebase deploy were performed.

## Phase 4 scope — myfitness-admin-webapp (2026-11-22)

Status: scoped, not built. The repository does not exist yet.

### Decisions

| Topic | Decision |
| --- | --- |
| Surface | New Next.js app `myfitness-admin-webapp`, its own App Hosting backend in `myfitness-test`, same pipeline shape as the trainer webapp. The Retool route used for entitlement admin is not reused. |
| MVP | OTP console only: lookup, status, disable, trust revocation, audit view. Entitlements and other ops tools come later on the same shell. |
| Read depth | New admin read callable plus contract DTOs. Summary-only was rejected because the admin requirements need the device list, last trust date and a real email. |
| Access | Google sign-in only; server-side `ops_staff` check on every callable (already in the backend). English-only internal UI. |

### Backend additions (ship before the admin app)

- Contract 0.18.0 (additive; 0.17.0 is already published with the OTP surface, and the clients' 0.15.1 pins plus the hand-mirrored mobile DTOs are unaffected):
  - `AdminGetOtpStatusRequest { uid?; email? }` — exactly one; email resolved with Admin SDK `getUserByEmail`.
  - `AdminGetOtpStatusResponse { uid; email; emailVerified; accountCreatedAt; lastSignInAt; otp: { enabled; enabledAt; emailAtEnrollment; backupCodesRemaining }; devices: AdminOtpDevice[]; audit: AdminOtpAuditRow[] }`.
  - `AdminOtpDevice { deviceId; app: OtpAppSurface | null; createdAt; lastSeenAt; trustedUntil }`.
  - `AdminOtpAuditRow { action; actor; at; reason }` — the most recent 20 `auditLogs` rows whose `target` is `users/{uid}`; `reason` is lifted from the stored `after.reason` by the backend.
- Callable `adminGetOtpStatus` in `src/infra/functions/otp.ts`, same guard as the other two admin callables (`canAdministerOtp(roleFromClaims(token))`, `ops_staff` only — `platform_owner` stays out, per Decision 11). Unknown uid or email maps to `not-found`. Reads are not written to `auditLogs` (log line only).
- Response whitelisting: the callable must never return backup-code hashes, challenge documents, claims, or `serverSecrets`; only the fields above are projected from `userSecurity/{uid}` and its `devices` subcollection.
- `auditLogs` query needs a composite index on `target` + `at` (descending) — add it to the backend's index config in the same change.
- Small alignment fix while in the file: `adminRevokeTrustedDevices` currently accepts any uid; make it verify the account with `getUser` and map `auth/user-not-found` to `not-found`, matching `adminDisableOtp`.

### Admin app shape

- App Router + TypeScript + Tailwind; Firebase web SDK for sign-in only. No direct Firestore access anywhere — every read and write goes through callables via a BFF route, mirroring `myfitness-trainer-webapp/src/app/api/auth/otp/route.ts`.
- Auth: Google popup, reject/resolve the role from the ID token claims (`role` or the `ops` alias, as `roleFromClaims` does) for routing UX only. A non-staff sign-in shows "this account is not authorised" and signs out. A callable `permission-denied` forces sign-out, because claims can change under a live session.
- BFF `src/app/api/otp-admin/route.ts`: closed action set `status | disable | revoke`, bearer token required, machine failure codes only (`unauthorized`, `forbidden`, `not-found`, `invalid`, `unavailable`, `failed`); upstream prose never reaches the browser. One ID-token refresh-and-retry on 401, then sign-out.
- Screens:
  - `/signin` — Google only.
  - `/` — lookup: one field accepting uid or email (a server `not-found` answers "no account matches").
  - `/accounts/[uid]` — overview card (email, verified, created, last sign-in, OTP enabled/disabled, enrolled date, email at enrollment, backup codes remaining, trusted-device count and last trust date); devices table (app, trusted until, created, last seen, per-row revoke); action bar with **Disable OTP** (only while enabled) and **Revoke all devices**; recent audit list.
- Disable and revoke dialogs: mandatory reason textarea (trimmed, ≤500 characters), destructive button red and right-aligned, and the dialog states the consequence — disable turns the second factor off, signs the user out everywhere and requires re-enrollment; revoke forces a code on every device or on the chosen device at the next sign-in. Both actions refetch status on success.
- No code entry, display, recovery or reset in the admin app. Backup-code regeneration stays in the user's own verified session; the admin screen shows only the remaining count.
- Copy: English-only operational wording; failure codes map to plain sentences (no callable messages, no internal identifiers). No locale machinery is added to this app.
- Provisioning: staff access is a Google account with `ops_staff` set through the Admin SDK (one-off script documented in the app's README; the claim is visible only after a fresh sign-in). No self-service and no UI for it.

### Environments and rollout

- `myfitness-test` first, with App Check in monitoring mode (separate reCAPTCHA Enterprise site key registered for the admin host; the trainer key is not reused). Production mirrors later through the same App Hosting + secret layout.
- Rollout order: backend contract 0.18.0 + callable deploy → admin app deploy. Steps 1–3 (webapp, mobile, backend) are untouched; the contract change is additive.

### Test plan

- Backend: unit tests for the guard (staff, `ops` alias, `platform_owner` denied, unauthenticated), uid/email resolution (found, not found, both or neither supplied), response whitelisting (no hashes, bounded device and audit lists), and the revoke not-found alignment; update the QA rules/callable registry if it enumerates callables.
- Admin app: Vitest for the BFF mapping (all six failure codes, prose never echoed), lookup validation, and the status components (empty devices, disabled state hides Disable, reason required, confirmation copy).
- End-to-end on `myfitness-test` with two real accounts — one staff account carrying `ops_staff`, one target account with OTP enabled and a trusted device: lookup by uid and by email, per-device revoke, revoke all, disable (verify the audit row, the user's next sign-in, and re-enrollment), and a non-staff Google account rejection.

### Out of scope for Phase 4

- Entitlement and other ops screens; admin-initiated email change; code or backup-code display, reset or regeneration; creating trust; bulk lookup or export; non-English copy; admin access from the mobile apps; changes to the coach-facing surfaces.

