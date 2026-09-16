# 04 — Scalability

Inherited by every feature. Phase 1 targets a modest concurrent load (SRS §5.1: ~50 browsing / ~10
checking out); this doc is about not painting the design into a corner, not about over-building now.

---

## 1. Statelessness

The backend holds **no session state in memory**. Everything needed to serve a request is either in
the verified JWT (identity, role) or the database. This means:

- More than one backend instance can run behind a load balancer with zero sticky-session config
  (SRS §6.2).
- A backend instance can be killed and replaced mid-traffic without losing anyone's session.
- The only per-instance state is the DB connection pool, which is cheap to re-establish.

## 2. Caching

- **HTTP-level:** catalog read endpoints (products, categories) carry cache headers — that data
  changes on staff action, not per-request (`03` §1).
- **No application-level cache (Redis etc.) in Phase 1** — the dataset is small (≥40 products) and
  query latency at that scale doesn't need it. Documented as the first thing to add if catalog size or
  read volume grows past what indexes alone handle — see §4.

## 3. Connection pooling

`database/sql`'s built-in pool (`SetMaxOpenConns`, `SetMaxIdleConns`, `SetConnMaxLifetime`), sized so
concurrent checkouts don't queue on connection acquisition (`03` §1). MySQL's own `max_connections`
must be set comfortably above `backend_instances × MaxOpenConns`.

## 4. Growth path (documented now, not built now)

| If this grows... | The move is... | Why it's cheap later |
|---|---|---|
| Read traffic (catalog browse) | Add a MySQL read replica; `reporting` and catalog-read queries point at it | Reporting already goes through views and is already read-only — routing it to a replica is a connection-string change, not a code change |
| Backend CPU/concurrency | Run more backend instances behind the load balancer | Already stateless (§1) |
| Product catalog size (10k+ SKUs) | Add full-text search (MySQL `FULLTEXT` index, or an external search index) | `catalog`'s search is behind its own service interface (`06` §1) — swapping the implementation doesn't touch callers |
| Warehouses (multi-warehouse routing) | Out of scope for Phase 1 by SRS §1.1/§2.7 — would need a `warehouse_id` on stock and a routing rule | Stock already lives on `product_variant`, not scattered — adding a dimension is a schema migration, not a redesign |
| A module needing independent scaling/deploy | Extract that module's package into its own service | Module boundary (`00` §3) already forbids cross-module table access — extraction is a network hop, not a rewrite |

## 5. What's explicitly *not* done in Phase 1 (and why that's fine)

- No message queue / async processing — every use case in Phase 1 (checkout, stock adjustment) is a
  single synchronous transaction that must succeed or fail atomically; there's no background job yet.
- No CDN for product images — Phase 1's image strategy is itself an open question (OQ-3,
  `10_BA_AUDIT_OPEN_QUESTIONS.md`); resolve that before optimizing its delivery.
- No horizontal database sharding — a single MySQL 8 instance comfortably serves this dataset and
  load; sharding solves a problem this project doesn't have yet.
