# 12 — Containerization & CI/CD

Inherited by both repositories (`brightbuy-backend`, `brightbuy-frontend` — resolved OQ-2). Fixed by
SRS §6.2 (container image, health endpoint, stateless, structured logs, migrations-before-traffic,
CI that refuses to publish on a failing test) and this project's own goal of being a real, deployable
industry-shaped pipeline, not just `go run` on a laptop.

---

## 1. Containerization

### 1.1 Backend — `brightbuy-backend/Dockerfile`

Multi-stage: a full Go toolchain builds a static binary; the runtime image ships only that binary, no
compiler, no shell if avoidable — smaller attack surface and smaller image.

```dockerfile
# --- build stage ---
FROM golang:1.23-alpine AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /out/api ./cmd/api

# --- runtime stage ---
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /out/api /api
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/api"]
```

- **Distroless, non-root, `CGO_ENABLED=0`:** no shell, no package manager, no root user in the final
  image — closes off a whole class of "shell into the container and pivot" post-compromise moves
  (`02_SECURITY_BASELINE.md` §6.1's least-privilege principle applied to the runtime, not just the DB
  role).
- **Health endpoint:** `GET /healthz` (liveness: process is up) and `GET /readyz` (readiness: DB pool
  can reach MySQL) — SRS §6.2's "expose a health endpoint" requirement, and what any orchestrator
  (Compose healthcheck, or a real scheduler later) polls before routing traffic to an instance.
- **Migrations run as a separate step, not inside this image's `ENTRYPOINT`** — see §3.3. Baking
  `golang-migrate` into the app image would mean every replica tries to migrate on boot, which is
  exactly the race SRS §6.2 ("migrations run... before the new version begins serving traffic") is
  worried about.

### 1.2 Frontend — `brightbuy-frontend/Dockerfile`

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build   # next.config.js: output: 'standalone'

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup -S app && adduser -S app -G app
COPY --from=builder --chown=app:app /app/.next/standalone ./
COPY --from=builder --chown=app:app /app/.next/static ./.next/static
COPY --from=builder --chown=app:app /app/public ./public
USER app
EXPOSE 3000
CMD ["node", "server.js"]
```

- Next.js `output: 'standalone'` produces a minimal self-contained server — the runtime image doesn't
  need the full `node_modules` tree or the Next.js CLI.
- No secret is ever baked into the frontend image at build time — anything the browser needs
  (a public API base URL) is a build-time **public** env var (`NEXT_PUBLIC_*`), never a credential;
  anything sensitive stays server-side only (`01_TECH_STACK.md` §3).

### 1.3 Local development — Docker Compose (one file per repo, resolved OQ-2)

`brightbuy-backend/docker-compose.yml`:
```yaml
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: brightbuy
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
    ports: ["3306:3306"]
    volumes: ["mysql-data:/var/lib/mysql"]
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 5s
      retries: 10

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: ${S3_ACCESS_KEY}
      MINIO_ROOT_PASSWORD: ${S3_SECRET_KEY}
    ports: ["9000:9000", "9001:9001"]     # 01_TECH_STACK.md's local S3-compatible store, resolved OQ-3
    volumes: ["minio-data:/data"]

  api:
    build: .
    depends_on:
      mysql: { condition: service_healthy }
    environment:
      DB_DSN: "app:${DB_APP_PASSWORD}@tcp(mysql:3306)/brightbuy?parseTime=true"
      S3_ENDPOINT_URL: "http://minio:9000"
      JWT_SIGNING_KEY: ${JWT_SIGNING_KEY}
    ports: ["8080:8080"]
    networks:
      default: {}
      brightbuy-shared: {}                # joined by brightbuy-frontend's compose, see below

volumes: { mysql-data: {}, minio-data: {} }
networks:
  brightbuy-shared:
    name: brightbuy-shared
    external: true    # created once: `docker network create brightbuy-shared`
