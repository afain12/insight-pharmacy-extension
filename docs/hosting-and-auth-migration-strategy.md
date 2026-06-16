# Hosting and Auth Migration Strategy

Date: 2026-06-16
Status: Draft decision note for implementation
Owner: Insight Specialty Pharmacy

## Executive decision

Use a HIPAA-capable hosting boundary and move authentication from SuperTokens to Better Auth before any real patient data or external provider pilot.

Recommended first production stack:

- App runtime: Fly.io HIPAA compliance package
- Database: existing/self-hosted MySQL, if it can be placed inside the HIPAA-covered production boundary with documented backups, restore testing, access controls, and encryption
- Auth: Better Auth self-hosted inside the backend, storing users and sessions in the same covered MySQL database
- Email: no PHI-bearing email until a HIPAA-capable email vendor and BAA are selected
- Current Railway usage: keep only for local-style development or synthetic/internal testing unless Railway's HIPAA threshold is accepted

This replaces the SuperTokens assumption in `BUILD-SPEC-v0.3.md` for production planning. The build spec still contains useful security invariants: fail-closed auth, account scoping, masked DOB in list views, no PHI logs, rate limiting, audit events, and production bootstrap. Those invariants should remain, but the auth implementation should be Better Auth rather than SuperTokens.

This also updates the earlier Postgres/Neon default. Neon is not required if the team already has operational comfort with a self-hosted MySQL database. The durable requirement is a BAA-covered relational database with reliable operations, not a specific Postgres vendor.

## Why this change

The current spec names SuperTokens as the auth layer. SuperTokens is workable, but it adds another auth-specific service, another deployment component, and potentially another BAA dependency if managed services are used.

The preferred direction is now:

- Use an open-source auth library inside the application.
- Store auth state in the same HIPAA-covered relational database as the application.
- Avoid hosted auth vendors entirely for V1.
- Keep the vendor/BAA chain limited to the infrastructure providers that actually store or process PHI.

Better Auth fits this better than SuperTokens for this project because it runs inside the TypeScript backend, supports MySQL and PostgreSQL, supports cookie sessions, and has bearer-token support for API clients such as a Chrome extension.

LabAide/Natalie is the closest internal precedent: it already uses a MySQL backend and ChatGPT-style query history. That is meaningful experience. For this project, the cheapest and fastest path should build on that operating comfort unless HIPAA boundary or database reliability requirements make a managed database more practical.

Important compliance caveat: self-hosting auth does not make the product HIPAA-compliant by itself. It only removes a hosted auth vendor from the compliance chain. HIPAA still requires BAAs, access controls, auditability, risk analysis, retention rules, breach response procedures, and operational safeguards.

## Product context

The app will handle referral and status data for specialty pharmacy workflows. Expected sensitive fields include:

- Patient name
- Patient DOB
- Medication/referral details
- Provider office association
- Referral status and next-action owner
- Upload files from hospital exports
- Audit and adoption events tied to user accounts

Because this is PHI/ePHI once real patient data is loaded, the repo's existing P0 blocker remains correct: no real patient data and no external provider pilot until HIPAA controls and vendor agreements are in place.

## Vendor comparison

### Recommended: Fly.io + self-hosted MySQL + Better Auth

This is the best balance of fast, cheap, and operationally understandable.

Use Fly.io for:

- Backend API
- Provider web view
- Internal admin UI
- Background/cron jobs if needed
- Optional same-org service for file-processing workers

Use self-hosted MySQL for:

- Application database
- Better Auth user/session tables
- Prisma application tables, using Prisma's MySQL provider
- Audit logs
- Uploaded batch metadata and validation errors

Use Better Auth for:

- Email/password login
- Session handling
- Cookie sessions for the web view and admin UI
- Bearer-token flow for the Chrome extension
- Optional 2FA/TOTP for internal admins

Current source checks:

- Fly.io says its HIPAA/BAA compliance package is available for HIPAA-compliant workloads and is priced at $99/month on its pricing page.
- Fly.io provides a pre-signed BAA flow through its compliance documents.
- Better Auth documents MySQL support, DB-backed users/sessions, cookie sessions, bearer token auth, and 2FA plugins.
- Prisma documents support for self-hosted MySQL/MariaDB through the `mysql` provider.

Key links:

