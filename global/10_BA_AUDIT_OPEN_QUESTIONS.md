# 10 — BA Audit & Open Questions

Every open question that blocks a feature `spec.md` from being written accurately lives here first.
`TBD-1..7` are carried verbatim from SRS Appendix C (the requirements owner's own open-question list);
`OQ-*` are gaps found while translating the SRS into this spec system. **A feature spec should either
resolve the relevant item here with an explicit assumption (stated in that feature's `spec.md` §2, same
as the SRS's own §2.7 pattern) or block on it — never silently guess.**

---

## From SRS Appendix C (business decisions BrightBuy itself needs to make)

| ID | Open item | Blocks | Working assumption until resolved |
|---|---|---|---|
| TBD-1 | Standard Delivery flat fee amount, and the Texas sales tax rate (and whether tax is itemized separately at checkout) | `04-checkout-orders` spec | Both held as **configuration values** (an app-config table or env-driven constant, not a hard-coded literal) so a decision later is a data change, not a code change. Placeholder for development: fee = $9.99, tax = 8.25% (a plausible TX combined rate) — must be replaced with a real BrightBuy-confirmed value before this is "done," not just "working." |
| TBD-2 | The definitive list of Main Cities beyond the six metros assumed (Houston, San Antonio, Dallas, Austin, Fort Worth, El Paso) | `05-delivery-estimation` spec | Ship the six as `city` seed data with `classification='Main'`, every other Texas city `'Other'`. Table-driven per SRS §2.7, so growing the list is a data change (`INSERT`/`UPDATE` on `city`), never a code change. |
| TBD-3 | Pickup readiness timing when an ordered item is out of stock (REQ-6.6 only defines the in-stock case) | `05-delivery-estimation` spec | Working assumption: Store Pickup with an out-of-stock line uses the **same +3-day rule as Standard Delivery** (REQ-6.3) rather than "same business day." Flag this explicitly as an assumption in that feature's spec, not a confirmed rule. |
| TBD-4 | Whether customer-initiated returns/cancellations are supported (even in a later phase), and refund handling | Out of scope for Phase 1 features; affects only whether the state model leaves room for it | Already accommodated: `Cancelled` is a valid terminal state reachable by **staff** (REQ-8.6); a customer-initiated cancel/return endpoint is explicitly not built in Phase 1. No feature spec should add one without this being resolved first. |
| TBD-5 | Detailed interface wireframes, as a separate UI specification | `frontend` feature specs' UI sections | Feature specs describe screens/flows in prose + the SRS §3.1 interface list; pixel-level wireframes are out of scope for this spec system unless produced separately. |
| TBD-6 | Backup schedule and retention policy | Deployment readiness, not any single feature | Deferred to `11_BUILD_PLAN.md`'s deployment phase; not a Phase-1-feature blocker. |
| TBD-7 | Hosting provider/plan choice, which determines managed-DB availability and backend instance count | Deployment readiness | Deferred, same as TBD-6. Local dev (`Docker Compose`) is unaffected either way. |

## Gaps found while writing this spec system (`OQ-*`)

| ID | Question | Why it matters | Recommendation |
|---|---|---|---|
| OQ-1 | SRS §2.3 lists "Warehouse/Inventory Staff" and "Store/Order Manager" as separate actors, but the ER diagram's `ROLE` table has only `CUSTOMER, STAFF, MANAGER, ADMIN` — no functional split is defined anywhere in SRS §4. Are they the same account/role in Phase 1? | Determines whether `identity`/`02_SECURITY_BASELINE.md` needs a 5th role or a permission-sub-scheme within `STAFF` | ✅ **Resolved (2026-09-02): separate roles, plus a full dynamic permission system.** Roles become `CUSTOMER, WAREHOUSE_STAFF, ORDER_MANAGER, MANAGER, ADMIN` (five, not four — a deviation from the ER diagram's `ROLE` table, logged as `RBAC-1` in `09_ISSUES_REGISTER.md`). More significantly: **what a role can do is no longer hardcoded** — a `permission`/`role_permission` schema lets an `ADMIN` grant/revoke individual permissions per role at runtime from the admin console, so adding a capability to `ORDER_MANAGER` later is a data change, not a redeploy. Full design in `02_SECURITY_BASELINE.md` §1 and `07_SQL_DATABASE_STANDARDS.md` §1a. |
| OQ-3 | Product image storage: local filesystem, an object store (S3-compatible), or deferred entirely (placeholder images) for Phase 1? | Blocks `01-catalog` and `09-admin-catalog` spec/plan (upload endpoint, storage adapter) | ✅ **Resolved (2026-09-02): real upload pipeline, S3-compatible object storage.** A free-tier S3-compatible bucket in prod (AWS S3 or Cloudflare R2 free tier — pick one before Phase 5 deploy) and **MinIO** (S3-API-compatible, self-hosted, free) in local dev via Docker Compose — same Go S3 client code against both, only the endpoint URL changes per environment. Upload is gated behind the new `catalog:image:write` permission (not a fixed role), so it inherits OQ-1's dynamic model. Full design in `01_TECH_STACK.md` §1, `07_SQL_DATABASE_STANDARDS.md` §1a, `08_THREAT_MODEL.md` T-16/T-17. |
| OQ-4 | Does the seed dataset (≥40 products / ≥10 categories, SRS §6.1) need to reflect a specific real BrightBuy catalog, or is realistic-but-invented data acceptable? | Blocks `db/seed` script content | Assume invented-but-plausible electronics/toy data is acceptable for a Phase 1 pilot/course project unless told otherwise. |

## How to use this doc

When drafting a feature `spec.md` that touches an item above: state the working assumption in that
feature's §2 (mirroring SRS §2.7's own pattern), and leave the row here open until BrightBuy (or, for
this course project, the module instructor/team) actually confirms it. Don't let an unresolved row
silently block progress — build against the working assumption, and make the assumption loud in the
spec so it's cheap to revise later.
