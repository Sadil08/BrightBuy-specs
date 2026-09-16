# 07-admin-catalog — Tasks (DO IT)

**Feature slug:** `ADMINCATALOG` · **Status:** ❌ not started · **Companion:** [spec.md](spec.md) · [plan.md](plan.md)

---

## 1. Build order

### Backend
- [ ] **T1 `[BE]`** `internal/catalog/mysql`: `ProductRepository.Create/Update/SetActive`,
      `VariantRepository.Create/Update/SetActive` (SKU-conflict mapping, `plan.md` §5.3),
      `CategoryRepository.Create/Update/SetActive`. — *Test:* integration.
- [ ] **T2 `[BE]`** `internal/catalog/app.CatalogAdminService`: product/variant/category use cases,
      deactivate-not-delete (`plan.md` §5.1), category-assignment validation (`plan.md` §6). — *Test:*
      unit with a fake repository.
- [ ] **T3 `[BE]`** `internal/catalog/httpapi`: the eight product/variant/category admin endpoints,
      all behind `RequirePermission("catalog:write")`. — *Test:* `httptest` — `403` without the
      permission, `409` on SKU conflict.
- [ ] **T4 `[BE]`** `internal/catalog/mysql.ImageStorage` (S3/MinIO adapter, implements the port from
      `06_ENGINEERING_STANDARDS.md` §2). — *Test:* integration against a local MinIO.
- [ ] **T5 `[BE]`** `internal/catalog/app.ImageService`: `PresignUpload`, `Confirm` (magic-byte
      verification + EXIF strip, `plan.md` §5.2), `Delete`. — *Test:* unit (magic-byte rejection with
      fixture files) + integration (full presign→upload→confirm cycle against MinIO).
- [ ] **T6 `[BE]`** `internal/catalog/httpapi.ImageHandler`: the three image endpoints, behind
      `RequirePermission("catalog:image:write")` — distinct from `catalog:write` (AC-ADMINCATALOG-5).
      — *Test:* `httptest`.

### Frontend
- [ ] **T7 `[FE]`** Admin console: product create/edit form (variants, categories), category
      management, image upload widget (direct-to-S3 PUT from the browser using the presigned URL). —
      *Test:* RTL + Playwright.

### Cross-cutting
- [ ] **T8 `[INFRA]`** Verify `openapi.yaml`'s `admin-catalog` paths match the shipped API.

## 2. Acceptance-criteria coverage map

| AC | Covered by |
|---|---|
| AC-ADMINCATALOG-1 | T1, T2, T3 |
| AC-ADMINCATALOG-2 | T1, T3 |
| AC-ADMINCATALOG-3 | T1, T2, T3 |
| AC-ADMINCATALOG-4 | T1, T2 (cross-checked against `04-checkout-orders`) |
| AC-ADMINCATALOG-5 | T6 |
| AC-ADMINCATALOG-6 | T5, T6 |

## 3. Definition of Done

- [ ] Every `FR-ADMINCATALOG-*` has a passing unit and integration test.
- [ ] Every `AC-ADMINCATALOG-*` above is green.
- [ ] No hard-delete code path exists for product/variant/category anywhere in this feature
      (SEC-ADMINCATALOG behavior verified by absence, not just by a passing test).
- [ ] `gofmt`/`go vet`/ESLint clean; `openapi.yaml` matches the shipped API.
