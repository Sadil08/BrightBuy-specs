# 03 — Non-Functional Requirements Baseline

Inherited by every feature; restated from SRS §5.1/§5.2/§5.4 with the concrete mechanism. Feature
`spec.md` files list only a delta (tighter budget, or a new NFR the feature introduces).

---

## 1. Performance (SRS §5.1 — goals to test against, not contractual SLAs for a Phase 1 pilot)

| Budget | Target | Mechanism |
|---|---|---|
| Catalog browse/search | < 2s at the API, normal load | Indexes on filtered/joined columns (`07` §6); pagination (REQ-1.6) |
| Checkout transaction | commits or rolls back in < 3s, excluding payment-stub latency | Single stored-procedure call inside one DB transaction (REQ-4.6) |
| Report queries | < 5s against the seeded dataset | Reads go through views, not ad-hoc joins in application code (`07` §5) |
| Concurrency | ≥50 concurrent browsing users, ≥10 concurrent checkouts, no lost updates | Row-level locking in the stock procedure (REQ-4.7); connection pool sizing below |
| Connection pool | starting point 10 conns/backend instance, tuned against measurement | `database/sql` `SetMaxOpenConns`/`SetMaxIdleConns` in `shared/dbx` |

Catalog read endpoints (`GET /products`, `GET /categories`) should carry HTTP cache headers
(`Cache-Control`, `ETag`) — that data changes far less often than it's read.

## 2. Safety (SRS §5.2)

- Stock **shall never go below zero** — enforced twice: the stored procedure's own logic, and a
  `BEFORE UPDATE` trigger on `product_variant` as a backstop (`07` §4) that rejects the write outright
  even if a future code path bypasses the procedure.
- No transaction leaves the database inconsistent on failure — guaranteed by InnoDB ACID + explicit
  `ROLLBACK` on any error path in the checkout use-case (REQ-4.3).
- Every schema/procedure/trigger/view change is a versioned migration (`golang-migrate`) — any
  environment can be rebuilt from source, and a bad change is identifiable and reversible (SRS §5.2,
  §6.1).
- Backup schedule/retention: **open, tracked as OQ-6** in `10_BA_AUDIT_OPEN_QUESTIONS.md`.

## 3. Software quality attributes (SRS §5.4)

| Attribute | How this build achieves it |
|---|---|
| Reliability | ACID transactions; idempotency key on checkout (REQ-4.9) — a retried request can't double-order |
| Availability | Stateless backend (JWT, no server-side session memory) — any instance can serve any request |
| Modularity | Module-owns-its-tables boundary (`00` §3); a delivery-rule change touches only `internal/delivery` |
| Maintainability | Schema normalized to 3NF (deviations documented, `07` §1); one place for each business rule |
| Scalability | See `04_SCALABILITY.md` |
| Testability | Ports/interfaces let domain+app logic be unit-tested without a database; `testcontainers-go` for what actually needs MySQL (triggers, procedures, locking) |
| Portability | Backend ships as a container image, config from environment; same image runs local/staging/prod |
| Usability | A first-time customer completes browse→cart→checkout with no help, using conventions from any mainstream storefront |

## 4. Business rules that are also NFRs (SRS §5.5, restated once here so features don't repeat them)

- Only registered customers may check out; enforced in the application layer *and* by a mandatory,
  non-nullable `customer_id` foreign key on `order` (defense in depth, matches SRS §2.5).
- Price and stock are held at the variant level, never the product level.
- Stock is decremented at order confirmation only; a cart never reserves stock.
- Each order records exactly one delivery mode and exactly one payment method.
- A cancelled order returns its stock; a delivered order is terminal (until Completed).
- Catalog records referenced by an existing order are **deactivated**, never deleted — order history
  must stay intact and readable (REQ-10.5).
