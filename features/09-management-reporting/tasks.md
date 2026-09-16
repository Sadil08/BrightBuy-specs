# 09-management-reporting — Tasks (DO IT)

**Feature slug:** `REPORTING` · **Status:** ❌ not started · **Companion:** [spec.md](spec.md) · [plan.md](plan.md)

---

## 1. Build order

### Database
- [ ] **T1 `[DB]`** Migration `0009_reporting_views`: the five views
      (`07_SQL_DATABASE_STANDARDS.md` §6). — *Test:* each view queried directly against a known seed
      dataset, asserted against a hand-computed expected result (not just "returns rows").
- [ ] **T2 `[DB]`** `brightbuy_reporting` DB role + grants (`plan.md` §2.1), applied as a deploy step. —
      *Test:* connecting as this role and attempting `SELECT * FROM order` is denied; querying the
      five views succeeds.

### Backend
- [ ] **T3 `[BE]`** `internal/reporting/mysql.ReportingRepository`: one method per view, own
      connection pool using the `brightbuy_reporting` credential. — *Test:* integration.
- [ ] **T4 `[BE]`** `internal/reporting/app.ReportingService`: thin pass-through per report, category
      report's note text (`plan.md` §5.2). — *Test:* unit.
- [ ] **T5 `[BE]`** `internal/reporting/httpapi`: the five `GET` endpoints, shared CSV/JSON format
      helper (`plan.md` §5.1), all behind `RequirePermission("reports:view")`. — *Test:* `httptest` —
      `403` without the permission, CSV output is valid and header-matched.

### Frontend
- [ ] **T6 `[FE]`** Manager reports dashboard: five report views/tables, date-range filters where
      applicable, CSV export buttons. — *Test:* RTL + Playwright.

### Cross-cutting
- [ ] **T7 `[INFRA]`** Verify `openapi.yaml`'s `reports` paths match the shipped API.

## 2. Acceptance-criteria coverage map

| AC | Covered by |
|---|---|
| AC-REPORTING-1 | T1, T3, T4 |
| AC-REPORTING-2 | T5 |
| AC-REPORTING-3 | T1, T4 |
| AC-REPORTING-4 | T5 |
| AC-REPORTING-5 | T2 |

## 3. Definition of Done

- [ ] Every `FR-REPORTING-*` has a passing integration test against a known dataset.
- [ ] Every `AC-REPORTING-*` above is green.
- [ ] The `brightbuy_reporting` role's restricted grant is verified by an actual denied-query test
      (T2), not just documented.
- [ ] `gofmt`/`go vet`/ESLint clean; `openapi.yaml` matches the shipped API.
