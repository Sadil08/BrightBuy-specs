# 11 — Build Plan

Build sequence and milestones. Order chosen so each phase unblocks the next with the least rework, and
so the DBMS-module-critical SQL work (triggers/procedures/views) is built and tested early, against a
real schema, rather than bolted on at the end.

---

## Phase 0 — Repo & environment (no feature code yet)

- Create the two repositories (`brightbuy-backend`, `brightbuy-frontend` — OQ-2, resolved). Both
  resolved: image strategy (OQ-3, real S3/MinIO pipeline) and role model (OQ-1, dynamic RBAC) — both
  now baked into `01_TECH_STACK.md`, `02_SECURITY_BASELINE.md`, `07_SQL_DATABASE_STANDARDS.md` §1a.
- `brightbuy-frontend/` Next.js app scaffold, boots, calls the health check.
- Dockerfiles + per-repo `docker-compose.yml` (backend: MySQL + MinIO + api; frontend: web, joined to
  the backend's network) — full detail in `12_DEVOPS_CICD.md` §1.
- CI skeleton in both repos (lint + unit tests green on an empty/near-empty codebase) — `12
  _DEVOPS_CICD.md` §2, so every feature from here on lands with CI already enforcing it, not bolted on
  later.
- **Schema arrives incrementally, one feature's migration at a time**, not as one upfront
  `0001_init` covering the whole ER diagram — `01-catalog`'s `plan.md` §2 owns the first migration
  (category/product/variant/attribute tables); `02-auth` adds `user_account`/`role`/`customer`/
  `permission`/`role_permission`; and so on. Each migration is reviewed alongside the feature and test
  that needs it, per `07_SQL_DATABASE_STANDARDS.md`, rather than as one unreviewable upfront dump.
- `db/seed/` — deterministic seed script, ≥40 products / ≥10 categories (SRS §6.1, resolves `OQ-4`).

## Phase 1 — Foundation features (unblock everything else)

1. **`01-catalog`** — browse/search/detail (REQ-1.*). No auth needed; proves the read path,
   `product`/`product_variant`/`category` schema, and `fn_is_variant_in_stock`.
2. **`02-auth`** — registration/login/logout/roles (REQ-2.*). Unblocks every authenticated feature
   after this point. Establishes `shared/auth` (JWT issue/verify, BCrypt, middleware) once, reused by
   every later feature per `06_ENGINEERING_STANDARDS.md` §5.

## Phase 2 — The core commerce path

3. **`03-cart`** — REQ-3.*. Depends on `01-catalog` (pricing/stock lookups) and `02-auth` (customer
   cart persistence + guest merge).
4. **`04-checkout-orders`** — REQ-4.*, REQ-7.* (payment), REQ-8.* (status/tracking) — the single
   biggest feature, and where `sp_place_order`, the stock-related triggers, and the payment/idempotency
   logic are built and integration-tested (`05_TESTING_STRATEGY.md` §2). Resolves `TBD-1` (fee/tax as
   config) as part of its design.
5. **`05-delivery-estimation`** — REQ-6.*. Small in isolation but depends on `04`'s procedure calling
   `fn_estimate_delivery_days`; resolves `TBD-2`/`TBD-3` as working assumptions.

## Phase 3 — Operations

6. **`06-inventory-stock`** — REQ-5.*. `sp_adjust_stock`, `stock_movement` audit trail, staff UI.
7. **`07-admin-catalog`** — REQ-10.*. Staff CRUD on products/variants/categories; deactivate-not-delete
   rule (REQ-10.5).
8. **`08-order-tracking-status`** — REQ-8.1–8.6 staff-facing half (customer-facing tracking view ships
   with `04`). `sp_cancel_order`, the two order-status triggers.

## Phase 4 — Reporting & polish

9. **`09-management-reporting`** — REQ-9.*. All five views from `07_SQL_DATABASE_STANDARDS.md` §6,
   CSV export. Deliberately last among the "real" features — every view reads data the earlier phases
   already produce, so seeding realistic report output doesn't require synthetic data.
10. Hardening pass across all features: rate limiting, CORS allow-list finalized, secret-scanner wired
    into CI, `staticcheck`/ESLint gates enforced (`05_TESTING_STRATEGY.md` §4).

## Phase 5 — Deployment

- Resolve `TBD-7` (hosting provider — `12_DEVOPS_CICD.md` §3.4 has a working recommendation) and
  `TBD-6` (backup schedule, which depends on that choice).
- CD wiring per `12_DEVOPS_CICD.md` §3: promotion (not rebuild) from staging to production, migrations
  as an explicit pre-traffic deploy step, rollback = redeploy the previous tag.
- CI is not new work here — it's been enforcing every PR since Phase 0 (`12_DEVOPS_CICD.md` §2); this
  phase is deployment, not "add CI."

---

## Milestone definition of done

A phase is done when every feature in it has ✅ status across `spec.md`/`plan.md`/`tasks.md`, every
`FR-*` traces to a passing test, and — for any feature touching `07_SQL_DATABASE_STANDARDS.md`
objects — the relevant `testcontainers-go` integration test exists and is green (never "the endpoint
returns 200," always "the trigger/procedure does the specific thing REQ-* says it must").
