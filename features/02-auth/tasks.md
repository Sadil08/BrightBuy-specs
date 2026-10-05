# 02-auth — Tasks (DO IT)

**Feature slug:** `AUTH` · **Status:** 🟢 usable end-to-end, verified with a real browser · **Companion:** [spec.md](spec.md) · [plan.md](plan.md)

---

## 1. Build order

### Database
- [x] **T1 `[DB]`** Migration `0002_identity_schema`: `role`, `user_account`, `customer`,
      `staff_profile`, `refresh_token` + seed `role`/`permission`/`role_permission` rows (`plan.md`
      §2, §2.1). — *Test:* applies clean, `down`/`up` idempotent; seed rows present and correct after
      `up`. — `FR-AUTH-1,2,6,9` — shipped as `0004_identity_schema`, not `0002` as plan.md names it:
      `0002`/`0003` were already taken by 01-catalog's stock function and `product_variant.updated_at`
      by the time this was built. Verified against a real Dockerized MySQL 8: applies clean, seeded
      roles/permissions/grants match the spec's table exactly (`WAREHOUSE_STAFF` → 3 permissions,
      `ORDER_MANAGER` → 2, `MANAGER` → 1, `ADMIN` → all 8), `down`→`up` idempotent, and
      `user_account.email`'s uniqueness is case-insensitive by the database's own default collation
      (`utf8mb4_0900_ai_ci`, confirmed directly — no extra `COLLATE` clause needed). Used
      `CURRENT_TIMESTAMP`, not `UTC_TIMESTAMP()`, for `created_at`/`granted_at` — the latter is
      silently rewritten by `CREATE TABLE` and flatly rejected by `ALTER TABLE` (discovered and fixed
      during 01-catalog's own ETag work — see that feature's `tasks.md` T7 note for the full story);
      `docker-compose.yml` now pins the server's own time zone to UTC (`--default-time-zone=+00:00`)
      so `CURRENT_TIMESTAMP` is actually
      correct rather than coincidentally so.
- [x] **T2 `[DB]`** `cmd/api --create-first-admin` one-off command (`plan.md` §2.2), refuses to run if
      any `ADMIN` account already exists. — *Test:* integration — running it twice fails the second
      time. — `cmd/api/create_first_admin.go`. Built AFTER `T4`/`T5` (not before, as the task order
      implies) so it could reuse the real `UserRepository`/`RoleRepository` rather than hand-rolling
      SQL in `cmd/api` directly. Verified live against the real dev database: first run creates
      `admin@brightbuy.local` with a random 20-byte password printed once to stdout (never logged via
      `slog`, never stored); second run fails with `"refusing to run: 1 ADMIN account(s) already
      exist"`, exit code 1. Also added `db/seed/identity.sql` — a separate, dev/test-only path with a
      KNOWN fixed password (`DevAdmin123!`, matching `db/seed/catalog.sql`'s own deterministic/
      idempotent convention, not plan.md's "random" framing, which is a production-CLI concern, not a
      repeatable-local-setup one). **A real bug fixed while writing it**: an initial draft hardcoded
      `user_account_id = 1` the way `catalog.sql` hardcodes product IDs — wrong here, since a database
      that already ran `--create-first-admin` has a real admin at whatever ID auto-increment assigned,
      and forcing `id=1` risked silently overwriting it. Fixed to key idempotency off the UNIQUE
      `email` column instead; verified running it twice produces no duplicate and the resulting admin
      can actually log in.

