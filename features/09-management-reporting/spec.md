# 09-management-reporting — Spec (WHAT & WHY)

**Feature slug:** `REPORTING` · **Status:** ⚠️ draft-for-review · **Companion:** [plan.md](plan.md) · [tasks.md](tasks.md)
**Inherits:** `global/02_SECURITY_BASELINE.md`, `global/03_NFR_BASELINE.md`
**SRS section:** §4.9 Management Reporting (Medium priority)

---

## 1. Summary

The five reports `MANAGER` uses to see how the store is performing without asking a developer to run a
query (`00_OVERVIEW.md` §1's stated goal). Deliberately last in the build order — every view this
feature exposes reads data the earlier features already produce, so there's a realistic dataset to
report on by the time this is built, not synthetic fixtures. All five underlying views are already
fully specified in `07_SQL_DATABASE_STANDARDS.md` §6; this feature is the thin read-only API surface
over them plus CSV export.

## 2. Actors & scope

- **In scope:** the five report endpoints, CSV export, `reports:view` permission gating.
- **Out of scope (explicit):** the views themselves (`07_SQL_DATABASE_STANDARDS.md` §6, already
  written); any report that would require writing to operational data (none do, by design — REQ-9.8).
- **Working assumptions used:** none new.

## 3. User stories

| ID | As a… | I want… | so that… | Priority |
|---|---|---|---|---|
| US-REPORTING-1 | manager | to see total sales per quarter for a year | I can track revenue trends | 🟡 P2 |
| US-REPORTING-2 | manager | to see which products sell best, by quantity and by revenue | I can plan stock and promotions | 🟡 P2 |
| US-REPORTING-3 | manager | to see how many orders touch each category | I understand demand by category | 🟡 P2 |
| US-REPORTING-4 | manager | to see upcoming unfulfilled orders and their estimated delivery | I can anticipate fulfilment load | 🟡 P2 |
| US-REPORTING-5 | manager | to see a customer's order and payment history | I can answer a customer service question quickly | 🟡 P2 |
| US-REPORTING-6 | manager | to export any report as CSV | I can analyze it in a spreadsheet | 🟡 P2 |

## 4. Functional requirements

| ID | Requirement | SRS ref | Priority |
|---|---|---|---|
| FR-REPORTING-1 | The system MUST produce a quarterly sales report for a given year, showing total sales value per quarter Q1–Q4. | REQ-9.1 | 🟡 P2 |
| FR-REPORTING-2 | The system MUST produce a top-selling-products report for a given period, ranked by quantity sold and by revenue. | REQ-9.2 | 🟡 P2 |
| FR-REPORTING-3 | The system MUST produce a category-wise report showing the number of orders containing at least one item from each category, and MUST state that the column does not sum to the total order count. | REQ-9.3 | 🟡 P2 |
| FR-REPORTING-4 | The system MUST produce a report listing upcoming, unfulfilled orders with their estimated delivery dates. | REQ-9.4 | 🟡 P2 |
| FR-REPORTING-5 | The system MUST produce a customer-wise summary showing each customer's orders and the payment status of each. | REQ-9.5 | 🟡 P2 |
| FR-REPORTING-6 | All five reports MUST be restricted to accounts holding `reports:view`, enforced server-side. | REQ-9.6 | 🟡 P2 |
| FR-REPORTING-7 | Sales figures MUST be based on order date, include every order except Cancelled, and sales value MUST exclude tax and delivery fee — every report MUST label the basis it uses. | REQ-9.7 | 🟡 P2 |
| FR-REPORTING-8 | Reports MUST read through database views or read-only queries and MUST NEVER modify operational data. | REQ-9.8 | 🟡 P2 |
| FR-REPORTING-9 | Report output MUST be exportable as CSV. | REQ-9.9 | 🟡 P2 |

## 5. NFR deltas (feature-specific only)

| ID | NFR | Target |
|---|---|---|
| NFR-REPORTING-1 | Report queries complete within 5s against the seeded Phase 1 dataset. | `global/03` §1, restated as this feature's own headline budget. |

## 6. Security deltas (feature-specific only)

| ID | Requirement | Closes |
|---|---|---|
| SEC-REPORTING-1 | The DB role the backend connects as has no `INSERT`/`UPDATE`/`DELETE` grant on the five report views — a bug in this feature's code cannot become a write path even in principle (`02_SECURITY_BASELINE.md` §6.3). | — |

## 7. Acceptance criteria (Given/When/Then)

- **AC-REPORTING-1** — *Given* orders across all four quarters of a year, *When* the quarterly sales
  report is requested for that year, *Then* each quarter's total excludes tax and delivery fee and
  excludes any Cancelled order.
- **AC-REPORTING-2** — *Given* a caller without `reports:view`, *When* any of the five endpoints is
  called, *Then* the response is `403`.
- **AC-REPORTING-3** — *Given* an order spanning three categories, *When* the category-wise report is
  requested, *Then* that order is counted once in each of the three categories' counts, and the
  response's own text notes the counts don't sum to total orders.
- **AC-REPORTING-4** — *Given* any report endpoint, *When* called with `format=csv`, *Then* the
  response is a valid CSV with a header row matching the JSON response's field names.
- **AC-REPORTING-5** — *Given* the reporting DB role, *When* its grants are inspected, *Then* it holds
  `SELECT` only on the five views — no write privilege anywhere in this feature's path.

## 8. Open questions / decisions needed

- [ ] None outstanding.
