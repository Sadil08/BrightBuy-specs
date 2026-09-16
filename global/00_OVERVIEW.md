# 00 — Overview

Business context and product model every feature inherits. Source of truth: `BrightBuy_SRS_v1.1`
(the PDF in the repo root) — this doc restates it in spec-system terms and does not override it.

---

## 1. What BrightBuy is

BrightBuy is a Texas-based consumer electronics retail chain moving into e-commerce. Phase 1 is a
**single-warehouse** storefront: browse a limited catalog (electronics + toys, ≥40 products / ≥10
categories), register, build a cart, check out with Store Pickup or Standard Delivery and Cash on
Delivery or Card Payment, and track the order. Staff manage catalog and stock; management pulls five
operational reports.

**Two outcomes matter most:** BrightBuy must never oversell stock it doesn't have, and management
must see how the store is performing without asking a developer to run a query. A third, implicit
outcome for this build: the codebase and schema are a **teaching artifact** — architecture and SQL
technique are graded, not just working software.

## 2. Actors (from SRS §2.3) and roles

| Actor | Role | Notes |
|---|---|---|
| Guest | none | Browse + local (client-only) cart. Cannot check out. |
| Registered Customer | `CUSTOMER` | Cart, checkout, own order history only. Not permission-table-driven — ownership-scoped, not capability-scoped (`02_SECURITY_BASELINE.md` §1). |
| Warehouse/Inventory Staff | `WAREHOUSE_STAFF` | Products, variants, categories, images, stock adjustments — via permissions `catalog:write`, `catalog:image:write`, `stock:adjust`. |
| Store/Order Manager | `ORDER_MANAGER` | Order status, fulfilment, staff-initiated cancellation — via permissions `order:status:update`, `order:cancel`. |
| Business/Management | `MANAGER` | Five reports, read-only — via permission `reports:view`. |
| System Administrator | `ADMIN` | Accounts, roles, **and now permission assignment** — unconditional bypass of the permission system (never table-driven, so it can't be locked out of itself). Highest privilege. |

**Resolved (OQ-1, 2026-09-02):** five roles, and — more significantly — **what each role may do is no
longer hardcoded.** A `permission`/`role_permission` schema (`07_SQL_DATABASE_STANDARDS.md` §1a) lets
`ADMIN` grant or revoke individual permissions per role at runtime from the admin console, so BrightBuy
can hand `ORDER_MANAGER` a new capability later without a redeploy. Logged as `RBAC-1` in
`09_ISSUES_REGISTER.md`. Full authorization design: `02_SECURITY_BASELINE.md` §1.

**Correction (2026-09-04):** an earlier version of this doc claimed roles "match the ER diagram's
`ROLE` table." That was wrong — the SRS's ER diagram (Appendix B, Figure B.1) covers only
`Category, Product, Customer, Order, ProductVariant, OrderItem, Delivery, Payment`, by its own caption
("the main entities"); it does not depict authentication/role/staff entities at all. The five-role
model, `user_account`, `customer`, `staff_profile`, `permission`, and `role_permission` are all
designed from the SRS's *text* requirements (REQ-2.1–2.8) since no diagram exists to follow for this
part of the schema — see `02-auth`'s `plan.md` for the actual DDL, and `DB-ERD-1` in
`09_ISSUES_REGISTER.md` for this correction's record.

## 3. Module map (owns which tables, mirrors SRS §2.1.2)

The backend is split into modules by business capability, not technical type. Each module owns its
tables and is reached only through its own package's exported functions — never another module's
internals. See `06_ENGINEERING_STANDARDS.md` for how this maps to Go packages.

| Module | Responsibility | Owns (tables, from the ER diagram) |
|---|---|---|
| `catalog` | Products, variants, categories, search, images | `product`, `product_variant`, `category`, `product_category`, `attribute_name`, `attribute_value`, `variant_attribute`, `product_image` |
| `identity` | Registration, login, roles, sessions, **permission assignment** | `user_account`, `role`, `customer`, `staff_profile`, `permission`, `role_permission` |
| `cart` | Cart lines and subtotals before checkout | `cart`, `cart_item` |
| `ordering` | Checkout, order creation, order status | `order`, `order_item`, `order_status_history` |
| `inventory` | Stock levels, adjustments, audit log | (stock lives on `product_variant.stock_quantity`), `stock_movement` |
| `delivery` | Delivery modes, city classification, estimates | `delivery`, `city` |
| `payment` | Payment method and lifecycle | `payment` |
| `reporting` | The five management reports (read-only) | no tables; reads through views (`07_SQL_DATABASE_STANDARDS.md` §5) |
| `shared` | Cross-cutting: errors, auth, config, logging | no business tables |

Two rules keep the boundaries honest: a module never queries another module's tables directly (it
calls that module's exported service), and `reporting` is read-only through views so a report can
never corrupt operational data.

## 4. Product model

- A product belongs to **one or more** categories (many-to-many, `product_category`).
- A product has **one or more variants**; price and stock live on the **variant**, not the product
  (REQ business rule, SRS §5.5). A single-configuration product still has exactly one variant.
- A variant's distinguishing attributes (color, storage size, …) are modeled as
  `attribute_name`/`attribute_value`/`variant_attribute` — an EAV-style side table, not fixed columns,
  so new attribute types don't require a schema change.
- SKU is unique at the variant level (`product_variant.sku`, unique index).
- Stock is decremented **only** at order confirmation, atomically, via a stored procedure — never on
  add-to-cart (a cart does not reserve stock).

## 5. Order lifecycle (SRS Appendix B.3, REQ-8.1/8.2)

```
Placed → Confirmed → Processing → Shipped / ReadyForPickup → Delivered → Completed
                                                                   ↓
                                                              Cancelled (terminal, from any
                                                              pre-Delivered state; returns stock)
```

Enforced by an explicit transition table, checked in the application layer **and** backstopped by a
database trigger (`07_SQL_DATABASE_STANDARDS.md` §4) — the DB never accepts an invalid transition
even if application logic has a bug.

## 6. Glossary

See SRS Appendix A for the full glossary (ACID, BCrypt, Cart, COD, CORS, DTO, Idempotency key, InnoDB,
JWT, Main City, ORM, Port and adapter, REST, SKU, SOLID, Store Pickup, Stored procedure/trigger,
Variant). Terms specific to this spec system:

- **Port**: a Go interface declared by the `app` layer of a module, describing what it needs from the
  outside world (e.g. a `StockRepository` interface). See `06_ENGINEERING_STANDARDS.md`.
- **Adapter**: the concrete implementation of a port, living in that module's `mysql` package.
- **REQ-\***: a requirement ID owned by the SRS. **FR/NFR/SEC-\***: feature-local IDs owned by this
  spec system, each tracing to one or more `REQ-*`.
