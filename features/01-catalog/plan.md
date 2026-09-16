# 01-catalog — Plan (HOW)

**Feature slug:** `CATALOG` · **Status:** ⚠️ draft-for-review · **Companion:** [spec.md](spec.md) · [tasks.md](tasks.md)
**Applies:** `global/01_TECH_STACK.md`, `global/06_ENGINEERING_STANDARDS.md`, `global/07_SQL_DATABASE_STANDARDS.md`

---

## 1. Approach & architecture

`catalog` is a pure-read module for this feature (writes come later in `07-admin-catalog`). Handlers
call one service method each; the service composes repository calls; nothing below `httpapi` knows
about HTTP, and nothing above `mysql` knows about SQL.

```
[FE product list/detail pages] --GET--> [httpapi.CatalogHandler] --> [app.CatalogService] --> [mysql.ProductRepository / CategoryRepository] --> MySQL
                                                                                                   (fn_is_variant_in_stock, §4 below)
```

- `CatalogService` has no port dependencies beyond its own repositories in this feature (no payment,
  no stock-write, no image-write) — the smallest possible surface, per `06_ENGINEERING_STANDARDS.md`
  §2's ISP rule.
- Read endpoints are the ones `04_SCALABILITY.md` §2 says should carry cache headers — `httpapi`
  sets `Cache-Control: public, max-age=60` (products/categories change on staff action, not per
  request) plus a weak `ETag` derived from the row's `updated_at`.

## 2. Data model / migrations

This is the first feature to touch these tables, so this is their authoritative first-cut DDL —
later features (`03-cart`, `04-checkout-orders`, `07-admin-catalog`) reference them without
redefining them. Ships as `db/migrations/0001_catalog_schema.up.sql` (part of the Phase 0 baseline,
`11_BUILD_PLAN.md` Phase 0).

```sql
CREATE TABLE category (
    category_id INT AUTO_INCREMENT PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    description VARCHAR(500) NULL,
    is_active   BOOLEAN NOT NULL DEFAULT TRUE     -- REQ-10.5: deactivate, never hard-delete (07-admin-catalog)
);

CREATE TABLE product (
    product_id  INT AUTO_INCREMENT PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    description TEXT NULL,
    is_active   BOOLEAN NOT NULL DEFAULT TRUE,     -- REQ-10.5: deactivate, never hard-delete
    created_at  DATETIME NOT NULL DEFAULT UTC_TIMESTAMP(),
    updated_at  DATETIME NOT NULL DEFAULT UTC_TIMESTAMP() ON UPDATE UTC_TIMESTAMP()
);
FULLTEXT INDEX ft_product_name_description ON product(name, description);  -- REQ-1.3 keyword search

CREATE TABLE product_category (                    -- many-to-many, 00_OVERVIEW.md §4
    product_id  INT NOT NULL,
    category_id INT NOT NULL,
    PRIMARY KEY (product_id, category_id),
    FOREIGN KEY (product_id)  REFERENCES product(product_id)   ON DELETE CASCADE,
    FOREIGN KEY (category_id) REFERENCES category(category_id) ON DELETE CASCADE
);
CREATE INDEX ix_product_category_category ON product_category(category_id);  -- reverse-direction lookup

CREATE TABLE product_variant (
    variant_id      INT AUTO_INCREMENT PRIMARY KEY,
    product_id      INT NOT NULL,
    sku             VARCHAR(64) NOT NULL,
    price           DECIMAL(10,2) NOT NULL,
    stock_quantity  INT NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    FOREIGN KEY (product_id) REFERENCES product(product_id) ON DELETE CASCADE,
    UNIQUE KEY uq_product_variant_sku (sku)          -- REQ-5.1, REQ-10.4
);

CREATE TABLE attribute_name (                        -- EAV side table, 00_OVERVIEW.md §4
    attribute_name_id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE                  -- e.g. 'Color', 'Storage Size'
);

CREATE TABLE attribute_value (
    attribute_value_id INT AUTO_INCREMENT PRIMARY KEY,
    attribute_name_id   INT NOT NULL,
    value VARCHAR(100) NOT NULL,                      -- e.g. 'Black'
    FOREIGN KEY (attribute_name_id) REFERENCES attribute_name(attribute_name_id) ON DELETE CASCADE,
    UNIQUE KEY uq_attribute_name_value (attribute_name_id, value)
);

CREATE TABLE variant_attribute (
    variant_id          INT NOT NULL,
    attribute_value_id  INT NOT NULL,
    PRIMARY KEY (variant_id, attribute_value_id),
    FOREIGN KEY (variant_id)         REFERENCES product_variant(variant_id)     ON DELETE CASCADE,
    FOREIGN KEY (attribute_value_id) REFERENCES attribute_value(attribute_value_id) ON DELETE CASCADE
);

-- product_image: written by 07-admin-catalog, only READ here. Full DDL owned by that feature's
-- plan.md (07_SQL_DATABASE_STANDARDS.md §1a) — restated here only because 01-catalog's SELECT joins it.
```

