# 07-admin-catalog — Plan (HOW)

**Feature slug:** `ADMINCATALOG` · **Status:** ⚠️ draft-for-review · **Companion:** [spec.md](spec.md) · [tasks.md](tasks.md)
**Applies:** `global/01_TECH_STACK.md`, `global/06_ENGINEERING_STANDARDS.md`, `global/07_SQL_DATABASE_STANDARDS.md` §1a, `global/02_SECURITY_BASELINE.md` §4

---

## 1. Approach & architecture

```
[Admin console] --POST /staff/products--> [httpapi.AdminCatalogHandler] --> [app.CatalogAdminService] --> [mysql] (catalog module, same tables 01-catalog reads)

[Admin console] --POST .../images/upload-url--> [httpapi.ImageHandler] --> [app.ImageService] --> [ImageStorage port] --> S3/MinIO
                --POST .../images (confirm)-----> fetch object, verify magic bytes, strip EXIF --> INSERT product_image
```

`CatalogAdminService` lives in `internal/catalog/app` alongside `01-catalog`'s `CatalogService` — same
module, same tables, split by **use case** (read vs. write), not by package, since both need the same
domain entities and the module boundary is `catalog` as a whole, not "browse" vs. "admin"
(`06_ENGINEERING_STANDARDS.md` §4: package-by-feature means package-by-*module*, and admin-catalog is
the write half of the `catalog` module, not a separate module).

## 2. Data model / migrations

No new tables — everything (`product`, `product_variant`, `category`, `product_category`,
`attribute_name`/`attribute_value`/`variant_attribute`, `product_image`) already exists
(`01-catalog/plan.md` §2, `07_SQL_DATABASE_STANDARDS.md` §1a). This feature ships
`db/migrations/0007_admin_catalog_indexes` only if a write-path query reveals a missing index during
testing — none anticipated, since all the write paths are single-row or small-batch inserts/updates
against already-indexed keys (`sku`, `product_id`, `category_id`).

## 3. API contract

Already in `specs/openapi/openapi.yaml` (`admin-catalog` tag): `POST /staff/products`,
`PATCH/DELETE /staff/products/{productId}`, `POST /staff/products/{productId}/variants`,
`PATCH/DELETE /staff/products/{productId}/variants/{variantId}`, `POST/PATCH/DELETE
/staff/categories[/{categoryId}]`, and the three image endpoints
(`POST .../images/upload-url`, `POST .../images`, `DELETE .../images/{imageId}`).

| Method | Path | Auth | FR |
|---|---|---|---|
| POST | `/api/v1/staff/products` | `catalog:write` | FR-ADMINCATALOG-1,2,3,4 |
| PATCH/DELETE | `/api/v1/staff/products/{id}` | `catalog:write` | FR-ADMINCATALOG-1,5,6 |
| POST/PATCH/DELETE | `/api/v1/staff/products/{id}/variants[/{id}]` | `catalog:write` | FR-ADMINCATALOG-1,3,4,5,6 |
| POST/PATCH/DELETE | `/api/v1/staff/categories[/{id}]` | `catalog:write` | FR-ADMINCATALOG-1,2,5 |
| POST | `/api/v1/staff/products/{id}/images/upload-url` | `catalog:image:write` | FR-ADMINCATALOG-7 |
| POST | `/api/v1/staff/products/{id}/images` | `catalog:image:write` | FR-ADMINCATALOG-7 |
| DELETE | `/api/v1/staff/products/{id}/images/{imageId}` | `catalog:image:write` | FR-ADMINCATALOG-7 |

## 4. Stored procedures / functions / triggers / views used

None — every write here is a single-table `INSERT`/`UPDATE` (or a small multi-row `INSERT` for
category assignment), well within plain parameterized SQL. No atomicity concern beyond what a single
statement or a short `dbx.WithTx` (product + its variants + its category assignments, one transaction)
already provides.