- Fly pricing: https://fly.io/pricing/
- Fly compliance: https://fly.io/compliance
- Fly healthcare production guide: https://fly.io/docs/blueprints/going-to-production-with-healthcare-apps/
- Better Auth docs: https://www.better-auth.com/docs
- Better Auth MySQL adapter: https://www.better-auth.com/docs/adapters/mysql
- Better Auth bearer plugin: https://better-auth.com/docs/plugins/bearer
- Better Auth sessions: https://www.better-auth.com/docs/concepts/session-management
- Better Auth 2FA: https://www.better-auth.com/docs/plugins/2fa
- Prisma MySQL connector: https://www.prisma.io/docs/orm/core-concepts/supported-databases/mysql

Pros:

- Lower starting cost than Render HIPAA or Railway HIPAA.
- No separate hosted auth vendor.
- Builds on the team's existing MySQL operating experience.
- Keeps the persistent data store familiar instead of introducing Neon/Postgres.
- Matches the existing Express/Prisma direction with a database provider change from Postgres to MySQL.
- Supports the Chrome extension via bearer tokens.
- Keeps deployment closer to a simple app-hosting model than AWS/GCP.

Cons:

- More database operations responsibility than a managed database.
- Need to document MySQL backups, restore testing, upgrades, monitoring, encryption, and admin access.
- Need to confirm that the MySQL host and storage live inside a BAA-covered production boundary before PHI.
- Need a HIPAA-capable email vendor before sending patient-specific digests.

### Optional managed-database fallback: Fly.io + Neon + Better Auth

Neon should remain an option, not the default.

Use Neon only if:

- The self-hosted MySQL environment cannot be brought into the HIPAA-covered boundary.
- Backup/restore, uptime, or database administration would distract from getting the pharmacy pilot live.
- The team decides managed Postgres is worth the additional vendor and migration work.

Current source checks:

- Neon says HIPAA is a self-serve feature on the Scale plan, currently with no additional HIPAA charge, though it notes a future 15% surcharge may apply.
- Better Auth and Prisma both support PostgreSQL if the team later chooses that path.

Key links:

- Neon HIPAA: https://neon.com/docs/security/hipaa
- Neon pricing: https://neon.com/pricing
- Better Auth PostgreSQL adapter: https://better-auth.com/docs/adapters/postgresql

Pros:

- Managed database operations.
- Clear HIPAA self-serve flow on the supported plan.
- Good fallback if self-hosted MySQL is operationally risky.

Cons:

- Introduces a database vendor the team does not currently know.
- Requires adapting the product database direction back to Postgres.
- Adds another BAA/vendor relationship.

### Alternative: Render HIPAA

Render is the easiest PaaS-style option if budget is less important than simplicity.

Use Render for:

- Backend web service
- Static/provider web app
- Static/admin app
- Managed Postgres
- Cron jobs/workers

Current source checks:

- Render supports HIPAA-enabled workspaces on Scale or Enterprise.
- Render states that a HIPAA-enabled workspace does not automatically make applications HIPAA-compliant.
- Render HIPAA workspaces keep HIPAA workflows inside the enabled workspace.

Key links:

- Render HIPAA docs: https://render.com/docs/hipaa-compliance
- Render pricing: https://render.com/pricing

Pros:

- Easiest migration path from a Railway-like mental model.
- Managed database and app hosting are in one platform.
- Less deployment complexity than Fly plus a separately operated database.

Cons:

- More expensive entry point.
- Still requires application controls and a BAA.
- Better Auth still needs careful integration and testing.

### Not recommended for first PHI pilot: Railway

Railway remains fine for synthetic testing and rapid prototypes, but should not be used for real PHI unless the HIPAA BAA threshold is accepted and the account is configured under that agreement.

Current source checks:

- Railway docs say HIPAA BAA is an add-on with a paid monthly spend threshold.
- Railway directs customers needing a BAA to contact its solutions team.

Key links:

- Railway compliance: https://docs.railway.com/enterprise/compliance
- Railway pricing plans: https://docs.railway.com/pricing/plans

Pros:

- Familiar to current builder.
- Fast development loop.

Cons:

- HIPAA is not the normal low-cost path.
- Spend threshold makes it a poor first pilot choice.
- Keeping real PHI on Railway without the HIPAA agreement is not acceptable.

### Later-stage option: AWS or Google Cloud

AWS or Google Cloud are strong long-term platforms, but not the recommended first step for this project because they introduce too much cloud configuration work.

