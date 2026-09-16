# 08-order-tracking-status — Spec (WHAT & WHY)

**Feature slug:** `ORDERSTATUS` · **Status:** ⚠️ draft-for-review · **Companion:** [plan.md](plan.md) · [tasks.md](tasks.md)
**Inherits:** `global/02_SECURITY_BASELINE.md`, `global/03_NFR_BASELINE.md`
**SRS section:** §4.8 Order Tracking and Status Management (Medium priority) — staff-facing half

---

## 1. Summary

Lets `ORDER_MANAGER` advance an order through its lifecycle and cancel it when needed, with every
transition validated and every change automatically audited. The customer-facing half of REQ-8
(REQ-8.4, viewing one's own orders) already shipped with `04-checkout-orders`, since it needed no
staff involvement to build. This feature is specifically the **write** side: status transitions and
cancellation.

## 2. Actors & scope

- **In scope:** staff status advancement with transition validation; order cancellation with stock
  return; the Cash-on-Delivery → Paid transition on delivery (REQ-7.4, deferred here from
  `04-checkout-orders`).
- **Out of scope, already owned elsewhere:** order creation (`04`), customer order history/detail
  (`04`), the SQL-level triggers' *existence* (they ship with `04`'s migration per `SQL-BUG-3`'s
  resolution — this feature is what actually *invokes* them by updating order status for the first
  time in the system).
- **Working assumptions used:** none new.

## 3. User stories

| ID | As a… | I want… | so that… | Priority |
|---|---|---|---|---|
| US-ORDERSTATUS-1 | order manager | to move an order from Confirmed to Processing to Shipped | fulfilment progresses visibly | 🟡 P2 |
| US-ORDERSTATUS-2 | order manager | to be stopped from making an invalid transition (e.g. Delivered back to Processing) | the status stays trustworthy | 🟡 P2 |
| US-ORDERSTATUS-3 | order manager | to cancel an order and have its stock returned automatically | inventory doesn't need a manual fix afterward | 🟡 P2 |
| US-ORDERSTATUS-4 | order manager | a Cash-on-Delivery order's payment to be marked Paid once it's Delivered | the payment record reflects reality without a separate manual step | 🟡 P2 |

## 4. Functional requirements

| ID | Requirement | SRS ref | Priority |
|---|---|---|---|
| FR-ORDERSTATUS-1 | The system MUST maintain an order status for every order, drawn from Placed, Confirmed, Processing, Shipped, Ready for Pickup, Delivered, Completed, Cancelled. | REQ-8.1 | 🟡 P2 |
| FR-ORDERSTATUS-2 | The system MUST permit only valid status transitions; a Delivered order MUST NOT return to Processing, and a Cancelled order is terminal. | REQ-8.2 | 🟡 P2 |
| FR-ORDERSTATUS-3 | `ORDER_MANAGER` MUST be able to update an order's status, and an invalid transition MUST be rejected with a clear message. | REQ-8.3 | 🟡 P2 |
| FR-ORDERSTATUS-4 | Every status change MUST be recorded in an order status history with a timestamp and the acting user. | REQ-8.5 | 🟡 P2 |
| FR-ORDERSTATUS-5 | Cancelling an order that has already decremented stock MUST return the reserved quantities to the affected variants within the same transaction. | REQ-8.6 | 🟡 P2 |
| FR-ORDERSTATUS-6 | For Cash on Delivery, payment status MUST become Paid when staff mark the order Delivered. | REQ-7.4 | 🟡 P2 |

## 5. NFR deltas (feature-specific only)

None beyond `global/03_NFR_BASELINE.md`.

## 6. Security deltas (feature-specific only)

| ID | Requirement | Closes |
|---|---|---|
| SEC-ORDERSTATUS-1 | Status update and cancellation both require permission `order:status:update`/`order:cancel` — default-granted to `ORDER_MANAGER` (RBAC-1). | — |

## 7. Acceptance criteria (Given/When/Then)

- **AC-ORDERSTATUS-1** — *Given* an order at `Confirmed`, *When* staff advance it to `Processing`,
  *Then* the status updates and an `order_status_history` row is appended automatically.
- **AC-ORDERSTATUS-2** — *Given* an order at `Delivered`, *When* staff attempt to set it to
  `Processing`, *Then* the request is rejected (`409`) and the status is unchanged.
- **AC-ORDERSTATUS-3** — *Given* a `Cancelled` order, *When* staff attempt any status change on it,
  *Then* the request is rejected — Cancelled is terminal.
- **AC-ORDERSTATUS-4** — *Given* an order that decremented 2 units of a variant, *When* staff cancel
  it, *Then* the variant's stock increases by 2 and a `stock_movement` row records the return with
  `related_order_id` set.
- **AC-ORDERSTATUS-5** — *Given* a Cash-on-Delivery order at any pre-Delivered status, *When* staff
  set it to `Delivered`, *Then* `payment.status` becomes `Paid` in the same request — not a separate
  staff action.
- **AC-ORDERSTATUS-6** — *Given* a Card-paid order (already `Paid` at checkout), *When* staff set it
  to `Delivered`, *Then* `payment.status` remains `Paid` — this rule only fires for COD.

## 8. Open questions / decisions needed

- [ ] None outstanding. Note: orders in this system are created directly at status `Confirmed`
      (`sp_place_order` — REQ-4.2 treats atomic creation *as* confirmation); `Placed` exists in the
      SRS's status enum but the SRS's own Appendix B.3 state diagram shows no distinct "Placed" state
      either (`ActiveCart → Checkout → OrderConfirmed`). `Placed` is kept in the schema enum for
      fidelity to REQ-8.1's exact wording but is not expected to appear in Phase 1 data — flagged here
      so it isn't mistaken for a bug when it never shows up in a test fixture.
