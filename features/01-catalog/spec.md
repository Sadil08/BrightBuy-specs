# 01-catalog — Spec (WHAT & WHY)

**Feature slug:** `CATALOG` · **Status:** ⚠️ draft-for-review · **Companion:** [plan.md](plan.md) · [tasks.md](tasks.md)
**Inherits:** `global/02_SECURITY_BASELINE.md`, `global/03_NFR_BASELINE.md`
**SRS section:** §4.1 Product Catalog Browsing and Search (High priority)

---

## 1. Summary

Lets **guests and registered customers** browse and search BrightBuy's catalog — categories,
products, and each product's variants with live price and stock status — with no account required.
This is the entry point for everything else in the storefront (cart, checkout all start from here).

**Explicitly not this feature:** creating/editing/deactivating products, variants or categories
(that's staff-only write access → `07-admin-catalog`, REQ-10.*), and product image upload
(also `07-admin-catalog`, resolved `IMG-1`). This feature only *reads* `product_image` rows that
`07-admin-catalog` writes — it never writes them. Building `01` before `07` works because Phase 0
seeds the catalog directly (`db/seed`), so browsing has real data to read before any admin write path
exists.

## 2. Actors & scope

- **In scope:** category listing; product list with pagination, keyword search, category filter;
  product detail with variants, attributes, price, derived stock status, images.
- **Out of scope (explicit):** product/variant/category CRUD (`07-admin-catalog`); cart (`03-cart`);
  exact stock quantity to customers (staff-only, `06-inventory-stock`).
- **Working assumptions used:** none from `global/10_BA_AUDIT_OPEN_QUESTIONS.md` — this feature has no
  open dependency on TBD-1/2/3/4 or OQ-4; it only needs OQ-3's resolution (real image pipeline,
  `IMG-1`), which is closed.

## 3. User stories

| ID | As a… | I want… | so that… | Priority |
|---|---|---|---|---|
| US-CATALOG-1 | guest | to browse products by category without logging in | I can shop before committing to an account | 🔴 P0 |
| US-CATALOG-2 | guest | to search by keyword | I can find a specific product quickly | 🔴 P0 |
| US-CATALOG-3 | guest | to see a product's variants (e.g. color) with price and whether it's in stock | I can decide what to buy | 🔴 P0 |
| US-CATALOG-4 | guest | a single-configuration product to just show its price, not force me to pick a variant | the UI doesn't ask me a pointless question | 🟠 P1 |
| US-CATALOG-5 | guest | large result sets to load in pages | the page stays fast as the catalog grows | 🟠 P1 |

## 4. Functional requirements

| ID | Requirement | SRS ref | Derived from | Priority |
|---|---|---|---|---|
| FR-CATALOG-1 | The system MUST let visitors and registered customers browse and search the catalog without authentication. | REQ-1.1 | US-CATALOG-1 | 🔴 P0 |
| FR-CATALOG-2 | The system MUST display, for each product, its assigned category/categories, and for each variant its distinguishing attributes, price, and stock status (In Stock / Out of Stock only). | REQ-1.2 | US-CATALOG-3 | 🔴 P0 |
| FR-CATALOG-3 | The system MUST support keyword search across product name and description. | REQ-1.3 | US-CATALOG-2 | 🔴 P0 |
| FR-CATALOG-4 | The system MUST allow the catalog to be filtered by one or more categories. | REQ-1.4 | US-CATALOG-1 | 🔴 P0 |
| FR-CATALOG-5 | A product with a single variant MUST be presented without asking the user to choose a variant. | REQ-1.5 | US-CATALOG-4 | 🟠 P1 |
| FR-CATALOG-6 | Catalog and search results MUST be paginated, default page size 20. | REQ-1.6 | US-CATALOG-5 | 🟠 P1 |
| FR-CATALOG-7 | The exact stock quantity MUST NOT be exposed to customers — only In Stock / Out of Stock. | REQ-1.7 | global business rule | 🔴 P0 |
| FR-CATALOG-8 | A deactivated product, variant, or category MUST NOT appear in browse/search results. | REQ-10.5 (consumed here, owned by `07`) | — | 🔴 P0 |

## 5. NFR deltas (feature-specific only)

None — inherits `global/03_NFR_BASELINE.md` §1 verbatim (catalog browse/search < 2s at the API,
indexes + pagination as the mechanism). Catalog reads additionally carry HTTP cache headers per
`global/04_SCALABILITY.md` §2.

## 6. Security deltas (feature-specific only)

| ID | Requirement | Closes |
|---|---|---|
| SEC-CATALOG-1 | Exact `stock_quantity` is never serialized to a customer-facing response — the API returns only the derived `IN_STOCK`/`OUT_OF_STOCK` enum (`fn_is_variant_in_stock`, `07_SQL_DATABASE_STANDARDS.md` §5). This is an information-disclosure control (REQ-1.7), not just a UX choice: exact stock reveals sales velocity to competitors. | — |

## 7. Acceptance criteria (Given/When/Then)

- **AC-CATALOG-1** — *Given* a guest with no session, *When* they call `GET /api/v1/products`, *Then*
  the response is `200` with a paginated list — no `401` for lacking a session.
- **AC-CATALOG-2** — *Given* the catalog has a product named "Bluetooth Speaker", *When* a user
  searches `q=bluetooth`, *Then* that product appears in the results regardless of case.
- **AC-CATALOG-3** — *Given* a product with variants Black ($49.99, 3 in stock) and White ($49.99, 0
  in stock), *When* its detail is fetched, *Then* the response shows both variants with
  `stockStatus: IN_STOCK` / `OUT_OF_STOCK` respectively, and no numeric quantity anywhere in the payload.
- **AC-CATALOG-4** — *Given* a product with exactly one variant, *When* its detail is fetched, *Then*
  the frontend renders price/stock directly without a variant selector control (FE-only behavior,
  driven by `variants.length === 1`; no special API field needed for this).
- **AC-CATALOG-5** — *Given* 45 products in a category, *When* a client requests page 1 with the
  default size, *Then* it receives 20 items and a `page.total` of 45.
- **AC-CATALOG-6** — *Given* a product has been deactivated by staff, *When* the catalog is browsed or
  searched, *Then* that product does not appear, and its direct detail URL returns `404`.

## 8. Open questions / decisions needed

- [ ] None outstanding for this feature.