## 3. API contract

Already in `specs/openapi/openapi.yaml` (`catalog` tag): `GET /categories`, `GET /products`,
`GET /products/{productId}`. No changes needed for this feature — this table is a summary.

| Method | Path | Auth | FR |
|---|---|---|---|
| GET | `/api/v1/categories` | public | FR-CATALOG-4 |
| GET | `/api/v1/products` | public | FR-CATALOG-1,3,4,6 |
| GET | `/api/v1/products/{productId}` | public | FR-CATALOG-2,5,7 |

## 4. Stored procedures / functions / triggers / views used

- Uses: `fn_is_variant_in_stock` (`07_SQL_DATABASE_STANDARDS.md` §5) — called per variant in the
  detail query to derive `stockStatus` server-side (SEC-CATALOG-1); never returns `stock_quantity`.
- New: none.

## 5. Key logic / code snippets (Go, illustrative)

### 5.1 Search + category filter — parameterized, appended clauses (07 §2)
```go
func (r *ProductRepository) List(ctx context.Context, f ListFilter) ([]Product, int, error) {
    where := "WHERE p.is_active = 1"
    args := []any{}
    if f.Query != "" {
        where += " AND MATCH(p.name, p.description) AGAINST (? IN NATURAL LANGUAGE MODE)"
        args = append(args, f.Query)                                    // FR-CATALOG-3
    }
    if f.CategoryID != nil {
        where += " AND EXISTS (SELECT 1 FROM product_category pc WHERE pc.product_id = p.product_id AND pc.category_id = ?)"
        args = append(args, *f.CategoryID)                               // FR-CATALOG-4
    }
    // total count for pagination, then LIMIT/OFFSET with the same `where`/`args` — never string-built.
    ...
}
```

### 5.2 Deriving stock status — never leak the quantity (SEC-CATALOG-1, FR-CATALOG-7)
```go
type VariantDTO struct {
    VariantID   int    `json:"variantId"`
    SKU         string `json:"sku"`
    Price       string `json:"price"`
    StockStatus string `json:"stockStatus"` // "IN_STOCK" | "OUT_OF_STOCK" — never a number
}

func toVariantDTO(v domain.Variant) VariantDTO {
    status := "OUT_OF_STOCK"
    if v.InStock { status = "IN_STOCK" }   // v.InStock computed from fn_is_variant_in_stock, not a raw count
    return VariantDTO{VariantID: v.ID, SKU: v.SKU, Price: v.Price.String(), StockStatus: status}
}
```

## 6. Edge cases & failure handling

- Empty search results → `200` with an empty `items` array, not `404` (searching is not "getting a
  resource," it's a query).
- Unknown `categoryId` filter → empty result set, not an error (consistent with "search that matches
  nothing" rather than "bad request").
- `GET /categories` filters `WHERE is_active = 1`, same as products — a category deactivated by
  `07-admin-catalog` disappears from the storefront's filter list immediately, though products already
  assigned to it are unaffected (the `product_category` row isn't touched, only the category's own
  visibility).
- Product with zero variants should never occur (REQ-10.3: every variant requires opening stock at
  creation, enforced by `07-admin-catalog`) — if it does (data issue), the detail endpoint treats it as
  fully out of stock rather than erroring.
- `GET /products/{productId}` for a deactivated or nonexistent product → `404` either way (no
  distinction — nothing here is ownership-sensitive like `02` §1's IDOR case, but consistency of "gone
  means gone" still matters for a clean client experience, REQ-10.5).
- Pagination beyond the last page → `200` with an empty `items` array and the correct `page.total`.

## 7. Which global rules apply (do not restate, just reference)

- Security: `02` §1 (public, no auth), §4 (parameterized SQL only — every query in this feature).
- SQL: `07` §2 (raw SQL policy), §5 (`fn_is_variant_in_stock`), §7 (`product_category` composite
  index, `product_variant.sku` unique index — both already in the DDL above).
- NFR: `03` §1 (< 2s budget), `04` §2 (cache headers).

## 8. Test hooks (→ tasks.md)

- Unit: DTO mapping never includes `stock_quantity` field name/value anywhere in the JSON output
  (a reflection-based test over the serialized struct, not just a manual check — this is the kind of
  rule that's easy to accidentally break in a later refactor, so the test should catch a field rename
  as well as an obvious mistake).
- Integration (`testcontainers-go`): search matches name and description, case-insensitively; category
  filter; pagination boundaries; deactivated product excluded from both list and direct detail fetch;
  `fn_is_variant_in_stock` returns the right enum at quantities 0 and >0.
