# BrightBuy — Spec System & Conventions

This folder is the **single source of truth** for BrightBuy (Retail Inventory and Online Order
Management System, Phase 1 — University of Moratuwa DBMS module, Group 11). It is spec-driven:
every feature flows **spec → plan → tasks → code → tests**, and every requirement is traceable
end-to-end by a stable ID back to the SRS (`BrightBuy_SRS_v1.1`).

> **Two runtimes:** a **Go** backend and a **Next.js** frontend (storefront + admin console), over
> **MySQL 8 / InnoDB**, with the SQL hand-written — **no ORM**. Specs describe both; tasks are
> tagged `[BE]`/`[FE]`/`[DB]`/`[INFRA]` so ownership is never ambiguous.
>
> This spec system mirrors the structure of a prior project (`ScanLanka`, Spring Boot + Postgres)
> but is **not a copy of its content** — the tech stack, security model (no RLS in MySQL), and SQL
> standards below are BrightBuy-specific and follow the SRS, which is the contractual source of
> requirements. Where this spec system and the SRS conflict, the SRS wins; open the conflict as an
> item in `10_BA_AUDIT_OPEN_QUESTIONS.md` rather than silently picking one.

---

## 1. Layout

```
specs/
├─ global/                          ← cross-cutting baseline EVERY feature inherits
│  ├─ 00_OVERVIEW.md                business context, actors/roles, module map, glossary
│  ├─ 01_TECH_STACK.md              Go/Next.js/MySQL stack, versions, repo layout, conventions
│  ├─ 02_SECURITY_BASELINE.md       authn/authz model, OWASP controls, secrets, MySQL defense-in-depth
│  ├─ 03_NFR_BASELINE.md            performance/availability/reliability budgets inherited by all
│  ├─ 04_SCALABILITY.md             statelessness, caching, connection pooling, growth path
│  ├─ 05_TESTING_STRATEGY.md        test pyramid, testcontainers-go, CI gates
│  ├─ 06_ENGINEERING_STANDARDS.md   clean architecture + SOLID in Go, package layout, lint rules
│  ├─ 07_SQL_DATABASE_STANDARDS.md  raw SQL policy, triggers/procedures/functions/views, indexing
│  ├─ 08_THREAT_MODEL.md            attack surface, threats (T-*), controls, OWASP Top Ten map
│  ├─ 09_ISSUES_REGISTER.md         running issues register + verdict (audit log)
│  ├─ 10_BA_AUDIT_OPEN_QUESTIONS.md SRS Appendix C TBDs + gaps, tracked to a resolution
│  ├─ 11_BUILD_PLAN.md              build sequence, phases, milestones, which specs to follow
│  └─ 12_DEVOPS_CICD.md             Dockerfiles, Compose, CI pipeline, deploy/rollback
│
├─ openapi/
│  └─ openapi.yaml                  the API contract ("swag" file) — hand-authored OpenAPI 3.0
│
└─ features/
   ├─ _TEMPLATE/                    ← clone this for every new feature
   │  ├─ spec.md                    WHAT & WHY
   │  ├─ plan.md                    HOW
   │  └─ tasks.md                   DO IT
   └─ NN-feature-slug/              e.g. 01-catalog, 02-auth, 04-checkout-orders
      ├─ spec.md
      ├─ plan.md
      └─ tasks.md
```

**Rule of thumb:** *global = inherited baseline; feature = the delta.* A feature spec restates a
security/NFR/SQL rule **only** when it adds to or overrides the baseline — otherwise it links to
global. This avoids duplication and drift, and keeps each feature file short enough to actually read.

---

## 2. The three feature files

| File | Question | Owner hat | Contains | Must NOT contain |
|---|---|---|---|---|
| `spec.md` | **What & why** | Business Analyst | User stories, FR, NFR-delta, Security-delta, Acceptance Criteria, SRS `REQ-*` trace | Tech choices, code, SQL |
| `plan.md` | **How** | Architect | Package layout, DB schema/migrations, triggers/procedures used, API contract (linked to `openapi.yaml`), code snippets | New requirements |
| `tasks.md` | **Do it** | Tech lead / QA | Ordered `[BE]/[FE]/[DB]/[INFRA]` tasks, per-task test mechanism, DoD | Design rationale |

A feature is **ready to code** only when all three exist and every `FR-*` in `spec.md` is addressed
in `plan.md` and has at least one task + test in `tasks.md`.

---

## 3. ID scheme (the traceability spine)

Every requirement gets a stable ID that threads through all artifacts, the SRS, and into test names.

| Prefix | Meaning | Example |
|---|---|---|
| `REQ-<n>.<m>` | Requirement inherited verbatim from the SRS (Section 4) | `REQ-4.6` |
| `US-<feat>-N` | User story | `US-CART-1` |
| `FR-<feat>-N` | Functional requirement (feature-local, maps to one or more `REQ-*`) | `FR-CART-3` |
| `NFR-<feat>-N` | Non-functional requirement (feature delta) | `NFR-CHECKOUT-1` |
| `SEC-<feat>-N` | Security requirement (feature delta) | `SEC-PAYMENT-2` |
| `AC-<feat>-N` | Acceptance criterion (Given/When/Then) | `AC-CART-1` |

- `<feat>` is the feature slug in SCREAMING-KEBAB without the number (e.g. `CART`, `CHECKOUT`).
- IDs are **append-only**. Never renumber; mark removed ones `~~FR-CART-3 (dropped)~~`.
- Every `FR-*` cites the `REQ-*` it implements. A requirement in the SRS with no `FR-*` anywhere is a
  gap — surface it in `10_BA_AUDIT_OPEN_QUESTIONS.md`.
- Cross-cutting global issues keep their register IDs from `global/09_ISSUES_REGISTER.md`. A feature
  that closes one links it: `Closes SEC-STOCK-1`.

**Traceability requirement:** `FR-CART-3` appears in `spec.md` (defined, cites `REQ-*`) → `plan.md`
(designed) → `tasks.md` (task + test `fr_cart_3_*`). A reviewer — or the module instructor — can grep
an ID across the three files and the SRS and find a hit in each.

---

## 4. Tagging tasks by runtime

In `tasks.md`, every task is prefixed with its runtime:

- `[BE]` Go backend (handlers, services, domain logic)
- `[FE]` Next.js frontend (storefront and/or admin console)
- `[DB]` schema, migrations, stored procedures, functions, triggers, views — the DBMS-module-graded
  artifacts. Kept distinct from `[INFRA]` because these are core deliverables, not plumbing.
- `[INFRA]` CI, containerization, env/secrets, deploy config
- `[BE+FE]` a contract change requiring both sides in lockstep (state the contract explicitly,
  point at the relevant `openapi.yaml` path)

---

## 5. Lifecycle of a feature

1. **Draft `spec.md`** — stories + FR/NFR/SEC + acceptance criteria, each `FR-*` traced to a SRS `REQ-*`.
2. **Draft `plan.md`** — design + API contract (added to `openapi/openapi.yaml`) + schema/procedures.
3. **Draft `tasks.md`** — granular tasks + tests, tagged `[BE]/[FE]/[DB]/[INFRA]`.
4. **Build** — implement tasks; each lands with its failing-first test (see `global/05`).
5. **Verify** — all ACs green; update `09_ISSUES_REGISTER.md` if the feature closed a register item.

Only steps 1–3 happen before any code is written — that's the point of this folder.

---

## 6. Status & severity tags (use consistently everywhere)

✅ done & tested · ⚠️ partial · ❌ missing/draft · 🔴 P0 · 🟠 P1 · 🟡 P2 · 🟢 P3
