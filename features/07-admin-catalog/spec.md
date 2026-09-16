# 07-admin-catalog — Spec (WHAT & WHY)

**Feature slug:** `ADMINCATALOG` · **Status:** ⚠️ draft-for-review · **Companion:** [plan.md](plan.md) · [tasks.md](tasks.md)
**Inherits:** `global/02_SECURITY_BASELINE.md`, `global/03_NFR_BASELINE.md`
**SRS section:** §4.10 Category and Product Administration (Medium priority)

---

## 1. Summary

Lets `WAREHOUSE_STAFF` create and maintain the catalog `01-catalog` reads: products, variants,
categories, and product images. This is the write side of a feature already split from its read side
back in Phase 1 — `01-catalog`'s tables are defined there, this feature only adds the mutating
endpoints against them. Also where the real S3/MinIO image upload pipeline (resolved `OQ-3`/`IMG-1`)
gets its handlers.

## 2. Actors & scope

- **In scope:** product/variant/category create, update, deactivate; product image
  upload/confirm/delete.
- **Out of scope (explicit):** stock quantity adjustment (`06-inventory-stock` — a variant's *price*
  and *attributes* are this feature's concern, its *stock count* is inventory's); browsing/search
  (`01-catalog`, read-only consumer of what this feature writes).
- **Working assumptions used:** `IMG-1` (`global/09_ISSUES_REGISTER.md`) — this is where it's actually
  implemented.

## 3. User stories

| ID | As a… | I want… | so that… | Priority |
|---|---|---|---|---|
| US-ADMINCATALOG-1 | warehouse staff | to create a new product with its variants and categories | it can be sold | 🟡 P2 |
| US-ADMINCATALOG-2 | warehouse staff | to update a product's price or details | the catalog stays accurate | 🟡 P2 |
| US-ADMINCATALOG-3 | warehouse staff | to deactivate a discontinued product without breaking past orders that reference it | order history stays intact | 🟡 P2 |
| US-ADMINCATALOG-4 | warehouse staff | to upload photos for a product | customers can see what they're buying | 🟡 P2 |

## 4. Functional requirements

| ID | Requirement | SRS ref | Priority |
|---|---|---|---|
| FR-ADMINCATALOG-1 | `WAREHOUSE_STAFF` MUST be able to create, update, and deactivate products, variants, and categories. | REQ-10.1 | 🟡 P2 |
| FR-ADMINCATALOG-2 | A product MUST be assignable to one or more categories. | REQ-10.2 | 🟡 P2 |
| FR-ADMINCATALOG-3 | Every variant MUST require a price and an opening stock quantity at creation. | REQ-10.3 | 🟡 P2 |
| FR-ADMINCATALOG-4 | SKU uniqueness MUST be enforced across all variants, at creation and at update. | REQ-10.4 | 🟡 P2 |
| FR-ADMINCATALOG-5 | A product, variant, or category referenced by an existing order MUST be deactivated rather than deleted; a deactivated item MUST NOT appear in the storefront. | REQ-10.5 | 🟡 P2 |
| FR-ADMINCATALOG-6 | A price change MUST apply to future orders only and MUST NOT alter the recorded value of orders already placed. | REQ-10.6 | 🟡 P2 |
| FR-ADMINCATALOG-7 | `WAREHOUSE_STAFF` MUST be able to upload, replace, and delete a product's images through a pre-signed direct-to-storage flow. | resolved `OQ-3`/`IMG-1` | 🟡 P2 |

## 5. NFR deltas (feature-specific only)

None beyond `global/03_NFR_BASELINE.md` — this is a low-frequency, staff-only write path.

## 6. Security deltas (feature-specific only)

| ID | Requirement | Closes |
|---|---|---|
| SEC-ADMINCATALOG-1 | All product/variant/category write endpoints require permission `catalog:write`; image endpoints require `catalog:image:write` — separate permissions, so a role can be given catalog editing without image upload or vice versa (`06_ENGINEERING_STANDARDS.md` §2 ISP). | — |
| SEC-ADMINCATALOG-2 | Uploaded images are verified server-side by magic bytes before being linked to a product; object keys are server-generated, never from client input. | T-16 |

## 7. Acceptance criteria (Given/When/Then)

- **AC-ADMINCATALOG-1** — *Given* valid input, *When* staff create a product with two variants and
  two category ids, *Then* the product appears in `01-catalog`'s browse results with both categories
  and both variants.
- **AC-ADMINCATALOG-2** — *Given* a SKU already in use, *When* staff create or update a variant with
  that SKU, *Then* the request is rejected with `409`.
- **AC-ADMINCATALOG-3** — *Given* a product referenced by an existing order, *When* staff deactivate
  it, *Then* it disappears from `01-catalog`'s browse/search but the existing order's detail view
  still shows the product's name and its `unit_price_at_order` unchanged.
- **AC-ADMINCATALOG-4** — *Given* a variant's price is changed, *When* an order placed before the
  change is viewed, *Then* its recorded price is unchanged (already guaranteed by
  `04-checkout-orders`'s `unit_price_at_order` snapshot — this AC verifies it from the admin-write
  side too).
- **AC-ADMINCATALOG-5** — *Given* a staff member with `catalog:write` but not `catalog:image:write`,
  *When* they attempt to upload an image, *Then* the request is rejected with `403`, while product
  edits still succeed.
- **AC-ADMINCATALOG-6** — *Given* a file renamed to look like a `.jpg` but containing executable
  bytes, *When* it's uploaded and confirmed, *Then* the confirm step rejects it with `400` and no
  `product_image` row is created.

## 8. Open questions / decisions needed

- [ ] None outstanding.
