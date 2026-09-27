# 02 — Security Baseline

Inherited by every feature. Feature `spec.md` lists only its **security delta**. Fixed requirements
come from SRS §5.3 (Security Requirements) and are restated here with the concrete Go/MySQL control
that satisfies each one, mapped to `08_THREAT_MODEL.md` T-IDs and OWASP Top Ten (2021).

---


Phase 1 does **not** hardcode "what a role can do" in application code. Two schema objects
(`07_SQL_DATABASE_STANDARDS.md` §1a) make it data instead:

- **`permission`** — a fixed catalog of capability codes: `catalog:write`, `catalog:image:write`,
  `stock:adjust`, `order:status:update`, `order:cancel`, `reports:view`, `account:manage_roles`,
  `account:manage_users`. New capabilities added later are new rows here, not new `if` branches.
- **`role_permission`** — which roles currently hold which permissions. **Editable at runtime by
  `ADMIN`** through `PUT /api/v1/admin/roles/{roleId}/permissions` — this is the "modular, loosely
  coupled" requirement: a handler asks `RequirePermission("stock:adjust")`, never `if role == "..."`,
  so regrouping who can do what is a data change.

**Roles** (five, not the ER diagram's four — deviation logged as `RBAC-1`,
`09_ISSUES_REGISTER.md`): `CUSTOMER`, `WAREHOUSE_STAFF`, `ORDER_MANAGER`, `MANAGER`, `ADMIN`.

| Role | Default permissions (seeded; admin-adjustable thereafter) |
|---|---|
| `CUSTOMER` | none — customer actions are **ownership-scoped**, not permission-scoped (see below) |
| `WAREHOUSE_STAFF` | `catalog:write`, `catalog:image:write`, `stock:adjust` (REQ-10.1, REQ-5.2) |
| `ORDER_MANAGER` | `order:status:update`, `order:cancel` (REQ-8.3, REQ-8.6) |
| `MANAGER` | `reports:view` (REQ-9.6) |
| `ADMIN` | **all**, and unconditionally — see below |

- **Public (no auth):** catalog browse/search/detail (REQ-1.1). Guests may hold a client-only cart but
  **cannot** reach any endpoint that reads/writes `customer`, `cart` (server-side), `order`, `payment`.
- **Customer:** own cart, own checkout, own order history/detail only — **object-level ownership
  enforced on every request**, not just at login (REQ-8.4: a request for another customer's order is
  refused whether or not the ID is guessed correctly → **404, not 403**, so existence isn't leaked —
  the standard IDOR/BOLA control, OWASP A01). Ownership checks are independent of the permission
  system below; a customer is never granted a "permission" to see their own data, they simply own it.
- **`ADMIN` is an unconditional bypass, enforced in code, not by table rows.** `role_permission` rows
  are still seeded for `ADMIN` so the admin console can *display* what it can do, but the middleware
  never consults them for this role — it short-circuits on `role == ADMIN`. This is deliberate: if
  permission rows were the only gate, an admin could revoke `account:manage_roles` from `ADMIN`
  (accidentally, or via a bug in the admin UI) and lock every admin out of fixing it. The bypass makes
  that failure mode structurally impossible.