Use later if:

- There is a dedicated technical operator.
- Procurement or hospital IT prefers a major cloud.
- The product needs deeper network controls, enterprise IAM, or regional resilience.

## Target production topology

Use one registrable domain and subdomains:

- `api.insightpharmacy.example` - Express backend and Better Auth endpoints
- `app.insightpharmacy.example` - provider web view
- `admin.insightpharmacy.example` - internal admin tool

The Chrome extension calls `api.insightpharmacy.example` directly.

Cookie strategy:

- Web view and admin UI use secure, HTTP-only, SameSite=Lax cookies.
- Keep API, app, and admin under the same registrable domain.
- Do not use cross-site cookie topology for V1.

Extension strategy:

- The extension uses bearer tokens through Better Auth's bearer-token support.
- Store extension tokens only in Chrome extension storage.
- Do not store patient data in extension storage.
- Clear tokens on sign-out.
- Keep patient data fetched on demand from the API.

Database strategy:

- One production MySQL database.
- One staging/synthetic MySQL database.
- No PHI in development databases.
- Better Auth tables live in the same database as the application tables.
- Prisma migrations own application tables.
- Better Auth CLI or generated schema owns auth tables, but migrations must be reviewed and committed like app migrations.
- Do not put PHI in the existing MySQL instance until the host, storage, backups, and access paths are confirmed inside the HIPAA-covered production boundary.

## SuperTokens to Better Auth migration

The app is not implemented yet, so this should be treated as an implementation change, not a data migration. If SuperTokens has not been deployed with real users, do not build migration scripts from SuperTokens. Replace the spec before implementation.

### Files and concepts to replace

Remove or avoid:

- `packages/backend/src/supertokens.ts`
- SuperTokens Core service in `docker-compose.yml`
- `supertokens-node`
- `supertokens-web-js`
- `supertokens-auth-react`
- `User.supertokensId`
- SuperTokens-specific seed flow
- SuperTokens middleware and error handler wiring
- SuperTokens-managed `/auth/*` assumptions

Add:

- `packages/backend/src/auth.ts` for Better Auth server config
- `packages/backend/src/middleware/requireSession.ts`
- `packages/backend/src/middleware/resolveLocalUser.ts`
- `packages/backend/src/middleware/accountScope.ts`
- `packages/backend/src/routes/auth.ts` if Express routing needs an adapter mount
- Better Auth client setup in extension/web/admin packages
- Better Auth generated tables or reviewed migration files

### Data model changes

Replace:

```prisma
model User {
  id            String  @id @default(uuid())
  supertokensId String  @unique
  email         String  @unique
  firstName     String
  lastName      String
  role          Role
  isActive      Boolean @default(true)
  ...
}
```

With:

```prisma
model User {
  id              String    @id @default(uuid())
  betterAuthUserId String   @unique
  email           String    @unique
  firstName       String
  lastName        String
  role            Role
  isActive        Boolean   @default(true)
  lastLoginAt     DateTime?
  digestOptIn     Boolean   @default(true)
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt
  accounts        UserAccount[]
  fileBatches     FileBatch[]
  auditLogs       AuditLog[]
  adoptionEvents  AdoptionEvent[]
  digestDeliveries DigestDelivery[]
}
```

Implementation note: if Better Auth's generated user table can be extended safely, consider making the app's local user profile reference Better Auth's user ID. Keep role/account authorization in application-owned tables, not in client-editable metadata.

### Auth route shape

Recommended:

- Better Auth mounted under `/auth/*` if practical, to preserve existing spec language and rate-limit boundaries.
- Keep API endpoints under `/api/v1/*`.
- All protected routes use:
  - Better Auth session verification
  - local user resolution
  - active-user check
  - role/account authorization

The route names can change internally, but the public security behavior must not.

### Required invariants to preserve

Do not weaken these from the existing build spec:

- Missing session means 401.
- Missing local user row means 401.
- Inactive user means 403.
- Provider users must pass an `X-Account-Id` header.
- Provider users can only access accounts linked through `UserAccount`.
- Cross-account patient access returns 404, not 403.
- Inactive accounts are rejected even if a stale session exists.
- `req.localUser.id` is the only user ID stored in audit logs.
- No patient name, DOB, medication, referral ID, or status note appears in server logs.
- Session idle timeout target remains 12 hours or shorter.
- `/auth/*` remains rate-limited.
- `/upload` remains rate-limited per authenticated user.
- First production admin is bootstrapped by a one-time script, not seeded credentials.

