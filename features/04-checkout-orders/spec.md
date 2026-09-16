# 04-checkout-orders — Spec (WHAT & WHY)

**Feature slug:** `CHECKOUT` · **Status:** ⚠️ draft-for-review · **Companion:** [plan.md](plan.md) · [tasks.md](tasks.md)
**Inherits:** `global/02_SECURITY_BASELINE.md`, `global/03_NFR_BASELINE.md`, `global/07_SQL_DATABASE_STANDARDS.md`
**SRS section:** §4.4 Checkout and Order Placement (High), §4.7 Payment Processing (High), §4.8 (customer-facing half only)

---

## 1. Summary

The single most business-critical feature: turns a cart into a confirmed order, atomically —
BrightBuy must never sell stock it doesn't have, and a retried/double-clicked confirmation must never
create two orders. Also covers the two supported payment methods and the customer's own read-only
order history/detail. This is where `sp_place_order` (`07_SQL_DATABASE_STANDARDS.md` §3.1) is actually
called from Go, and where `TBD-1` (delivery fee/tax rate) becomes real configuration.

**Explicitly not this feature:** staff-side order status advancement, cancellation, and the audit
trail (`08-order-tracking-status`, REQ-8.1/8.2/8.3/8.5/8.6) — this feature only *creates* an order at
status `Confirmed` and lets the owning customer *read* it back.

## 2. Actors & scope

- **In scope:** checkout (stock re-check, delivery capture, payment method capture, atomic order
  creation, idempotency), Card/COD payment lifecycle up to order creation, customer order
  history/detail (own orders only).
- **Out of scope (explicit):** order status transitions after `Confirmed` (`08`); delivery-estimate
  *preview* before checkout and city-list curation (`05-delivery-estimation` — this feature calls the
  same underlying function but doesn't own the preview UX or `TBD-2`/`TBD-3`'s resolution).
- **Working assumptions used:** `TBD-1` (`global/10_BA_AUDIT_OPEN_QUESTIONS.md`) — delivery fee and
  tax rate are real `app_config` rows in this feature, seeded with the SRS's own placeholder values
  ($9.99 flat fee, 8.25%), explicitly not yet BrightBuy-confirmed. `SQL-BUG-1`
  (`global/09_ISSUES_REGISTER.md`) — the delivery-estimate function this feature calls was fixed to be
  mode-aware while drafting this spec.

## 3. User stories

| ID | As a… | I want… | so that… | Priority |
|---|---|---|---|---|
| US-CHECKOUT-1 | registered customer | to check out my cart with a delivery mode and payment method | I receive my order | 🔴 P0 |
| US-CHECKOUT-2 | registered customer | a double-click on "confirm" to not charge or order me twice | I don't get billed twice by an accident | 🔴 P0 |
| US-CHECKOUT-3 | registered customer | to be told exactly which items are unavailable if checkout fails | I can fix my cart and retry | 🟠 P1 |
| US-CHECKOUT-4 | registered customer | to see my past orders and their status | I can track what I bought | 🟠 P1 |

## 4. Functional requirements

