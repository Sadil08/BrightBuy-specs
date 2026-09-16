# 01 — Tech Stack & Conventions

Baseline every feature inherits. `plan.md` files reference this; they do not restate it. Fixed by
SRS §2.4/2.5/3.3 where the SRS is explicit (Go replaces the SRS's original Spring Boot placeholder —
see note below); left to this doc where the SRS is silent.

> **Deviation note:** SRS v1.1 §3.3 names Spring Boot/JPA/Hibernate as the backend stack. This
> project builds the backend in **Go** instead, by requirement of the person building it (learning
> goal) and because raw SQL — required by the DBMS module — is more natural without an ORM fighting
> it. Everything the SRS mandates *behaviorally* (REST/JSON, MySQL 8/InnoDB, stateless backend, JWT,
> BCrypt, stored-procedure-based atomic stock decrement, Flyway-equivalent versioned migrations)
> still holds; only the framework choice changes. This substitution is logged as a closed item in
> `09_ISSUES_REGISTER.md`.

---

## 1. Stack

| Layer | Choice | Notes |
|---|---|---|
| Backend language | **Go 1.23+** | Compiled, statically typed, small learning surface — good fit for "learn while building." |
| HTTP routing | **stdlib `net/http` + `chi` (`go-chi/chi/v5`)** | `chi` adds routing/middleware sugar over stdlib `http.Handler` without hiding it — you're still writing real Go `net/http`, not framework magic. Rejected Gin/Echo: both wrap the request in their own context type, which obscures the stdlib you're trying to learn. |
| SQL access | **`database/sql` + `sqlx`** | `sqlx` is a thin convenience layer (struct scanning, named params) over `database/sql` — **not an ORM**, no query builder, no lazy loading. All SQL is hand-written strings. Driver: `go-sql-driver/mysql`. |
| Migrations | **`golang-migrate/migrate`** | Versioned, forward-only `.sql` up/down files — the Go-world equivalent of the SRS's Flyway requirement (§2.5, §6.1). |
| Validation | **`go-playground/validator/v10`** | Struct-tag validation on request DTOs. |
| Auth | **`golang-jwt/jwt/v5`** (access+refresh tokens) + **`golang.org/x/crypto/bcrypt`** (password hashing) | Matches SRS §3.3/§5.3 (JWT, BCrypt) exactly; just a different library ecosystem. |
| Config | **`os.Getenv` + a typed `Config` struct loaded once at startup** (`github.com/joho/godotenv` for local `.env` only, never in prod) | 12-factor: config from environment, never from committed source (SRS §2.5, §6.2). |
| Logging | **stdlib `log/slog`**, JSON handler in prod, text handler in dev | Structured, one request-correlation-id field per line (SRS §6.2). |
| API docs | **Hand-authored OpenAPI 3.0** at `specs/openapi/openapi.yaml` | The "swag" file — the API contract, written *before* the handler that implements it. Optionally regenerated/cross-checked later with `swaggo/swag` doc-comments once code exists; the hand-authored file is authoritative during spec phase. |
| Database | **MySQL 8.0+, InnoDB** | Fixed by SRS §2.4/§6.1. No RLS (MySQL has none) — see `02_SECURITY_BASELINE.md` §6 for the compensating control. |
| Image storage | **S3-compatible object storage**, via `aws-sdk-go-v2/service/s3` | Resolved OQ-3. **MinIO** (self-hosted, S3-API-compatible, free) in local dev, added as a service in `brightbuy-backend`'s Docker Compose; a real free-tier bucket (AWS S3 or Cloudflare R2) in staging/prod. Same Go client code against both — only `S3_ENDPOINT_URL` differs by environment. Behind an `ImageStorage` port (`06_ENGINEERING_STANDARDS.md` §2) so the adapter is swappable without touching `catalog`. Upload security: `02_SECURITY_BASELINE.md` §4. |
| Frontend | **Next.js 14+ (App Router), React 18+, TypeScript strict** | Storefront + admin console in one app, route-group-segregated, per SRS §3.3. |
| Frontend styling | **Tailwind CSS** | Utility-first, fast to learn, matches SRS §3.3. |
| Tests (BE) | **stdlib `testing` + `testify`** (assertions) + **`testcontainers-go`** (real MySQL 8 in Docker for integration tests) | No mocked database for anything that touches a trigger/procedure — see `05_TESTING_STRATEGY.md`. |
| Tests (FE) | **Vitest + React Testing Library** (unit/component) + **Playwright** (E2E) | |
| Containerization | **Docker + Docker Compose** for local dev | SRS §2.4, §6.2. `brightbuy-backend`'s compose file owns MySQL + backend (that's the pairing that matters — a developer working backend-only never needs the frontend running); `brightbuy-frontend`'s compose file runs the frontend alone, pointed at the backend's URL via env config. Both join a shared external Docker network so `docker compose up` in each repo is enough — no single command spans both repos, matching the two-repo split (OQ-2). |

