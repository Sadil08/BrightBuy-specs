# 01-catalog — Tasks (DO IT)

**Feature slug:** `CATALOG` · **Status:** ❌ not started · **Companion:** [spec.md](spec.md) · [plan.md](plan.md)

---
## 1. Build order

### Database
- [ ] **T1 `[DB]`** Migration `0001_catalog_schema`: `category`, `product` (+ `FULLTEXT` index),
      `product_category`, `product_variant`, `attribute_name`, `attribute_value`, `variant_attribute`
      — full DDL in `plan.md` §2. — *Test:* applies clean on a fresh `testcontainers` MySQL 8; `down`
      then `up` again is idempotent. — `FR-CATALOG-1..7`
- [ ] **T2 `[DB]`** `fn_is_variant_in_stock` function (`07_SQL_DATABASE_STANDARDS.md` §5). — *Test:*
      direct SQL call at `stock_quantity` = 0 and > 0. — `FR-CATALOG-2`, `SEC-CATALOG-1`
- [ ] **T3 `[DB]`** `db/seed`: deterministic seed script, ≥40 products across ≥10 categories, a mix of
      single-variant and multi-variant products, at least one product with an out-of-stock variant and
      one fully out-of-stock product, at least one deactivated product (`is_active = 0`) to exercise
      AC-CATALOG-6. — *Test:* running the script twice against a fresh DB produces identical row counts
      (deterministic, not random). — SRS §6.1

### Backend
- [ ] **T4 `[BE]`** `internal/catalog/domain`: `Product`, `Variant`, `Category` entities;
      `StockStatus` derivation type. — *Test:* unit, no DB.
- [ ] **T5 `[BE]`** `internal/catalog/mysql`: `ProductRepository.List` (search+filter+paginate, `plan.md`
      §5.1), `ProductRepository.GetByID`, `CategoryRepository.List` — all parameterized. — *Test:*
      integration against real MySQL 8 (`testcontainers-go`) per `plan.md` §8.
- [ ] **T6 `[BE]`** `internal/catalog/app.CatalogService`: `ListProducts`, `GetProduct`,
      `ListCategories`. — *Test:* unit with a fake repository.
- [ ] **T7 `[BE]`** `internal/catalog/httpapi`: `CatalogHandler` + DTOs (never serialize
      `stock_quantity` — `plan.md` §5.2) + route registration (`GET /categories`, `GET /products`,
      `GET /products/{productId}`) + cache headers. — *Test:* `httptest` — status codes, pagination
      response shape, DTO field-name reflection check (plan.md §8). — `FR-CATALOG-1,2,3,4,5,6,7`

### Frontend
- [ ] **T8 `[FE]`** Typed API client methods for the three endpoints (generated or hand-written against
      `openapi.yaml`). — *Test:* Vitest, mocked responses.
- [ ] **T9 `[FE]`** Category nav / filter component. — *Test:* RTL.
- [ ] **T10 `[FE]`** Product list page: search box, pagination controls, stock-status badge. — *Test:*
      RTL + Playwright happy path.
- [ ] **T11 `[FE]`** Product detail page: variant selector that **auto-selects and hides the chooser
      UI when `variants.length === 1`** (FR-CATALOG-5, no backend field needed — `plan.md` §7). —
      *Test:* RTL for both the single- and multi-variant render paths (AC-CATALOG-4).

### Cross-cutting
- [ ] **T12 `[INFRA]`** Verify `specs/openapi/openapi.yaml`'s `catalog` paths match the shipped
      handler exactly (response shape, status codes) — no drift between spec and implementation.

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

- [ ] Every `FR-CATALOG-*` has a passing unit and integration test.
- [ ] Every `AC-CATALOG-*` above is green.
- [ ] No response payload anywhere contains `stock_quantity`/a numeric stock field (SEC-CATALOG-1) —
      verified by test, not just code review.
- [ ] `gofmt`/`go vet`/ESLint clean; `openapi.yaml`'s `catalog` section matches the shipped API.
- [ ] Seed script (`db/seed`) runs clean and produces the dataset T3 describes.