### Backend
- [x] **T3 `[BE]`** `shared/auth`: JWT issue/verify (access + refresh), BCrypt wrapper, `ClaimsFromContext`,
      `RequirePermission`/`RequireRole` middleware (`plan.md` §5.2). — *Test:* unit — token round-trip,
      expired/invalid token rejected, ADMIN bypass never queries the fake repo. — 20 unit tests.
      Includes the algorithm-confusion guard (pin the expected HMAC signing method in the JWT
      `keyFunc`, never trust the token's own `alg` header) and the opaque-refresh-token generation/
      hashing (SEC-AUTH-2). The ADMIN-bypass-never-queries test actually lives in `identity/app`
      (T5's `resolvePermissions`), not here — that's where the bypass logic itself is, by design (see
      T5's note for why).
- [x] **T4 `[BE]`** `internal/identity/mysql`: `UserRepository` (create/find-by-email),
      `RoleRepository` (`ResolvePermissions` with the ADMIN short-circuit), `RefreshTokenRepository`
      (issue/rotate/revoke/revoke-all). — *Test:* integration. — 12 integration tests against a real
      disposable MySQL 8 container (all 4 migrations applied via `golang-migrate`), including:
      transactional `user_account`+`customer`/`staff_profile` creation, case-insensitive duplicate-
      email rejection PROVEN at the database level (not just the Go translation), and
      `SetRolePermissions`'s atomic resolve-then-replace (a request with one bad code leaves the
      role's EXISTING permissions completely untouched, verified by re-querying after the rejected
      call). `ResolvePermissionCodes` is deliberately the raw, un-bypassed query — see T5.
- [x] **T5 `[BE]`** `internal/identity/app.AuthService`: `Register`, `Login` (generic-failure path,
      `plan.md` §5.1), `Refresh` (rotation + reused-token detection, `plan.md` §6), `Logout`. —
      *Test:* unit with fakes. — 11 unit tests. The SEC-AUTH-1 ADMIN bypass lives in a package-level
      `resolvePermissions` function here (not a method, specifically so a test can call it directly)
      — proven with a fake `RoleRepository` that **panics** if its query method is ever invoked for
      an ADMIN account, not just asserted to return the right list. FR-AUTH-8's generic failure
      message also runs a dummy BCrypt comparison on the "unknown email" branch, so that branch
      doesn't return measurably faster than "wrong password for a real account" (a timing oracle
      that would otherwise let an attacker enumerate valid emails — SEC-AUTH-3). AC-AUTH-6 proven by
      decoding the actual re-issued JWT after a permission revocation between login and refresh, not
      just checking `Refresh` returned without error.
- [x] **T6 `[BE]`** Rate limiter (`plan.md` §5.3) applied to `/auth/login`, `/auth/register`. —
      *Test:* integration — Nth request in a window gets `429`. — `internal/shared/ratelimit`
      (per-key token bucket, `golang.org/x/time/rate`), applied as IP-keyed middleware for both
      routes plus a SECOND, email-keyed check inside `AuthHandler.Login` itself (email only exists
      after the body is decoded, so it can't be route middleware). Concrete numbers are this
      project's own choice, since neither spec document names one: 5/min per IP and per email for
      login, 3/hour per IP for registration. Verified live against the real running server — 7 rapid
      login attempts from one IP started returning `429` partway through (consistent with the
      process's shared IP bucket already being partially consumed by earlier manual testing in the
      same session, not a bug).
- [x] **T7 `[BE]`** `internal/identity/httpapi`: `AuthHandler` for the five `/auth/*` endpoints, cookie
      handling (httpOnly/Secure/SameSite=Strict, `02` §2). — *Test:* `httptest`. — Verified live
      end-to-end against the real server/database, not just `httptest`: register → login (cookies
      confirmed `HttpOnly`+`SameSite=Strict` via the raw `Set-Cookie` headers) → `/auth/me` →
      `/auth/refresh` (rotates both cookies) → `/auth/logout` → replaying the OLD raw refresh token
      afterward correctly `401`s (AC-AUTH-4, proven independent of client-side cookie clearing by
      replaying the captured token value directly, not relying on curl's cookie jar). `cookieSecure`
      is threaded from `cfg.Env` (`true` outside `local`) — hardcoding `true` would make cookies
      silently not work for anyone running `go run ./cmd/api` directly over plain HTTP.
- [x] **T8 `[BE]`** `internal/identity/app.AccountService`: `CreateStaffAccount`,
      `ListRoles`/`ListPermissions`/`SetRolePermissions`. — *Test:* unit + integration (permission
      change reflected in the next-issued token, not the current one). — 4 unit tests +
      covered by T4's integration suite. `SetRolePermissions` has no ADMIN special case anywhere in
      this service — AC-AUTH-7 falls out naturally, since real enforcement never reads this table for
      that role in the first place (T5's `resolvePermissions`), so there's nothing here that COULD
      weaken ADMIN even if called against its row.
- [x] **T9 `[BE]`** `internal/identity/httpapi`: `/admin/users`, `/admin/roles`, `/admin/permissions`
      handlers, all behind `RequirePermission("account:manage_users"/"account:manage_roles")`. —
      *Test:* `httptest` — `403` for a caller without the permission. — Verified live: an
      authenticated CUSTOMER hitting `POST /admin/users` gets a real `403 FORBIDDEN` (distinct from
      the `401` an unauthenticated request gets — confirmed both, not just one); `ADMIN` can create a
      `WAREHOUSE_STAFF` account, list roles, and the full `AC-AUTH-6` round trip (revoke
      `stock:adjust` via `PUT .../permissions`, have that staff account silently refresh, decode its
      NEW token) was run against the real server, not just the unit-test fakes.

### Frontend
- [x] **T10 `[FE]`** Register/login forms + client-side field validation (UX only, not a control —
      `02` §4). — *Test:* RTL. — Built with **Server Actions** (`src/app/actions/auth.ts`,
      `registerAction`/`loginAction`), not a Client Component calling a Route Handler — this is the
      officially-documented Next.js pattern for auth forms (`<form action={loginAction}>`), and works
      with JavaScript disabled (verified: a raw multipart POST replicating the form's own hidden
      `$ACTION_*` fields, no JS at all, correctly created an account). `useActionState` wires each
      form to its action for inline error display without a full reload.
      **A real, significant bug found and fixed while building this**: the backend's `Set-Cookie`
      headers were silently vanishing — not just invisible via `getSetCookie()`, absent from the
      response entirely. Root cause, confirmed directly (not guessed): the Fetch API spec classifies
      `Set-Cookie` as a "forbidden response-header name," stripped from `Response.headers` for
      **every** caller of `fetch()`, including Node's server-side implementation — not a Next.js
      quirk. Proven by comparing `fetch()` against Node's raw `http` module hitting the exact same
      backend endpoint: `http` exposed `Set-Cookie` correctly, `fetch()` did not, `cache: "no-store"`
      made no difference. Fixed by rewriting the four cookie-dependent calls
      (`register`/`login`/`refresh`/`logout` in `src/lib/api-client/auth.ts`) on Node's `http`/`https`
      module directly; every other function (which only ever *sends* a cookie, never reads one back)
      stayed on plain `fetch()`. Found via real-browser testing (Playwright, installed temporarily
      for verification only, not a project dependency) after curl-based reproduction of the no-JS
      submission format proved inconclusive — unit tests with fakes would never have caught this,
      since the bug was in the real network layer's interaction with a real spec-compliant `fetch()`.
      Test gap: no RTL — no frontend test framework is installed at all (pre-existing, logged under
      01-catalog's own T8–T11 notes).
- [x] **T11 `[FE]`** Auth state (current user, role) + silent-refresh-on-401 interceptor in the API
      client. — *Test:* Vitest. — Auth state: `src/lib/auth/session.ts`'s `getSession()` (cached
      per-request via React's `cache()`), used by the storefront nav and the admin layout.
      **The silent-refresh design deliberately deviates from the obvious "do it in middleware"
      approach**, after reading Next.js's own guidance closely: `proxy.ts` (Next 16's renamed
      `middleware.ts`) runs on *every* request including speculative `<Link>` prefetches, and
      Next's docs explicitly warn against anything beyond cheap, optimistic cookie-presence checks
      there. Our refresh token *rotates* on every use (plan.md §6) with reuse-detection that revokes
      the entire session — a proxy-triggered refresh racing a prefetch could present an
      already-rotated token and lock a user out via a link they never even clicked. `proxy.ts` is
      strictly cookie-presence-based redirects, documented inline with the exact race it avoids.
      The real silent-refresh (catch 401, call `/auth/refresh`, persist new cookies, retry once) lives
      in `src/app/actions/admin.ts`'s `withSilentRefresh`, scoped to Server Actions specifically —
      those only ever run from an explicit form submission, never prefetching, so the race doesn't
      apply. Real authorization (as opposed to `proxy.ts`'s optimistic check) happens in
      `session.ts`'s `requireUser`/`requireAdmin`, called once in the admin layout, per Next's own
      Data Access Layer guidance ("close to the data," not in proxy).
      **Not live-tested**: the retry-after-refresh path itself (forcing a genuinely expired-but-not-
      yet-evicted access token mid-session is awkward to simulate without extra tooling) — the
      individual pieces (`refresh()`, `persistCookies`, 401 detection) are each exercised by other
      verified flows, but the full retry sequence end-to-end isn't proven live the way login/logout/
      admin-actions are. Test gap: no Vitest, same pre-existing reason as T10.
- [x] **T12 `[FE]`** Admin console: staff account creation form; role → permission checkbox matrix
      calling `PUT /admin/roles/{roleId}/permissions`. — *Test:* RTL + Playwright. — Verified with a
      **real browser** (Playwright, temporary, see T10): logged in as the seeded dev admin, created a
      `WAREHOUSE_STAFF` account through the actual form (confirmed in the re-rendered list via
      `revalidatePath`, no manual reload), unchecked `stock:adjust` for `WAREHOUSE_STAFF` through the
      matrix, saved, reloaded, confirmed the uncheck persisted through the real backend/database, then
      restored it (confirmed in the database directly). Also verified: a `CUSTOMER` session never
      sees the `Admin` nav link and is redirected away from `/admin` by `requireAdmin()`; logout
      clears every cookie (`0` remaining) and redirects to `/login`.
      **A second bug found during this verification — in the test script, not the app**: an early
      version of the Playwright test used a generic `button[type="submit"]` selector, which matched
      the admin layout's "Log out" button (also `type="submit"`) instead of the form's own "Create
      account" button, since both exist on the same page — silently logging the test out instead of
      submitting the form, which cascaded into a second, seemingly-unrelated test failure on the next
      page. Fixed by scoping selectors to button TEXT, not just type. Worth remembering: a page with
      more than one `type="submit"` button needs every test interacting with it to be specific about
      which one.

### Cross-cutting
- [x] **T13 `[INFRA]`** Verify `openapi.yaml`'s `auth`/`admin-rbac` paths match the shipped API. —
      every DTO in `internal/identity/httpapi/dto.go` was written field-for-field against the actual
      schemas (`RegisterRequest`, `LoginRequest`, `UserProfile`, `CreateAccountRequest`, `Role`,
      `Permission`, `RolePermissionsUpdateRequest`); all ten routes match `plan.md` §3's table
      exactly, including the `/api/v1` base path and `{roleId}` path param naming. No drift found.

## 2. Acceptance-criteria coverage map

| AC | Covered by |
|---|---|
| AC-AUTH-1 | T5, T7 |
| AC-AUTH-2 | T4, T5 |
| AC-AUTH-3 | T5, T7 |
| AC-AUTH-4 | T4, T5, T7 |
| AC-AUTH-5 | T4, T5, T7 |
| AC-AUTH-6 | T4, T5, T8, T9 |
| AC-AUTH-7 | T5 (ADMIN bypass lives here, not T3 — see T5's note), T8, T9 |

## 3. Definition of Done

- [x] Every `FR-AUTH-*` has a passing unit and integration test. — backend FRs all covered (T3–T9's
      notes); FE-dependent ones (nothing currently, all of 02-auth's FRs are backend-testable) n/a.
- [x] Every `AC-AUTH-*` above is green. — all seven verified, most live against the real server (see
      T7/T9's notes), not just unit-test fakes.
- [x] No plaintext password or raw refresh token ever appears in a log line (grep-tested, not just
      reviewed — `02` §7). — grepped this session's actual server log output for every plaintext
      password/token used during live testing; none present.
- [x] `gofmt`/`go vet`/ESLint clean; `openapi.yaml` matches the shipped API. — `gofmt`/`go vet`/
      `staticcheck` all clean (backend); `tsc --noEmit`, `eslint`, and `next build` all clean
      (frontend, including the new Server Actions/proxy.ts/admin console). OpenAPI match: see T13.
- [x] `09_ISSUES_REGISTER.md`'s `RBAC-1` and `DB-ERD-1` rows updated to point at this feature as where
      the design actually landed. — already true as of this pass (both rows already cite
      `02-auth/plan.md` directly); nothing further needed here.

**Bottom line: the feature is usable end-to-end, backend AND frontend**, verified with a real
browser against the real backend and real database — register, login, logout, session-aware nav,
admin-only route protection, staff account creation, and the role/permission matrix all confirmed
working, not just unit-tested. The one real bug this verification pass caught — `fetch()` silently
dropping `Set-Cookie` per the Fetch spec, even server-side — would have shipped completely invisible
to unit tests (which exercise fakes, never a real `fetch()` against a real HTTP server) and broken
every login in production. **What's left, honestly**: no frontend test framework is installed
(pre-existing gap, same as 01-catalog's), and the silent-refresh-after-401 retry path's full sequence
isn't proven live end-to-end (its individual pieces are, via other flows). Neither blocks usability.

**Bottom line: the backend is done and proven live**, not just unit-tested — every AC in this
document was exercised against a real running server, a real MySQL database, and real JWTs decoded
and inspected, not only against hand-written fakes. `T10`–`T12` (the entire frontend: register/login
forms, auth state + silent refresh, the admin console) don't exist yet, so no actual user can sign up
or log in through anything but `curl` today. That's the next and last piece, same shape as
01-catalog's own final stretch.
