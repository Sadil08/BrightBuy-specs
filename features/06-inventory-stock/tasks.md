# 06-inventory-stock — Tasks (DO IT)

**Feature slug:** `INVENTORY` · **Status:** ❌ not started · **Companion:** [spec.md](spec.md) · [plan.md](plan.md)

---

## 1. Build order

### Database
- [ ] **T1 `[DB]`** Migration `0006_inventory_procedures`: `sp_adjust_stock` (`07_SQL_DATABASE_STANDARDS.md`
      §3.2). — *Test:* direct SQL — successful adjustment updates quantity + writes `stock_movement`;
      would-go-negative adjustment rejected, no row written, quantity unchanged.

### Backend
- [ ] **T2 `[BE]`** `internal/inventory/mysql.VariantRepository`: `Search` (SKU/name, `plan.md` §5),
      `InventoryRepository.CallAdjustStock`. — *Test:* integration.
- [ ] **T3 `[BE]`** `internal/inventory/app.InventoryService`: `Search`, `Adjust` (error mapping,
      `plan.md` §5). — *Test:* unit with a fake repository.
- [ ] **T4 `[BE]`** `internal/inventory/httpapi.InventoryHandler`: `GET /staff/variants`,
      `POST /staff/variants/{variantId}/stock`, both behind `RequirePermission("stock:adjust")`. —
      *Test:* `httptest` — `403` without the permission.

### Frontend
- [ ] **T5 `[FE]`** Staff stock-adjustment screen: SKU/name search, current quantity display, delta +
      reason form. — *Test:* RTL + Playwright.

### Cross-cutting
- [ ] **T6 `[INFRA]`** Verify `openapi.yaml`'s `inventory` paths match the shipped API.

## 2. Acceptance-criteria coverage map

| AC | Covered by |
|---|---|
| AC-INVENTORY-1 | T2, T4 |
| AC-INVENTORY-2 | T1, T2, T3, T4 |
| AC-INVENTORY-3 | T1, T3, T4 |
| AC-INVENTORY-4 | T1 (integration test spans this + `04-checkout-orders`'s seeded data) |

## 3. Definition of Done

- [ ] Every `FR-INVENTORY-*` has a passing unit and integration test.
- [ ] Every `AC-INVENTORY-*` above is green.
- [ ] `gofmt`/`go vet`/ESLint clean; `openapi.yaml` matches the shipped API.