- Every `/api/v1/admin/**` and `/api/v1/staff/**` route is gated **inside Go middleware, server-side**,
  by calling `RequirePermission(code)` (or the ownership check, for customer routes) — a hidden button
  in the Next.js admin console is a UX nicety, never the access control (SRS §3.1, "hiding a button is
  presentation, not access control").

### 1.1 How permissions reach the request (and the staleness this implies)

The access token's JWT claims carry the caller's **resolved permission list at issue/refresh time**
(`{"sub": userId, "role": "WAREHOUSE_STAFF", "perms": ["catalog:write", "catalog:image:write",
"stock:adjust"]}`, or `{"role": "ADMIN", "perms": ["*"]}`) rather than the backend re-querying
`role_permission` on every request — consistent with the stateless-backend requirement (`04
_SCALABILITY.md` §1) and avoiding a DB round-trip per request for something that changes rarely.

**Consequence:** a permission revoked by `ADMIN` takes effect for an already-logged-in user within, at
most, the access token TTL (≤15 min, `02_SECURITY_BASELINE.md` §2) — the next silent refresh
re-resolves the list from the table. For a case that needs to be immediate (e.g. suspected account
compromise), the existing "log out everywhere" refresh-token revocation (REQ-2.5) forces re-auth right
away, which also re-resolves permissions. This tradeoff — short bounded staleness vs. a DB hit on every
request — is the right one at Phase 1 scale; revisit only if permission changes need to be instant and
that bound stops being acceptable.

## 2. Identity, tokens & sessions

- The acting user is resolved **only** from a verified JWT's subject claim, injected into the request
  context by `shared/auth` middleware before the handler runs. Any `customer_id`/`user_id`/`role`
  field present in a request body is a validation error, not a hint — this closes the classic
  "trust the body" mass-assignment hole (OWASP A01, T-1).
- **Access token:** short-lived (≤15 min), signed HS256 (or RS256 once there's a reason to verify
  without the signing key), carries `sub` (user id) and `role`. **Refresh token:** longer-lived,
  opaque, stored server-side (a `refresh_token` row or Redis later) so it can be revoked — "log out
  everywhere" (REQ-2.5) is a real revoke, not just discarding a client-side token.
- Tokens are returned to the Next.js client and held in an **httpOnly, Secure, SameSite=Strict**
  cookie — never `localStorage` — so a successful XSS cannot read and exfiltrate the token (OWASP A03,
  T-4). This is a deliberate deviation from "convenient to read in JS"; convenience loses here.
- Every protected endpoint validates `exp`, `iss`, signature — expired/invalid → `401`. Role mismatch
  on an otherwise-valid token → `403`.
- Login failure returns the **same generic message** whether the email exists or not (REQ-2.8) —
  no user enumeration oracle.

## 3. Passwords & sensitive data

- Passwords: **BCrypt**, cost factor ≥12, never logged, never returned in any response, never stored
  reversibly (REQ-2.3, SRS §5.3). Minimum length 8 (REQ-2.7).
- Card data: **never stored.** Only a payment-provider reference and the last four digits may be
  retained (REQ-7.5, REQ-7.6) — the payment stub in Phase 1 enforces this by construction (its
  interface has no field for a full card number). Out of PCI scope by design, not by discipline.
- PII collection is minimized to what checkout/delivery needs (SRS §6.3) — no field is added to a
  DTO "in case it's useful later."

## 4. Input validation, injection & abuse (the DBMS-module-critical section)

- **SQL injection is prevented structurally, not by discipline:** every query goes through
  `database/sql`/`sqlx` **placeholder parameters** (`?`), never `fmt.Sprintf`/string concatenation
  into SQL. This is a hard rule enforced by code review and (where feasible) a `go vet`/lint check for
  raw string concatenation near `.Query`/`.Exec` calls. See `07_SQL_DATABASE_STANDARDS.md` §2 for the
  full policy — this is the direct answer to the SRS's "prevented by using parameterized queries"
  requirement (§5.3), translated from JPA to raw Go SQL.
- **Every request body is validated server-side** via `go-playground/validator` struct tags
  (required, min/max, enum, email format) before it touches a service. Client-side validation in
  Next.js is a UX nicety, not a control (mirrors §1's button-hiding rule).
- **XSS:** React/Next.js escapes by default; `dangerouslySetInnerHTML` is **banned** (ESLint rule) —
  no feature needs raw HTML injection in Phase 1. Free-text fields (product description) are rendered
  as text, never interpreted as HTML.
- **CORS:** explicit allow-list of the frontend origin(s) per environment. A wildcard origin is never
  acceptable in any deployed environment (SRS §3.4).
- **Rate limiting:** login, registration, and checkout endpoints are rate-limited per IP/account
  (in-memory token bucket in Phase 1 — single instance; move to a shared store if the backend ever
  scales beyond one instance, `04_SCALABILITY.md`) to slow credential stuffing and cart/checkout abuse.
  The "IP" used is `r.RemoteAddr` (the real TCP peer), **not** a client-supplied
  `X-Forwarded-For`/`X-Real-IP` header — those are spoofable and would make the limiter trivially
  bypassable until a trusted reverse proxy is in place to strip and re-set them (`12_DEVOPS_CICD.md`
  §3). Caught by `staticcheck` flagging chi's `middleware.RealIP` as unsafe without one.
- **CSRF:** mitigated by `SameSite=Strict` cookies + checking the request's `Origin`/`Referer` header
  on state-changing requests, since the JWT lives in a cookie rather than a header the browser won't
  auto-attach.
- **File uploads (product images, resolved OQ-3):** the backend never proxies the binary — it issues a
  short-lived (≤5 min) pre-signed S3 `PUT` URL scoped to one object key, gated behind
  `catalog:image:write` (§1), with the content-type and a max size (`Content-Length` condition on the
  presigned policy) fixed at issue time so the client can't upload an arbitrary file type or an
  oversized object. After the client uploads directly to S3/MinIO, the confirm call
  (`POST .../images`) fetches the object back server-side and **verifies the magic bytes match an
  allowed image type** (JPEG/PNG/WebP) before persisting the `product_image` row — a renamed
  `.exe`/`.php` with a `.jpg` extension is rejected even though the presigned policy's content-type
  check alone wouldn't catch it. Object keys are **server-generated** (`products/{productId}/{uuid}.
  {ext}`), never derived from the client-supplied filename, which closes path traversal
  (`../../etc/passwd`-style keys) by construction. EXIF metadata is stripped on confirm (privacy —
  uploaded photos may carry GPS/device data unrelated to the product).

## 5. Secrets management

- Every secret — MySQL credentials, JWT signing key — comes from environment variables, injected at
  process start, never committed (SRS §2.5, §6.2). `.env` is git-ignored; a secret-scanner (e.g.
  `gitleaks`) is a required pre-commit/CI check once CI exists (`11_BUILD_PLAN.md`).
- Secrets are rotatable without a code change or redeploy of application logic — only a config change.

## 6. Database-level defense in depth (MySQL has no RLS — read this carefully)

The ScanLanka reference project (Postgres) used **Row-Level Security** as a database-enforced
backstop for ownership scoping. **MySQL 8 has no RLS.** The compensating controls, in order of
strength:

1. **Least-privilege DB user.** The application connects as a role with `SELECT/INSERT/UPDATE/DELETE`
   and `EXECUTE` (for stored procedures/functions) on BrightBuy's schema only — **no `DROP`, `ALTER`,
   `GRANT`, `SUPER`, `FILE`**. A separate, more-privileged role runs migrations
   (`golang-migrate`), used only from CI/deploy, never by the running application. This limits blast
   radius if the application's DB credential leaks — it cannot alter the schema or read other schemas.
2. **Ownership checked in every query's `WHERE` clause**, not just in application logic that decides
   *whether* to run the query. E.g. `SELECT * FROM order WHERE order_id = ? AND customer_id = ?` —
   the customer_id from the verified JWT is always a filter, never just a post-fetch check. A query
   that fetches by primary key alone and checks ownership in Go *after* the fact is a code-review
   rejection (this is the concrete rule that replaces RLS).
3. **Reporting views are read-only** and contain no `customer_id`-scoped row filtering need (reports
   are manager-only and see all customers by design, REQ-9.5) — so this concern doesn't apply there,
   but the view itself has no `INSERT`/`UPDATE`/`DELETE` grant, which prevents "reporting" code from
   ever becoming a write path (`00_OVERVIEW.md` §3).
4. **Stored procedures run with `SQL SECURITY INVOKER`** (not `DEFINER`) where practical, so a
   procedure never silently grants the caller more privilege than their own DB role has. Documented
   per-procedure in `07_SQL_DATABASE_STANDARDS.md` §3 with the reasoning if `DEFINER` is used instead
   (needed when the procedure must do something the low-privilege app role can't, e.g. write to an
   audit table the app role has no direct `INSERT` on).

This is logged as closed item `SEC-RLS-1` in `09_ISSUES_REGISTER.md` — "no RLS in MySQL" is a known,
deliberate gap with an explicit compensating control, not an oversight.

## 7. Logging & auditing

- Security-relevant events — logins, failed logins, role changes, stock adjustments, order
  cancellations — are logged with timestamp and acting user (SRS §5.3, REQ-5.2). Logs never contain
  passwords, tokens, or card data.
- `stock_movement` (inventory) and `order_status_history` (ordering) are the two audit tables the SRS
  explicitly requires (REQ-5.5, REQ-8.5) — see `07_SQL_DATABASE_STANDARDS.md` §4 for how triggers keep
  `order_status_history` populated automatically.

## 8. OWASP Top Ten (2021) map

| OWASP | Control here |
|---|---|
| A01 Broken Access Control | §1 role gates (server-side), §2 identity-from-token, §6.2 ownership-in-WHERE |
| A02 Cryptographic Failures | §3 BCrypt, TLS everywhere (SRS §3.4), no card storage |
| A03 Injection | §4 parameterized SQL only, §4 XSS via React escaping |
| A04 Insecure Design | REQ-4.6–4.8 stock atomicity, idempotency keys, this whole doc |
| A05 Security Misconfiguration | §5 secrets via env, §6.1 least-privilege DB role |
| A07 Identification/Auth Failures | §2 short-lived JWT + revocable refresh, §4 rate limiting, §2 generic login error |
| A08 Software/Data Integrity | §6.1 DB role separation, migrations only via `golang-migrate` |
| A09 Logging/Monitoring Failures | §7 |
