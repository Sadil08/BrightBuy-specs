# 06 — Engineering Standards (Clean Architecture · SOLID · Go conventions)

The non-negotiable **how we write code** rules, so every module looks the same, is testable without a
database, and can be graded/reviewed as "proper architecture" rather than a pile of handlers. Inherited
by every feature. Follows SRS §2.1.1's architectural approach (which itself follows Martin, *Clean
Architecture*) — translated from the SRS's Java/Spring framing to idiomatic Go, since Go has no
annotations, no DI container, and no inheritance to reach for.

> If a plan deviates from this doc, it must say why in that feature's `plan.md` §6.

---

## 1. The dependency rule (four layers, same as SRS §2.1.1)

```
┌─ httpapi ──── handlers, request/response DTOs, routing, auth middleware      (depends inward only)
├─ app ───────  use-case services, port interfaces, transaction boundaries
├─ domain ────  entities, value objects, business rules, state machines
└─ mysql ─────  port implementations: raw SQL via database/sql/sqlx, stored-procedure calls
```

- **Dependencies point inward.** `domain` imports nothing but the Go standard library — it has never
  heard of HTTP or SQL. `app` imports `domain` and declares the interfaces (**ports**) it needs from
  the outside world. `mysql` imports `domain` (to return domain types) and implements `app`'s port
  interfaces. `httpapi` imports `app` (calls one service method per handler) and never imports `mysql`
  directly.
- **Handlers are thin:** decode request → validate → call exactly one `app` service method → map the
  result to a response DTO. **No SQL and no business rule ever appears in a `httpapi` file.**
