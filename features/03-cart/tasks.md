# 03-cart — Tasks (DO IT)

**Feature slug:** `CART` · **Status:** ❌ not started · **Companion:** [spec.md](spec.md) · [plan.md](plan.md)

---

## 1. Build order

### Database
- [ ] **T1 `[DB]`** Migration `0003_cart_schema`: `cart`, `cart_item` (`plan.md` §2). — *Test:* applies
      clean; `down`/`up` idempotent; `uq_cart_item_variant` enforced. — `FR-CART-1,2`

### Backend
- [ ] **T2 `[BE]`** `internal/cart/mysql.CartRepository`: `Get`, `UpsertLine`, `FindLine`,
      `DeleteLine` — parameterized, subtotal query (`plan.md` §5.3). — *Test:* integration.
- [ ] **T3 `[BE]`** `internal/cart/app.CartService`: `AddItem` (stock cap, `plan.md` §5.1),
      `UpdateQuantity`, `RemoveLine`, `Merge` (`plan.md` §5.2), consuming `catalog`'s public service
      for price/stock. — *Test:* unit with a fake `catalog` port.
- [ ] **T4 `[BE]`** `internal/cart/httpapi.CartHandler`: the five endpoints (`plan.md` §3), ownership
      from JWT context (SEC-CART-1), unavailable-line flagging (`plan.md` §6). — *Test:* `httptest` —
      status codes, 404-on-non-owner.

### Frontend
- [ ] **T5 `[FE]`** Guest cart: `localStorage`-backed cart hook, re-prices against
      `GET /products/{id}` on render (no backend cart endpoint involved). — *Test:* Vitest.
- [ ] **T6 `[FE]`** Customer cart: typed API client calls to the five endpoints; call `POST
      /cart/merge` once, immediately after login, with the current guest cart. — *Test:* Vitest.
- [ ] **T7 `[FE]`** Cart page: line items, quantity stepper, remove, subtotal, stock-warning message,
      unavailable-line affordance. — *Test:* RTL + Playwright (add → update qty → remove → subtotal
      updates at each step).

### Cross-cutting
- [ ] **T8 `[INFRA]`** Verify `openapi.yaml`'s `cart` paths match the shipped API.

## 2. Acceptance-criteria coverage map

| AC | Covered by |
|---|---|
| AC-CART-1 | T3, T4, T5 |
| AC-CART-2 | T3, T4 |
| AC-CART-3 | T2, T4 |
| AC-CART-4 | T3, T4, T6 |
| AC-CART-5 | T2, T3 |

## 3. Definition of Done

- [ ] Every `FR-CART-*` has a passing unit and integration test.
- [ ] Every `AC-CART-*` above is green.
- [ ] No client-sent price is ever trusted anywhere in this feature (SEC-CART-2) — verified by a test
      that sends a bogus price field and asserts it's ignored, not just absent from the DTO struct.
- [ ] `gofmt`/`go vet`/ESLint clean; `openapi.yaml` matches the shipped API.
