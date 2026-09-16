# 02-auth — Tasks (DO IT)

**Feature slug:** `AUTH` · **Status:** ❌ not started · **Companion:** [spec.md](spec.md) · [plan.md](plan.md)

---

## 1. Build order

### Database
- [ ] **T1 `[DB]`** Migration `0002_identity_schema`: `role`, `user_account`, `customer`,
      `staff_profile`, `refresh_token` + seed `role`/`permission`/`role_permission` rows (`plan.md`
      §2, §2.1). — *Test:* applies clean, `down`/`up` idempotent; seed rows present and correct after
      `up`. — `FR-AUTH-1,2,6,9`
- [ ] **T2 `[DB]`** `cmd/api --create-first-admin` one-off command (`plan.md` §2.2), refuses to run if
      any `ADMIN` account already exists. — *Test:* integration — running it twice fails the second
      time.

### Backend
- [ ] **T3 `[BE]`** `shared/auth`: JWT issue/verify (access + refresh), BCrypt wrapper, `ClaimsFromContext`,
      `RequirePermission`/`RequireRole` middleware (`plan.md` §5.2). — *Test:* unit — token round-trip,
      expired/invalid token rejected, ADMIN bypass never queries the fake repo.
- [ ] **T4 `[BE]`** `internal/identity/mysql`: `UserRepository` (create/find-by-email),
      `RoleRepository` (`ResolvePermissions` with the ADMIN short-circuit), `RefreshTokenRepository`
      (issue/rotate/revoke/revoke-all). — *Test:* integration.
- [ ] **T5 `[BE]`** `internal/identity/app.AuthService`: `Register`, `Login` (generic-failure path,
      `plan.md` §5.1), `Refresh` (rotation + reused-token detection, `plan.md` §6), `Logout`. —
      *Test:* unit with fakes.
- [ ] **T6 `[BE]`** Rate limiter (`plan.md` §5.3) applied to `/auth/login`, `/auth/register`. —
      *Test:* integration — Nth request in a window gets `429`.
- [ ] **T7 `[BE]`** `internal/identity/httpapi`: `AuthHandler` for the five `/auth/*` endpoints, cookie
      handling (httpOnly/Secure/SameSite=Strict, `02` §2). — *Test:* `httptest`.
- [ ] **T8 `[BE]`** `internal/identity/app.AccountService`: `CreateStaffAccount`,
      `ListRoles`/`ListPermissions`/`SetRolePermissions`. — *Test:* unit + integration (permission
      change reflected in the next-issued token, not the current one).
- [ ] **T9 `[BE]`** `internal/identity/httpapi`: `/admin/users`, `/admin/roles`, `/admin/permissions`
      handlers, all behind `RequirePermission("account:manage_users"/"account:manage_roles")`. —
      *Test:* `httptest` — `403` for a caller without the permission.

### Frontend
- [ ] **T10 `[FE]`** Register/login forms + client-side field validation (UX only, not a control —
      `02` §4). — *Test:* RTL.
- [ ] **T11 `[FE]`** Auth state (current user, role) + silent-refresh-on-401 interceptor in the API
      client. — *Test:* Vitest.
- [ ] **T12 `[FE]`** Admin console: staff account creation form; role → permission checkbox matrix
      calling `PUT /admin/roles/{roleId}/permissions`. — *Test:* RTL + Playwright.

### Cross-cutting
- [ ] **T13 `[INFRA]`** Verify `openapi.yaml`'s `auth`/`admin-rbac` paths match the shipped API.

## 2. Acceptance-criteria coverage map

| AC | Covered by |
|---|---|
| AC-AUTH-1 | T5, T7 |
| AC-AUTH-2 | T4, T5 |
| AC-AUTH-3 | T5, T7 |
| AC-AUTH-4 | T4, T5, T7 |
| AC-AUTH-5 | T3, T4, T5 |
| AC-AUTH-6 | T3, T8, T9 |
| AC-AUTH-7 | T3 (ADMIN bypass), T9 |

## 3. Definition of Done

- [ ] Every `FR-AUTH-*` has a passing unit and integration test.
- [ ] Every `AC-AUTH-*` above is green.
- [ ] No plaintext password or raw refresh token ever appears in a log line (grep-tested, not just
      reviewed — `02` §7).
- [ ] `gofmt`/`go vet`/ESLint clean; `openapi.yaml` matches the shipped API.
- [ ] `09_ISSUES_REGISTER.md`'s `RBAC-1` and `DB-ERD-1` rows updated to point at this feature as where
      the design actually landed.
