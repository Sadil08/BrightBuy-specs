# 09-management-reporting — Plan (HOW)

**Feature slug:** `REPORTING` · **Status:** ⚠️ draft-for-review · **Companion:** [spec.md](spec.md) · [tasks.md](tasks.md)
**Applies:** `global/01_TECH_STACK.md`, `global/06_ENGINEERING_STANDARDS.md`, `global/07_SQL_DATABASE_STANDARDS.md` §6

---

## 1. Approach & architecture

```
[Manager console] --GET /reports/*--> [httpapi.ReportingHandler] --> [app.ReportingService] --> [mysql.ReportingRepository]
                                                                                                        |
                                                                                          SELECT * FROM vw_* WHERE ...
```

`internal/reporting` has no domain entities of its own and no writes anywhere in the module
(`00_OVERVIEW.md` §3: "reporting owns no tables; reads through views"). Every query is a `SELECT`
against one of the five views in `07_SQL_DATABASE_STANDARDS.md` §6 — this module exists to be thin.

## 2. Data model / migrations

None new. Ships `db/migrations/0009_reporting_views.up.sql` containing the five `CREATE VIEW`
statements already fully specified in `07_SQL_DATABASE_STANDARDS.md` §6
(`vw_quarterly_sales`, `vw_top_selling_products`, `vw_category_wise_orders`,
`vw_upcoming_deliveries`, `vw_customer_order_summary`) — this feature is the first to actually need
them created.

### 2.1 Read-only DB role for defense in depth (SEC-REPORTING-1)

```sql
-- Run once, as part of this feature's deploy step (not a golang-migrate migration — a role grant,
-- not a schema object): the application's normal DB role already lacks broad privileges (02 §6.1),
-- but reporting queries specifically are additionally guaranteed safe by using a SECOND, narrower
-- credential for this module's connection pool.
GRANT SELECT ON brightbuy.vw_quarterly_sales        TO 'brightbuy_reporting'@'%';
GRANT SELECT ON brightbuy.vw_top_selling_products   TO 'brightbuy_reporting'@'%';
GRANT SELECT ON brightbuy.vw_category_wise_orders   TO 'brightbuy_reporting'@'%';
GRANT SELECT ON brightbuy.vw_upcoming_deliveries    TO 'brightbuy_reporting'@'%';
GRANT SELECT ON brightbuy.vw_customer_order_summary TO 'brightbuy_reporting'@'%';
-- No grant on any base table — this role cannot even read `order`/`payment` directly, only through
-- the views, which is what makes SEC-REPORTING-1 a database-enforced fact, not just an application
-- convention.
```

`internal/reporting/mysql` opens its own small connection pool with this credential, separate from
the pool every other module shares — the one place in the codebase with two DB credentials in play,
justified because reporting's blast radius should be structurally smaller than everything else's.

## 3. API contract

Already in `specs/openapi/openapi.yaml` (`reports` tag): `GET /reports/quarterly-sales`,
`GET /reports/top-selling-products`, `GET /reports/category-wise-orders`,
`GET /reports/upcoming-deliveries`, `GET /reports/customer-order-summary` — all accept
`?format=csv`.

| Method | Path | Auth | FR |
|---|---|---|---|
| GET | `/api/v1/reports/quarterly-sales` | `reports:view` | FR-REPORTING-1,7 |
| GET | `/api/v1/reports/top-selling-products` | `reports:view` | FR-REPORTING-2 |
| GET | `/api/v1/reports/category-wise-orders` | `reports:view` | FR-REPORTING-3 |
| GET | `/api/v1/reports/upcoming-deliveries` | `reports:view` | FR-REPORTING-4 |
| GET | `/api/v1/reports/customer-order-summary` | `reports:view` | FR-REPORTING-5 |

## 4. Stored procedures / functions / triggers / views used

- Uses: all five views (`07` §6 — this feature's entire reason for existing).
- New: none — the views are new *to the schema* here, but their design is already complete in `07`.

## 5. Key logic / code snippets (Go, illustrative)

### 5.1 One handler shape, format-aware (FR-REPORTING-9)
```go
func (h *ReportingHandler) QuarterlySales(w http.ResponseWriter, r *http.Request) {
    rows, err := h.service.QuarterlySales(r.Context(), yearParam(r))
    if err != nil { httpx.WriteError(w, ...); return }
    if r.URL.Query().Get("format") == "csv" {
        httpx.WriteCSV(w, rows) // one shared helper, reused by all five handlers — not five copies
        return
    }
    httpx.WriteJSON(w, http.StatusOK, rows)
}
```

### 5.2 Category-wise report — the label REQ-9.3 requires isn't optional (FR-REPORTING-3)
```go
type CategoryOrderCount struct {
    CategoryID int    `json:"categoryId"`
    Name       string `json:"name"`
    OrderCount int    `json:"orderCount"`
}
type CategoryWiseReport struct {
    Categories []CategoryOrderCount `json:"categories"`
    Note       string                `json:"note"` // "Counts do not sum to total orders — an order spanning multiple categories is counted once per category."
}
```

## 6. Edge cases & failure handling

- Quarterly sales for a year with no orders → `200` with all four quarters at `0`, not an empty
  response (a manager comparing years shouldn't see a missing row and wonder if it's a bug).
- `top-selling-products` with no `from`/`to` params → defaults to all-time, documented in the response
  (echoes the effective date range used, so a manager knows what they're looking at).
- CSV export of a report with zero rows → still a valid CSV with just the header row, never an empty
  body (a spreadsheet import shouldn't choke on it).

## 7. Which global rules apply (do not restate, just reference)

- Security: `02` §1 (`reports:view`), §6.3 (views are read-only by construction), this feature's own
  `SEC-REPORTING-1`.
- SQL: `07` §6 (the views themselves — window functions, `WITH ROLLUP`, the category CTE).
- NFR: `03` §1, this feature's own `NFR-REPORTING-1`.

## 8. Test hooks (→ tasks.md)

- Integration (`testcontainers-go`), against the shared seed dataset (`db/seed`, `01-catalog` T3) plus
  orders created by `04-checkout-orders`'s test suite where feasible, or a dedicated reporting fixture
  otherwise: each of the five views' output checked against a hand-computed expected result for a
  known dataset — not just "returns 200 with some rows." `SEC-REPORTING-1`'s grant is verified by
  attempting a `SELECT` on a base table (e.g. `order`) with the `brightbuy_reporting` credential and
  asserting it's denied.
