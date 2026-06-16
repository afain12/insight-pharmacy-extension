# Deployment Runbook — Fly.io + self-hosted MySQL 8 + in-process Better Auth

Operational companion to [`hosting-and-auth-migration-strategy.md`](hosting-and-auth-migration-strategy.md) (the *why*) and the build contract [`../BUILD-SPEC-v0.4.md`](../BUILD-SPEC-v0.4.md) (the *what*). This file is the *how* — the concrete steps to stand up staging and production.

> **Hard gate.** Synthetic / internal-only deploys may proceed now. **No real patient data and no external provider office may use any environment until the HIPAA control checklist (§7) is complete** — this is the `TODOS.md` P0 blocker. Treat every environment as PHI-free until that gate is signed off.

---

## 1. Topology

| Component | Hosting | Notes |
|---|---|---|
| Backend (Express + Prisma + **Better Auth in-process**) | Fly.io app (HIPAA package for prod) | Serves `/auth/*` (Better Auth) and `/api/v1/*`. No separate auth-service. |
| Provider web view + internal tool (static bundles) | Fly static / CDN, or served by the backend | Must be **subdomains of one registrable domain** so `sameSite=lax; secure` cookies work (§4.4 of the spec). |
| Database | **Self-hosted MySQL 8** inside the BAA-covered boundary | Holds both the app tables *and* Better Auth's `user`/`session`/`account`/`verification` tables. |
| Email (digests/alerts) | Deferred | No PHI-bearing email until a HIPAA-capable vendor + BAA exist. Use Mailpit/synthetic only until then. |

Domain layout (example): `api.insightpharmacy.example`, `app.…`, `admin.…` — all under `.insightpharmacy.example` (`COOKIE_DOMAIN`).

---

## 2. Prerequisites (once)

- `flyctl` installed and authenticated (`fly auth login`).
- A Fly **organization per environment** — keep staging and production in **separate orgs** (Phase 4 requirement).
- MySQL 8 reachable from the Fly app over TLS, inside the same covered boundary (Fly MySQL machine + volume, or an existing self-hosted MySQL host that the Fly BAA covers — confirm coverage before PHI, §7).
- `mysql` client locally for migrations/restore tests.

---

## 3. Provision MySQL 8

Self-hosted on a Fly machine with a persistent volume (simplest path that stays inside the Fly boundary):

```bash
fly volumes create mysql_data --size 10 --region <region> --app <db-app>   # encrypted at rest by Fly
# Run mysql:8 as a Fly app mounting mysql_data at /var/lib/mysql, or use your existing MySQL host.
```

Create the database with the charset the spec pins (§5.0):

```sql
CREATE DATABASE insight_pharmacy
  CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
CREATE USER 'insight_app'@'%' IDENTIFIED BY '<strong-random>';
GRANT ALL PRIVILEGES ON insight_pharmacy.* TO 'insight_app'@'%';
FLUSH PRIVILEGES;
```

`DATABASE_URL=mysql://insight_app:<strong-random>@<db-host>:3306/insight_pharmacy` (require TLS in prod).

---

## 4. Secrets (Fly secret manager only — never in repo)

```bash
fly secrets set --app <backend-app> \
  NODE_ENV=production \
  DATABASE_URL='mysql://insight_app:...@<db-host>:3306/insight_pharmacy' \
  BETTER_AUTH_SECRET="$(openssl rand -base64 32)" \
  BETTER_AUTH_URL='https://api.insightpharmacy.example' \
  API_ORIGIN='https://api.insightpharmacy.example' \
  WEB_ORIGIN='https://app.insightpharmacy.example' \
  ADMIN_ORIGIN='https://admin.insightpharmacy.example' \
  ALLOWED_ORIGINS='https://app.insightpharmacy.example,https://admin.insightpharmacy.example,chrome-extension://<id>' \
  COOKIE_DOMAIN='.insightpharmacy.example' \
  COOKIE_SECURE=true \
  SESSION_IDLE_HOURS=12 \
  UPLOAD_MAX_BYTES=10485760 \
  PUBLISH_DEACTIVATE_MISSING=false \
  NO_PHI_LOGS=true
```

