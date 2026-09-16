# 02-auth — Plan (HOW)

**Feature slug:** `AUTH` · **Status:** ⚠️ draft-for-review · **Companion:** [spec.md](spec.md) · [tasks.md](tasks.md)
**Applies:** `global/01_TECH_STACK.md`, `global/02_SECURITY_BASELINE.md` §1–2, `global/06_ENGINEERING_STANDARDS.md`, `global/07_SQL_DATABASE_STANDARDS.md` §1a

---

## 1. Approach & architecture

```
[FE login/register forms] --> [httpapi.AuthHandler] --> [app.AuthService]  --> [mysql.UserRepository / RefreshTokenRepository]
                                                              |
                                                        [shared/auth: JWT issue/verify, BCrypt]
                                                              |
[every other module's middleware] <-- RequirePermission(code) / RequireRole / customer-ownership check
```

`shared/auth` (not `internal/identity`) is where `RequirePermission`/`RequireRole` middleware and JWT
verification live, since **every** module's `httpapi` package needs them — putting them in `identity`
would make every other module depend on `identity`'s internals, breaking the module boundary
(`06_ENGINEERING_STANDARDS.md` §4). `identity` owns the *data* (accounts, roles, permissions);
`shared/auth` owns the *mechanism* (tokens, middleware) that reads claims already put there by
`identity` at login/refresh time.

## 2. Data model / migrations

Ships as `db/migrations/0002_identity_schema.up.sql`. This is the authoritative DDL for `role` (only
referenced, not defined, elsewhere) plus the auth tables the SRS's ER diagram doesn't cover
(`DB-ERD-1`, `09_ISSUES_REGISTER.md`) — `permission`/`role_permission` were already given in
`07_SQL_DATABASE_STANDARDS.md` §1a and are not repeated here, just seeded (§2.2).

```sql
CREATE TABLE role (
    role_id INT AUTO_INCREMENT PRIMARY KEY,
    name    VARCHAR(30) NOT NULL UNIQUE          -- CUSTOMER, WAREHOUSE_STAFF, ORDER_MANAGER, MANAGER, ADMIN
);

CREATE TABLE user_account (
    user_account_id INT AUTO_INCREMENT PRIMARY KEY,
    email           VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255) NOT NULL,        -- BCrypt, cost >= 12 (REQ-2.3)
    role_id         INT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      DATETIME NOT NULL DEFAULT UTC_TIMESTAMP(),
    FOREIGN KEY (role_id) REFERENCES role(role_id),
    UNIQUE KEY uq_user_account_email (email)      -- REQ-2.2; case-insensitive via utf8mb4_0900_ai_ci
);

-- The ER diagram's single "Customer" entity (id, name, email, phone, address) is split in two:
-- user_account carries login mechanics (shared by every role); customer carries the
-- customer-specific profile. DB-ERD-1: the diagram never modeled auth at all, so this split follows
-- the SRS's *text* requirements, not a diagram to stay literally consistent with.
CREATE TABLE customer (
    customer_id     INT AUTO_INCREMENT PRIMARY KEY,
    user_account_id INT NOT NULL,
    name            VARCHAR(150) NOT NULL,
    phone           VARCHAR(30) NOT NULL,
    address         VARCHAR(255) NULL,             -- required later at checkout for Standard Delivery, not at registration
    FOREIGN KEY (user_account_id) REFERENCES user_account(user_account_id) ON DELETE CASCADE,
    UNIQUE KEY uq_customer_user_account (user_account_id)
);

CREATE TABLE staff_profile (
    staff_profile_id INT AUTO_INCREMENT PRIMARY KEY,
    user_account_id  INT NOT NULL,
    name             VARCHAR(150) NOT NULL,
    FOREIGN KEY (user_account_id) REFERENCES user_account(user_account_id) ON DELETE CASCADE,
    UNIQUE KEY uq_staff_profile_user_account (user_account_id)
);

CREATE TABLE refresh_token (
    refresh_token_id INT AUTO_INCREMENT PRIMARY KEY,
    user_account_id  INT NOT NULL,
    token_hash       VARCHAR(255) NOT NULL,        -- SHA-256 of the opaque token; raw value never stored (SEC-AUTH-2)
    expires_at       DATETIME NOT NULL,
    revoked_at       DATETIME NULL,                -- REQ-2.5 "log out everywhere": set on logout
    created_at       DATETIME NOT NULL DEFAULT UTC_TIMESTAMP(),
    FOREIGN KEY (user_account_id) REFERENCES user_account(user_account_id) ON DELETE CASCADE
);
CREATE INDEX ix_refresh_token_user ON refresh_token(user_account_id);
CREATE INDEX ix_refresh_token_hash ON refresh_token(token_hash);
```

