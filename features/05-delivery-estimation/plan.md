# 05-delivery-estimation — Plan (HOW)

**Feature slug:** `DELIVERY` · **Status:** ⚠️ draft-for-review · **Companion:** [spec.md](spec.md) · [tasks.md](tasks.md)
**Applies:** `global/01_TECH_STACK.md`, `global/06_ENGINEERING_STANDARDS.md`, `global/07_SQL_DATABASE_STANDARDS.md` §5

---

## 1. Approach & architecture

```
[FE cart/checkout page, on mode/city change] --POST /delivery/estimate--> [httpapi.DeliveryHandler]
                                                                                 |
                                                                    [app.DeliveryService.Preview]
                                                                                 |
                                                        SELECT fn_estimate_delivery_days(?, ?, ?) AS days
```

Deliberately **not** a repeat of the estimation logic in Go — the preview calls the exact same MySQL
function `sp_place_order` calls (`04-checkout-orders/plan.md` §2/§4). Reimplementing the arithmetic in
Go would create two places the 5/7/+3/0/+3 rule could drift out of sync (`SQL-BUG-1` was exactly this
kind of drift, caught before it shipped) — a single `SELECT fn_estimate_delivery_days(...)` avoids the
whole class of bug by construction rather than by discipline.

## 2. Data model / migrations

None — `city`, `fn_classify_city`, `fn_estimate_delivery_days` all already exist
(`04-checkout-orders/plan.md` §2). This feature only extends the **seed data** for `city` if `TBD-2`'s
list grows beyond the six metros (`db/seed`, not a schema migration — REQ-6.1's whole point is that
this is a data change).

## 3. API contract

Already in `specs/openapi/openapi.yaml` (`delivery` tag): `POST /delivery/estimate`, public.

| Method | Path | Auth | FR |
|---|---|---|---|
| POST | `/api/v1/delivery/estimate` | public | FR-DELIVERY-1..6 |

## 4. Stored procedures / functions / triggers / views used

- Uses (read-only call, no new object): `fn_estimate_delivery_days`, `fn_classify_city`
  (`07_SQL_DATABASE_STANDARDS.md` §5).
- New: none.

## 5. Key logic / code snippets (Go, illustrative)

```go
func (s *DeliveryService) Preview(ctx context.Context, mode string, cityID *int, items []LineInput) (Estimate, error) {
    itemsJSON, _ := json.Marshal(items) // same [{"variant_id":.., "quantity":..}] shape sp_place_order takes
    var days int
    err := s.db.QueryRowContext(ctx,
        `SELECT fn_estimate_delivery_days(?, ?, ?)`, mode, cityID, itemsJSON, // cityID nil-safe for StorePickup
    ).Scan(&days)
    if err != nil { return Estimate{}, err }
    return Estimate{Mode: mode, Days: days, Date: time.Now().AddDate(0, 0, days)}, nil
}
```

`cityID` as a Go `*int` bound to a `?` placeholder passes `NULL` correctly for Store Pickup — no
special-casing needed at the SQL boundary beyond what `database/sql` already does for a nil pointer.

## 6. Edge cases & failure handling

- Store Pickup request that also supplies a `cityId` → the value is accepted but ignored (REQ-6.6);
  the response never varies with it. Covered by AC-DELIVERY-4 specifically because this is an easy
  regression to introduce (someone "fixing" the function to use the city anyway).
- Unknown `cityId` for Standard Delivery → `fn_classify_city` already `COALESCE`s to `'Other'`
  (`07` §5) — the preview degrades to the conservative 7-day estimate rather than erroring, matching
  what checkout itself would do with the same input.
- Empty `items` array → `0` out-of-stock lines by definition; base estimate only, no `+3`.

## 7. Which global rules apply (do not restate, just reference)

- Security: `02` §1 (public, no auth — nothing sensitive is revealed).
- SQL: `07` §5 (the function itself, `SQL-BUG-1`'s fix).
- NFR: `03` §1.

## 8. Test hooks (→ tasks.md)

- Integration (`testcontainers-go`) — this feature has no meaningful *unit* test of its own (the logic
  under test lives in the SQL function, already covered by `04-checkout-orders`'s tests): all six
  `AC-DELIVERY-*` cases, run directly against the endpoint; **AC-DELIVERY-6** specifically as a
  same-request-twice comparison (preview, then checkout, assert equal) to prove the "can't diverge"
  claim isn't just architectural intent.
