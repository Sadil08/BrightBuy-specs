# 01-catalog — Tasks (DO IT)

**Feature slug:** `CATALOG` · **Status:** 🟢 usable end-to-end, frontend test coverage still missing · **Companion:** [spec.md](spec.md) · [plan.md](plan.md)

---
## 1. Build order

### Database
- [x] **T1 `[DB]`** Migration `0001_catalog_schema`: `category`, `product` (+ `FULLTEXT` index),
      `product_category`, `product_variant`, `attribute_name`, `attribute_value`, `variant_attribute`
      — full DDL in `plan.md` §2. — *Test:* applies clean on a fresh `testcontainers` MySQL 8; `down`
      then `up` again is idempotent. — `FR-CATALOG-1..7` — verified manually against a real
      Dockerized MySQL 8 (`migrate up` → `down -all` → `up` again, table set identical each time); a
      `testcontainers-go` version of this same check still belongs in the automated suite (T5/CI).
- [x] **T2 `[DB]`** `fn_is_variant_in_stock` function (`07_SQL_DATABASE_STANDARDS.md` §5). — *Test:*
      direct SQL call at `stock_quantity` = 0 and > 0. — `FR-CATALOG-2`, `SEC-CATALOG-1` — verified:
      returns `1` at quantity 5, `0` at quantity 0, and `0` (not `NULL`) for a nonexistent
      `variant_id`. Fixed a real bug found while verifying: the migration wrapped the function body in
      `DELIMITER $$` / `DELIMITER ;`, which is a `mysql`-CLI-only convention, not SQL — golang-migrate
      sends the file straight to the server, which rejected the literal word `DELIMITER`. Removed; the
      server parses the semicolons inside `CREATE FUNCTION ... BEGIN ... END` correctly without it.