```

`brightbuy-frontend/docker-compose.yml`:
```yaml
services:
  web:
    build: .
    environment:
      BACKEND_URL: "http://api:8080"      # reaches the backend repo's `api` service over the shared network
    ports: ["3000:3000"]
    networks: { brightbuy-shared: {} }

networks:
  brightbuy-shared: { name: brightbuy-shared, external: true }
```

A developer runs `docker network create brightbuy-shared` once, then `docker compose up` in each repo
independently — matches the two-repo split (`01_TECH_STACK.md` §2) without needing a single command
that spans both.

## 2. CI (per repository, both pipelines shaped the same way)

**Assumption:** GitHub Actions, since both repos are assumed to live on GitHub (matches the
`gitleaks`/CI conventions already referenced elsewhere in these specs). If BrightBuy ends up hosted
elsewhere, the stages below are the actual contract — just ported to that platform's syntax.

### 2.1 `brightbuy-backend/.github/workflows/ci.yml` — stage sequence

1. **Checkout + setup Go** (pinned version matching `go.mod`).
2. **Static checks:** `gofmt -l .` (fails if anything is unformatted), `go vet ./...`, `staticcheck
   ./...` — `06_ENGINEERING_STANDARDS.md` §6.
3. **Secret scan:** `gitleaks detect` — `02_SECURITY_BASELINE.md` §5. Required check; blocks merge.
4. **Unit tests:** `go test ./... -short` (excludes anything tagged `integration`) — fast, no DB.
5. **Integration tests:** spin up MySQL 8 as a GitHub Actions **service container** (or let
   `testcontainers-go` manage its own — either works; a service container is simpler in CI since the
   Docker-in-Docker layer GitHub Actions runners provide handles it natively), run migrations, then
   `go test ./... -run Integration -tags=integration`. This is where every trigger/procedure/view test
   from `05_TESTING_STRATEGY.md` §2 actually executes — **a red integration suite blocks the build**,
   full stop, since this is the DBMS-module-graded logic.
6. **Build the Docker image** (§1.1), tagged `ghcr.io/<org>/brightbuy-backend:<git-sha>` and, on the
   default branch only, also `:latest`.
7. **On the default branch only:** push the image to GHCR (GitHub Container Registry — free for public
   repos, and free at low volume for private ones, matching the "freely available" constraint already
   used for image storage).

A PR that fails any of steps 2–5 is blocked from merging (branch protection: required status checks).
Step 6 building successfully is itself a check — an image that doesn't build doesn't merge either.

### 2.2 `brightbuy-frontend/.github/workflows/ci.yml` — stage sequence

1. Checkout + setup Node (pinned to match `.nvmrc`/`package.json` engines).
2. `npm ci` (never `npm install` in CI — exact lockfile versions).
3. `eslint .`, `tsc --noEmit` — `06_ENGINEERING_STANDARDS.md` §6.
4. `vitest run` (component/unit).
5. `next build` (also the correctness check that the app actually compiles for production).
6. Playwright E2E — **against a docker-compose'd backend + seeded MySQL**, spun up as a CI step before
   this one runs. Given the runtime cost, this can be scoped to the default branch and nightly runs
   rather than every PR, if PR feedback speed becomes a problem — start with "every PR," relax only if
   it's actually too slow.
7. Build the Docker image (§1.2), same tagging scheme as §2.1, pushed to GHCR on the default branch.

### 2.3 Keeping the two pipelines honest against the shared contract

Since the two repos are independent (resolved OQ-2), nothing stops them drifting from
`specs/openapi/openapi.yaml` except discipline — until there's a generated client to make drift a
compile error. Two cheap steps close most of the gap without that investment yet:

- Both CI workflows lint `openapi.yaml` itself (`redocly lint` or `spectral lint`, run from wherever
  `specs/` is checked out — a git submodule, or the file mirrored into each repo via a small sync
  script) as a required check — a broken contract file fails both builds, not just whichever repo
  happened to touch it last.
- Every PR that changes a handler's request/response shape **must** touch the corresponding
  `openapi.yaml` path in the same PR (code-review-enforced, not yet tool-enforced) — `06 §7`'s
  Definition of Done already says this per-feature; CI catching it mechanically (e.g. a generated
  Go client from `openapi.yaml` that fails to compile against a changed handler) is the natural next
  step once there's enough traffic through this pipeline to justify building it.

## 3. CD (deployment)

**Hosting provider is still open — SRS TBD-7 (`10_BA_AUDIT_OPEN_QUESTIONS.md`).** The stages below are
the actual, provider-agnostic contract; §3.4 gives a concrete recommendation for *this* project's scale.

### 3.1 Promotion, not rebuild

The exact image tagged and pushed by CI (§2.1/§2.2 step 6/7) is what gets deployed — staging and
production run the **same image**, promoted, never rebuilt from source a second time (SRS §6.2: "the
same image is promoted from staging to production without a rebuild"). This is what makes "it passed
CI" actually mean something about what's running in prod.

### 3.2 Environments

`local` (Docker Compose, §1.3) → `staging` → `production`. Only configuration differs
(`01_TECH_STACK.md` §4) — connection strings, secrets, the S3 bucket/MinIO endpoint. Promotion from
staging to production is a manual approval gate (a human clicks "deploy to prod"), not automatic on
every merge — appropriate for a small team/solo project where a bad deploy is expensive to notice fast.

### 3.3 Migrations before traffic

A dedicated deploy step — not baked into the app container's `ENTRYPOINT` (§1.1) — runs
`golang-migrate -database $DB_DSN -path db/migrations up` **before** the new application version
starts receiving traffic (SRS §6.2). In a simple single-VM deploy (§3.4) this is a step in the deploy
script; on a platform with a "release phase" concept (Railway, Render, Heroku-style), it's that
platform's release-phase hook.

### 3.4 Recommendation for this project's scale

Given "freely available" is already the operating constraint (S3 free tier, GHCR free tier): a single
small VM (or a platform's free/hobby tier — Railway and Render both offer one, and both support
"deploy this container image, run a release-phase command first") running:
- the backend container,
- a managed or self-hosted MySQL 8 (a platform-managed MySQL add-on is worth the small cost over
  self-hosting it on the same VM, once there's a real deployment target chosen — self-hosting is fine
  to start and matches "freely available"),
- the frontend container (or the frontend deployed to Vercel's free tier instead, which is a very
  natural fit for a Next.js app specifically and decouples frontend deploys from backend ones entirely).

This is a recommendation to resolve TBD-7 with, not a decision made on BrightBuy's behalf — confirm
before Phase 5 (`11_BUILD_PLAN.md`) executes.

### 3.5 Rollback

Because deploys are tagged images (§3.1), rollback is "point the deploy at the previous git-sha tag,"
not a rebuild or a `git revert` race — this is the direct payoff of tag-per-commit image builds.

## 4. Secrets in CI/CD

Every value in §1.3's `${...}` placeholders — DB passwords, `JWT_SIGNING_KEY`, S3/MinIO keys, the
registry push credential — lives in GitHub Actions **encrypted secrets** (repo or environment-scoped),
never in the workflow YAML, never in a committed `.env` (`02_SECURITY_BASELINE.md` §5). Production and
staging use **separate** secret values, not the same credentials reused across environments — a
staging leak should never compromise production.

## 5. Definition of done (this doc)

- Both repos build a working container image locally (`docker build .`) before any CI is wired up.
- CI blocks merge on: format/lint, unit tests, integration tests (backend), E2E (frontend, at least on
  the default branch), and a successful image build.
- No secret appears in any workflow file, Dockerfile, or committed source — `gitleaks` enforces this
  mechanically, not just by review.
- A deploy is: pull the CI-built tag → run migrations → start the new version → health-check before
  routing traffic → old version terminated only after the new one is ready (this is what makes it a
  zero-downtime deploy, whichever hosting choice resolves TBD-7).