- **Services own the use case** and the transaction boundary (`shared/dbx`'s `WithTx` helper); they
  depend on interfaces (ports), never on a concrete `*sql.DB` or a specific driver type.
- **Domain entities are never serialized directly to the client** — `httpapi` DTOs are separate
  structs with their own `json` tags, mapped explicitly. This is what keeps an internal field (e.g. a
  password hash, an internal-only status) from ever accidentally reaching a response.

**One caveat, stated plainly because it looks like a contradiction otherwise:** SRS REQ-4.6/4.7
requires the stock check-and-decrement to run *inside the database* as a stored procedure, which does
place a piece of business logic outside the Go domain layer. That's deliberate — atomicity under
concurrent checkout is a correctness guarantee the database is better positioned to make (row
locking), and the Go domain layer keeps its own copy of the same rule for validation and unit testing
without a database. **The database is the authority for atomicity; the domain model is the readable,
testable statement of intent.** Same reasoning the SRS gives for the Spring Boot version — it doesn't
change with the language.

## 2. SOLID, in Go terms

Go has no classes or inheritance, so SOLID shows up through **interfaces and composition**, not
subclassing. Treat these as design requirements a reviewer checks for, not style advice.

- **S — Single responsibility.** One service per use case (`OrderService.PlaceOrder`,
  `DeliveryService.EstimateDelivery`, `InventoryService.AdjustStock`). A service method does one
  thing; if a handler needs three unrelated things to happen, that's three service calls, not one
  service method that does three things.
- **O — Open/closed.** Extension happens by adding a new implementation of an existing small
  interface, not by adding another branch to an existing `switch`. Example: adding a third payment
  method means a new `payment.Processor` implementation, not editing the one that already works.
  (Go idiom: "the bigger the interface, the weaker the abstraction" — a `Processor` interface with one
  or two methods is easy to add a new case for; a ten-method interface is not.)
- **L — Liskov substitution.** Every implementation of a port must be swappable without the caller
  knowing — e.g. the Phase 1 `payment.StubProcessor` and any later real gateway both satisfy
  `payment.Processor`, and `OrderService` never type-switches on which one it has. Verified in
  practice: `app`-layer tests run against a hand-written fake implementing the same interface, and
  pass unchanged when the real adapter is substituted at wire-up time.
- **I — Interface segregation.** Ports are declared **where they're consumed** (in `app`), and kept
  narrow: `StockLocker`, `PaymentProcessor`, `DeliveryEstimator`, `ImageStorage` (presigned-URL issue +
  delete, `01_TECH_STACK.md`'s S3/MinIO adapter) are separate one-or-two-method interfaces, not one
  `Repository` god-interface that most callers only use a third of. A Go interface should be defined by
  the consumer's needs, never pre-emptively by the implementer's full capability.
- **D — Dependency inversion.** `app` services take their dependencies as constructor
  arguments typed as interfaces; `mysql` implementations are constructed and wired together in
  `cmd/api/main.go` (the **composition root** — the only place in the codebase that knows every
  concrete type). No package reaches for a global/singleton DB handle; everything is passed in
  explicitly. This is what makes a provider swap (or pulling a module into its own service later)
  cheap — see `04_SCALABILITY.md` §4.

## 3. Dependency injection: manual, not a framework

No `wire`/`fx`/reflection-based DI container. `cmd/api/main.go` constructs every concrete adapter,
passes it into the service that needs it, and passes the wired service into the handler that needs it
— plain Go function calls, fully readable top to bottom, no generated code. Example shape (illustrative,
not the actual file):

```go
func main() {
    cfg := config.Load()
    db := dbx.MustOpen(cfg.DatabaseURL)

    stockRepo := catalogmysql.NewStockRepository(db)
    deliveryRepo := deliverymysql.NewCityRepository(db)

    orderSvc := ordering.NewService(orderingmysql.NewOrderRepository(db), stockRepo, deliveryRepo)
    orderHandler := orderinghttp.NewHandler(orderSvc)

    r := chi.NewRouter()
    r.Use(shared_auth.Middleware(cfg.JWTSecret))
    orderinghttp.RegisterRoutes(r, orderHandler)

    http.ListenAndServe(cfg.Addr, r)
}
```

At project scale (9 modules, Phase 1) this stays readable by hand. If it stops being readable, that's
a signal the module boundaries are wrong before it's a signal to reach for a DI framework.

## 4. Package structure — package-by-feature (module), not package-by-layer

```
internal/<module>/domain/     entities, value objects, state machines — no imports beyond stdlib
internal/<module>/app/        use-case services + port interfaces
internal/<module>/mysql/      port implementations (raw SQL)
internal/<module>/httpapi/    handlers, DTOs, route registration
internal/shared/              auth, httpx (errors/pagination), config, logging, dbx — no business rules
```

- **Package-by-feature, not layer-first at the top level** — keeps a module's code together and its
  boundary clean (`00_OVERVIEW.md` §3): a delivery-estimate rule change touches only
  `internal/delivery/*`, never `internal/catalog`.
- A module may call another module's **exported `app` service**, never reach into its `mysql` or
  `domain` package directly (Go's `internal/` visibility only blocks the outside world; *within*
  `internal/`, this is a convention enforced by review, not the compiler — call it out explicitly in
  `plan.md` §6 when a feature needs cross-module data, e.g. reporting reading catalog+order data
  through its own views rather than through catalog's/ordering's Go packages).
- Go's own `internal/` directory rule already gives one enforcement point for free: nothing outside
  this repository can import any of it.

## 5. Coding consistency (reuse these patterns; don't invent variants)

- **Identity** from the JWT principal in request context (`shared/auth`), never a body field.
- **Errors:** every `app`-layer error is a typed sentinel or wrapped error (`errors.Is`/`errors.As`
  friendly) that `httpapi`'s central error mapper turns into the one JSON error shape (`01` §3). No
  raw Go `error.Error()` string reaches the client.
- **Money:** always MySQL `DECIMAL`; in Go, a small `Money` value type (backed by a fixed-point
  integer or `shopspring/decimal`) — never `float64` for a price or total, anywhere.
- **Transactions:** one helper (`shared/dbx.WithTx`) wraps `BEGIN`/`COMMIT`/`ROLLBACK`; a service
  never calls `db.Begin()` directly. This is also where the checkout stored-procedure call lives —
  inside the same transaction boundary as everything else in that use case (REQ-4.8: no DB transaction
  held open across a call to an external payment provider — authorize *before* opening the transaction).
- **Migrations:** only via `golang-migrate` numbered files (`01` §3); no hand-edited schema, ever, in
  any environment.
- **Idempotency:** the checkout endpoint's idempotency key (REQ-4.9) is checked inside the same
  transaction as order creation — a unique index on `order.idempotency_key`, not just an
  application-level in-memory check.

## 6. Style & formatting (auto-enforced, zero bikeshedding)

- **Backend (Go):** `gofmt`/`goimports` (non-negotiable — Go tooling assumes it), `go vet`,
  `staticcheck`. Constructor functions (`NewXxx`) return interfaces where callers only need the
  interface, concrete structs where they need the concrete type (composition root only). No naked
  `panic` outside `main()`/truly unrecoverable startup failure. Errors are always checked, never
  discarded with `_`.
- **Frontend (TS/Next.js):** ESLint + Prettier, TypeScript strict, no `any`, no
  `dangerouslySetInnerHTML`, one central API client, feature-folder structure.
- Formatting/lint failures block the build once CI exists (`05` §4).

## 7. Definition of done (engineering standards)

- Dependency rule respected: `domain` imports nothing but stdlib; `httpapi` never imports `mysql`.
- Handlers are thin; no SQL or business rule inside `httpapi`.
- Every port is a small, consumer-defined interface; every adapter is swappable in tests via a fake.
- `cmd/api/main.go` is the only place concrete adapters are constructed.
- `gofmt`/`go vet`/`staticcheck` clean; ESLint/`tsc --noEmit` clean.
- New code reuses the shared patterns (§5) — auth-from-context, error shape, `Money`, `WithTx`,
  idempotency-by-unique-index — rather than inventing a new one per feature.
