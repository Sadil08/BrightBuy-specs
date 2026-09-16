# 05 — Testing Strategy

Inherited by every feature. The rule that matters most for this project: **anything that touches a
trigger, stored procedure, or row-lock behavior is tested against a real MySQL 8 instance, never a
mock.** Mocking the database for that logic would mean the DBMS-module deliverables — the actual
graded artifacts — are never exercised by a test at all.

---

## 1. Test pyramid

| Layer | Tool | Scope | Runs against |
|---|---|---|---|
| Unit (Go) | stdlib `testing` + `testify` | `domain`/`app` package logic: pricing, delivery-estimate calculation, state-transition validation, validator rules | nothing — pure functions, ports mocked with hand-written fakes |
| Integration (Go) | `testing` + `testcontainers-go` (real MySQL 8 container) | `mysql` package: every query, every stored procedure/function/trigger call, migrations applying cleanly | a disposable containerized MySQL, migrated from `db/migrations` fresh each run |
| API (Go) | `testing` + `net/http/httptest` | `httpapi` handlers: status codes, auth/role gating, error shape, pagination | in-process server, real DB (via testcontainers) or the integration layer's fakes for unit-level handler tests |
| Component (FE) | Vitest + React Testing Library | Individual components/hooks in isolation | mocked API client |
| E2E (FE) | Playwright | Full user journeys: browse → cart → checkout → order tracking; admin: create product → adjust stock → order fulfillment | full stack, seeded DB |

## 2. What must be an integration test, never a unit test with a mock

- **Stock decrement atomicity (REQ-4.2, 4.6, 4.7):** two concurrent checkouts against the same
  variant, asserting neither oversells and the row-locking order (ascending `variant_id`) prevents
  deadlock. This is exactly the kind of behavior a mock cannot reproduce — it only exists under a real
  engine's locking.
- **Stock-never-negative trigger (`07` §4):** attempt a write that would take stock below zero
  directly at the SQL layer (bypassing the procedure) and assert the trigger rejects it.
- **Order status transition trigger:** attempt an invalid transition and assert rejection at the DB
  layer, independent of whatever the application layer does.
- **`order_status_history` auto-population trigger:** update `order.status` and assert a history row
  appears without the application inserting one explicitly.
- **Reporting views:** each of the five views (`07` §5), queried directly, checked against a known
  seeded dataset for correct aggregation (including the window-function ranking and `ROLLUP` outputs).
- **Migrations:** `golang-migrate up` then `down` then `up` again on a fresh container — a migration
  that isn't reversible or idempotent-on-reapply is a bug.

## 3. What's fine as a unit test with fakes

- Delivery-estimate calculation (REQ-6.1–6.4) as a pure function once city classification is fetched —
  the classification lookup itself is faked.
- Cart subtotal computation, DTO validation, JWT issuance/verification, BCrypt wrapping.
- Order state-machine transition table, tested as pure Go logic (independent of the DB trigger that
  backstops the same rule — both are tested, separately, on purpose, since they're two independent
  layers of the same guarantee).

## 4. CI gates (once CI exists — see `11_BUILD_PLAN.md`)

- `go vet`, `staticcheck`, `gofmt -l` (fails on unformatted code) for the backend.
- ESLint + `tsc --noEmit` for the frontend.
- Unit + integration test suites both green; integration suite runs against a `testcontainers`-managed
  MySQL 8, not a shared dev database.
- A migration that fails to apply cleanly on a fresh container blocks the build.
- No secret-scanner finding (`gitleaks`).

## 5. Seed data for tests

`db/seed/` (SRS §6.1: ≥40 products / ≥10 categories) is a **repeatable script**, run to build the
integration/E2E baseline dataset — never hand-populated. The reporting-view integration tests assert
against known aggregates computed from this exact seed set, so the seed script's output must be
deterministic (fixed, not random, sample data).