### Client behavior

Provider web view:

- Cookie session.
- Calls `/api/v1/me` after login.
- Shows account picker if multiple accounts exist.

Internal admin tool:

- Cookie session.
- Requires `INTERNAL_ADMIN` or `INTERNAL_STAFF`.
- Require 2FA for `INTERNAL_ADMIN` before real PHI.

Chrome extension:

- Bearer token flow.
- No PHI persisted locally.
- Sign-out clears token storage.
- API wrapper attaches `Authorization: Bearer <token>` and `X-Account-Id`.

## Implementation sequence

### Phase 0 - Keep development safe

Goal: allow fast building without creating a compliance incident.

- Continue using local Docker or Railway only for synthetic data.
- Put `NO_REAL_PHI=true` or equivalent in development/staging envs.
- Add README warnings that dev/staging are not for real PHI unless explicitly covered by BAA.
- Keep sample users and sample referrals synthetic only.

Exit criteria:

- No production secrets in repo.
- No real patient data in dev/staging.
- Synthetic seed flow works.

### Phase 1 - Replace SuperTokens in the spec

Goal: update implementation direction before code is written.

- Update `BUILD-SPEC-v0.3.md` or create `BUILD-SPEC-v0.4.md`.
- Replace the tech stack auth row with Better Auth.
- Replace Section 4 auth details with Better Auth equivalents.
- Replace SuperTokens references in build steps, tests, seed scripts, and internal tool login.
- Update `TODOS.md` P0 text from "SuperTokens core" to "auth stack".

Exit criteria:

- No new implementation starts from a SuperTokens assumption.
- The fail-closed auth tests are still listed as ship-blocking.

### Phase 2 - Implement Better Auth locally

Goal: prove auth and authorization behavior before hosting.

- Add Better Auth packages.
- Configure MySQL adapter.
- Generate/review auth tables.
- Mount Better Auth under `/auth`.
- Implement session middleware.
- Implement local user resolution.
- Implement account scoping.
- Implement admin bootstrap.
- Implement provider account assignment.
- Add 2FA for internal admins if plugin compatibility is confirmed.

Exit criteria:

- Sign-in works locally.
- `/api/v1/me` returns local role and account IDs.
- Deactivated user cannot access protected routes.
- Provider cross-account access returns 404.
- Admin-only routes reject provider users.
- Extension bearer-token flow works locally.

### Phase 3 - Deploy synthetic staging

Goal: validate hosting without PHI.

- Create Fly organization/project for staging.
- Create a staging MySQL database inside the same hosting model, or use synthetic-only local/Railway development if staging is not BAA-covered.
- Deploy backend and frontends.
- Configure domain names.
- Configure Better Auth secret.
- Configure CORS and cookie domain.
- Run migrations.
- Run synthetic seed.
- Run smoke tests.

Exit criteria:

- `GET /healthz` passes.
- Login passes.
- Scoped `/patients` works.
- Cross-tenant smoke test returns 404.
- Upload/publish smoke test works on synthetic CSV/XLSX.

### Phase 4 - Prepare PHI production boundary

Goal: complete the minimum compliance gate before any real patient data.

- Sign Fly BAA or selected app-host BAA.
- Confirm MySQL host coverage under the selected BAA, or sign the selected managed database BAA.
- Select and sign BAA for any email/SMS/logging/support vendor that may touch PHI.
- Confirm encryption at rest and TLS in transit.
- Create production organization separate from staging.
- Restrict admin access.
- Store secrets in platform secret manager only.
- Disable debug logging.
- Configure backups and restore testing.
- Configure retention policy and purge jobs.
- Document breach-response owner and wrong-account-exposure triage.
- Confirm monthly HIPAA audit checklist owner.

Exit criteria:

- BAA chain is documented.
- Production deploy has no synthetic/default credentials.
- Restore procedure is tested.
- Retention job is implemented or explicitly scheduled before PHI.
- Compliance owner approves pilot.

### Phase 5 - External pilot

Goal: start small without creating operational sprawl.

- Limit to 1-2 trusted provider offices.
- Side-load extension or use Chrome Web Store private/unlisted path if approved.
- Use least-privilege provider accounts.
- Monitor audit logs daily.
- Keep PHI-bearing email digests disabled unless email BAA is complete.
- Review every upload/publish before providers see it.

