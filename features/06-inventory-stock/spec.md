# 06-inventory-stock — Spec (WHAT & WHY)

**Feature slug:** `INVENTORY` · **Status:** ⚠️ draft-for-review · **Companion:** [plan.md](plan.md) · [tasks.md](tasks.md)
**Inherits:** `global/02_SECURITY_BASELINE.md`, `global/03_NFR_BASELINE.md`
**SRS section:** §4.5 Inventory and Stock Management (High priority)

---

## 1. Summary

Lets `WAREHOUSE_STAFF` look up a variant by SKU or product name (seeing its **exact** quantity, unlike
the public catalog) and manually adjust stock — restocks, corrections — with every change logged.
Several of REQ-5.*'s requirements are already implemented by earlier features because they had to be
(atomicity can't wait); this spec is explicit about which half is new here and which is being
verified, not re-owned.

## 2. Actors & scope

- **In scope:** staff variant lookup (exact quantity); manual stock adjustment (`sp_adjust_stock`);
  the negative-adjustment rejection.
- **Out of scope, already owned elsewhere (verified, not redefined, by this feature's tests):**
  SKU uniqueness (REQ-5.1, `01-catalog`), automatic order-driven decrement + never-negative guarantee
  (REQ-5.3, `04-checkout-orders`), derived In-Stock/Out-of-Stock status (REQ-5.4, `01-catalog`), the
  order-driven half of the stock movement log (REQ-5.5, `04-checkout-orders`/`08-order-tracking-status`).
- **Working assumptions used:** none new.

## 3. User stories

| ID | As a… | I want… | so that… | Priority |
|---|---|---|---|---|
| US-INVENTORY-1 | warehouse staff | to look up a variant by SKU and see its exact stock | I know what to restock | 🔴 P0 |
| US-INVENTORY-2 | warehouse staff | to record a stock increase after a delivery arrives | inventory stays accurate | 🔴 P0 |
| US-INVENTORY-3 | warehouse staff | to correct a stock count if it's wrong | discrepancies don't compound | 🔴 P0 |
| US-INVENTORY-4 | warehouse staff | to be stopped from adjusting stock below zero | I can't accidentally corrupt the count | 🟠 P1 |

## 4. Functional requirements

| ID | Requirement | SRS ref | Priority |
|---|---|---|---|
| FR-INVENTORY-1 | `WAREHOUSE_STAFF` MUST be able to look up a variant by SKU or product name and see its exact current stock quantity (not the customer-facing In Stock/Out of Stock enum). | SRS §4.5.2 (stimulus 3) | 🔴 P0 |
| FR-INVENTORY-2 | `WAREHOUSE_STAFF` MUST be able to adjust a variant's stock quantity for restocking or correction, and the system MUST record who made the change and why. | REQ-5.2 | 🔴 P0 |
| FR-INVENTORY-3 | An adjustment that would take a variant's quantity below zero MUST be rejected with a clear message, leaving the current quantity unchanged. | REQ-5.6 | 🟠 P1 |
| FR-INVENTORY-4 | Every stock increase and decrease — manual or order-driven — MUST be logged with a timestamp, the acting user (where one exists), the reason, and the related order where one applies. | REQ-5.5 | 🔴 P0 |

## 5. NFR deltas (feature-specific only)

None beyond `global/03_NFR_BASELINE.md`.

## 6. Security deltas (feature-specific only)

| ID | Requirement | Closes |
|---|---|---|
| SEC-INVENTORY-1 | Both endpoints in this feature require permission `stock:adjust` — default-granted to `WAREHOUSE_STAFF` (RBAC-1), never hardcoded to a role name. | — |
| SEC-INVENTORY-2 | Exact stock quantity is available only behind `stock:adjust` — the same information-disclosure boundary `SEC-CATALOG-1` establishes from the other side. | T-13-adjacent (role-gated data exposure) |

## 7. Acceptance criteria (Given/When/Then)

- **AC-INVENTORY-1** — *Given* a variant with SKU `BB-1001`, *When* staff search `q=BB-1001`, *Then*
  the result shows the exact `stockQuantity` (e.g. `7`), not `IN_STOCK`.
- **AC-INVENTORY-2** — *Given* a variant at quantity 10, *When* staff record a `+20` adjustment with
  reason "restock," *Then* the quantity becomes 30 and a `stock_movement` row exists with `change_qty
  = 20`, that reason, and the acting staff member's id.
- **AC-INVENTORY-3** — *Given* a variant at quantity 3, *When* staff attempt a `-5` adjustment, *Then*
  the request is rejected, the quantity remains 3, and no `stock_movement` row is written for the
  rejected attempt.
- **AC-INVENTORY-4** — *Given* an order that decremented a variant by 2 and a later manual `+5`
  adjustment on the same variant, *When* the `stock_movement` history is queried, *Then* both rows are
  present, correctly distinguished by `related_order_id` (set vs. NULL) and `acting_user_id` (NULL vs.
  set).

## 8. Open questions / decisions needed

- [ ] None outstanding.
