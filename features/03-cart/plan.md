# 03-cart — Plan (HOW)

**Feature slug:** `CART` · **Status:** ⚠️ draft-for-review · **Companion:** [spec.md](spec.md) · [tasks.md](tasks.md)
**Applies:** `global/01_TECH_STACK.md`, `global/06_ENGINEERING_STANDARDS.md`, `global/07_SQL_DATABASE_STANDARDS.md`

---

## 1. Approach & architecture

```
Guest:    localStorage[{variantId,quantity}] --GET /products/{id}--> re-priced client-side on render
                                              (no server cart exists yet)
Customer: [FE cart UI] --> [httpapi.CartHandler] --> [app.CartService] --> [mysql.CartRepository] --> MySQL
                                                            |
                                                    [catalog.app.CatalogService] (price + stock lookup)
Login:    [FE] posts localStorage items --> POST /cart/merge --> merged into the now-authenticated customer's cart
```

- `cart`'s `app` layer depends on `catalog`'s exported service for price/stock lookups — the one
  cross-module call this feature makes, and it goes through `catalog`'s public `app` interface, never
  its `mysql` package directly (`06_ENGINEERING_STANDARDS.md` §4).
- The client **never** sends a price, only `variantId` + `quantity` — the server resolves price via
  `catalog` and stock via `fn_is_variant_in_stock`'s underlying quantity (SEC-CART-2).
- A guest cart is never written to the database — it doesn't exist server-side until the owner logs in
  and it's merged (FR-CART-5). This keeps `cart`'s only real complexity server-side to "one row per
  customer," not "sessions for anonymous visitors too."

## 2. Data model / migrations

Ships as `db/migrations/0003_cart_schema.up.sql`.

```sql
CREATE TABLE cart (
    cart_id     INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT NOT NULL,
    updated_at  DATETIME NOT NULL DEFAULT UTC_TIMESTAMP() ON UPDATE UTC_TIMESTAMP(),
    FOREIGN KEY (customer_id) REFERENCES customer(customer_id) ON DELETE CASCADE,
    UNIQUE KEY uq_cart_customer (customer_id)          -- one cart per customer
);

CREATE TABLE cart_item (
    cart_item_id INT AUTO_INCREMENT PRIMARY KEY,
    cart_id      INT NOT NULL,
    variant_id   INT NOT NULL,
    quantity     INT NOT NULL,
    FOREIGN KEY (cart_id)    REFERENCES cart(cart_id)               ON DELETE CASCADE,
    FOREIGN KEY (variant_id) REFERENCES product_variant(variant_id) ON DELETE CASCADE,
    CHECK (quantity >= 1),
    UNIQUE KEY uq_cart_item_variant (cart_id, variant_id)            -- one line per variant; re-add = update
);
```

No price column on `cart_item` — deliberately. FR-CART-6 requires the current price at view time, so
storing a price would just be a value that's wrong the moment it's stale; the repository always joins
to `product_variant.price` on read.

## 3. API contract

Already in `specs/openapi/openapi.yaml` (`cart` tag): `GET /cart`, `POST /cart/items`,
`PATCH /cart/items/{cartItemId}`, `DELETE /cart/items/{cartItemId}`, `POST /cart/merge`.

| Method | Path | Auth | FR |
|---|---|---|---|
| GET | `/api/v1/cart` | customer | FR-CART-3,6 |
| POST | `/api/v1/cart/items` | customer | FR-CART-1,4 |
| PATCH | `/api/v1/cart/items/{cartItemId}` | customer | FR-CART-2 |
| DELETE | `/api/v1/cart/items/{cartItemId}` | customer | FR-CART-2 |
| POST | `/api/v1/cart/merge` | customer | FR-CART-5 |

Guest add/update/remove has **no backend endpoint** — it's pure frontend state (`localStorage`)
until login, per FR-CART-5. `01-catalog`'s public `GET /products/{id}` is what the guest cart's
frontend logic re-reads for current price/stock on each render.