Exit criteria:

- No cross-account visibility issues.
- Providers can log in and see only their accounts.
- Office staff can use the flow without portal switching.
- Insight ops can correct upload errors without engineering help.

## Minimum production environment variables

Exact names can change during implementation, but these concepts are required:

```text
NODE_ENV=production
DATABASE_URL=mysql://...
BETTER_AUTH_SECRET=...
BETTER_AUTH_URL=https://api.insightpharmacy.example
API_ORIGIN=https://api.insightpharmacy.example
WEB_ORIGIN=https://app.insightpharmacy.example
ADMIN_ORIGIN=https://admin.insightpharmacy.example
ALLOWED_ORIGINS=https://app.insightpharmacy.example,https://admin.insightpharmacy.example,chrome-extension://...
COOKIE_DOMAIN=.insightpharmacy.example
COOKIE_SECURE=true
SESSION_IDLE_HOURS=12
UPLOAD_MAX_BYTES=10485760
PUBLISH_DEACTIVATE_MISSING=false
NO_PHI_LOGS=true
```

## HIPAA control checklist

Before real PHI:

- Signed BAA with app host.
- Signed BAA with database provider or documented proof that the self-hosted MySQL host/storage/backups are inside the app-host BAA boundary.
- Signed BAA with email vendor if PHI appears in email.
- Signed BAA with logging/monitoring vendor if logs may include PHI.
- No PHI in logs, metrics labels, traces, resource names, branch names, database names, or support tickets.
- TLS required for all public endpoints.
- Encryption at rest confirmed for database and backups.
- Least-privilege admin access.
- 2FA required for internal admins.
- Separate production and staging organizations/projects.
- No real PHI in staging unless staging is also covered and approved.
- Audit logs for login, patient list fetch, patient detail fetch, upload, validation, publish, account/user changes.
- Retention policy for uploaded files, validation errors, and audit logs.
- Backup restore test documented.
- Breach-response owner named.
- Wrong-account exposure runbook written.
- Provider termination/deactivation process written.

## Email and digest decision

The existing spec includes daily email digests. Keep the feature, but gate it:

- Digest emails may be enabled for synthetic/internal testing.
- For real PHI, do not send patient names, DOBs, medication names, or specific referral status through ordinary SMTP.
- If email must include patient-specific content, choose a HIPAA-capable email vendor with a signed BAA.
- Safer first pilot option: send generic "You have N Insight updates" emails with a link to the authenticated app, not patient-specific details.

## Logging and observability

Use platform logs only if they are inside the BAA boundary or guaranteed not to contain PHI.

Log allowed:

- Request method and route template
- Status code
- Timing
- User role
- Local user ID
- Account ID only if approved as operational metadata
- Batch ID
- Error class without raw input values

Do not log:

- Patient names
- DOB
- Medication
- Referral ID
- Status note
- Raw uploaded rows
- Validation raw values
- Authorization headers
- Cookies
- Session tokens

## Open decisions

- Final app host: Fly.io vs Render.
- Final database host: existing/self-hosted MySQL vs managed fallback.
- Final email vendor for PHI-capable digests.
- Whether extension pilot is side-loaded, private Chrome Web Store, or public listing.
- Whether internal admins require TOTP on day one or before first external pilot.
- Whether staging will ever contain real PHI. Default answer should be no.

## Recommended next repo changes

1. Create `BUILD-SPEC-v0.4.md` that replaces SuperTokens with Better Auth.
2. Update `TODOS.md` P0 HIPAA item to reference "auth stack" instead of "SuperTokens core".
3. Update build steps to remove SuperTokens Core from local Docker.
4. Add Better Auth fail-closed test cases to the test plan.
5. Add a deployment runbook for Fly + self-hosted MySQL.

## Bottom line

The cheapest credible path is not a hosted auth provider and not Railway HIPAA. It is a small HIPAA-covered app/database boundary with auth running inside the app and data stored in a database the team already knows how to operate.

Start with Fly.io + self-hosted MySQL + Better Auth if the MySQL environment can be covered and operated safely. Keep Neon only as the managed-database fallback, Railway for synthetic development only, Render if simplicity becomes more important than monthly cost, and AWS/GCP only when the business is ready for more cloud operations.