## 5. Key logic / code snippets (Go, illustrative)

### 5.1 Deactivate-not-delete (FR-ADMINCATALOG-5)
```go
func (s *CatalogAdminService) DeactivateProduct(ctx context.Context, productID int) error {
    return s.repo.SetActive(ctx, productID, false) // UPDATE product SET is_active = 0 — never DELETE
}
```
There is deliberately **no** `DeleteProduct` method anywhere in this service — REQ-10.5 isn't a policy
to remember to follow, it's the only code path that exists.

### 5.2 Image confirm — verification before the row is written (SEC-ADMINCATALOG-2, `02_SECURITY_BASELINE.md` §4)
```go
func (s *ImageService) Confirm(ctx context.Context, productID int, objectKey string) (*ProductImage, error) {
    data, err := s.storage.Fetch(ctx, objectKey)         // re-fetch what the client uploaded to S3/MinIO
    if err != nil { return nil, ErrObjectNotFound }
    if !isAllowedImageType(data) {                        // magic-byte check: JPEG/PNG/WebP signatures only
        s.storage.Delete(ctx, objectKey)                   // don't leave a rejected upload sitting in the bucket
        return nil, ErrInvalidImageType
    }
    clean := stripEXIF(data)
    s.storage.Put(ctx, objectKey, clean)                   // overwrite with the EXIF-stripped version
    return s.repo.InsertImage(ctx, productID, objectKey)
}
```

### 5.3 SKU uniqueness — surfaced as 409, not a generic 500 (FR-ADMINCATALOG-4)
```go
func (r *VariantRepository) Create(ctx context.Context, v Variant) (int, error) {
    res, err := r.db.ExecContext(ctx, `INSERT INTO product_variant (...) VALUES (...)`, ...)
    if isDuplicateKeyError(err, "uq_product_variant_sku") { return 0, ErrSKUTaken } // -> 409
    return int(lastInsertID(res)), err
}
```

## 6. Edge cases & failure handling

- Creating a product with zero variants → `400` (REQ-10.3 requires at least one, matching
  `00_OVERVIEW.md` §4's business rule that every product has ≥1 variant).
- Assigning a `categoryId` that doesn't exist or is deactivated → `400`, not a silent no-op FK failure.
- Deactivating a variant that's the product's *only* active variant → allowed (the product itself then
  effectively has zero visible variants and disappears from browsing by consequence, not by a special
  rule — `01-catalog`'s query already excludes it naturally since there's nothing left to show).
- Re-uploading an image to replace an existing one → a new `product_image` row (new object key), old
  one explicitly deleted via the `DELETE` endpoint — no implicit "replace" semantics, since that would
  need to define what "the same image slot" means (this feature doesn't have image slots, just an
  ordered list).
- Upload confirmed but the object key was never actually uploaded to S3/MinIO (client skipped step 1's
  actual PUT) → `Fetch` returns not-found → `400`, no row created.

## 7. Which global rules apply (do not restate, just reference)

- Security: `02` §1 (`catalog:write`/`catalog:image:write`), §4 (file-upload controls — this is where
  they're actually implemented).
- SQL: `07` §1a (`product_image`), §2 (parameterized only).
- NFR: `03` §1.

## 8. Test hooks (→ tasks.md)

- Unit: deactivate-not-delete (no delete code path exists — this is testable as "the method doesn't
  exist," not just "the method isn't called"), magic-byte rejection logic, SKU-conflict error mapping.
- Integration (`testcontainers-go` + a local MinIO in the test compose): full create → verify visible
  in `01-catalog` → deactivate → verify hidden cycle; SKU uniqueness at both create and update;
  price-change-doesn't-alter-past-orders (cross-checked against a `04-checkout-orders`-seeded order);
  image upload-url → (test uploads directly to MinIO, simulating the client) → confirm → magic-byte
  rejection path with a deliberately mismatched file.
