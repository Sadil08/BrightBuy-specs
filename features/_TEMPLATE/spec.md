# <NN-feature-slug> — Spec (WHAT & WHY)

> Clone this folder per feature. Fill every section; delete the `> guidance` lines.
> Owner hat: **Business Analyst.** No tech choices, no code, no SQL here — that's `plan.md`.
> Examples below use a CART slice for illustration; replace with your feature.

**Feature slug:** `<FEAT>` (e.g. `CART`) · **Status:** ❌ draft · **Companion:** [plan.md](plan.md) · [tasks.md](tasks.md)
**Inherits:** `global/02_SECURITY_BASELINE.md`, `global/03_NFR_BASELINE.md`
**SRS section:** `<SRS §4.x — feature name>`

---

## 1. Summary
> 2–4 sentences: what this feature is, which actor(s) it serves, the outcome it enables.

A comprehensive cart that lets **guests and logged-in customers** collect product variants and
quantities before checkout, persisting across the session for a logged-in customer.

## 2. Actors & scope
- **In scope:** add/update/remove line items; quantity; running subtotal.
- **Out of scope (explicit):** payment & delivery charge (→ `04-checkout-orders`).
- **Working assumptions used** (cite the `OQ-*`/`TBD-*` from `global/10_BA_AUDIT_OPEN_QUESTIONS.md`
  this spec had to assume an answer for, if any): —

## 3. User stories
| ID | As a… | I want… | so that… | Priority |
|---|---|---|---|---|
| US-<FEAT>-1 | guest | to add a product variant to my cart without logging in | I can shop quickly | 🔴 P0 |
| US-<FEAT>-2 | customer | my cart to survive a page refresh | I don't lose my selection | 🟠 P1 |

## 4. Functional requirements
> Numbered, atomic, verifiable. Each becomes design in `plan.md` + a task+test in `tasks.md`, and
> **must cite the SRS `REQ-*` it implements** — that's the traceability spine (`README.md` §3).

| ID | Requirement | SRS ref | Derived from | Priority |
|---|---|---|---|---|
| FR-<FEAT>-1 | The system MUST let an unauthenticated user add a variant+quantity to a cart. | REQ-3.1 | US-<FEAT>-1 | 🔴 P0 |
| FR-<FEAT>-2 | The system MUST persist a logged-in customer's cart server-side across sessions/devices. | REQ-3.5 | US-<FEAT>-2 | 🟠 P1 |

## 5. NFR deltas (feature-specific only)
> Inherits `global/03`. List ONLY additions/overrides.

| ID | NFR | Target |
|---|---|---|
| NFR-<FEAT>-1 | Cart read/update p95 < 300ms. | snappy UX |

## 6. Security deltas (feature-specific only)
> Inherits `global/02`. List ONLY additions/overrides + register items closed.

| ID | Requirement | Closes |
|---|---|---|
| SEC-<FEAT>-1 | A customer's server-side cart is ownership-scoped (cannot read another customer's cart). | — |

## 7. Acceptance criteria (Given/When/Then — the contract)
- **AC-<FEAT>-1** — *Given* a guest, *When* they add variant V (qty 2), *Then* the cart shows that
  line with the server-computed line price.
- **AC-<FEAT>-2** — *Given* a variant with stock 3, *When* the user sets quantity 4, *Then* the system
  warns and does not add the excess (REQ-3.4 — advisory only; the authoritative check is at checkout).

## 8. Open questions / decisions needed
- [ ] Anything not already covered in `global/10_BA_AUDIT_OPEN_QUESTIONS.md`.
