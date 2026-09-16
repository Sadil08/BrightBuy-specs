# 06-inventory-stock — Plan (HOW)

**Feature slug:** `INVENTORY` · **Status:** ⚠️ draft-for-review · **Companion:** [spec.md](spec.md) · [tasks.md](tasks.md)
**Applies:** `global/01_TECH_STACK.md`, `global/06_ENGINEERING_STANDARDS.md`, `global/07_SQL_DATABASE_STANDARDS.md` §3.2

---

## 1. Approach & architecture

```
[Staff admin console] --GET /staff/variants?q=--> [httpapi.InventoryHandler] --> [app.InventoryService.Search] --> [mysql]
                       --POST /staff/variants/{id}/stock--> [app.InventoryService.Adjust] --CALL sp_adjust_stock--> MySQL
```

Small module — most of the hard part (atomicity, the audit trail's schema) was already built by
`04-checkout-orders` out of necessity. `internal/inventory` owns exactly two use cases.

## 2. Data model / migrations

None new. `stock_movement` was already created by `04-checkout-orders/plan.md` §2 (it had to exist
before `sp_place_order` could write to it, `SQL-BUG-2`) — this feature reuses it without redefining it.
`sp_adjust_stock` (`07_SQL_DATABASE_STANDARDS.md` §3.2) ships here as
`db/migrations/0006_inventory_procedures.up.sql`, since it's the first feature that actually needs it.

## 3. API contract

Already in `specs/openapi/openapi.yaml` (`inventory` tag): `GET /staff/variants`,
`POST /staff/variants/{variantId}/stock`.

| Method | Path | Auth | FR |
|---|---|---|---|
| GET | `/api/v1/staff/variants` | `stock:adjust` | FR-INVENTORY-1 |
| POST | `/api/v1/staff/variants/{variantId}/stock` | `stock:adjust` | FR-INVENTORY-2,3,4 |

## 4. Stored procedures / functions / triggers / views used

- Uses: `sp_adjust_stock` (`07` §3.2 — this feature's reason for existing), backstopped by
  `trg_stock_nonnegative` (`07` §4, shipped with `04`).
- New: none beyond `sp_adjust_stock` itself.

## 5. Key logic / code snippets (Go, illustrative)

```go
func (s *InventoryService) Adjust(ctx context.Context, actingUserID, variantID, delta int, reason string) error {
    err := s.repo.CallAdjustStock(ctx, variantID, delta, reason, actingUserID) // CALL sp_adjust_stock(...)
    if isAdjustmentBelowZero(err) {
        return &AdjustmentRejectedError{Reason: "would take stock below zero"} // FR-INVENTORY-3 -> 409
    }
    return err
}
```

Search (`FR-INVENTORY-1`) is a plain parameterized `LIKE`/exact-match query against `product_variant`
joined to `product` — no stored object needed, it's a simple read.

## 6. Edge cases & failure handling

- Adjustment of `0` → accepted as a no-op (writes a `stock_movement` row with `change_qty = 0` and
  whatever reason was given — unusual but not invalid; simpler than special-casing it away).
- Search with no query string → returns a paginated list of all variants rather than erroring, useful
  for a staff "browse everything" view.
- Adjusting a **deactivated** variant (`07-admin-catalog`'s soft-delete) → allowed; a deactivated
  product can still have its stock corrected (e.g. before reactivating it), it just doesn't appear in
  the public catalog while inactive.

## 7. Which global rules apply (do not restate, just reference)

- Security: `02` §1 (`stock:adjust` permission), §6.2 (n/a — no per-row ownership here, this is a
  staff-wide capability, not customer-owned data).
- SQL: `07` §3.2, §4 (trigger backstop), §2 (parameterized search query).
- NFR: `03` §1.

## 8. Test hooks (→ tasks.md)

- Unit: `Adjust`'s error mapping (`AdjustmentBelowZero` → `409`).
- Integration (`testcontainers-go`): search by SKU (exact and prefix) and by product name; a
  successful adjustment updates both `product_variant.stock_quantity` and writes the `stock_movement`
  row correctly; a would-go-negative adjustment is rejected and writes **no** row (AC-INVENTORY-3);
  the combined view of an order-driven row (from `04`'s tests) and a manual row for the same variant
  are both queryable and correctly attributed (AC-INVENTORY-4).