`BETTER_AUTH_SECRET` must be a strong random value (rotating it invalidates all sessions). `BETTER_AUTH_URL`/`API_ORIGIN` are the public HTTPS API origin. Full env reference: BUILD-SPEC §20 + the migration doc's "Minimum production environment variables".

---

## 5. Deploy order

Run for **staging first** (synthetic data), then production after the §7 gate.

```bash
# 1. Generate/verify Better Auth tables are part of the Prisma schema (committed), then:
fly deploy --app <backend-app>          # build + ship the backend image

# 2. Apply migrations against the target DB (additive in V1):
fly ssh console --app <backend-app> -C "pnpm prisma migrate deploy"

# 3. (staging only) synthetic seed:
fly ssh console --app <backend-app> -C "pnpm db:seed"   # foundation + synthetic users + referrals

# 4. First admin (prod): one-time bootstrap, refuses to run if an INTERNAL_ADMIN exists:
fly ssh console --app <backend-app> -C "pnpm bootstrap:admin"   # reads BOOTSTRAP_ADMIN_* secrets

# 5. Deploy the web view + internal tool bundles (subdomains per §1).

# 6. Smoke (must pass before anyone uses the env):
pnpm smoke   # signin → scoped /patients → cross-tenant 404 → healthz
```

**Smoke exit criteria** (Phase 3): `GET /healthz` → `{ db: "ok", auth: "ok", ... }`; login works; scoped `/patients` works; cross-tenant request returns 404; synthetic upload/publish works.

---

## 6. MySQL operations (document + test before PHI)

- **Backups:** automated daily logical dump (`mysqldump --single-transaction`) or volume snapshots; store inside the covered boundary.
- **Restore testing:** restore the latest backup into a throwaway DB and run `pnpm smoke` against it — schedule this regularly, not just once.
- **Encryption:** confirm encryption at rest (Fly volume) and TLS in transit on the connection string.
- **Upgrades / monitoring / access:** patch cadence for MySQL 8; least-privilege DB users; restrict admin access; no shared root in app config.
- **Retention/purge:** implement the FileBatch/ValidationError 90-day purge job before PHI (TODOS P0).

---

## 7. HIPAA control checklist (the production gate)

Before **any** real PHI or external pilot (this is the `TODOS.md` P0 hard blocker — engineering hygiene already in V1 does **not** satisfy it):

- [ ] Signed BAA with the app host (Fly.io HIPAA package, ~$99/mo).
- [ ] Signed BAA with the DB provider, **or** documented proof the self-hosted MySQL host/storage/backups sit inside the Fly BAA boundary.
- [ ] Signed BAA with any email/SMS/logging/support vendor that could touch PHI (else keep PHI-bearing email **off**).
- [ ] Encryption at rest + TLS in transit confirmed.
- [ ] Production org separate from staging; secrets in the platform secret manager only; debug logging disabled (`NO_PHI_LOGS=true`).
- [ ] Backups + tested restore procedure.
- [ ] Retention policy + purge job implemented or explicitly scheduled before PHI.
- [ ] Breach-response owner + wrong-account-exposure triage documented.
- [ ] Monthly HIPAA audit checklist owner assigned (per SOP-SPI-001).
- [ ] Compliance owner sign-off on the pilot.

---

## 8. Rollback

- **App:** `fly releases --app <backend-app>` then `fly deploy --image <previous>` (or `fly releases rollback`). Migrations are additive in V1, so a code rollback is safe without a schema downgrade.
- **Publish-level:** recover via FileBatch history + `ReferralStatusEvent` (BUILD-SPEC §9), not DB surgery.
- **DB:** restore from the most recent tested backup (§6) only as a last resort.

---

## 9. References

- Fly pricing / compliance / healthcare guide: https://fly.io/pricing/ · https://fly.io/compliance · https://fly.io/docs/blueprints/going-to-production-with-healthcare-apps/
- Better Auth MySQL adapter: https://www.better-auth.com/docs/adapters/mysql
- Prisma MySQL connector: https://www.prisma.io/docs/orm/core-concepts/supported-databases/mysql
- Strategy / rationale: [`hosting-and-auth-migration-strategy.md`](hosting-and-auth-migration-strategy.md)
