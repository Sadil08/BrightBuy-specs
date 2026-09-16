# 04-checkout-orders — Tasks (DO IT)

**Feature slug:** `CHECKOUT` · **Status:** ❌ not started · **Companion:** [spec.md](spec.md) · [plan.md](plan.md)

---

## 1. Build order

### Database
- [ ] **T1 `[DB]`** Migration `0004_checkout_schema`: `city`, `app_config`, `order`, `order_item`,
      `delivery`, `stock_movement`, `order_status_history`, `payment` (`plan.md` §2) + seed
      `city`/`app_config` rows. — *Test:* applies clean; `down`/`up` idempotent. —
      `FR-CHECKOUT-4,5,10,11`
- [ ] **T2 `[DB]`** Migration `0004a_checkout_procedures`: `trg_stock_nonnegative` (`07` §4),
      `fn_classify_city`, `fn_estimate_delivery_days` (mode-aware, `SQL-BUG-1`), `sp_place_order`
      (`07_SQL_DATABASE_STANDARDS.md` §3.1, §5, now writing `stock_movement` per `SQL-BUG-2` and the
      initial `order_status_history` row per `SQL-BUG-3`). — *Test:* direct SQL — Store Pickup
      estimate ignores city entirely (regression test for `SQL-BUG-1`); Main vs. Other City base days;
      +3 out-of-stock addend for both modes; a successful `sp_place_order` call produces one
      `stock_movement` row per line with the correct `related_order_id` (regression test for
      `SQL-BUG-2`) **and** exactly one `order_status_history` row at status `Confirmed` (regression
      test for `SQL-BUG-3`); a direct `UPDATE` that would take stock negative is rejected by the
      trigger independent of the procedure.

### Backend
- [ ] **T3 `[BE]`** `internal/payment`: `PaymentProcessor` port + `StubProcessor` (`plan.md` §5.2). —
      *Test:* unit.
- [ ] **T4 `[BE]`** `internal/ordering/mysql.OrderRepository`: `CallPlaceOrder` (procedure call +
      error mapping for `STOCK_EXCEEDED`), `GetByID`, `ListByCustomer`,
      `DiagnoseUnavailableLines` (`plan.md` §6). — *Test:* integration.
- [ ] **T5 `[BE]`** `internal/ordering/app.CheckoutService.PlaceOrder` (`plan.md` §5.1): cart
      cross-module call, delivery-field validation, tax/fee from `app_config`, auth-before-transaction
      ordering, `dbx.WithTx` orchestration. — *Test:* unit with fakes for `cart`/`payment`/`orders`.
- [ ] **T6 `[BE]`** `internal/ordering/httpapi`: `POST /checkout`, `GET /orders`, `GET
      /orders/{orderId}` — idempotency-key header handling, ownership filter on the two `GET`s (`02`
      §6.2). — *Test:* `httptest`.

### Frontend
- [ ] **T7 `[FE]`** Checkout flow: delivery mode → (city/address if Standard) → payment method → review
      → confirm, with the idempotency key generated once per checkout attempt and reused on any retry
      of the same attempt (not regenerated on a network-error retry). — *Test:* Playwright.
- [ ] **T8 `[FE]`** Order history + detail pages. — *Test:* RTL + Playwright.

### Cross-cutting
- [ ] **T9 `[INFRA]`** Verify `openapi.yaml`'s `checkout`/`orders` paths match the shipped API.

## 2. Acceptance-criteria coverage map

| AC | Covered by |
|---|---|
| AC-CHECKOUT-1 | T4, T5, T6 |
| AC-CHECKOUT-2 | T4, T5 |
| AC-CHECKOUT-3 | T2, T4 |
| AC-CHECKOUT-4 | T3, T5 |
| AC-CHECKOUT-5 | T2, T4 (the concurrency integration test — see `plan.md` §8) |
| AC-CHECKOUT-6 | T2, T4 |
| AC-CHECKOUT-7 | T6 |

## 3. Definition of Done

- [ ] Every `FR-CHECKOUT-*` has a passing unit and integration test.
- [ ] Every `AC-CHECKOUT-*` above is green, **especially AC-CHECKOUT-5** — the concurrent-checkout
      race test is the single most important test in this entire project; it must actually spin up two
      goroutines/connections against a real MySQL 8 and assert the row-locking behavior, not simulate
      it.
- [ ] No client-sent price/total is ever trusted anywhere in this feature (SEC-CHECKOUT-1).
- [ ] `gofmt`/`go vet`/ESLint clean; `openapi.yaml` matches the shipped API.
- [ ] `SQL-BUG-1`, `SQL-BUG-2`, and `SQL-BUG-3` in `09_ISSUES_REGISTER.md` confirmed fixed by T2's
      regression tests, not just by re-reading the SQL.
