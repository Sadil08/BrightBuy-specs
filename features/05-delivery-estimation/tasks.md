# 05-delivery-estimation — Tasks (DO IT)

**Feature slug:** `DELIVERY` · **Status:** ❌ not started · **Companion:** [spec.md](spec.md) · [plan.md](plan.md)

---

## 1. Build order

Depends on `04-checkout-orders` T2 (`fn_estimate_delivery_days`, `city` table) already landed.

### Database
- [ ] **T1 `[DB]`** Extend `db/seed` city list if BrightBuy has resolved `TBD-2` beyond the six seeded
      metros by this point; otherwise no-op (already seeded in `04`). — *Test:* n/a unless data changes.

### Backend
- [ ] **T2 `[BE]`** `internal/delivery/app.DeliveryService.Preview` (`plan.md` §5). — *Test:* unit with
      a fake DB call, plus the full integration suite below.
- [ ] **T3 `[BE]`** `internal/delivery/httpapi`: `POST /delivery/estimate`, public route (no auth
      middleware applied). — *Test:* `httptest`.

### Frontend
- [ ] **T4 `[FE]`** Cart/checkout page: call the preview endpoint on delivery-mode/city change, show
      the estimated date inline before the confirm step (REQ-6.5). — *Test:* RTL + Playwright.

### Cross-cutting
- [ ] **T5 `[INFRA]`** Verify `openapi.yaml`'s `delivery` path matches the shipped API.

## 2. Acceptance-criteria coverage map

| AC | Covered by |
|---|---|
| AC-DELIVERY-1 | T2, T3 |
| AC-DELIVERY-2 | T2, T3 |
| AC-DELIVERY-3 | T2, T3 |
| AC-DELIVERY-4 | T2, T3 |
| AC-DELIVERY-5 | T2, T3 |
| AC-DELIVERY-6 | T2, T3 (cross-checked against `04-checkout-orders`'s checkout endpoint) |

## 3. Definition of Done

- [ ] Every `FR-DELIVERY-*` has a passing integration test (per `plan.md` §8, unit tests aren't the
      meaningful layer here).
- [ ] Every `AC-DELIVERY-*` above is green, including the cross-feature AC-DELIVERY-6 comparison.
- [ ] `gofmt`/`go vet`/ESLint clean; `openapi.yaml` matches the shipped API.
