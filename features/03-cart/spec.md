# 03-cart — Spec (WHAT & WHY)

**Feature slug:** `CART` · **Status:** ⚠️ draft-for-review · **Companion:** [plan.md](plan.md) · [tasks.md](tasks.md)
**Inherits:** `global/02_SECURITY_BASELINE.md`, `global/03_NFR_BASELINE.md`
**SRS section:** §4.3 Shopping Cart Management (High priority)

---

## 1. Summary

Lets a guest or a registered customer assemble variants and quantities before checkout, with a live
subtotal. A guest's cart lives entirely in the browser; a customer's cart is server-side and survives
across devices/sessions, merging on login. Depends on `01-catalog` (variant lookup/pricing) and
`02-auth` (customer identity for the server-side cart).

## 2. Actors & scope

- **In scope:** add/update/remove lines; running subtotal; advisory stock warning; guest→customer
  cart merge on login.
- **Out of scope (explicit):** checkout itself, payment, delivery charge (`04-checkout-orders`); the
  cart never reserves stock — the authoritative check happens at checkout (REQ-4.1).
- **Working assumptions used:** none new. Note: like `02-auth`'s tables, `cart`/`cart_item` are not in
  the SRS's ER diagram (Figure B.1 covers only 8 "main entities," `DB-ERD-1`) — designed from the
  REQ-3.* text, following the same precedent.

## 3. User stories

| ID | As a… | I want… | so that… | Priority |
|---|---|---|---|---|
| US-CART-1 | guest | to add a variant to my cart without logging in | I can shop before committing to an account | 🔴 P0 |
| US-CART-2 | guest | to change a quantity or remove a line | I can fix a mistake before checkout | 🔴 P0 |
| US-CART-3 | registered customer | my cart to be there when I log in on another device | I don't lose my selection | 🟠 P1 |
| US-CART-4 | guest who registers/logs in mid-shop | my cart items to carry over | I don't have to re-add everything | 🟠 P1 |

## 4. Functional requirements

| ID | Requirement | SRS ref | Derived from | Priority |
|---|---|---|---|---|
| FR-CART-1 | The system MUST allow a specific variant and quantity to be added to the cart. | REQ-3.1 | US-CART-1 | 🔴 P0 |
| FR-CART-2 | The system MUST allow a cart line's quantity to be updated and allow the line to be removed. | REQ-3.2 | US-CART-2 | 🔴 P0 |
| FR-CART-3 | The system MUST display a running subtotal reflecting current variant prices and quantities. | REQ-3.3 | — | 🔴 P0 |
| FR-CART-4 | The system MUST warn when a cart quantity exceeds the variant's last known stock and MUST NOT allow adding that quantity; this check is advisory — the authoritative check is at checkout confirmation. | REQ-3.4 | — | 🔴 P0 |
| FR-CART-5 | A guest cart MUST be held in browser storage for that session only; a logged-in customer's cart MUST be stored server-side and persist across sessions/devices; on login, a guest cart MUST merge into the account cart, keeping the higher quantity where the same variant appears in both. | REQ-3.5 | US-CART-3, US-CART-4 | 🟠 P1 |
| FR-CART-6 | The cart MUST show prices from the current variant record at the time it is viewed; a price change between add and checkout MUST be reflected before confirmation, not after. | REQ-3.6 | — | 🔴 P0 |

## 5. NFR deltas (feature-specific only)

| ID | NFR | Target |
|---|---|---|
| NFR-CART-1 | Cart read/update p95 < 300ms. | Snappy UX — this is the highest-frequency interaction in the storefront. |

## 6. Security deltas (feature-specific only)

| ID | Requirement | Closes |
|---|---|---|
| SEC-CART-1 | A customer's server-side cart is ownership-scoped — `cart_item` operations filter by the caller's `customer_id` in the query itself, never a post-fetch check (`02_SECURITY_BASELINE.md` §6.2); a request for another customer's cart line returns `404`. | T-2 |
| SEC-CART-2 | No price is ever accepted from the client — every subtotal is computed server-side from `product_variant.price` at read time (FR-CART-6). | T-8 |

## 7. Acceptance criteria (Given/When/Then)

- **AC-CART-1** — *Given* a guest, *When* they add variant V (qty 2), *Then* the cart (client-held)
  shows that line with the server-validated price from `POST /cart/validate`-equivalent behavior
  (this feature's `GET/POST` responses always return server-computed prices, never echo a client price).
- **AC-CART-2** — *Given* a variant with 3 in stock, *When* the user sets quantity 4, *Then* the system
  rejects it and surfaces the available count — advisory only, not a hard block on viewing the cart.
- **AC-CART-3** — *Given* a logged-in customer with a saved cart, *When* they log in on a second
  device, *Then* the same cart is returned.
- **AC-CART-4** — *Given* a guest with 2 of variant V in a local cart who then logs into an account
  that already has 1 of variant V server-side, *When* the merge runs, *Then* the resulting line has
  quantity 2 (the higher of the two, not 3 — FR-CART-5).
- **AC-CART-5** — *Given* a variant's price changes after it was added to a cart, *When* the cart is
  next viewed, *Then* the new price is shown, not the price at add-time.

## 8. Open questions / decisions needed

- [ ] None outstanding.