| ID | Requirement | SRS ref | Priority |
|---|---|---|---|
| FR-CHECKOUT-1 | The system MUST re-check stock availability for every cart line at the moment of order confirmation, immediately before committing. | REQ-4.1 | 🔴 P0 |
| FR-CHECKOUT-2 | The system MUST create the order, its items, and decrement matching variant stock inside a single atomic database transaction. | REQ-4.2 | 🔴 P0 |
| FR-CHECKOUT-3 | If any cart line cannot be fulfilled at confirmation, the system MUST roll back the entire transaction and place no partial order. | REQ-4.3 | 🔴 P0 |
| FR-CHECKOUT-4 | The system MUST capture delivery details (mode, and for Standard Delivery a destination city and address) and a payment method before an order can be confirmed. | REQ-4.4 | 🔴 P0 |
| FR-CHECKOUT-5 | The system MUST generate a unique order identifier and creation timestamp for every confirmed order. | REQ-4.5 | 🔴 P0 |
| FR-CHECKOUT-6 | The stock check and decrement MUST run as a MySQL stored procedure invoked inside the application transaction. | REQ-4.6 | 🔴 P0 |
| FR-CHECKOUT-7 | The stored procedure MUST lock affected `product_variant` rows `FOR UPDATE`, acquired in ascending `variant_id` order, so concurrent checkouts covering the same variants cannot deadlock. | REQ-4.7 | 🔴 P0 |
| FR-CHECKOUT-8 | For Card Payment, authorization MUST complete before the order transaction opens; no database transaction may be held open across a call to the payment provider. | REQ-4.8 | 🔴 P0 |
| FR-CHECKOUT-9 | The checkout endpoint MUST accept an idempotency key and return the previously created order when the same key is replayed by the same customer. | REQ-4.9 | 🔴 P0 |
| FR-CHECKOUT-10 | The order MUST record each variant's unit price as it stood at confirmation; a later catalog price change must not alter an order already placed. | REQ-4.10 | 🔴 P0 |
| FR-CHECKOUT-11 | The order total MUST be the sum of line values plus tax and any delivery fee, computed at confirmation from configuration. | REQ-4.11 | 🔴 P0 |
| FR-CHECKOUT-12 | The system MUST support exactly two payment methods per order — Cash on Delivery and Card Payment — recording exactly one per order. | REQ-7.1 | 🔴 P0 |
| FR-CHECKOUT-13 | The system MUST record and update a payment status through its lifecycle: Pending, Authorized, Paid, Failed, Refunded. | REQ-7.2 | 🔴 P0 |
| FR-CHECKOUT-14 | For Card Payment, the system MUST NOT create the order unless authorization succeeds. | REQ-7.3 | 🔴 P0 |
| FR-CHECKOUT-15 | For Cash on Delivery, payment status MUST remain Pending until staff mark the order Delivered (implemented in `08-order-tracking-status`; this feature only sets the initial Pending state). | REQ-7.4 | 🔴 P0 |
| FR-CHECKOUT-16 | The system MUST NOT store a raw card number, CVV, or expiry date — only a provider reference and the last four digits may be retained. | REQ-7.5 | 🔴 P0 |
| FR-CHECKOUT-17 | Card details MUST pass through the payment port and MUST NOT be written to application logs. | REQ-7.6 | 🔴 P0 |
| FR-CHECKOUT-18 | The payment integration MUST sit behind an interface with a stub implementation for Phase 1. | REQ-7.7 | 🔴 P0 |
| FR-CHECKOUT-19 | A registered customer MUST be able to view the status and detail of their own orders only; a request for another customer's order MUST be refused (404) whether or not the identifier is valid. | REQ-8.4 | 🔴 P0 |
| FR-CHECKOUT-20 | Only registered customers MAY check out; unauthenticated checkout attempts MUST be rejected. | global business rule (`03_NFR_BASELINE.md` §4) | 🔴 P0 |

## 5. NFR deltas (feature-specific only)

| ID | NFR | Target |
|---|---|---|
| NFR-CHECKOUT-1 | Checkout transaction commits or rolls back within 3s, excluding time waiting on the payment stub. | `global/03` §1, restated because it's this feature's own headline budget. |

## 6. Security deltas (feature-specific only)

| ID | Requirement | Closes |
|---|---|---|
| SEC-CHECKOUT-1 | The order total is always server-recomputed from `product_variant.price` + `app_config` tax/fee — a client-sent total or unit price is never trusted, even if present in the request. | T-8 |
| SEC-CHECKOUT-2 | The idempotency key is scoped per-customer (`unique(customer_id, idempotency_key)`) — one customer cannot replay another's key to probe for an order. | — |
| SEC-CHECKOUT-3 | Card token/details reach the backend only to be forwarded to `PaymentProcessor.Authorize` — never logged, never persisted beyond `provider_reference`/`last_four` (REQ-7.5/7.6). | T-14 |

## 7. Acceptance criteria (Given/When/Then)

- **AC-CHECKOUT-1** — *Given* an authenticated customer with a valid cart, *When* they confirm
  checkout with Standard Delivery + COD, *Then* an order is created at status `Confirmed`, stock is
  decremented for every line, and the response includes the order id, total, and estimated delivery
  date.
- **AC-CHECKOUT-2** — *Given* a cart line whose stock dropped below the cart quantity between add and
  checkout, *When* checkout is confirmed, *Then* the whole order is rejected (`409`), no stock is
  decremented for *any* line, and the response names the unavailable line(s).
- **AC-CHECKOUT-3** — *Given* a customer double-clicks confirm (same `Idempotency-Key` header sent
  twice), *When* both requests reach the server, *Then* exactly one order exists and both responses
  return the same order id.
- **AC-CHECKOUT-4** — *Given* Card Payment is selected and authorization fails, *When* checkout is
  confirmed, *Then* no order is created, the cart is left intact, and the response indicates payment
  failure distinctly from a stock failure.
- **AC-CHECKOUT-5** — *Given* two customers simultaneously check out carts that both include the last
  unit of the same variant, *When* both requests are confirmed concurrently, *Then* exactly one
  succeeds and the other receives a `409` stock-exceeded response — never both succeeding, never a
  deadlock.
- **AC-CHECKOUT-6** — *Given* a confirmed order, *When* its owning customer fetches it later, *Then*
  its line items show the unit price **at the time of the order**, even if the catalog price has since
  changed.
- **AC-CHECKOUT-7** — *Given* customer A's order, *When* customer B requests it by id, *Then* the
  response is `404`.

## 8. Open questions / decisions needed

- [ ] `TBD-1` remains open at the BrightBuy-business-decision level (this feature just makes it a real,
      changeable config row rather than a blocker) — flag before this feature is considered "done" for
      a real launch, not just "working" for the course deliverable.
