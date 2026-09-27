# Session Progress - 2026-09-27


- Verified the backend health endpoint:
  - `GET /healthz` returns `200 OK` and `ok`.
  - `GET /readyz` returns `200 OK` and `ok`, confirming MySQL connectivity.
- Verified the frontend health route reaches the backend and returns `backend: "reachable"`.
- Resolved the local Docker setup issues:
  - Started Docker Desktop successfully.
  - Used the backend Compose services from `BrightBuy-backend`, not the frontend directory.
  - Worked around the host port `3307` conflict.
- Completed catalog database baseline work:
  - T1: created and applied `0001_catalog_schema` with seven catalog tables and the product FULLTEXT index.
  - Verified the migration with down/up execution.
  - T2: created and verified `fn_is_variant_in_stock`.
  - Created and executed the deterministic catalog seed at `db/seed/catalog.sql`.

## Current Status

The project has moved from Phase 0 setup into Phase 1, feature `01-catalog`.

The catalog schema, stock function, and seed are in place. The seed count and repeat-run checks should still be verified explicitly if not already recorded.
## Next Task

T4: create and test the catalog domain entities in:

`BrightBuy-backend/internal/catalog/domain/`

The domain should define `Category`, `Product`, `Variant`, `StockStatus`, and the stock-status conversion test without importing HTTP, SQL, or JSON packages.

## Useful Commands

Run backend checks from `BrightBuy-backend`:

```powershell
go test ./...
go vet ./...
```

Verify seed counts:

```powershell
docker compose exec -T mysql mysql -ubrightbuy_app -pdevapppass brightbuy -e "SELECT COUNT(*) AS categories FROM category; SELECT COUNT(*) AS products FROM product; SELECT COUNT(*) AS variants FROM product_variant;"
```