### 2.1 Seed data (migration, not `db/seed` — these rows are required for the app to function, not sample data)

```sql
INSERT INTO role (name) VALUES ('CUSTOMER'), ('WAREHOUSE_STAFF'), ('ORDER_MANAGER'), ('MANAGER'), ('ADMIN');

INSERT INTO permission (code, description) VALUES
  ('catalog:write',        'Create/update/deactivate products, variants, categories'),
  ('catalog:image:write',  'Upload/replace/delete product images'),
  ('stock:adjust',         'Manually adjust variant stock quantity'),
  ('order:status:update',  'Advance or cancel an order''s status'),
  ('order:cancel',         'Cancel an order and return its reserved stock'),
  ('reports:view',         'View the five management reports'),
  ('account:manage_roles', 'View/edit which permissions a role holds'),
  ('account:manage_users', 'Create staff/manager/admin accounts');

-- Default grants (02_SECURITY_BASELINE.md §1 table) — admin-editable after this point.
INSERT INTO role_permission (role_id, permission_id)
SELECT r.role_id, p.permission_id FROM role r JOIN permission p ON (
    (r.name = 'WAREHOUSE_STAFF' AND p.code IN ('catalog:write','catalog:image:write','stock:adjust'))
 OR (r.name = 'ORDER_MANAGER'   AND p.code IN ('order:status:update','order:cancel'))
 OR (r.name = 'MANAGER'         AND p.code = 'reports:view')
 OR (r.name = 'ADMIN'                                          -- all, for console display only (SEC-AUTH-1)
);
```

### 2.2 A first ADMIN account must exist before any admin console can be used

A migration cannot know a real password. `db/seed` (dev/test only, never run against production)
creates a throwaway admin (`admin@brightbuy.test` / a random generated password printed once to
stdout); a **production** first-admin is created by a one-off CLI command
(`go run ./cmd/api --create-first-admin`, guarded to refuse running if any `ADMIN` account already
exists) — documented here so it isn't improvised later. This command lives in `cmd/api`, not behind
an HTTP endpoint, since "create the first admin with no existing admin to authorize it" has no safe
API shape.

## 3. API contract

Already in `specs/openapi/openapi.yaml` (`auth` and `admin-rbac` tags):
`POST /auth/register`, `POST /auth/login`, `POST /auth/refresh`, `POST /auth/logout`, `GET /auth/me`,
`GET /admin/users`, `POST /admin/users`, `GET /admin/roles`, `GET /admin/permissions`,
`PUT /admin/roles/{roleId}/permissions`. No changes needed — this table is a summary.

| Method | Path | Auth | FR |
|---|---|---|---|
| POST | `/api/v1/auth/register` | public | FR-AUTH-1,2,3,7 |
| POST | `/api/v1/auth/login` | public | FR-AUTH-8,11 |
| POST | `/api/v1/auth/refresh` | cookie (refresh) | FR-AUTH-11 |
| POST | `/api/v1/auth/logout` | customer/staff | FR-AUTH-5 |
| POST | `/api/v1/admin/users` | `account:manage_users` | FR-AUTH-9 |
| PUT | `/api/v1/admin/roles/{roleId}/permissions` | `account:manage_roles` | FR-AUTH-10 |

## 4. Stored procedures / functions / triggers / views used

