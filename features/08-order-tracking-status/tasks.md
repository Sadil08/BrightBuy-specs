# 08-order-tracking-status — Tasks (DO IT)

**Feature slug:** `ORDERSTATUS` · **Status:** ❌ not started · **Companion:** [spec.md](spec.md) · [plan.md](plan.md)

---

## 1. Build order

### Database
- [ ] **T1 `[DB]`** Migration `0008_order_status_procedures`: `trg_order_status_transition`,
      `trg_order_status_history`, `sp_cancel_order` (`07_SQL_DATABASE_STANDARDS.md` §3.3, §4). —
      *Test:* direct SQL — invalid transition rejected independent of any application code; a valid
      `UPDATE` produces exactly one history row; `sp_cancel_order` returns stock, writes
      `stock_movement`, and is itself rejected by the transition trigger when attempted on a
      `Delivered` order.

### Backend
- [ ] **T2 `[BE]`** `internal/ordering/mysql`: `OrderRepository.SetStatus`, `CallCancelOrder`,
      `PaymentRepository.SetStatus`. — *Test:* integration.
- [ ] **T3 `[BE]`** `internal/ordering/app.OrderStatusService`: `UpdateStatus` (transition table +
      COD→Paid side effect, `plan.md` §5.1), `Cancel`, `@acting_user_id` session-variable wiring
      (`plan.md` §5.2). — *Test:* unit (transition-table completeness, table-driven) + integration
      (full status walk).
- [ ] **T4 `[BE]`** `internal/ordering/httpapi`: `PATCH /staff/orders/{orderId}/status`, routing
      `status: "Cancelled"` to `Cancel()` internally (`plan.md` §3), behind
      `RequirePermission("order:status:update")`/`RequirePermission("order:cancel")` as appropriate. —
      *Test:* `httptest` — `403` without the permission, `409` on invalid transition.

### Frontend
- [ ] **T5 `[FE]`** Staff order-management screen: status dropdown constrained to valid next states
      (client-side convenience only — the real guard is server-side, `plan.md` §1), cancel action. —
      *Test:* RTL + Playwright.

### Cross-cutting
- [ ] **T6 `[INFRA]`** Verify `openapi.yaml`'s staff-order-status path matches the shipped API.

## 2. Acceptance-criteria coverage map

| AC | Covered by |
|---|---|
| AC-ORDERSTATUS-1 | T1, T2, T3, T4 |
| AC-ORDERSTATUS-2 | T1, T3, T4 |
| AC-ORDERSTATUS-3 | T1, T3, T4 |
| AC-ORDERSTATUS-4 | T1, T2, T3 |
| AC-ORDERSTATUS-5 | T2, T3 |
| AC-ORDERSTATUS-6 | T2, T3 |

## 3. Definition of Done

- [ ] Every `FR-ORDERSTATUS-*` has a passing unit and integration test.
- [ ] Every `AC-ORDERSTATUS-*` above is green.
- [ ] The invalid-transition test in T1 is run **directly against SQL**, not just through the Go
      service — proving the trigger is the real backstop, not just the Go table.
- [ ] `gofmt`/`go vet`/ESLint clean; `openapi.yaml` matches the shipped API.
