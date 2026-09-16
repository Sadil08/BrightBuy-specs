# <NN-feature-slug> — Plan (HOW)

> Owner hat: **Architect.** Turns every `FR-*`/`SEC-*`/`NFR-*` in `spec.md` into a design.
> No new requirements here — if you discover one, add it to `spec.md` first. Examples use CART.

**Feature slug:** `<FEAT>` · **Status:** ❌ draft · **Companion:** [spec.md](spec.md) · [tasks.md](tasks.md)
**Applies:** `global/01_TECH_STACK.md` (Go + chi + sqlx + MySQL 8), `global/06_ENGINEERING_STANDARDS.md`,
`global/07_SQL_DATABASE_STANDARDS.md`

---

## 1. Approach & architecture
> 1 paragraph + a diagram/flow. Which module(s) own what; which layer does which job.

```
[FE cart UI] --add/update--> [httpapi.CartHandler] --> [app.CartService] --> [mysql.CartRepository] --> MySQL
```

## 2. Data model / migrations
> Tables, columns, constraints, indexes — new in this feature only; link to the ER diagram / `07` for
> anything already defined globally. One `golang-migrate` file pair per change.

| Table | Key columns | Notes | Serves |
|---|---|---|---|
| `cart` | `cart_id`, `customer_id` (nullable) | matches ER diagram | FR-<FEAT>-2 |
| `cart_item` | `cart_item_id`, `cart_id`, `variant_id`, `quantity` | matches ER diagram | FR-<FEAT>-1 |

## 3. API contract
> Every endpoint: method, path, auth, request/response, status codes, FR ids. **Add the actual
> OpenAPI path to `specs/openapi/openapi.yaml` at the same time — this table is a summary, the YAML
> is authoritative.**

| Method | Path | Auth | Req | Resp | Codes | FR |
|---|---|---|---|---|---|---|
| POST | `/api/v1/cart/items` | customer | `{variantId, quantity}` | `CartDTO` | 200/400/409 | FR-<FEAT>-1 |

## 4. Stored procedures / functions / triggers / views used
> Cite `07_SQL_DATABASE_STANDARDS.md` objects this feature calls, and any **new** ones it introduces
> (with the full SQL, same style as `07`). This section is what a DBMS-module reviewer checks first.

- Uses: `fn_is_variant_in_stock` (`07` §5).
- New: none.

## 5. Key logic / code snippets (Go, illustrative)

```go
// Server-side price/stock resolution — never trust a client-sent price (SEC-<FEAT>-1)
variant, err := s.catalog.GetVariant(ctx, req.VariantID)
```

## 6. Edge cases & failure handling
- …

## 7. Which global rules apply (do not restate, just reference)
- Security: `02` §1 (identity from token). SQL: `07` §2 (parameterized only).

## 8. Test hooks (the plan QA will turn into tasks)
- Unit: … Integration (`testcontainers-go`): …
