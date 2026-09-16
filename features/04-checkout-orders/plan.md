# 04-checkout-orders — Plan (HOW)

**Feature slug:** `CHECKOUT` · **Status:** ⚠️ draft-for-review · **Companion:** [spec.md](spec.md) · [tasks.md](tasks.md)
**Applies:** `global/01_TECH_STACK.md`, `global/06_ENGINEERING_STANDARDS.md`, `global/07_SQL_DATABASE_STANDARDS.md` §3.1

---

## 1. Approach & architecture

```
[FE checkout page] --POST /checkout--> [httpapi.CheckoutHandler]
                                              |
                                       [app.CheckoutService.PlaceOrder]
                                        1. load cart (cart.app port)
                                        2. validate delivery fields (mode/city/address combo)
                                        3. IF Card: PaymentProcessor.Authorize(...)  <- OUTSIDE any DB tx (REQ-4.8)
                                        4. dbx.WithTx( CALL sp_place_order(...); DELETE cart_item; )
                                        5. map result -> OrderDTO
                                              |
                                      [mysql.OrderRepository] --CALL sp_place_order--> MySQL
```

- `ordering` depends on `cart`'s and `catalog`'s exported `app` services (cross-module calls through
  public interfaces only, `06_ENGINEERING_STANDARDS.md` §4) — never their `mysql` packages.