None. Identity is plain CRUD + a JOIN to resolve permissions — nothing here needs atomicity beyond a
single-row `INSERT`/`UPDATE`, so no stored procedure earns its place (`07_SQL_DATABASE_STANDARDS.md`
§3's own framing: procedures exist where the DB is *better positioned*, not by default).

## 5. Key logic / code snippets (Go, illustrative)

### 5.1 Login — generic failure message regardless of which factor failed (REQ-2.8, FR-AUTH-8)
```go
func (s *AuthService) Login(ctx context.Context, email, password string) (*Tokens, error) {
    acct, err := s.users.FindByEmail(ctx, email)
    if err != nil || acct == nil || !bcrypt_Compare(acct.PasswordHash, password) {
        return nil, ErrInvalidCredentials // one error, one message, whether email unknown or password wrong
    }
    if !acct.IsActive {
        return nil, ErrInvalidCredentials // deactivated accounts fail the same way — no extra signal
    }
    perms, err := s.roles.ResolvePermissions(ctx, acct.RoleID) // "*" sentinel if role == ADMIN
    ...
    return s.tokens.Issue(ctx, acct, perms) // access (~15min) + refresh (rotated, hashed at rest)
}
```

### 5.2 Permission resolution — the ADMIN bypass, enforced in code (SEC-AUTH-1)
```go
func (s *RoleService) ResolvePermissions(ctx context.Context, roleID int) ([]string, error) {
    role, _ := s.roles.Get(ctx, roleID)
    if role.Name == "ADMIN" {
        return []string{"*"}, nil // never queries role_permission for this role — can't be locked out
    }
    return s.roles.PermissionCodes(ctx, roleID) // real query for every other role
}

// shared/auth middleware:
func RequirePermission(code string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            perms := ClaimsFromContext(r.Context()).Permissions
            if !slices.Contains(perms, "*") && !slices.Contains(perms, code) {
                httpx.WriteError(w, http.StatusForbidden, "FORBIDDEN", "missing permission: "+code)
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}
```

### 5.3 Rate limiting login/register (02 §4)
An in-memory token-bucket keyed on `IP` and, for login, also on the submitted `email` — Phase 1 is a
single backend instance (`04_SCALABILITY.md` §1), so in-memory is sufficient; move to a shared store
(e.g. Redis) only if/when the backend actually scales to multiple instances, which would make a
per-instance in-memory bucket ineffective.

## 6. Edge cases & failure handling

- Registering with an email that differs only in case (`A@x.com` vs `a@x.com`) → rejected as a
  duplicate, because `utf8mb4_0900_ai_ci` is case-insensitive (`07` §1) — no separate app-side
  lowercasing needed, but the app still normalizes to lowercase before insert for readability of admin
  listings.
- Refresh called with an already-rotated (superseded) refresh token → treated as a possible token-theft
  signal: revoke **all** of that account's refresh tokens and require a fresh login, rather than
  silently accepting it.
- `POST /admin/users` with a `roleId` that doesn't exist → `400`, not `500`.
- Deactivating a `user_account` (future admin action, not in this feature's endpoint list yet) should
  cascade to revoking all its refresh tokens — noted for `07-admin-catalog`/future account-management
  work, not built here since SRS §4.2 doesn't ask for account deactivation, only creation.

## 7. Which global rules apply (do not restate, just reference)

- Security: `02` §1 (full authorization model — this feature implements it), §2 (tokens/sessions), §4
  (rate limiting, generic errors).
- SQL: `07` §1a (`permission`/`role_permission` DDL, seeded here), §2 (parameterized only).
- NFR: `03` §1.

## 8. Test hooks (→ tasks.md)

- Unit: BCrypt wrapping, JWT issue/verify round-trip, `ResolvePermissions`'s ADMIN bypass path (must
  never issue a query for that role — assert via a fake repository that panics if called).
- Integration (`testcontainers-go`): unique-email constraint; full register→login→refresh→logout
  cycle; refresh-token revocation actually blocks a subsequent refresh; `PUT
  /admin/roles/{roleId}/permissions` changes what a *newly issued* token carries without touching
  code; reused/rotated refresh token triggers the revoke-all response (§6).
