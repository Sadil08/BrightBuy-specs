# 05-delivery-estimation — Spec (WHAT & WHY)

**Feature slug:** `DELIVERY` · **Status:** ⚠️ draft-for-review · **Companion:** [plan.md](plan.md) · [tasks.md](tasks.md)
**Inherits:** `global/02_SECURITY_BASELINE.md`, `global/03_NFR_BASELINE.md`
**SRS section:** §4.6 Delivery Estimation (Medium priority)

---

## 1. Summary

Lets a customer see an estimated delivery date **before** confirming checkout. The estimation rule
itself (`fn_estimate_delivery_days`/`fn_classify_city`) and the `city` reference table were built as
part of `04-checkout-orders` — they had to exist for `sp_place_order` to work at all. This feature is
the **preview** surface: a public endpoint the cart/checkout UI calls as the user picks a delivery
mode/city, calling the exact same DB function checkout itself uses, so the number shown pre-checkout
can never diverge from what actually gets stored on the order.

## 2. Actors & scope

- **In scope:** `POST /delivery/estimate` preview endpoint; curating the seeded Main-City list
  (`TBD-2`); confirming the Store Pickup out-of-stock assumption (`TBD-3`) is correctly wired end to
  end from the API, not just inside the SQL function.
- **Out of scope (explicit):** the function/table DDL itself (`04-checkout-orders/plan.md` §2, already
  built); persisting the final estimate on an order (also `04`, via `sp_place_order`'s `INSERT INTO
  delivery`); a city-management admin UI (no SRS requirement asks for one — `city` is
  migration-seeded/maintained directly for Phase 1, revisited only if a real need surfaces).

## 3. User stories

| ID | As a… | I want… | so that… | Priority |
|---|---|---|---|---|
| US-DELIVERY-1 | registered customer | to see an estimated delivery date before I confirm my order | I know when to expect it | 🟡 P2 |
| US-DELIVERY-2 | registered customer | Store Pickup to tell me if it's ready today or delayed | I know whether to drive over immediately | 🟡 P2 |

## 4. Functional requirements

| ID | Requirement | SRS ref | Priority |
|---|---|---|---|
| FR-DELIVERY-1 | The system MUST classify every Texas destination city as Main or Other using a maintainable reference table, not a hard-coded list. | REQ-6.1 | 🟡 P2 |
| FR-DELIVERY-2 | For Standard Delivery, the system MUST compute a base estimate of 5 days for a Main City and 7 days for an Other City when every ordered item is in stock. | REQ-6.2 | 🟡 P2 |
| FR-DELIVERY-3 | The system MUST add 3 days to the base estimate, once per order, when one or more items are out of stock at estimation time. | REQ-6.3 | 🟡 P2 |
| FR-DELIVERY-4 | Estimates MUST be expressed in calendar days from the order date. | REQ-6.4 | 🟡 P2 |
| FR-DELIVERY-5 | The system MUST display the estimated delivery date to the customer **before** order confirmation, via a preview call that uses the same computation checkout itself will use. | REQ-6.5 | 🟡 P2 |
| FR-DELIVERY-6 | Store Pickup MUST NOT use the Main/Other City rule; where every item is in stock, it is ready the same business day; when an item is out of stock, it follows the same +3-day rule as Standard Delivery (working assumption, `TBD-3`). | REQ-6.6 | 🟡 P2 |

## 5. NFR deltas (feature-specific only)

None — a single scalar SQL function call, well within `global/03` §1's budgets.

## 6. Security deltas (feature-specific only)

None beyond `global/02` §1 — this endpoint is intentionally public (a guest previews before
registering), and it reveals nothing sensitive: no stock quantities, no pricing, just a day count.

## 7. Acceptance criteria (Given/When/Then)

- **AC-DELIVERY-1** — *Given* a Standard Delivery request to a Main City with every item in stock,
  *When* the estimate is previewed, *Then* the response shows 5 days.
- **AC-DELIVERY-2** — *Given* the same request but to an Other City, *When* previewed, *Then* the
  response shows 7 days.
- **AC-DELIVERY-3** — *Given* a Standard Delivery request where one item is out of stock, *When*
  previewed, *Then* 3 days are added once (not per out-of-stock line).
- **AC-DELIVERY-4** — *Given* a Store Pickup request with every item in stock, *When* previewed,
  *Then* the response shows 0 days / ready today, regardless of any `cityId` present in the request
  (ignored per REQ-6.6).
- **AC-DELIVERY-5** — *Given* a Store Pickup request with an out-of-stock item, *When* previewed,
  *Then* the response shows 3 days (the `TBD-3` working assumption).
- **AC-DELIVERY-6** — *Given* a previewed estimate and the same cart confirmed at checkout moments
  later with no stock change in between, *When* the two are compared, *Then* they match exactly — same
  function, same inputs, same output (this is the actual point of the feature).

## 8. Open questions / decisions needed

- [ ] `TBD-2` (definitive Main-City list) and `TBD-3` (pickup-out-of-stock timing) remain open as
      *business* decisions — this feature ships against the working assumptions already recorded in
      `global/10_BA_AUDIT_OPEN_QUESTIONS.md`; only the underlying seed data or the `IF` branch in
      `fn_estimate_delivery_days` would need to change if BrightBuy answers differently, not the API
      shape.