- `PaymentProcessor` is a port owned by `ordering/app` (ISP: one method,
  `06_ENGINEERING_STANDARDS.md` §2); its Phase 1 implementation lives in `internal/payment/mysql` as
  `StubProcessor` and is wired in `cmd/api/main.go` — this is the concrete instance of the "database
  is the authority for atomicity, Go domain is the readable statement of intent" split
  (`06_ENGINEERING_STANDARDS.md` §1's caveat): authorization is a plain Go call outside any
  transaction; the stock+order write is entirely inside `sp_place_order`.

## 2. Data model / migrations

Ships as `db/migrations/0004_checkout_schema.up.sql`. Includes `city` (needed by `delivery.city_id`,
and by `fn_estimate_delivery_days` which `sp_place_order` calls) even though `05-delivery-estimation`
owns the *estimate preview UX and city-list curation* — the table and function have to exist for
checkout to work at all, so they ship here; `05` extends the seed data and TBD-2/3 resolution later
without touching this DDL.

```sql
CREATE TABLE city (
    city_id        INT AUTO_INCREMENT PRIMARY KEY,
    name           VARCHAR(100) NOT NULL UNIQUE,
    classification ENUM('Main','Other') NOT NULL DEFAULT 'Other'   -- REQ-6.1
);
-- seed: Houston, San Antonio, Dallas, Austin, Fort Worth, El Paso = 'Main' (TBD-2 working assumption)

CREATE TABLE app_config (                                          -- TBD-1: a data change, not a code change
    config_key   VARCHAR(64) PRIMARY KEY,
    config_value VARCHAR(255) NOT NULL
);
-- seed: ('standard_delivery_fee','9.99'), ('tax_rate_percent','8.25')  -- explicit placeholders, not confirmed

CREATE TABLE `order` (
    order_id        INT AUTO_INCREMENT PRIMARY KEY,
    customer_id     INT NOT NULL,
    status          ENUM('Placed','Confirmed','Processing','Shipped','ReadyForPickup',
                          'Delivered','Completed','Cancelled') NOT NULL DEFAULT 'Confirmed',
    subtotal        DECIMAL(10,2) NOT NULL,
    tax_amount      DECIMAL(10,2) NOT NULL,
    delivery_fee    DECIMAL(10,2) NOT NULL,
    total_amount    DECIMAL(10,2) NOT NULL,
    idempotency_key VARCHAR(64) NOT NULL,                          -- REQ-4.9
    order_date      DATETIME NOT NULL DEFAULT UTC_TIMESTAMP(),     -- REQ-4.5; also what reporting (09) filters on
    FOREIGN KEY (customer_id) REFERENCES customer(customer_id),
    UNIQUE KEY uq_order_idempotency (customer_id, idempotency_key)  -- SEC-CHECKOUT-2: scoped per customer
);
CREATE INDEX ix_order_customer ON `order`(customer_id);            -- REQ-8.4 ownership lookups
CREATE INDEX ix_order_date ON `order`(order_date);                 -- 09-management-reporting date-range reports

CREATE TABLE order_item (
    order_item_id       INT AUTO_INCREMENT PRIMARY KEY,
    order_id            INT NOT NULL,
    variant_id          INT NOT NULL,
    quantity             INT NOT NULL,
    unit_price_at_order  DECIMAL(10,2) NOT NULL,                    -- REQ-4.10
    FOREIGN KEY (order_id)   REFERENCES `order`(order_id)          ON DELETE CASCADE,
    FOREIGN KEY (variant_id) REFERENCES product_variant(variant_id)
);
CREATE INDEX ix_order_item_order   ON order_item(order_id);
CREATE INDEX ix_order_item_variant ON order_item(variant_id);

CREATE TABLE delivery (
    delivery_id    INT AUTO_INCREMENT PRIMARY KEY,
    order_id       INT NOT NULL,
    mode           ENUM('StorePickup','StandardDelivery') NOT NULL,
    city_id        INT NULL,       -- NULL for StorePickup (SQL-BUG-1 fix depends on this being nullable)
    address        VARCHAR(255) NULL,
    estimated_days INT NOT NULL,
    estimated_date DATE NOT NULL,
    FOREIGN KEY (order_id) REFERENCES `order`(order_id) ON DELETE CASCADE,
    FOREIGN KEY (city_id)  REFERENCES city(city_id),
    UNIQUE KEY uq_delivery_order (order_id)
);

-- Owned conceptually by 06-inventory-stock (00_OVERVIEW.md §3), but shipped here: sp_place_order
-- (below) writes to it directly (REQ-5.5, SQL-BUG-2), so it must exist before this feature's
-- migration runs — same reasoning as city/app_config above. 06 reuses this table without redefining it.
CREATE TABLE stock_movement (
    stock_movement_id INT AUTO_INCREMENT PRIMARY KEY,
    variant_id         INT NOT NULL,
    change_qty          INT NOT NULL,                              -- positive = increase, negative = decrease
    reason               VARCHAR(255) NOT NULL,
    related_order_id    INT NULL,                                  -- REQ-5.5: "the related order where one applies"
    acting_user_id       INT NULL,                                 -- NULL for system-driven (checkout) movements
    created_at           DATETIME NOT NULL DEFAULT UTC_TIMESTAMP(),
    FOREIGN KEY (variant_id)        REFERENCES product_variant(variant_id),
    FOREIGN KEY (related_order_id)  REFERENCES `order`(order_id),
    FOREIGN KEY (acting_user_id)    REFERENCES user_account(user_account_id)
);
CREATE INDEX ix_stock_movement_variant ON stock_movement(variant_id);
CREATE INDEX ix_stock_movement_order   ON stock_movement(related_order_id);

-- Owned conceptually by 08-order-tracking-status, shipped here for the same reason as
-- stock_movement above: sp_place_order writes this table's first row directly (SQL-BUG-3), since
-- trg_order_status_history (07 §4, added by 08) is an AFTER UPDATE trigger and never fires for the
-- INSERT that creates the order. 08 adds the triggers that keep it updated after this point.
CREATE TABLE order_status_history (
    order_status_history_id INT AUTO_INCREMENT PRIMARY KEY,
    order_id            INT NOT NULL,
    status               ENUM('Placed','Confirmed','Processing','Shipped','ReadyForPickup',
                               'Delivered','Completed','Cancelled') NOT NULL,
    changed_by_user_id  INT NULL,        -- NULL for the system-created initial row (SQL-BUG-3)
    changed_at           DATETIME NOT NULL DEFAULT UTC_TIMESTAMP(),
    FOREIGN KEY (order_id)           REFERENCES `order`(order_id)       ON DELETE CASCADE,
    FOREIGN KEY (changed_by_user_id) REFERENCES user_account(user_account_id)
);
CREATE INDEX ix_order_status_history_order ON order_status_history(order_id);

CREATE TABLE payment (
    payment_id         INT AUTO_INCREMENT PRIMARY KEY,
    order_id           INT NOT NULL,
    method              ENUM('COD','Card') NOT NULL,                -- REQ-7.1
    status              ENUM('Pending','Authorized','Paid','Failed','Refunded') NOT NULL,  -- REQ-7.2
    amount              DECIMAL(10,2) NOT NULL,
    provider_reference  VARCHAR(100) NULL,                          -- REQ-7.5
    card_last_four      CHAR(4) NULL,                               -- REQ-7.5 — never more than this
    created_at          DATETIME NOT NULL DEFAULT UTC_TIMESTAMP(),
    FOREIGN KEY (order_id) REFERENCES `order`(order_id) ON DELETE CASCADE,
    UNIQUE KEY uq_payment_order (order_id)
);
```

Plus `fn_classify_city`, `fn_estimate_delivery_days` (mode-aware, `SQL-BUG-1`), `sp_place_order` — all
already fully specified in `07_SQL_DATABASE_STANDARDS.md` §3.1/§5 and shipped as part of this same
migration's companion procedure/function migration file
(`db/migrations/0004a_checkout_procedures.up.sql` — kept in a **separate** file from the table DDL per
`07` §1's naming convention, so a procedure-only change doesn't touch table migrations).

## 3. API contract

Already in `specs/openapi/openapi.yaml`: `POST /checkout` (`checkout` tag), `GET /orders`,
`GET /orders/{orderId}` (`orders` tag). No changes needed — this table is a summary.

| Method | Path | Auth | FR |
|---|---|---|---|
| POST | `/api/v1/checkout` | customer | FR-CHECKOUT-1..18,20 |
| GET | `/api/v1/orders` | customer | FR-CHECKOUT-19 |
| GET | `/api/v1/orders/{orderId}` | customer | FR-CHECKOUT-19 |

## 4. Stored procedures / functions / triggers / views used

- Uses: `sp_place_order` (`07` §3.1 — this feature's reason for existing, now also writing
  `stock_movement` rows per `SQL-BUG-2` and the initial `order_status_history` row per `SQL-BUG-3`),
  `fn_estimate_delivery_days` (`07` §5, mode-aware per `SQL-BUG-1`).
- New (ships with this feature's migration, per §2): `trg_stock_nonnegative` (`07` §4) — must exist
  before `sp_place_order` ever runs, since it's that procedure's own backstop; conceptually a
  `06-inventory-stock` rule, but this is the feature that first needs it live.
- Beyond what `07` already specifies: the schema in §2 above (`07` didn't carry `order`/`order_item`/
  `delivery`/`payment`/`stock_movement`/`city`/`app_config` DDL, only the procedures/functions that
  operate on them).

## 5. Key logic / code snippets (Go, illustrative)

### 5.1 The orchestration — auth outside the transaction, everything else inside it (REQ-4.8)
```go
func (s *CheckoutService) PlaceOrder(ctx context.Context, customerID int, req PlaceOrderRequest) (*Order, error) {
    cart, err := s.cart.Get(ctx, customerID)              // cross-module call, cart's public app service
    if err != nil || len(cart.Items) == 0 { return nil, ErrEmptyCart }

    if err := validateDeliveryFields(req); err != nil { return nil, err }         // FR-CHECKOUT-4

    taxRate := s.config.Get("tax_rate_percent")
    deliveryFee := decimal.Zero
    if req.DeliveryMode == "StandardDelivery" { deliveryFee = s.config.Get("standard_delivery_fee") }
    subtotal := cart.Subtotal()
    taxAmount := subtotal.Mul(taxRate).Div(100)                                    // FR-CHECKOUT-11

    var paymentRef PaymentAuth
    if req.PaymentMethod == "Card" {
        paymentRef, err = s.payments.Authorize(ctx, subtotal.Add(taxAmount).Add(deliveryFee), req.CardToken)
        if err != nil { return nil, &PaymentFailedError{Cause: err} }              // FR-CHECKOUT-14; no DB tx opened yet
    }

    var orderID int
    err = s.tx.WithTx(ctx, func(tx dbx.Tx) error {
        var procErr error
        orderID, procErr = s.orders.CallPlaceOrder(ctx, tx, sp_place_order.Args{   // FR-CHECKOUT-2,6,7
            CustomerID: customerID, Items: cart.ItemsAsJSON(), DeliveryMode: req.DeliveryMode,
            CityID: req.DeliveryCityID, Address: req.DeliveryAddress, PaymentMethod: req.PaymentMethod,
            IdempotencyKey: idemKeyFromHeader(ctx), TaxAmount: taxAmount, DeliveryFee: deliveryFee,
        })
        if procErr != nil { return procErr }              // STOCK_EXCEEDED -> rolled back automatically (FR-CHECKOUT-3)
        return s.carts.ClearIfNewOrder(ctx, tx, customerID, orderID) // skip on idempotent replay, §6
    })
    if isStockExceeded(err) {
        unavailable, _ := s.orders.DiagnoseUnavailableLines(ctx, cart) // read-only re-check, §6 — for the error body only
        return nil, &StockExceededError{Lines: unavailable}
    }
    if err != nil { return nil, err }
    return s.orders.GetByID(ctx, orderID)
}
```

### 5.2 Payment stub (REQ-7.7) — the interface Card/COD both satisfy, real gateway drops in later
```go
type PaymentProcessor interface {
    Authorize(ctx context.Context, amount Money, cardToken string) (PaymentAuth, error)
}

type PaymentAuth struct { ProviderReference string; LastFour string }

// internal/payment/mysql/stub_processor.go — Phase 1 only, never sees a real card network
func (StubProcessor) Authorize(ctx context.Context, amount Money, cardToken string) (PaymentAuth, error) {
    if cardToken == "" { return PaymentAuth{}, ErrCardTokenRequired }
    return PaymentAuth{ProviderReference: "stub-" + uuid.NewString(), LastFour: "4242"}, nil
    // REQ-7.5/7.6: this type has no field for a card number/CVV/expiry by construction — there is
    // nothing for a future mistake to accidentally log or persist.
}
```

## 6. Edge cases & failure handling

- **`STOCK_EXCEEDED` from `sp_place_order`:** the transaction is already rolled back by the time Go
  sees the error (InnoDB rolls back on an unhandled `SIGNAL` inside the procedure/transaction). Go then
  runs a **separate, read-only** diagnostic query re-checking each cart line's current stock, purely to
  build a useful error body (`{unavailableLines: [...]}`) — this query is not part of the atomicity
  guarantee, just UX (FR-CHECKOUT-3, AC-CHECKOUT-2).
- **Card authorization fails:** no DB transaction was ever opened (§5.1) — the cart is untouched by
  construction, not by a rollback (AC-CHECKOUT-4).
- **Idempotent replay:** `sp_place_order` itself detects the existing `(customer_id,
  idempotency_key)` row and returns the original `order_id` without re-running the stock logic
  (`07` §3.1). Go's cart-clearing step is skipped on this path (`ClearIfNewOrder` checks whether this
  call actually created a new order vs. returned an existing one) — a replay must not re-clear
  (harmlessly idempotent here, but modeled explicitly rather than relying on that).
- **Standard Delivery with missing city/address, or Store Pickup with a city/address supplied
  anyway:** `400` at validation, before any payment authorization or DB work (§5.1 step 2).
- **Empty cart at checkout:** `400`, distinct from a stock failure — nothing to charge or order.
- **Two concurrent checkouts, same last-unit variant:** both call `sp_place_order`; MySQL's `FOR
  UPDATE` row lock serializes them (whichever transaction locks the row first proceeds, the second
  sees the now-decremented `stock_quantity` and fails its own check) — this is exactly REQ-4.7's
  guarantee, exercised as a real concurrency integration test, not just reasoned about (AC-CHECKOUT-5).

## 7. Which global rules apply (do not restate, just reference)

- Security: `02` §1 (customer-only), §3 (card data never stored raw), §6.2 (ownership on `GET
  /orders/{orderId}`).
- SQL: `07` §3.1 (the procedure this feature is built around), §4 (`trg_stock_nonnegative` backstop).
- NFR: `03` §1, this feature's own `NFR-CHECKOUT-1`.

## 8. Test hooks (→ tasks.md)

- Unit: tax/fee computation from `app_config`; delivery-field validation matrix (mode × required
  fields); `PaymentFailedError`/`StockExceededError` mapping to `409`.
- Integration (`testcontainers-go`, the ones that matter most in this whole project):
  **concurrent-checkout race on the last unit of a variant** (AC-CHECKOUT-5 — this is the test that
  actually exercises REQ-4.7's row-locking guarantee); idempotency replay returns the same order and
  does not double-decrement stock; card-auth-failure leaves stock and cart untouched; `unit_price_at_order`
  survives a later catalog price change (AC-CHECKOUT-6); ownership on `GET /orders/{orderId}`
  (AC-CHECKOUT-7).
