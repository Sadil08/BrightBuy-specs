# 02-auth — Spec (WHAT & WHY)

**Feature slug:** `AUTH` · **Status:** ⚠️ draft-for-review · **Companion:** [plan.md](plan.md) · [tasks.md](tasks.md)
**Inherits:** `global/02_SECURITY_BASELINE.md`, `global/03_NFR_BASELINE.md`
**SRS section:** §4.2 User Registration and Authentication (High priority)

---

## 1. Summary

Registration, login, logout, and token refresh for customers; account creation for staff/manager/admin
(no self-registration for those roles); and the dynamic RBAC machinery (`permission`/`role_permission`,
resolved OQ-1) every other feature's authorization depends on. This is the second foundation feature
(`11_BUILD_PLAN.md` Phase 1) — nothing requiring auth can be built before it.

## 2. Actors & scope

- **In scope:** customer self-registration/login/logout/refresh; BCrypt password storage; JWT
  issuance carrying role + resolved permissions; admin-created staff/manager/admin accounts; the
  permission-catalog and role-permission-assignment endpoints.
- **Out of scope (explicit):** password reset/forgot-password (not in SRS §4.2 — logged as OQ-5 below,
  not a Phase 1 blocker since there's no requirement asking for it); email verification (same — not a
  numbered SRS requirement).
- **Working assumptions used:** `RBAC-1` and `DB-ERD-1` (`global/09_ISSUES_REGISTER.md`) — this
  feature is where both are actually implemented.

## 3. User stories

| ID | As a… | I want… | so that… | Priority |
|---|---|---|---|---|
| US-AUTH-1 | visitor | to register with name, email, password, and phone | I can check out later | 🔴 P0 |
| US-AUTH-2 | registered customer | to log in and stay logged in across a session | I don't re-enter credentials on every visit | 🔴 P0 |
| US-AUTH-3 | registered customer | to log out, including "everywhere" | I can secure my account from a shared device | 🟠 P1 |
| US-AUTH-4 | ADMIN | to create a staff account and assign it a role | new hires can use the system on day one | 🔴 P0 |
| US-AUTH-5 | ADMIN | to grant or revoke a specific permission on a role without a redeploy | I can adjust who-can-do-what as the org changes | 🟠 P1 |

## 4. Functional requirements

| ID | Requirement | SRS ref | Derived from | Priority |
|---|---|---|---|---|
| FR-AUTH-1 | The system MUST let a visitor register as a customer with at minimum name, email, password, and phone. | REQ-2.1 | US-AUTH-1 | 🔴 P0 |
| FR-AUTH-2 | The system MUST enforce a unique email per account, at the application layer and via a unique index. | REQ-2.2 | — | 🔴 P0 |
| FR-AUTH-3 | The system MUST store passwords as a BCrypt salted hash and never store or log plaintext. | REQ-2.3 | — | 🔴 P0 |
| FR-AUTH-4 | The system MUST require authentication before checkout is permitted. | REQ-2.4 | — | 🔴 P0 (enforced in `04-checkout-orders`; this feature provides the token that makes enforcement possible) |
| FR-AUTH-5 | The system MUST let an account log out, after which its issued refresh token is no longer accepted. | REQ-2.5 | US-AUTH-3 | 🔴 P0 |
| FR-AUTH-6 | The system MUST assign every account a role and authorize every protected endpoint against it, server-side. | REQ-2.6 | — | 🔴 P0 |
| FR-AUTH-7 | The system MUST enforce a minimum password length of 8 characters at registration and account creation. | REQ-2.7 | — | 🔴 P0 |
| FR-AUTH-8 | The system MUST reject a failed login with the same generic message regardless of whether the email exists. | REQ-2.8 | — | 🔴 P0 |
| FR-AUTH-9 | `ADMIN` MUST be able to create a `WAREHOUSE_STAFF`/`ORDER_MANAGER`/`MANAGER`/`ADMIN` account directly (no self-registration path for these roles). | — (RBAC-1) | US-AUTH-4 | 🔴 P0 |
| FR-AUTH-10 | `ADMIN` MUST be able to view the permission catalog and replace a role's granted-permission set at runtime. | — (RBAC-1) | US-AUTH-5 | 🟠 P1 |
| FR-AUTH-11 | The system MUST issue a short-lived access token and a longer-lived, revocable refresh token, and MUST re-resolve the account's current permissions on every refresh. | — (supports REQ-2.5/2.6) | — | 🔴 P0 |

## 5. NFR deltas (feature-specific only)

None beyond `global/03_NFR_BASELINE.md`. Login/register are explicitly in the rate-limited tier
(`global/02_SECURITY_BASELINE.md` §4) — restated here only because this feature is what implements
that limiter, not because the budget differs.

## 6. Security deltas (feature-specific only)

| ID | Requirement | Closes |
|---|---|---|
| SEC-AUTH-1 | `ADMIN`'s effective permission set is never read from `role_permission` — it is an unconditional code-level bypass, so `PUT /admin/roles/{roleId}/permissions` against the ADMIN role can never lock every admin out. | T-17 |
| SEC-AUTH-2 | The refresh token is stored server-side as a hash (`refresh_token.token_hash`), never in plaintext — a leaked database dump doesn't hand out usable tokens directly. | — |
| SEC-AUTH-3 | Login and registration are rate-limited per IP and per email; failure responses are identical in shape and timing-insensitive enough not to leak which factor (email vs. password) was wrong. | T-5, T-15 |

## 7. Acceptance criteria (Given/When/Then)

- **AC-AUTH-1** — *Given* a visitor submits valid registration data with an email not already in use,
  *When* they submit, *Then* the account is created with role `CUSTOMER` and a `201` is returned.
- **AC-AUTH-2** — *Given* an email already registered, *When* a second registration attempt uses it,
  *Then* the system returns `409` and does not create a duplicate row.
- **AC-AUTH-3** — *Given* a registered customer, *When* they log in with the wrong password, *Then*
  the response is `401` with the same message as if the email didn't exist at all.
- **AC-AUTH-4** — *Given* a logged-in customer, *When* they call logout, *Then* their refresh token is
  revoked and a subsequent `/auth/refresh` with the old cookie returns `401`.
- **AC-AUTH-5** — *Given* `ADMIN` creates a `WAREHOUSE_STAFF` account, *When* that account logs in,
  *Then* its JWT carries `role: WAREHOUSE_STAFF` and the permissions currently on `role_permission`
  for that role — not a hardcoded list.
- **AC-AUTH-6** — *Given* `ADMIN` revokes `stock:adjust` from `WAREHOUSE_STAFF` via
  `PUT /admin/roles/{roleId}/permissions`, *When* a `WAREHOUSE_STAFF` account's access token expires
  and it silently refreshes, *Then* the new token no longer carries `stock:adjust`
  (`02_SECURITY_BASELINE.md` §1.1's bounded-staleness behavior).
- **AC-AUTH-7** — *Given* any caller, *When* they call `PUT /admin/roles/{adminRoleId}/permissions`
  attempting to strip `account:manage_roles` from `ADMIN`, *Then* the write is accepted (for console
  display consistency) but has no enforcement effect — `ADMIN` still bypasses the check (SEC-AUTH-1).

## 8. Open questions / decisions needed

- [ ] OQ-5 (new): Password reset / forgot-password flow — not in SRS §4.2, not built in Phase 1.
      Flag for BrightBuy: is this acceptable for a pilot, or must staff manually reset a customer's
      password via `/admin/users` in the meantime? Working assumption: acceptable gap for Phase 1;
      an admin-triggered reset (no email-based flow) is the fallback if this becomes a real complaint.