- [x] **T3 `[DB]`** `db/seed`: deterministic seed script, ≥40 products across ≥10 categories, a mix of
      single-variant and multi-variant products, at least one product with an out-of-stock variant and
      one fully out-of-stock product, at least one deactivated product (`is_active = 0`) to exercise
      AC-CATALOG-6. — *Test:* running the script twice against a fresh DB produces identical row counts
      (deterministic, not random). — SRS §6.1 — verified: two runs both produced 10 categories / 40
      products / 40 variants / 1 inactive product / 6 out-of-stock variants, identically.
      **Updated in a later pass**, closing the "thin multi-variant coverage" note this section used to
      have: now 50 variants across 40 products, with one multi-variant product per category (10 total,
      up from 1) — `attribute_name`/`attribute_value`/`variant_attribute` all populated to match (was
      entirely empty before, see T11's note). **A real, previously-undetected bug found and fixed
      along the way**: product 10 (`BrightPhone Secure`) had **zero** variants — a direct violation of
      `plan.md`'s own "a product with zero variants should never occur" (REQ-10.3), sitting undetected
      in the original seed data this whole time. Re-verified idempotency after the change: two
      consecutive runs produce identical counts (10 categories / 40 products / 50 variants / 8
      attribute values / 20 variant-attribute links).

### Backend
- [x] **T4 `[BE]`** `internal/catalog/domain`: `Product`, `Variant`, `Category` entities;
      `StockStatus` derivation type. — *Test:* unit, no DB. — done: entities are plain structs with
      no behavior yet, so the meaningful unit tests land with T6/T7; `internal/shared/money` (the
      `Money` value type referenced by `Variant.Price`) was built alongside this and has its own
      table-driven tests.
- [x] **T5 `[BE]`** `internal/catalog/mysql`: `ProductRepository.List` (search+filter+paginate, `plan.md`
      §5.1), `ProductRepository.GetByID`, `CategoryRepository.List` — all parameterized. — *Test:*
      integration against real MySQL 8 (`testcontainers-go`) per `plan.md` §8 — 7 tests, all green,
      against a disposable container with the real `db/migrations/*.up.sql` applied via the
      `golang-migrate` library (not re-typed SQL). `List` batches all of a page's variants in one
      extra query (keyed by `product_id IN (...)`) instead of one query per product, to avoid an N+1
      pattern on the paginated endpoint. Two real bugs found and fixed while building this:
      (1) the container needed `--log-bin-trust-function-creators=1`, same as `docker-compose.yml`
      already sets, or `CREATE FUNCTION` is rejected outright even though it's `DETERMINISTIC`;
      (2) an earlier attempt fed the migration files through testcontainers' `WithScripts` (which
      runs them via the `mysql` CLI, docker's `docker-entrypoint-initdb.d` convention) and hit the
      *same* `DELIMITER`-shaped failure as T2's original bug, for the mirror-image reason: the `mysql`
      CLI DOES split on every `;` (that's what `DELIMITER` exists to work around), while
      `golang-migrate`'s direct `database/sql` connection does not — so the fix there was routing the
      test through `golang-migrate` itself, the same engine dev/prod actually use, not an alternate
      runner with different semicolon rules.
- [x] **T6 `[BE]`** `internal/catalog/app.CatalogService`: `ListProducts`, `GetProduct`,
      `ListCategories`. — *Test:* unit with a fake repository — 4 tests (pagination defaulting/
      clamping, pass-through, not-found propagation), no DB, all green.
- [x] **T7 `[BE]`** `internal/catalog/httpapi`: `CatalogHandler` + DTOs (never serialize
      `stock_quantity` — `plan.md` §5.2) + route registration (`GET /categories`, `GET /products`,
      `GET /products/{productId}`) + cache headers. — *Test:* `httptest` — status codes, pagination
      response shape, DTO field-name reflection check (plan.md §8). — `FR-CATALOG-1,2,3,4,5,6,7` —
      done. Verified live (see below) AND now backed by 9 `httptest`-based tests in
      `internal/catalog/httpapi/handler_test.go`: category/product list/detail happy paths, 404/400
      error paths, `Cache-Control` header presence, pagination-envelope defaulting, and the plan.md §8
      reflection test (`TestDTOsNeverDeclareAStockQuantityField` — walks every DTO's actual `json`
      struct tags via `reflect`, so a future field rename reintroducing a quantity-shaped field fails
      immediately, not just "doesn't show up in today's grep"). Also found and fixed a real
      architecture gap while wiring this: `mysql.ErrNotFound` couldn't be checked from `httpapi`
      without violating "httpapi never imports mysql" — moved the sentinel to `app.ErrNotFound` (the
      port layer), since "not found" is repository-agnostic, not MySQL-specific.
      Live verification (real server, real restarted dev MySQL, real `curl`): `GET /categories` (10
      active, descriptions included), `GET /products?size=2` (`total: 39` — the one deactivated seed
      product correctly excluded), `GET /products?q=BRIGHTBOOK` (case-insensitive match,
      AC-CATALOG-2), `GET /products/1` (full detail correct), a real deactivated product's detail URL
      → `404`, `GET /products/999999` → `404`, `GET /products/notanumber` → `400`. Grepped full JSON
      responses for `quantity` — absent (SEC-CATALOG-1).
      `gofmt`/`go vet`/`staticcheck` all clean.
      **Two follow-ups fixed in this same pass, found by the live check above:**
      (1) the `attributes:null` vs `attributes:[]` bug — `mapper.go`'s `toVariantAttributeDTO` now
      initializes `:= []VariantAttributeDTO{}` instead of `var ... []VariantAttributeDTO`, so a
      variant with zero attributes serializes as `[]` (a non-nullable array per `openapi.yaml`), not
      `null`; covered by `TestGetProduct_Returns200WithFullDetail`'s raw-body check.
      (2) a deactivated **variant** (as opposed to product/category, both already tested) was
      completely untested — every `insertVariant` call in the integration suite passed `isActive:
      true`. Added `TestProductRepository_GetByID_ExcludesDeactivatedVariant` and
      `TestProductRepository_List_ExcludesDeactivatedVariantFromBatch` (the two separate SQL query
      paths that each independently filter `is_active = TRUE`), closing FR-CATALOG-8's variant case.
      **The deferred `ETag` was built in a follow-up pass** — `Cache-Control: public, max-age=60` on
      all three success paths, plus a weak `ETag` on `GetProduct` specifically, derived from
      `domain.Product.UpdatedAt`. `httpx.CheckETag(w, r, etag)` (new shared helper, `httpx.go`) sets
      the header and checks the request's `If-None-Match`, writing a real `304 Not Modified` (empty
      body) on a match — verified live against the real server: a second request replaying the first
      response's own `ETag` gets `304`, a stale/different one gets a normal `200`.
      `domain.Product` gained `UpdatedAt time.Time`, populated by `ProductRepository.GetByID` as
      `MAX(product.updated_at, MAX(variant.updated_at) across its variants)` — not just the product
      row — because a stock/price change only touches `product_variant`, never `product` itself; an
      ETag keyed on the product row alone would miss exactly the kind of change this feature cares
      most about (stock status). Proven by a dedicated integration test
      (`TestProductRepository_GetByID_UpdatedAtReflectsMostRecentVariantChange`) and a handler-level
      one (`TestGetProduct_SetsETagAndReturns304WhenIfNoneMatchMatches`). List/categories still have no
      ETag — no single natural "last modified" row for a collection, and `category` has no
      `updated_at` column at all; not something this pass invented a workaround for.
      **This required a new migration** (`0003_product_variant_updated_at`, since `product_variant`
      had no `updated_at` at all) — **which surfaced a second, separate, pre-existing bug**: the
      original `0001` migration's `product.updated_at DEFAULT UTC_TIMESTAMP() ON UPDATE
      UTC_TIMESTAMP()` doesn't actually do what its name claims. MySQL's automatic default/on-update
      column behavior only recognizes `CURRENT_TIMESTAMP` (and direct synonyms) — `UTC_TIMESTAMP()` is
      rejected outright by `ALTER TABLE` (confirmed directly), and `CREATE TABLE`'s more lenient parser
      silently REWRITES it to `CURRENT_TIMESTAMP` instead of erroring (`SHOW CREATE TABLE product`
      proves this) — which runs in the server's own time zone, not UTC. It only ever "worked" because
      this dev container's system time zone happened to already be UTC. Fixed properly:
      `docker-compose.yml`'s `mysql` service now also passes `--default-time-zone=+00:00`, so
      `CURRENT_TIMESTAMP` actually means UTC by configuration, not by accident of the base image.
      `product`'s own column wasn't rewritten (out of scope for an ETag task to go editing an
      unrelated, already-shipped migration) — logged here so it isn't mistaken for still being UTC by
      name alone.

### Frontend
- [x] **T8 `[FE]`** Typed API client methods for the three endpoints (generated or hand-written against
      `openapi.yaml`). — *Test:* Vitest, mocked responses. — `src/lib/api-client/catalog.ts`:
      `getCategories`/`getProducts`/`getProduct`, types hand-written 1:1 against the OpenAPI schemas
      (same field names/casing, so a mismatch is an easy diff to spot). No `images` field on
      `Product` — matches the backend's own deliberate, now-logged omission (T12).
      **Test gap, not silently skipped**: no test framework (Vitest, RTL, Playwright) is installed in
      this project AT ALL yet — `package.json` has none of the three. Verified correctness instead by
      running the real Next.js dev server against the real backend/MySQL and hitting it with `curl`
      (see T10/T11's notes) plus a full `next build` + `tsc --noEmit` + ESLint pass, all clean. Setting
      up a test framework from scratch is its own real chunk of work (config, jsdom, CI wiring) —
      worth doing as its own next step, not bundled silently into this one.
- [x] **T9 `[FE]`** Category nav / filter component. — *Test:* RTL. —
      `src/app/(storefront)/_components/CategoryNav.tsx`. Deliberately a plain Server Component, no
      `"use client"`: switching categories navigates to `/products?categoryId=N` (a `<Link>`), which
      Next.js re-renders server-side — no client state needed, works with JS disabled. Preserves an
      active search query when switching categories (`searchQuery` prop). Verified live: filtering by
      "Laptops" returns exactly the 5 real laptop products. RTL gap: same as T8, no test framework
      installed.
- [x] **T10 `[FE]`** Product list page: search box, pagination controls, stock-status badge. — *Test:*
      RTL + Playwright happy path. — `src/app/(storefront)/products/page.tsx`. Search is a plain
      `<form method="GET">` (no JS required); pagination is `<Link>`s preserving the active
      query/category. Verified against the real backend: default page shows exactly 20 of 39 active
      products (`page.total`); `?q=BrightPhone` returns exactly the 5 matching products; `?categoryId=`
      filtering verified too. Empty results render a plain message, not an error (mirrors
      AC-CATALOG-1's "200 with empty list" on the frontend). Test gap: same as T8/T9.
- [x] **T11 `[FE]`** Product detail page: variant selector that **auto-selects and hides the chooser
      UI when `variants.length === 1`** (FR-CATALOG-5, no backend field needed — `plan.md` §7). —
      *Test:* RTL for both the single- and multi-variant render paths (AC-CATALOG-4). —
      `src/app/(storefront)/products/[productId]/page.tsx` (Server Component, fetches + maps a 404
      `ApiError` to Next's `notFound()`) + `VariantSelector.tsx` (the ONE Client Component this whole
      feature needed — picking a variant updates displayed price/stock in-browser). Verified live: a
      single-variant product (`BrightBook Air 13`) renders zero picker buttons; a 2-variant product
      (`BrightPhone X1`) renders exactly 2, one pre-selected; `/products/999999` and `/products/banana`
      both correctly 404. **A real seed-data bug found and fixed along the way**: `BrightPhone X1`'s
      variant buttons initially showed raw SKUs (`PHONE-X1-BLK`) instead of "Black"/"White" — not a
      frontend bug, `attribute_name`/`attribute_value`/`variant_attribute` were ALL completely empty
      (0 rows) in `db/seed/catalog.sql`, for every product, not just this one. Added the missing
      `Color: Black/White` rows for this product's two variants; re-ran the (already-idempotent) seed
      script and confirmed live. Broadening *which other* products get a second variant is still the
      separate, already-logged T3 follow-up. Test gap: same as T8–T10.

### Cross-cutting
- [ ] **T12 `[INFRA]`** Verify `specs/openapi/openapi.yaml`'s `catalog` paths match the shipped
      handler exactly (response shape, status codes) — no drift between spec and implementation. —
      partially checked, not closed out: confirmed the `servers: /api/v1` base path, all three route
      paths (`/categories`, `/products`, `/products/{productId}`), and the `Error` schema
      (`code`/`message`/`fields`) all match exactly. **One confirmed, deliberate drift, not yet
      logged anywhere formal**: `Product`'s schema includes an `images` array; `ProductDTO` has none,
      because `product_image` is a table `07-admin-catalog` owns and no migration has created it yet
      — there's nothing to fetch. This should get a line in `09_ISSUES_REGISTER.md` (or similar)
      rather than just living in conversation history, so it's not mistaken for an oversight later.

## 2. Acceptance-criteria coverage map

| AC | Covered by |
|---|---|
| AC-CATALOG-1 | T7 |
| AC-CATALOG-2 | T5, T7 |
| AC-CATALOG-3 | T5, T6, T7 |
| AC-CATALOG-4 | T11 |
| AC-CATALOG-5 | T5, T7 |
| AC-CATALOG-6 | T5, T7 |

## 3. Definition of Done

- [x] Every `FR-CATALOG-*` has a passing unit and integration test. — including FR-CATALOG-8's
      variant case, closed in this pass (see T7).
- [x] Every `AC-CATALOG-*` above is green. — all six, including AC-CATALOG-4 now: verified live that
      a single-variant product renders no picker and a 2-variant product renders exactly 2 options.
- [x] No response payload anywhere contains `stock_quantity`/a numeric stock field (SEC-CATALOG-1) —
      verified by test, not just code review. — `TestDTOsNeverDeclareAStockQuantityField`
      (`handler_test.go`), plus the earlier live `grep` check.
- [x] `gofmt`/`go vet`/ESLint clean; `openapi.yaml`'s `catalog` section matches the shipped API. —
      `gofmt`/`go vet`/`staticcheck` all clean (backend); `tsc --noEmit`, `next build`, and `eslint`
      all clean (frontend). OpenAPI match: see T12 — matches, with one deliberate, documented gap
      (`images`).
- [x] Seed script (`db/seed`) runs clean and produces the dataset T3 describes — now also seeds
      `BrightPhone X1`'s Color attribute data (was entirely missing, all three EAV tables empty; see
      T11's note).

**Bottom line: the feature is now usable end-to-end** — a real visitor can load `/`, click through to
`/products`, search, filter by category, paginate, open a product, and (for any of the now-10
multi-variant products, one per category) pick a variant and see its price/stock update — all verified
against the real backend and real MySQL, not mocked. The `ETag`/`Cache-Control` pair and the
multi-variant seed breadth, previously tracked here as deferred, are both done (see T7/T3's notes) —
closing two real gaps (a product with zero variants, and `product.updated_at` never actually being UTC
despite its name) along the way, neither hypothetical.
**What's left, honestly**: no frontend test framework (Vitest/RTL/Playwright) is installed at all —
every frontend check in T8–T11 was done live with `curl`/build tooling, not as automated, repeatable
tests. That's the one remaining gap, and it's real setup work (config, `jsdom`, CI wiring), not a
quick addition.
