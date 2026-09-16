# 08 — Threat Model

Attack surface and threats for BrightBuy Phase 1, with the control that addresses each, cross-referenced
to `02_SECURITY_BASELINE.md` and OWASP Top Ten (2021). Every feature's `plan.md` §6 should cite the
T-IDs it closes or introduces.

---

## 1. Assets

| Asset | Why it matters |
|---|---|
| Customer PII (name, email, phone, address) | Privacy, legal exposure (SRS §6.3) |
| Password hashes | Credential compromise cascades to other services if reused |
| Payment references (provider token, last-4) | Never full card data, but still sensitive |
| Order/pricing data | Financial integrity — under/overcharging, phantom orders |
| Stock levels | Overselling breaks the core business guarantee (SRS §1.4) |
| JWT signing key, DB credentials | Full-system compromise if leaked |
| The database itself | Single point of truth for everything above |

## 2. Attackers

| Attacker | Motivation | Capability |
|---|---|---|
| Unauthenticated internet user | Data theft, fraud, vandalism, curiosity | Can reach any public endpoint |
| Registered customer (malicious) | Access another customer's data/orders, price manipulation, free/duplicate orders | Valid low-privilege credentials |
| Compromised staff/manager account | Broader catalog/order tampering, report data exfiltration | Valid mid-privilege credentials |
| Insider with DB access | Direct data exfiltration/tampering | Depends entirely on least-privilege enforcement (`02` §6) |

## 3. Threats & controls

| T-ID | Threat | Control | Ref |
|---|---|---|---|
| T-1 | Client sends `customer_id`/`role` in a request body to impersonate another user or escalate privilege | Identity resolved only from verified JWT; body identity fields rejected | `02` §1–2 |
| T-2 | IDOR/BOLA — customer A requests order/cart belonging to customer B by guessing/incrementing an ID | Ownership filter in every query's `WHERE`, not a post-fetch check; 404 not 403 | `02` §1, §6.2 |
| T-3 | SQL injection via product search, login, any free-text field | Parameterized queries only, structurally enforced | `02` §4, `07` §2 |
| T-4 | XSS via stored product description / review text (future) reads a token from `localStorage` | React auto-escaping, `dangerouslySetInnerHTML` banned, JWT in httpOnly cookie (not readable by JS) | `02` §2, §4 |
| T-5 | Credential stuffing / brute-force login | Rate limiting on auth endpoints, generic failure message (no enumeration), BCrypt cost factor | `02` §2, §4 |
| T-6 | Race condition: two concurrent checkouts oversell the same variant | Row-level locking (`FOR UPDATE`, ascending `variant_id`) inside `sp_place_order`, plus `trg_stock_nonnegative` backstop | `07` §3.1, §4 |
| T-7 | Double-order from a double-submitted checkout (double-click, retry) | Idempotency key, unique DB index, checked inside the same transaction | `07` §3.1, `06` §5 |
| T-8 | Client-sent price/total trusted at checkout, enabling underpayment | Server always recomputes price from `product_variant`; procedure snapshots `unit_price_at_order` itself | `07` §3.1 (REQ-4.10) |
| T-9 | Leaked application DB credential used to alter schema, drop tables, or read unrelated data | Least-privilege app DB role — no `DROP`/`ALTER`/`GRANT`; separate migration role | `02` §6.1 |
| T-10 | A future code path bypasses the service layer and writes stock/status directly, skipping business rules | `trg_stock_nonnegative`, `trg_order_status_transition` enforce the rule at the row level regardless of caller | `07` §4 |
| T-11 | Secrets (DB password, JWT key) committed to source control or logged | Env-var-only secrets, `.env` git-ignored, secret-scanner in CI, logs never contain secrets | `02` §5, §7 |
| T-12 | CSRF against a state-changing endpoint (place order, adjust stock) using the browser's ambient cookie | `SameSite=Strict` cookie + `Origin`/`Referer` check on state-changing requests | `02` §4 |
| T-13 | Report endpoint (customer-wise summary, full order history) accessed by a non-Manager role | Server-side role gate on every `/api/v1/admin/reports/**` route | `02` §1 |
| T-14 | Card data (PAN/CVV) accidentally logged or persisted by a future payment integration | Payment port interface has no field for raw card data by construction; REQ-7.5/7.6 | `02` §3 |
| T-15 | Enumerate valid customer emails via differing register/login error messages | Generic message regardless of whether the email exists | `02` §2 (REQ-2.8) |
| T-16 | Malicious file uploaded via the product-image pipeline (executable renamed `.jpg`, oversized file, path-traversal object key) to gain code execution or exhaust storage | Server-generated object keys (never client filenames), presigned-URL size/content-type conditions, server-side magic-byte verification + EXIF strip on confirm | `02` §4 (resolved OQ-3) |
| T-17 | Privilege escalation or self-lockout via the new permission-management endpoint — a non-admin granting itself `account:manage_roles`, or an admin accidentally revoking every role's access to that endpoint | `PUT /admin/roles/{roleId}/permissions` itself requires `account:manage_roles`; `ADMIN` is an unconditional code-level bypass of the permission table, so it can never be locked out regardless of what `role_permission` rows say | `02` §1 (resolved OQ-1) |

## 4. Explicitly accepted / deferred risk (Phase 1)

- **No RLS** (MySQL has none) — compensated by §6.2's WHERE-clause discipline; logged as `SEC-RLS-1`
  in `09_ISSUES_REGISTER.md`, not silently absent.
- **No live payment gateway** — Phase 1's payment stub means T-14 is largely theoretical until a real
  gateway is integrated; the interface boundary exists specifically so that integration doesn't require
  revisiting this threat model's payment-related controls.
- **Single MySQL instance, no read replica** — an availability risk accepted at Phase 1 scale
  (`04_SCALABILITY.md`).