## 4. Stored procedures / functions / triggers / views used

- Uses: `fn_is_variant_in_stock` (`07` §5) for the advisory stock warning (FR-CART-4). No new
  procedures/triggers/views — nothing here needs multi-statement atomicity; a cart line write is a
  single `INSERT ... ON DUPLICATE KEY UPDATE` against the `uq_cart_item_variant` unique key.

## 5. Key logic / code snippets (Go, illustrative)

### 5.1 Add/update line — server-resolved price, advisory stock cap (FR-CART-1,4, SEC-CART-2)
```go
func (s *CartService) AddItem(ctx context.Context, customerID, variantID, qty int) (*Cart, error) {
    variant, err := s.catalog.GetVariant(ctx, variantID) // price + stock_quantity, from catalog's app service
    if err != nil { return nil, ErrVariantNotFound }
    if qty > variant.StockQuantity {
        return nil, &StockWarningError{Available: variant.StockQuantity} // FR-CART-4: reject, don't clamp silently
    }
    return s.repo.UpsertLine(ctx, customerID, variantID, qty) // ON DUPLICATE KEY UPDATE quantity = ?
}
```

### 5.2 Merge on login — higher quantity wins, not summed (FR-CART-5, AC-CART-4)
```go
func (s *CartService) Merge(ctx context.Context, customerID int, guestItems []LineInput) (*Cart, error) {
    for _, g := range guestItems {
        existing := s.repo.FindLine(ctx, customerID, g.VariantID) // 0 if no existing line
        merged := max(existing.Quantity, g.Quantity)              // higher, not summed — SRS §3.5 explicit
        s.repo.UpsertLine(ctx, customerID, g.VariantID, merged)
    }
    return s.repo.Get(ctx, customerID)
}
```

### 5.3 Subtotal — always computed from current `product_variant.price` (FR-CART-3,6)
```sql
SELECT ci.cart_item_id, ci.variant_id, pv.price, ci.quantity, (pv.price * ci.quantity) AS line_total
FROM cart_item ci
JOIN product_variant pv ON pv.variant_id = ci.variant_id
WHERE ci.cart_id = ?;   -- subtotal = SUM(line_total) in Go, or SUM() in the same query — either is fine here
```

## 6. Edge cases & failure handling

- Adding a variant already in the cart → `UpsertLine` sets (not adds to) the new quantity — REQ-3.1
  describes "add" as the user action, but the API contract's semantics are "the line now has this
  quantity," matching `PATCH`'s own semantics and avoiding a confusing "add 2, add 2 again, get 4"
  surprise when the user meant to correct a typo.
- Variant deactivated/deleted after being added to cart (`07-admin-catalog`'s soft-delete) → line is
  flagged `unavailable` on read, excluded from the subtotal, with a clear removal affordance — never a
  500 on cart view.
- Quantity `PATCH` to `0` → treated as equivalent to `DELETE` (single code path, not two).
- Merge called with an empty `items` array (customer had no guest cart) → no-op, returns the existing
  server cart unchanged.
- Concurrent add of the same variant from two tabs → `ON DUPLICATE KEY UPDATE` on
  `uq_cart_item_variant` makes this safe without app-level locking.

## 7. Which global rules apply (do not restate, just reference)

- Security: `02` §1 (customer-only, ownership-scoped), §6.2 (ownership in `WHERE`, not post-fetch).
- SQL: `07` §2 (parameterized only), §5 (`fn_is_variant_in_stock`).
- NFR: `03` §1, this feature's own `NFR-CART-1`.

## 8. Test hooks (→ tasks.md)

- Unit: `AddItem`'s stock-cap rejection, `Merge`'s higher-quantity-wins logic, quantity-0-is-delete.
- Integration (`testcontainers-go`): ownership (customer B → 404 on A's `cart_item`); price always
  reflects current `product_variant.price`, never a stale add-time value; unavailable/deactivated
  variant excluded from subtotal; concurrent-add race via `ON DUPLICATE KEY UPDATE`.
