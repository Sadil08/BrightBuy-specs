# 08-order-tracking-status — Plan (HOW)

**Feature slug:** `ORDERSTATUS` · **Status:** ⚠️ draft-for-review · **Companion:** [spec.md](spec.md) · [tasks.md](tasks.md)
**Applies:** `global/01_TECH_STACK.md`, `global/06_ENGINEERING_STANDARDS.md`, `global/07_SQL_DATABASE_STANDARDS.md` §3.3, §4

---

## 1. Approach & architecture

```
[Staff console] --PATCH /staff/orders/{id}/status--> [httpapi.OrderStatusHandler] --> [app.OrderStatusService]
                                                              |
                                              1. Go transition-table check (fast, clear error)
                                              2. dbx.WithTx( UPDATE order SET status = ?;
                                                              IF Delivered AND COD: UPDATE payment SET status='Paid'; )
                                                    -> trg_order_status_transition backstops step 2 (07 §4)
                                                    -> trg_order_status_history auto-logs it (07 §4)

[Staff console] --POST /staff/orders/{id}/cancel--> [app.OrderStatusService.Cancel] --CALL sp_cancel_order--> MySQL
```

Two layers validate the same rule on purpose (`06_ENGINEERING_STANDARDS.md` §1's caveat, applied here
exactly as it was for stock): the Go transition table gives a fast, specific error message; the
database trigger is what actually guarantees an invalid transition can never be committed, even if the
Go check has a bug or a future code path skips it.

## 2. Data model / migrations

No new tables — `order_status_history` already exists (`04-checkout-orders/plan.md` §2, per
`SQL-BUG-3`'s resolution). This feature ships `db/migrations/0008_order_status_procedures.up.sql`:
`trg_order_status_transition`, `trg_order_status_history` (`07_SQL_DATABASE_STANDARDS.md` §4),
`sp_cancel_order` (`07` §3.3, already writing `stock_movement` per `SQL-BUG-2`).

## 3. API contract

Already in `specs/openapi/openapi.yaml` (`orders` tag): `PATCH /staff/orders/{orderId}/status`.
**One addition needed:** the spec's `OrderStatusUpdateRequest` only carries `status`, which is
sufficient for advancement but `sp_cancel_order` is a distinct procedure (not just "set status to
Cancelled" via the generic path) since it does the stock-return work — the handler routes a `status:
"Cancelled"` request to `Cancel()` rather than the plain `UpdateStatus()` path internally; no separate
endpoint or schema change needed, this is an implementation-routing detail (`plan.md` §5.1).

| Method | Path | Auth | FR |
|---|---|---|---|
| PATCH | `/api/v1/staff/orders/{orderId}/status` | `order:status:update` (advance) / `order:cancel` (→ Cancelled) | FR-ORDERSTATUS-1..6 |

## 4. Stored procedures / functions / triggers / views used

- Uses: `sp_cancel_order` (`07` §3.3 — this feature's reason for existing).
- New (ships with this feature's migration): `trg_order_status_transition`, `trg_order_status_history`
  (`07` §4) — these are the **first** things in the whole system that `UPDATE order.status`, since
  `04` only ever `INSERT`s an order; this feature is where they're actually exercised for the first
  time end to end.

## 5. Key logic / code snippets (Go, illustrative)

### 5.1 Status update — transition table + COD→Paid side effect (FR-ORDERSTATUS-2,3,6)
```go
var validTransitions = map[string][]string{
    "Confirmed":      {"Processing", "Cancelled"},
    "Processing":     {"Shipped", "ReadyForPickup", "Cancelled"},
    "Shipped":        {"Delivered", "Cancelled"},
    "ReadyForPickup": {"Delivered", "Cancelled"},
    "Delivered":      {"Completed"},
    // Completed, Cancelled: no outgoing transitions — terminal
}

func (s *OrderStatusService) UpdateStatus(ctx context.Context, actingUserID, orderID int, newStatus string) error {
    if newStatus == "Cancelled" {
        return s.Cancel(ctx, actingUserID, orderID) // routes to sp_cancel_order, not a plain UPDATE
    }
    order, _ := s.repo.GetByID(ctx, orderID)
    if !slices.Contains(validTransitions[order.Status], newStatus) {
        return &InvalidTransitionError{From: order.Status, To: newStatus} // -> 409, FR-ORDERSTATUS-3
    }
    return s.tx.WithTx(ctx, func(tx dbx.Tx) error {
        ctx = withActingUser(ctx, actingUserID) // SET @acting_user_id — trg_order_status_history reads this (07 §4)
        if err := s.repo.SetStatus(ctx, tx, orderID, newStatus); err != nil { return err }
        if newStatus == "Delivered" && order.PaymentMethod == "COD" {
            return s.payments.SetStatus(ctx, tx, orderID, "Paid") // REQ-7.4, FR-ORDERSTATUS-6
        }
        return nil
    })
}
```

### 5.2 `@acting_user_id` — how the trigger learns who made the change
```go
func withActingUser(ctx context.Context, userID int) context.Context {
    // executed as the first statement of the same session/transaction, per 07_SQL_DATABASE_STANDARDS.md §4
    return dbx.RunFirst(ctx, "SET @acting_user_id = ?", userID)
}
```

## 6. Edge cases & failure handling

- Cancelling an order that's already `Delivered` → `sp_cancel_order` would attempt `UPDATE order SET
  status = 'Cancelled'`, which `trg_order_status_transition` rejects — the procedure's own `UPDATE`
  fails, the whole call rolls back, **no stock is returned** (correctly — an already-delivered order
  shouldn't un-decrement stock that's physically out the door). Go's `Cancel()` doesn't need its own
  pre-check for this case; the trigger is authoritative here by design.
- COD order marked `Delivered` twice (double-click) → the second call is itself an invalid transition
  (`Delivered` has no outgoing edge to `Delivered`) — rejected before the payment side-effect would
  run again, so `payment.status` is never redundantly "set" (harmless either way, but the transition
  guard means it's not even attempted).
- Store Pickup order skipping `Shipped` and going straight `Processing → ReadyForPickup → Delivered` —
  this is exactly why `Shipped` and `ReadyForPickup` are both valid targets from `Processing` and both
  valid sources for `Delivered`, rather than a single linear chain.

## 7. Which global rules apply (do not restate, just reference)

- Security: `02` §1 (`order:status:update`/`order:cancel`).
- SQL: `07` §3.3, §4 (both triggers — this is their first real caller).
- NFR: `03` §1.

## 8. Test hooks (→ tasks.md)

- Unit: transition-table completeness (every non-terminal status has at least one valid outgoing
  edge, every terminal status has none — a table-driven test over the map itself, not just a few
  example cases); COD→Paid side-effect only fires for COD.
- Integration (`testcontainers-go`): a full Confirmed→Processing→Shipped→Delivered→Completed walk,
  asserting one `order_status_history` row per step with the correct acting user; an invalid
  transition attempted directly via SQL (bypassing Go entirely) is still rejected by the trigger;
  cancellation returns stock and writes the `stock_movement` row; cancellation of a `Delivered` order
  is rejected and returns no stock.