> **Why not a lighter/heavier stack?** This is a learning project as much as a deliverable: `chi` +
> `database/sql`/`sqlx` keeps every layer visible (no annotation magic, no generated code you can't
> read) while still being what real Go backends look like in industry. If later the DBMS grading
> pressure eases and velocity matters more than learning stdlib idioms, `sqlc` (generates typed Go
> from SQL you still write by hand) is the natural next step — noted, not adopted yet.

---

## 2. Repository layout

**Two repositories** (resolved OQ-2, `10_BA_AUDIT_OPEN_QUESTIONS.md`): `brightbuy-backend` and
`brightbuy-frontend`. `specs/` (this folder, including `openapi.yaml`) is the shared contract both
build against — it lives outside both, wherever the SRS/planning docs live, and is not duplicated
into either repo. A backend contract change and its frontend consumer are **two separate PRs in two
separate repos**; until there's CI to cross-check it automatically, each PR description names the
`openapi.yaml` path/commit it implements against, by convention.

```
brightbuy-backend/
├── cmd/
│   └── api/main.go            composition root: load config, open DB pool, wire dependencies, start server
├── internal/
│   ├── catalog/                one folder per module (00_OVERVIEW.md §3)
│   │   ├── domain/             entities + business rules; imports nothing but stdlib
│   │   ├── app/                use-case services + port interfaces; imports domain only
│   │   ├── mysql/               port implementations (raw SQL via sqlx); imports domain + app (for the port interface)
│   │   └── httpapi/             handlers + request/response DTOs + route registration; imports app only
│   ├── identity/  cart/  ordering/  inventory/  delivery/  payment/  reporting/   (same 4-folder shape)
│   └── shared/
│       ├── auth/                JWT issue/verify, bcrypt, middleware
│       ├── httpx/                error shape, response helpers, pagination helpers
│       ├── config/               env loading
│       ├── logging/              slog setup
│       └── dbx/                  sqlx connection setup, transaction helper
├── db/
│   ├── migrations/               golang-migrate `.up.sql`/`.down.sql`, one numbered file per change
│   └── seed/                     seed script — ≥40 products / ≥10 categories (SRS §6.1)
├── go.mod / go.sum
└── Dockerfile

brightbuy-frontend/
├── app/
│   ├── (storefront)/             customer-facing routes
│   └── (admin)/                  staff/manager console routes
├── components/                   shared UI primitives
├── lib/
│   └── api-client/                one generated/typed client wrapping every backend call
└── package.json
```

`internal/` is a real Go compiler boundary — nothing outside this module can import it, which is a
free enforcement of "don't let anything external depend on our internals" (ties to `06` DIP).

## 3. Conventions

- **API:** REST, JSON, plural lowercase nouns, versioned from day one (`/api/v1/products`), per SRS
  §3.5. Money as MySQL `DECIMAL`, never float, surfaced over the API as a JSON string
  (`"total_amount": "129.99"`) to avoid client-side float rounding — never a JSON number for currency.
  Timestamps ISO-8601 UTC.
- **DTOs:** request/response structs are defined in each module's `httpapi` package and are never the
  same struct as the `domain` entity — a domain field never accidentally leaks over the wire (e.g. a
  password hash) because there is no path for it to.
- **Identity from the token, never the body.** The JWT's subject claim is the only source of "who is
  making this request"; a `customer_id` field in a request body is ignored/rejected.
- **Errors:** one shape everywhere — `{"code": "STOCK_EXCEEDED", "message": "...", "fields": [...]}`.
  No Go error strings, no stack traces, reach the client (SRS §3.5, §5.3).
- **Pagination:** `page`/`size` query params, response includes total count (SRS §3.5, REQ-1.6).
- **Migrations:** every schema/procedure/trigger/view change is a new numbered `golang-migrate` file
  pair; no hand-edited schema in any environment (SRS §2.5, §6.1).
- **Frontend:** TypeScript strict, no `any`; one central typed API client so a backend contract change
  is a one-file fix; server components for read-only catalog pages where SSR helps; no secrets in the
  client bundle.

## 4. Environments

`local` (Docker Compose MySQL), `staging`, `production` — differ only in configuration (SRS §6.2).
Secrets via environment variables / a secret manager, never committed; `.env` is git-ignored.
