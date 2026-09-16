# <NN-feature-slug> — Tasks (DO IT)

> Owner hat: **Tech lead / QA.** Ordered, granular, runtime-tagged tasks. Each task names its
> **test mechanism** and the requirement id it satisfies. Failing-first, then green (`global/05`).
> Tags: `[BE]` `[FE]` `[DB]` `[INFRA]` `[BE+FE]` (`README.md` §4). Examples use CART.

**Feature slug:** `<FEAT>` · **Status:** ❌ not started · **Companion:** [spec.md](spec.md) · [plan.md](plan.md)

---

## 1. Build order (dependencies first)

### Database
- [ ] **T1 `[DB]`** `golang-migrate` migration: `cart`, `cart_item` tables + indexes. — *Test:* migration applies clean on a fresh `testcontainers` MySQL, `down` then `up` again. — `FR-<FEAT>-2`

### Backend
- [ ] **T2 `[BE]`** `CartRepository` (sqlx, parameterized queries only). — *Test:* integration test against real MySQL. — `FR-<FEAT>-1,2`
- [ ] **T3 `[BE]`** `CartService.AddItem` — server-side price/stock resolution. — *Test:* unit test with a fake `catalog` port. — `FR-<FEAT>-1`
- [ ] **T4 `[BE]`** `CartHandler` endpoints per `plan.md` §3; identity from JWT context. — *Test:* `httptest` handler test (status codes), ownership test (2nd customer → 404). — `FR-<FEAT>-1`, `SEC-<FEAT>-1`

### Frontend
- [ ] **T5 `[FE]`** Cart state + typed API client call. — *Test:* Vitest. — `FR-<FEAT>-1,2`
- [ ] **T6 `[FE]`** Cart UI: line items, quantity stepper, subtotal. — *Test:* RTL render per state. — `AC-<FEAT>-2`

### Cross-cutting
- [ ] **T7 `[BE]`** Error mapping to the shared error shape (`06` §5). — *Test:* error-shape assertion.
- [ ] **T8 `[INFRA]`** Add the endpoint(s) to `specs/openapi/openapi.yaml` (should already exist from `plan.md` §3 — verify it matches the shipped handler).

## 2. Acceptance-criteria coverage map
| AC | Covered by |
|---|---|
| AC-<FEAT>-1 | T3, T4 |
| AC-<FEAT>-2 | T3, T6 |

## 3. Definition of Done
- [ ] Every `FR-*`/`SEC-*` has a passing unit and (where stateful) integration test.
- [ ] Every `AC-*` mapped above is green.
- [ ] No client-trusted prices; server recomputes (`02` §4).
- [ ] `gofmt`/`go vet`/ESLint clean; `openapi.yaml` matches the shipped API; `09_ISSUES_REGISTER.md`
      updated if this feature closed a register item.
