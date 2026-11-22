# myfitness-admin-webapp

Internal ops console for MyFitness/Uppra. **Status: scoped 2026-11-22, not built** — this repository
currently holds only the scaffold and the governing spec. Decisions are registered in
`myfitness-api-backend/docs/decisions/README.md` (entries 89 and 90).

## Scope (v1)

The OTP admin console only:

- Look up an account by uid or email.
- See OTP status: enabled/disabled, enrollment date, email at enrollment, backup codes remaining,
  trusted-device count and last trust date, plus account email, verification and last sign-in.
- Disable OTP — identity-checked, mandatory reason, audit row (actor, target, before/after,
  timestamp); the user must re-enroll afterwards.
- Revoke one trusted device or all of them — mandatory reason, audit row.
- View the recent audit rows for the account.

Never in the UI: displaying, recovering or resetting codes or backup codes. Regeneration happens only
from the user's own verified session.

Out of scope for v1: entitlements and the wider ops console (those stay on the Retool/RBAC path per
Decision 18), admin email change, bulk actions, non-English copy.

## Spec

[`docs/email-otp-second-factor.md`](docs/email-otp-second-factor.md) — the cross-surface OTP spec,
mirrored from `myfitness-trainer-webapp/docs/email-otp-second-factor.md`. The Phase 4 section is this
app's functional spec. Where the two copies disagree, the trainer-webapp copy wins and this one is
re-synced.

## Access

- Sign in with Google only.
- Every callable enforces the existing `ops_staff` claim server-side (`canAdministerOtp` in
  `myfitness-api-backend/src/domain/otp.ts`); the client checks the claim only for routing.
- A staff account gets `ops_staff` set through the Admin SDK (one-off, documented script); the claim
  appears after a fresh sign-in. No self-service and no UI for it.

## Planned stack

- Next.js App Router + TypeScript + Tailwind, deployed on its own Firebase App Hosting backend in
  `myfitness-test` (mirrors the trainer webapp's pipeline).
- Firebase web SDK for sign-in only; no direct Firestore access. All reads and writes go through
  callables behind one BFF route (`status` / `disable` / `revoke`) that forwards machine failure
  codes only — upstream prose never reaches the browser.

## Backend prerequisites

Already live in `myfitness-test`: `adminDisableOtp`, `adminRevokeTrustedDevices`, plus the six
coach-facing OTP callables (contract 0.17.0).

Still required: an `adminGetOtpStatus` read callable and its DTOs (contract 0.18.0), because nothing
today returns the device list, enrollment date, email at enrollment or last-trust date to an
operator.

## Repository conventions

Branch model follows Decision 31 (`main` + long-lived `offset`, `offset` as the sole development
base) once development starts. Pushes, PRs and merges each need the owner's explicit go-ahead.
