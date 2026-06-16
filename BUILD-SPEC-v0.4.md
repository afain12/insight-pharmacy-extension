# Insight Specialty Pharmacy Extension — V1 Build Specification v0.4 (Consolidated)

This document is the complete, self-contained build specification for the Insight Specialty Pharmacy Extension V1. It supersedes build spec v0.3 (`BUILD-SPEC-v0.3.md`), which in turn superseded "Insight Specialty Pharmacy Extension PRD Structure v0_2.md" (build spec v0.2). Read top-to-bottom. Follow the architecture, file structure, schema, API contracts, and build order exactly. Where this document and an earlier spec disagree, this document wins.

**What changed from v0.3 (the only substantive deltas):** the authentication layer moves from **SuperTokens → Better Auth**, and the database moves from **PostgreSQL → self-hosted MySQL 8**, per `docs/hosting-and-auth-migration-strategy.md`. Every security invariant from v0.3 is preserved unchanged — fail-closed sessions, account scoping, cross-account 404, request-time `isActive` enforcement, masked DOB, no-PHI logs, rate limiting, audit events, and one-time admin bootstrap. Only the auth implementation and the database engine change. All other sections (status model §1, ingestion §10, publish §9, provider UI §12, digest §14, scope boundary §15) carry over from v0.3 verbatim except where they named SuperTokens, Postgres, or a Postgres-only schema construct. The auth-layer changes are summarized in §4.0; the MySQL schema deltas are called out in §5.0.

**Product context** (from the product PRD + SOP-SPI-001 v2.0): Insight Specialty Pharmacy is a hospital-based specialty pharmacy. Provider offices send referrals and currently have no self-serve status visibility — they call or email. This system gives office staff a daily account-scoped status view (Chrome extension + web view + email digest), fed by a daily hospital Google Sheets export that Insight ops uploads, validates, previews, and publishes through an internal admin tool. The SOP defines four phases: (1) Physician Enrollment & Submission, same-day SLA; (2) Intake & Prior Authorization, 24–48h (complex PA 2–3 weeks, measured from receipt of complete documentation); (3) Telehealth & Approval, 24h; (4) Fulfillment & Delivery, 24–48h.

---

## Section 0: Coding Principles

1. **Think before coding.** One-line purpose comment at the top of every file. Ambiguous requirement → implement the simplest interpretation.
2. **Simplicity first.** Minimum code per requirement. No utility libraries beyond the tech stack table. Native fetch, native Date (except date-only fields — see §5.1), built-in array methods. Functions and plain objects; no classes.
3. **Surgical changes.** Each build step produces exactly the files listed for that step. Do not refactor previous steps while building a new one.
4. **Goal-driven execution.** Every build step ends with a verification. Do not proceed until it passes.
5. **Tests are required.** Every build step that adds backend logic includes the tests named for it in §18. `pnpm test` green is part of step verification. (This replaces v0.2's "write tests only where the spec says to" — §18 is where the spec says to.)

Anti-patterns to avoid: barrel files; abstract base classes; logging frameworks (use `console.log`/`console.error` with structured prefixes per §16); request-ID middleware; custom error classes (plain `Error` with descriptive messages); comments that repeat the code. **Rate limiting on `/auth/*` and `/upload` is required (§4.6), not an anti-pattern.**

---

## Section 1: Canonical Status Model

### 1.1 The 18 canonical provider-facing statuses

Submission phase: `Physician enrollment pending`, `Referral received`, `eRx received`, `Missing information`
Intake/PA phase: `Intake in progress`, `Prior authorization initiated`, `Prior authorization pending`, `Prior authorization approved`, `Prior authorization denied`
Telehealth phase: `Telehealth pending`, `Telehealth scheduled`, `Telehealth no-show`, `Telehealth completed`
Fulfillment/Delivery phase: `Address confirmation pending`, `Fulfillment in progress`, `Medication shipped`, `Delivered`
Cross-phase: `Closed`

Workflow phases: `Submission`, `Intake/PA`, `Telehealth`, `Fulfillment/Delivery`.
Next action owners: `Insight`, `Provider office`, `Patient`, `Payer`, `Carrier`, `None`.

### 1.2 Status mapping layer

The uploaded file carries `internal_status` (REQUIRED). The backend maps internal → canonical via the `StatusMapping` table (seeded from `prisma/seed/status-mappings.ts`; **provisional fixture pending Phase-0 sample exports — see §22**). The file MAY carry `provider_status` as an explicit override; if present it must be one of the 18 canonical values. An `internal_status` with no mapping and no override is a CRITICAL validation error listing the known mappings.

### 1.3 Canonical status table (single source of truth for UI + API)

Lives in `packages/shared/src/statuses.ts`, consumed by backend and both frontends:

| Canonical status | Phase | Chip bucket | Chip color | Typical needs-action source (informational — §1.4 predicate is the ONLY source of truth) | Explanation (provider-facing) |
|---|---|---|---|---|---|
| Physician enrollment pending | Submission | All | neutral | per owner | Provider setup (CDTM agreement) is not yet complete for this prescriber. Insight will reach out if anything is needed from your office. |
| Referral received | Submission | All | neutral | no | Insight has received this patient's referral and it is being processed. |
| eRx received | Submission | All | neutral | no | The electronic prescription was received successfully. |
| Missing information | Submission | Needs Action | amber | YES | Required documents or information are incomplete. Please check the note below for what is needed. |
| Intake in progress | Intake/PA | All | neutral | no | Insight is verifying patient, prescriber, and medication details. |
| Prior authorization initiated | Intake/PA | PA Pending | indigo | no | The prior authorization process has been started with the insurance carrier. |
| Prior authorization pending | Intake/PA | PA Pending | indigo | no | We are waiting on a response from the insurance carrier. |
| Prior authorization approved | Intake/PA | PA Approved | emerald | no | Insurance has approved this medication. Moving to next steps. |
| Prior authorization denied | Intake/PA | PA Pending | neutral | no | The prior authorization was denied. Insight is reviewing next steps. |
| Telehealth pending | Telehealth | Telehealth | neutral | no | A telehealth visit needs to be scheduled or completed for this patient. |
| Telehealth scheduled | Telehealth | Telehealth | neutral | no | The telehealth visit has been scheduled. |
| Telehealth no-show | Telehealth | Telehealth | amber | per owner | The patient missed the scheduled telehealth visit. Insight is following up to reschedule. |
| Telehealth completed | Telehealth | Telehealth | emerald | no | The telehealth visit is complete. Fulfillment has been triggered. |
| Address confirmation pending | Fulfillment/Delivery | Needs Action | amber | per owner | The patient's delivery address needs to be confirmed before the medication can ship. |
| Fulfillment in progress | Fulfillment/Delivery | Shipping | blue | no | The medication is being prepared for shipment. |
| Medication shipped | Fulfillment/Delivery | Shipping | blue | no | The medication has been shipped and is in transit. |
| Delivered | Fulfillment/Delivery | Shipping | emerald | no | Delivery has been confirmed. |
| Closed | retains last phase | All | neutral | no | This case is no longer active. |

`Closed` invariant: a row mapped to `CLOSED` is deactivated on publish (`isActive=false`, `DEACTIVATED` event); `workflowPhase` keeps its last known value (no `CLOSED` phase exists). Closed rows are excluded from Total active and all default views.

Colors: neutral `#6B7280`, amber `#F59E0B`, indigo `#4F46E5`, emerald `#10B981`, blue `#3B82F6`. "PA denied" is deliberately NOT red — Insight is handling it; red would alarm office staff about something they don't own.

### 1.4 Server-computed predicates (never computed client-side)

- `needsAction(referral) := referral.nextActionOwner === PROVIDER_OFFICE`
- `newToday(referral) := referral has a ReferralStatusEvent of type CREATED or STATUS_CHANGED in the most recent PUBLISHED batch (by publishedAt) for its account`

---

## Section 2: Tech Stack (pinned)

| Layer | Technology | Version pin |
| --- | --- | --- |
| Runtime | Node.js | 20 LTS (`"engines": { "node": ">=20 <21" }`) |
| Package manager | pnpm workspaces | 9.x |
| Backend | Express + TypeScript, run with `tsx watch` (dev) / `tsc && node dist` (prod) | express 4.x, typescript 5.x, tsx 4.x |
| Database | MySQL | 8.x (docker image `mysql:8`) — Prisma `mysql` provider; utf8mb4 charset |
| ORM | Prisma | 5.x (`provider = "mysql"`) |
| Auth | Better Auth — `better-auth` (latest 1.x), run **in-process inside the Express backend** (no separate auth service). Prisma adapter on the same MySQL database; `emailAndPassword` enabled with public sign-up disabled; `bearer()` plugin (Chrome extension), cookie sessions (web view + internal tool); optional `twoFactor()` for internal admins. Client: `better-auth/client` + `better-auth/react`. No hosted auth vendor. |
| File parsing | `csv-parse` 5.x (CSV), **`exceljs` 4.x** (XLSX — replaces v0.2's `xlsx`, which is frozen at 0.18.5 on npm with CVE-2023-30533 + CVE-2024-22363), `iconv-lite` 0.6.x (Windows-1252 fallback decode), `yauzl` 3.x (XLSX zip-entry inspection / bomb guard) |
| Validation | zod 3.x |
| Uploads | multer 1.4.5-lts.x (+ @types/multer) |
| Rate limiting | express-rate-limit 7.x |
| Email | nodemailer 6.x (SMTP via env) |
| Scheduler | node-cron 3.x (ops alerts) |
| Frontends | React 18, Vite 5, Tailwind CSS 4 (`@tailwindcss/vite`) |
| Extension build | `@crxjs/vite-plugin` 2.x (MV3 multi-entry, manifest handling) |
| Tests | vitest 2.x + supertest 7.x (backend); @playwright/test 1.4x (3 E2E flows, §18) |

Packages per workspace are enumerated in §19 Step 1. Commit `pnpm-lock.yaml`.

---

## Section 3: Project Structure

```
insight-pharmacy-extension/
├── pnpm-workspace.yaml
├── package.json                      # root scripts: dev, db:setup, db:migrate, seed:*, test, smoke, build:extension
├── docker-compose.yml                # mysql:8 (app DB) + mailpit (SMTP :1025, UI :8025) — no auth service; Better Auth runs in-process
├── .env.example                      # zero placeholders — works as-is for local dev
├── .gitignore
├── README.md                         # quick start, env table, scripts, extension load procedure, troubleshooting
├── TODOS.md                          # deferred-scope register (pre-exists in repo; maintained, not produced by a build step)
├── docs/
│   ├── file-contract.md              # versioned ops API: headers, formats, valid values, mapping, templates, error meanings
│   ├── ops-runbook.md                # daily upload SOP, error triage, publish-failure recovery, user provisioning, alert response
│   ├── release-runbook.md            # migrations, extension packaging, side-load update checklist, smoke checks
│   └── architecture.md              # auth/session/account scoping, publish flow, data model invariants
├── fixtures/                         # test files: valid.csv, valid.xlsx, missing-column.csv, bad-dates.csv,
│                                     # duplicate-referral-id.csv, unknown-status.csv, unmatched-account.csv,
│                                     # bom-utf8.csv, win1252.csv, multi-sheet.xlsx, file-contract-v1.csv, file-contract-v1.xlsx
├── scripts/
│   └── smoke.ts                      # signin → scoped /patients → cross-tenant 404 → bad upload 400 → publish 200→409
├── prisma/
│   ├── schema.prisma                 # §5 — canonical
│   ├── seed/
│   │   ├── foundation.ts             # accounts, aliases, status mappings (no FK on users)
│   │   ├── users.ts                  # Better Auth user + app User rows (in-process; no core) — LOCAL-ONLY credentials
│   │   └── referrals.ts              # synthetic FileBatch + 17 referrals + status events (requires users)
│   └── migrations/
└── packages/
    ├── shared/                       # statuses.ts (canonical table §1.3), types.ts — consumed by all packages
    ├── backend/
    │   └── src/
    │       ├── index.ts              # Express wiring per §4.5
    │       ├── config.ts             # env via zod; fail fast on missing
    │       ├── prisma.ts             # single shared PrismaClient export
    │       ├── auth.ts               # §4.1 — Better Auth server instance
    │       ├── middleware/
    │       │   ├── resolveLocalUser.ts   # §4.2 — validates Better Auth session, fail-closed, isActive, attaches req.localUser
    │       │   ├── accountScope.ts       # §4.3 — X-Account-Id ∈ req.localUser.accountIds (DB-resolved)
    │       │   └── requireRole.ts
    │       ├── routes/ (healthz, upload, patients, accounts, users, digest-unsubscribe)
    │       ├── services/ (fileParser, validation, statusMapping, accountMatcher, publish, patients, audit, digest, alerts)
    │       └── utils/normalizeName.ts
    ├── extension/                    # builds TWO targets: MV3 zip (header tokens) and web view bundle (cookies)
    │   ├── manifest.json             # §11 — includes committed dev "key"
    │   ├── vite.config.ts            # crxjs + tailwind; web target via VITE_TARGET=web
    │   └── src/
    │       ├── popup/ (App, auth.ts, lib/api.ts, hooks/)
    │       │   └── components/ (Header, NeedsActionBanner, StatSummary, FilterChips, SearchBar,
    │       │                    PatientList, PatientRow, PatientDetail, AccountPicker, FirstRunCard,
    │       │                    EmptyState, ErrorState, SkeletonList, LoginForm, Footer)
    │       └── background/service-worker.ts   # badge on popup-open message only (V1)
    └── internal-tool/
        └── src/pages/ (Overview, Upload, Accounts, Users)
```

---

## Section 4: Authentication & Authorization

### 4.0 What changed from v0.3 (SuperTokens → Better Auth)

Better Auth runs **in-process inside the Express backend** (no separate auth-service container; the `supertokens-postgresql` core is gone — §17/§19). It owns its own tables (`user`, `session`, `account`, `verification`, plus `twoFactor` if enabled) in the **same MySQL database** via the Prisma adapter; those tables are generated by Better Auth's schema, reviewed, and committed alongside `prisma/schema.prisma`. The app's `User` table keeps a `betterAuthUserId String @unique` foreign reference (replacing `supertokensId` — §5).

**The fail-closed model is unchanged in behavior and actually simpler:** v0.3 minted `role`/`accountIds` into a SuperTokens access-token payload at login/refresh. Better Auth sessions carry no app claims; instead `resolveLocalUser` (4.2) reads `role` + active `accountIds` **from the database on every protected request**. Consequence: there is **no stale-claims window** — role/account/deactivation changes take effect on the very next request, not on next token refresh. Every v0.3 invariant still holds: missing session → 401, missing local user → 401, inactive user → 403, provider request without a valid in-scope `X-Account-Id` → 404 (no existence leak), deactivated account excluded everywhere.

### 4.1 Backend Better Auth init (`auth.ts`)

```typescript
import { betterAuth } from "better-auth";
import { prismaAdapter } from "better-auth/adapters/prisma";
import { bearer, twoFactor } from "better-auth/plugins";
import { prisma } from "./prisma";          // single shared client — never instantiate per-request

export const auth = betterAuth({
  database: prismaAdapter(prisma, { provider: "mysql" }),
  basePath: "/auth",                          // served under /auth/* (was SuperTokens' /auth)
  baseURL: process.env.BETTER_AUTH_URL,
  secret: process.env.BETTER_AUTH_SECRET,
  trustedOrigins: process.env.ALLOWED_ORIGINS!.split(","),
  emailAndPassword: {
    enabled: true,
    disableSignUp: true,                      // NO public registration — users are provisioned admin-side (§6)
    // password reset is initiated only server-side via the admin flow (§6.3); no public reset endpoint
  },
  session: {
    expiresIn: 60 * 60 * 12,                  // 12h idle ceiling (§4.6)
    updateAge: 60 * 60,                       // refresh sliding window at most hourly
  },
  plugins: [
    bearer(),                                 // Chrome extension: Authorization: Bearer <token>
    twoFactor(),                              // optional TOTP for INTERNAL_ADMIN (enrollment in internal tool)
  ],
});
```

Public sign-up is disabled at the recipe level (`disableSignUp: true`); the **server-side functions remain available** and are exactly what admin provisioning (§6.1) and temp-password reset (§6.3) call (`auth.api.createUser` via the admin plugin / `auth.api.setUserPassword`). There is no `try.*` hosted vendor — Better Auth is self-contained.

```typescript
// Authorization data is resolved from the DB per request (no token claims). Fail-closed helper:
export async function loadAuthContext(betterAuthUserId: string) {
  const user = await prisma.user.findUnique({
    where: { betterAuthUserId },
    include: { accounts: { include: { account: { select: { id: true, isActive: true } } } } },
  });
  if (!user || !user.isActive) return null;             // fail closed → caller denies
  return {
    localUserId: user.id,
    role: user.role,                                    // Prisma enum value
    accountIds: user.accounts
      .filter(a => a.account.isActive)                  // deactivated ACCOUNTS drop out
      .map(a => a.account.id),                          // [] for internal users
  };
}
// Account deactivation semantics: an inactive Account is excluded here, from GET /me,
// rejected by accountScope (re-checks Account.isActive), and excluded from upload matching (§8).
```

### 4.2 `resolveLocalUser` middleware (first middleware on every protected route)

```typescript
import { fromNodeHeaders } from "better-auth/node";
const session = await auth.api.getSession({ headers: fromNodeHeaders(req.headers) });
if (!session) return res.status(401).end();             // no/invalid session (cookie OR bearer)
const ctx = await loadAuthContext(session.user.id);     // session.user.id === User.betterAuthUserId
if (!ctx) return res.status(401).end();                 // orphan Better Auth user → no local row → 401
// (inactive local user is filtered inside loadAuthContext; a previously-active user that is now
//  inactive is treated as 403 by callers that distinguish — see below)
req.localUser = { id: ctx.localUserId, role: ctx.role, accountIds: ctx.accountIds };
```

Denies **401** when there is no session or no matching local `User`; denies **403** when the local row exists but `isActive === false` (resolve `isActive` explicitly so an active→inactive transition returns 403, not 401). Attaches `req.localUser` — the source of truth for all `AuditLog.userId` FKs (never the Better Auth user id). Updates `lastLoginAt` at most once per hour per user.

### 4.3 `accountScope` middleware (provider routes)

```typescript
// FAIL CLOSED: role must be a known enum value; provider requests must name an in-scope account.
const { role, accountIds } = req.localUser;
if (role !== "PROVIDER_STAFF" && role !== "INTERNAL_ADMIN" && role !== "INTERNAL_STAFF") return res.status(403).end();
if (role === "PROVIDER_STAFF") {
  const accountId = req.header("X-Account-Id");
  if (!accountId || !Array.isArray(accountIds) || !accountIds.includes(accountId)) {
    return res.status(404).json({ error: "Not found" });   // 404, never 403 — no existence leak
  }
  const account = await prisma.account.findUnique({ where: { id: accountId }, select: { isActive: true } });
  if (!account?.isActive) return res.status(404).json({ error: "Not found" });  // deactivated account
  req.scopedAccountId = accountId;                          // non-empty string, asserted
}
```

`accountIds` come from `resolveLocalUser` (DB-resolved this request), not a token — so there is no stale-claims path. The patients service MUST still assert `scopedAccountId` is a non-empty string before any query: `where: { accountId: undefined }` in Prisma silently drops the filter — that failure mode is why every guard above exists, and §18 ships a test for each.

### 4.4 Clients (explicit init configs)

- **Extension (MV3 popup):** `better-auth/client` with the **bearer** plugin — `createAuthClient({ baseURL: import.meta.env.VITE_API_DOMAIN, basePath: "/auth", plugins: [bearerClient()] })`. On sign-in the token is read from the `set-auth-token` response header, stored in extension-scoped storage, and sent as `Authorization: Bearer <token>` on every API call. No cookies.
- **Web view (same React app, `VITE_TARGET=web`):** identical client without the bearer plugin — cookie session (`credentials: "include"`), the default transport.
- **Internal tool:** `better-auth/react` (`createAuthClient` + `useSession`) with a **small custom email/password login form** (Better Auth ships no prebuilt UI — this replaces v0.3's SuperTokens prebuilt screen, §13). Cookie session. If 2FA is enabled for the admin, the form includes the TOTP step from the `twoFactor` client plugin.

**Cookie/site topology (binding):** dev — all web clients run on `localhost` ports (5173/5174) against `localhost:3001`; same host = same site, so default `sameSite=lax` cookies work over HTTP. Prod — all three web surfaces MUST be subdomains of one registrable domain (e.g. `api.insightrx.example`, `app.…`, `admin.…`) so `sameSite=lax; secure` works (`COOKIE_DOMAIN`/`COOKIE_SECURE` env). Cross-site hosting (different registrable domains) is NOT supported in V1 — it would require `SameSite=None` and is explicitly out of scope. The extension is unaffected (bearer transfer).

### 4.5 Express wiring order (exact)

```typescript
import { toNodeHandler } from "better-auth/node";
app.use(cors({ origin: ALLOWED_ORIGINS, allowedHeaders: ["content-type", "authorization", "x-account-id"], credentials: true }));
app.use("/auth", authRateLimiter);          // 10 req/min/IP — BEFORE the Better Auth handler
app.all("/auth/*", toNodeHandler(auth));     // Better Auth — parses its own bodies; MUST come before express.json()
app.use(express.json());                     // app routes only, AFTER the auth handler (Better Auth + Express gotcha)
app.post("/api/v1/upload", resolveLocalUser, requireRole([...]), uploadRateLimiter, multerSingle, uploadHandler);
// ...remaining /api/v1 routes, each starting with resolveLocalUser...
app.use(errorHandler());
```

`resolveLocalUser` replaces v0.3's `verifySession()` + `resolveLocalUser` pair — it both validates the Better Auth session and loads the local user in one step. `ALLOWED_ORIGINS` is an env CSV: internal tool origin, web view origin, extension origin (deterministic — §11).

### 4.6 Rate limits & session policy

`/auth/*`: 10 req/min/IP (express-rate-limit, in addition to Better Auth's built-in rate limiting). `/upload`: 10 req/hour/user. Session: **12h idle ceiling** (`session.expiresIn = 12h`, sliding via `updateAge`). Password policy for admin-created users: ≥12 chars, not in a small denylist (enforced in the §6 provisioning call). Visible signed-in-as + Sign out in the popup footer; sign-out calls `authClient.signOut()` and clears the bearer token from extension storage.

---

## Section 5: Database Schema (canonical)

### 5.0 MySQL deltas from v0.3 (read before the schema)

The schema below is v0.3's, ported to MySQL 8. Four engine-specific changes — everything else is identical:
1. **Provider:** `datasource db { provider = "mysql" }`. Create the database with `CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci` (full Unicode; case-insensitive collation, which is fine — `email`/`name` uniqueness becoming case-insensitive is desirable, and `normalizedName`/`AccountAlias.normalized` are already lowercased by `normalizeName()`).
2. **No scalar lists:** Prisma scalar lists (`String[]`) are **not supported on MySQL**. `Account.locations` becomes `Json?` (a JSON array). (If structured querying of locations is ever needed, promote to a child table — not required in V1.)
3. **No partial/filtered indexes:** MySQL has no `CREATE UNIQUE INDEX ... WHERE`. The "globally-unique among ACTIVE referrals" invariant is enforced with a **generated column + plain unique index** instead (see the `Referral` note below). Same guarantee, MySQL-native.
4. **`@db.Date` is retained** — it maps to MySQL `DATE`; the §5.1 controlled-UTC-midnight write/read discipline is unchanged.

Better Auth's own tables (`user`, `session`, `account`, `verification`, + `twoFactor` when enabled) live in this same database, generated by Better Auth's schema and committed alongside `prisma/schema.prisma` (§4.0). They are distinct from the app models below.

```prisma
generator client { provider = "prisma-client-js" }
datasource db { provider = "mysql"; url = env("DATABASE_URL") }   // mysql://… (was postgresql://…)

enum Role { INTERNAL_ADMIN INTERNAL_STAFF PROVIDER_STAFF }
enum WorkflowPhase { SUBMISSION INTAKE_PA TELEHEALTH FULFILLMENT_DELIVERY }
enum NextActionOwner { INSIGHT PROVIDER_OFFICE PATIENT PAYER CARRIER NONE }
enum BatchStatus { PENDING VALIDATED PUBLISHING PUBLISHED FAILED }
enum ErrorSeverity { CRITICAL WARNING }
enum StatusEventType { CREATED STATUS_CHANGED PHASE_CHANGED OWNER_CHANGED DEACTIVATED REACTIVATED }
enum ProviderStatus {
  PHYSICIAN_ENROLLMENT_PENDING REFERRAL_RECEIVED ERX_RECEIVED MISSING_INFORMATION
  INTAKE_IN_PROGRESS PA_INITIATED PA_PENDING PA_APPROVED PA_DENIED
  TELEHEALTH_PENDING TELEHEALTH_SCHEDULED TELEHEALTH_NO_SHOW TELEHEALTH_COMPLETED
  ADDRESS_CONFIRMATION_PENDING FULFILLMENT_IN_PROGRESS MEDICATION_SHIPPED DELIVERED CLOSED
}

model Account {
  id             String   @id @default(uuid())
  name           String   @unique
  normalizedName String   @unique   // maintained on create/edit via normalizeName(); collision → 409
  practiceNpi    String?  @unique
  locations   Json?              // JSON array of strings (MySQL has no scalar lists — §5.0)
  specialty   String?
  isActive    Boolean  @default(true)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  aliases     AccountAlias[]
  users       UserAccount[]
  referrals   Referral[]
}

model AccountAlias {
  id         String   @id @default(uuid())
  alias      String   @unique
  normalized String   @unique          // collisions rejected at creation with a clear error
  accountId  String
  account    Account  @relation(fields: [accountId], references: [id])
  createdAt  DateTime @default(now())
}

model User {
  id              String  @id @default(uuid())
  betterAuthUserId String @unique   // FK reference to Better Auth's user table (was supertokensId — §4.0)
  email         String    @unique
  firstName     String
  lastName      String
  role          Role
  isActive      Boolean   @default(true)
  lastLoginAt   DateTime?
  digestOptIn   Boolean   @default(true)   // provider users; email digest §14
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
  accounts      UserAccount[]
  fileBatches   FileBatch[]
  auditLogs     AuditLog[]
  adoptionEvents AdoptionEvent[]
  digestDeliveries DigestDelivery[]
}

model UserAccount {
  userId    String
  accountId String
  user      User    @relation(fields: [userId], references: [id])
  account   Account @relation(fields: [accountId], references: [id])
  @@id([userId, accountId])
}

model StatusMapping {
  id             String         @id @default(uuid())
  internalStatus String         @unique   // stored canonicalized: trimmed, lowercased
  providerStatus ProviderStatus
  phase          WorkflowPhase
  defaultOwner   NextActionOwner
  createdAt      DateTime       @default(now())
}

model FileBatch {
  id            String      @id @default(uuid())
  fileName      String
  schemaVersion String?                 // optional handshake from file metadata
  recordCount   Int         @default(0)
  errorCount    Int         @default(0)
  warningCount  Int         @default(0)
  status        BatchStatus @default(PENDING)
  uploadedById  String
  uploadedBy    User        @relation(fields: [uploadedById], references: [id])
  publishedAt   DateTime?
  failureReason String?
  createdAt     DateTime    @default(now())
  referrals     Referral[]
  errors        ValidationError[]
  events        ReferralStatusEvent[]
}

model ValidationError {
  id        String        @id @default(uuid())
  batchId   String
  batch     FileBatch     @relation(fields: [batchId], references: [id])
  rowNumber Int                         // matches the row number the admin sees in Excel
  field     String
  rawValue  String?                     // the offending value, shown to the admin
  message   String                      // problem + cause + fix copy (§10.4)
  severity  ErrorSeverity
  createdAt DateTime      @default(now())
}

model Referral {
  id               String          @id @default(uuid())
  referralId       String
  accountId        String
  account          Account         @relation(fields: [accountId], references: [id])
  batchId          String          // semantics: "last published by" — not full batch history
  batch            FileBatch       @relation(fields: [batchId], references: [id])
  prescriberName   String
  prescriberNpi    String?
  patientFirstName String
  patientLastName  String
  patientDob       DateTime?       @db.Date   // DATE-ONLY. Never round-trip through new Date()/toLocaleDateString.
  medicationName   String?
  providerStatus   ProviderStatus
  workflowPhase    WorkflowPhase
  nextActionOwner  NextActionOwner
  statusNote       String?
  statusUpdatedAt  DateTime
  isActive         Boolean         @default(true)
  createdAt        DateTime        @default(now())
  updatedAt        DateTime        @updatedAt
  events           ReferralStatusEvent[]
  @@unique([referralId, accountId])
  // INVARIANT: hospital referral IDs are globally unique among ACTIVE rows (PRD: "stable unique identifier").
  // MySQL has no partial index, so enforce it with a generated column + plain unique index, added by
  // a raw-SQL step in the migration (MySQL treats multiple NULLs as distinct, so inactive rows never collide):
  //   ALTER TABLE `Referral`
  //     ADD COLUMN `activeReferralKey` VARCHAR(191)
  //       GENERATED ALWAYS AS (IF(`isActive`, `referralId`, NULL)) STORED,
  //     ADD UNIQUE INDEX `referral_active_global` (`activeReferralKey`);
  // Historical inactive rows may share a referralId (account-move history, §9.4).
  @@index([accountId, isActive, statusUpdatedAt])
  @@index([accountId, providerStatus])
}

model ReferralStatusEvent {
  id         String          @id @default(uuid())
  referralId String
  referral   Referral        @relation(fields: [referralId], references: [id])
  type       StatusEventType
  fromStatus ProviderStatus?
  toStatus   ProviderStatus?
  fromOwner  NextActionOwner?
  toOwner    NextActionOwner?
  batchId    String
  batch      FileBatch       @relation(fields: [batchId], references: [id])
  createdAt  DateTime        @default(now())
  @@index([referralId, createdAt])
  @@index([batchId, type])
}

model AdoptionEvent {
  id        String   @id @default(uuid())
  userId    String
  user      User     @relation(fields: [userId], references: [id])
  kind      String   // "LOGIN" | "PATIENTS_FETCH" — no patient identifiers
  accountId String?
  createdAt DateTime @default(now())
  @@index([userId, createdAt])
}

model DigestDelivery {
  id         String   @id @default(uuid())
  userId     String
  user       User     @relation(fields: [userId], references: [id])
  digestDate DateTime @db.Date
  status     String   // SENT | FAILED
  attempts   Int      @default(0)
  createdAt  DateTime @default(now())
  @@unique([userId, digestDate])   // one aggregated digest per user per day, all their accounts in sections
}

model AuditLog {
  id         String   @id @default(uuid())
  userId     String   // ALWAYS req.localUser.id — never the Better Auth user id
  user       User     @relation(fields: [userId], references: [id])
  action     String
  resource   String
  resourceId String?
  details    Json?
  createdAt  DateTime @default(now())
}
```

### 5.1 Date discipline (boundary convention)
Date-only values cross three boundaries — fix the representation at each: (1) **Parser/DTOs:** date-only values are `YYYY-MM-DD` STRINGS (Excel serials converted with epoch 1899-12-30, timezone-naive). (2) **DB write:** convert the string to `new Date(s + "T00:00:00.000Z")` (controlled UTC midnight) for Prisma `@db.Date` — Prisma truncates to the date. (3) **API response:** serialize `patientDob` back to `YYYY-MM-DD` via `toISOString().slice(0,10)` before sending; clients format the string directly (`MM/DD/****` mask in lists) and NEVER pass it through `new Date()`/`toLocaleDateString()`. A DOB stored on the East Coast must render identically on the West Coast — an off-by-one DOB is a patient-identification hazard.

---

## Section 6: User Provisioning

### 6.1 Create user (internal admin only)
`POST /api/v1/users` — validate role + accountIds against zod BEFORE touching Better Auth. Then create the Better Auth user server-side (`auth.api.createUser` via the admin plugin / `auth.api.signUpEmail`); on success, create the app `User` (with `betterAuthUserId`) + `UserAccount` rows. **If the Prisma write fails, delete the just-created Better Auth user (compensating cleanup)** — orphan Better Auth users must not be able to sign in (and cannot anyway: 4.1 fails closed — no local `User` row → 401). Audit-log with `req.localUser.id`.

### 6.2 Deactivate
`DELETE /api/v1/users/:id` soft-deactivates (`isActive=false`). Takes effect on the user's next request (4.2), not at next session expiry.

### 6.3 Password reset
Admin-issued: admin sets a temp password via the Better Auth server API (`auth.api.setUserPassword` / admin plugin); user is told out-of-band. Public reset endpoints are disabled (4.1).

### 6.4 Production bootstrap
No seeded users in production. First admin is created by a one-time `pnpm bootstrap:admin` script reading `BOOTSTRAP_ADMIN_EMAIL`/`BOOTSTRAP_ADMIN_PASSWORD` env vars, refusing to run if any INTERNAL_ADMIN exists. Seed credentials in §19 are LOCAL-ONLY.

---

## Section 7: API Endpoints

Base `/api/v1`. Auth handled by Better Auth at `/auth/*` (in-process, §4.5) — no custom auth routes. All protected routes start with `resolveLocalUser` (validates the Better Auth session + loads the local user) → (`accountScope` | `requireRole`).

### Provider-facing (extension + web view)
- `GET /patients` (header `X-Account-Id` required) → `{ account, lastPublishedAt, summary: { totalActive, newToday, needsAction, paPending }, patients: [...] }`. Summary computed server-side from §1.4 predicates; rows capped at 500, sorted needs-action-pinned then `statusUpdatedAt` desc; query params `?status=&phase=&search=` (search = case-insensitive patient-name contains). Logs an `AdoptionEvent(PATIENTS_FETCH)`.
- `GET /patients/:id` → detail incl. status explanation key, `events` (status history), guaranteed fields always, conditional fields (history) only when present. Cross-account or unknown id → **404** (never 403; no existence leak).
- `GET /healthz` (public) → `{ db: "ok", auth: "ok", minExtensionVersion: "1.0.0" }` (single endpoint serves both readiness and client version policy; `auth` reflects Better Auth in-process readiness).
- `GET /me` (protected: `resolveLocalUser`) → `{ email, role, accountIds }` — used by clients post-login and as the Step 3 auth probe.

### Internal-facing (`requireRole([INTERNAL_ADMIN, INTERNAL_STAFF])`)
- `POST /upload` — multipart (multer, 10MB limit, extension+MIME check `.csv/.xlsx`). Parses (§10), validates, persists FileBatch + ValidationErrors. Returns counts, header-resolution report, errors (first 200 + totals), account preview, `canPublish`.
- `GET /upload/:batchId/preview` — account-first diff: per account `{ new, statusChanged, deactivated, needsAction }` in operator language, plus unmatched rows with did-you-mean suggestions.
- `GET /upload/:batchId/errors.csv` — downloadable error report (rowNumber, field, rawValue, problem/cause/fix).
- `POST /upload/:batchId/publish` — see §9.
- `GET /overview` — per-account cards: activeReferrals, newToday, needsAction, lastPublishedAt, lastProviderLoginAt. (Health labels are NOT in V1 — cut at final gate; formulas preserved in TODOS.md.)
- Accounts CRUD + `POST /accounts/:id/aliases` (alias creation validates normalized uniqueness; collision → 409 with the conflicting account named).
- Users CRUD per §6. `GET /batches` — batch history (who, file, counts, result, failureReason).

---

## Section 8: Account Matching

```typescript
// punctuation → SPACE (not empty: "Smith-Jones" must normalize like "Smith Jones"), then collapse whitespace
export function normalizeName(name: string): string {
  return name.toLowerCase().trim()
    .replace(/[.,\/#!$%\^&\*;:{}=\-_`~()]/g, " ")
    .replace(/\b(llc|inc|corp|pc|pllc|pa|md|do|dba)\b/gi, "")
    .replace(/\s+/g, " ").trim();
}
```

Matching priority (NPI-first — a typo'd name must not route patient data to the wrong practice). **Only `isActive` accounts participate in matching.**
1. `practice_npi` exact match on `Account.practiceNpi` (when the file provides it)
2. Exact `account_name` match on `Account.name`
3. `normalizeName(account_name)` match on `Account.normalizedName`
4. `normalizeName(account_name)` match on `AccountAlias.normalized`
5. Unmatched → CRITICAL error with did-you-mean suggestions (Levenshtein ≤ 2 on normalized names/aliases)

Collision integrity: `Account.normalizedName` and `AccountAlias.normalized` are each unique AND alias creation rejects (409, naming the conflict) any normalized value that equals another account's `normalizedName` — an alias can never shadow a different practice's name. Aliases are NEVER auto-created. Resolving an unmatched name is an explicit admin action in the preview UI (shows account NPI + prescriber context), which creates the alias and re-validates.

---

## Section 9: Publish Semantics

Inside one Prisma transaction with the `FileBatch` row locked:
1. Idempotency guard + lock in one statement (Prisma has no row-lock API on `findUnique`): `updateMany({ where: { id: batchId, status: "VALIDATED" }, data: { status: "PUBLISHING" } })` — affected count 0 → **409** ("Batch already published" / "Batch failed validation" / "Publish already in progress"). This conditional update IS the lock; concurrent publishers lose the race deterministically.
2. Upsert referrals by `(referralId, accountId)` in chunks of 200 (explicit transaction timeout 60s; validation caps files at 10,000 rows).
3. Write `ReferralStatusEvent` rows: `CREATED` for new; `STATUS_CHANGED`/`PHASE_CHANGED`/`OWNER_CHANGED` by diffing the locked prior row.
4. **Account-move guard:** if a `referralId` arrives under account B while an active row exists under account A, deactivate A's row (`DEACTIVATED` event) — a corrected assignment must not leave patient data visible to the wrong practice. Surfaced in the diff preview.
5. **Bulk deactivation is DISABLED by default.** `PUBLISH_DEACTIVATE_MISSING=false` until ops answers cumulative-vs-incremental in writing (owner: Insight operations admin). When enabled, deactivation is scoped to accounts present in the file. The preview's deactivation count requires an explicit confirmation checkbox when > 0. Referrals are never deleted.
6. Transition `PUBLISHING → PUBLISHED` + `publishedAt` inside the transaction. **Failure handling is two-phase:** if the transaction throws, it rolls back ALL of steps 2–6; the catch block — OUTSIDE the transaction — then writes `FileBatch.status = FAILED` + `failureReason` as a separate update. UI shows "Publish failed — no changes were applied."
7. Post-commit: audit log, ops-alert check (§16), digest enqueue (§14).

---

## Section 10: File Contract (the ops admin's API — full reference in docs/file-contract.md)

### 10.1 Formats
CSV: UTF-8 with BOM tolerated (`bom: true`); non-UTF-8 input is detected by strict UTF-8 decode failure and converted from Windows-1252 via `iconv-lite` (pinned, §2); anything else → clear encoding error. XLSX: the workbook must contain **exactly one non-empty sheet** — zero or more than one non-empty sheet is a validation error naming the sheets. 10MB upload cap, 10,000 row cap; XLSX zip-bomb guard: before workbook load, inspect ZIP entries with `yauzl` (pinned, §2) and reject if total uncompressed size > 50MB or any entry's compression ratio > 100×. Blank rows skipped. Duplicate headers → error. Formula cells: display values. Reported row numbers match what the admin sees in Excel (header = row 1).

### 10.2 Columns (case-insensitive, trimmed; canonical header → aliases)

| Maps to | Canonical header (aliases) | Required |
|---|---|---|
| referralId | `referral_id` (Referral ID) | YES |
| accountName | `account_name` (Account Name, Practice Name) | YES |
| practiceNpi | `practice_npi` (Practice NPI) | Recommended |
| prescriberName | `prescriber_name` (Prescriber Name, Provider Name) | YES |
| prescriberNpi | `prescriber_npi` (Prescriber NPI) | No |
| patientFirstName | `patient_first_name` (First Name) | YES |
| patientLastName | `patient_last_name` (Last Name) | YES |
| patientDob | `patient_dob` (DOB, Date of Birth) | YES |
| medicationName | `medication_name` (Medication, Drug Name) | Recommended |
| internalStatus | `internal_status` (Internal Status) | **YES** |
| providerStatus | `provider_status` (Status) | No — optional override of the mapped status |
| workflowPhase | `workflow_phase` (Phase) | No — IGNORED as an override: phase is ALWAYS derived from the (possibly overridden) provider status via §1.3. If present and contradicting the derived phase → WARNING naming both values |
| nextActionOwner | `next_action_owner` (Next Action, Action Owner) | No — override allowed (it drives needs-action); overridden values are highlighted in the preview |
| statusNote | `status_note` (Notes) | No (≤300 chars; shown provider-facing — keep controlled) |
| statusUpdatedAt | `status_updated_at` (Last Updated) | YES |
| schemaVersion | `schema_version` | No — when present, must be `v1` |

Ambiguous generic headers are resolved and ECHOED: the upload response includes a header-resolution report ("`Status` interpreted as `provider_status`; `internal_status` is still required"). Missing required columns fail the whole upload, listing exactly what's missing and the accepted aliases.

### 10.3 Value normalization
Dates: `YYYY-MM-DD`, `M/D/YYYY`, Excel serial (epoch 1899-12-30). Enums (`internal_status`, `workflow_phase`, `next_action_owner`): trimmed, case-insensitive ("Provider Office " → `PROVIDER_OFFICE`). NPI: digits only after stripping spaces/hyphens; must be 10 digits when present. The preview shows canonicalized values so ops sees what the system understood.

### 10.4 Error copy (every error = problem + cause + fix; full table in docs/file-contract.md)
Example row: `Row 14 · patient_dob · "13/44/2025" · Problem: invalid date · Cause: month/day out of range · Fix: use M/D/YYYY, e.g. 3/14/2025.` Unknown `internal_status` → "Problem: status not recognized · Cause: '<value>' has no mapping · Fix: correct the cell, or ask an Insight admin to add a mapping for it." Duplicate `referral_id` → both row numbers. CRITICAL blocks publish; WARNING does not (missing medication/prescriber NPI).

### 10.5 Templates
`fixtures/file-contract-v1.csv` and `.xlsx` are the canonical templates, linked from the Upload page.

---

## Section 11: Chrome Extension Manifest (MV3)

```json
{
  "manifest_version": 3,
  "name": "Insight Specialty Pharmacy",
  "version": "1.0.0",
  "key": "<COMMITTED_DEV_PUBLIC_KEY>",
  "description": "Patient referral status updates from Insight Specialty Pharmacy",
  "permissions": ["storage"],
  "action": { "default_popup": "src/popup/index.html", "default_icon": { "16": "icons/icon-16.png", "48": "icons/icon-48.png", "128": "icons/icon-128.png" } },
  "background": { "service_worker": "src/background/service-worker.ts", "type": "module" },
  "icons": { "16": "icons/icon-16.png", "48": "icons/icon-48.png", "128": "icons/icon-128.png" }
}
```

Build Step 1 includes `pnpm gen:extension-key`: generates the keypair once (openssl commands in `docs/release-runbook.md`), writes the public `key` into manifest.json, derives the stable extension ID, prints it in the README, and rewrites `EXTENSION_ORIGIN`/`ALLOWED_ORIGINS` in `.env.example` with the real value. Key + updated files are **committed in Step 1**, so every subsequent clone has zero placeholders. (`<derived-from-committed-key>` in §20 denotes the value this step bakes in — it exists only until Step 1's first run.) @crxjs/vite-plugin owns entry wiring. The popup footer shows the extension version; `GET /healthz` returns `minExtensionVersion` and the popup shows "Update required" when older. Badge: set from the popup on open (needs-action count via message to the SW; `chrome.action.setBadgeText`); a live SW-side fetch is V2 (TODOS — the auth client / bearer-token flow runs in the popup, not in an MV3 service worker).

Popup: 420×600. Web view target: same app, full viewport, served at `WEB_VIEW_ORIGIN`.

---

## Section 12: Provider UI (extension popup + web view)

### 12.1 Design system
Inter (retained deliberately — operational legibility over brand expressiveness). Background `#FAFAFA`, cards `#FFFFFF` r8, text `#111827`/`#6B7280`, accent indigo `#4F46E5` used sparingly, amber strictly reserved for needs-action/staleness. Chips r16. 4px spacing base. Dense list-first layout; no gradients, no icon-circles, no emoji; cards only where the card is the interaction.

### 12.2 Hierarchy (top → bottom; patient list above the fold, ≥5 rows visible)
1. **Header (one 32px line):** "Good morning, GI Medical Services · Updated 2h ago". Account name comes from the selected account. Sub-48h staleness is a subdued "as of" — amber + icon only at ≥48h, copy: "Insight hasn't published an update since Tue 4:12 PM. Your patient list is accurate as of then."
2. **AccountPicker context bar** (only when `accountIds.length > 1`): fixed bar, current account unmistakable, switch = confirm + refetch; last selection persisted.
3. **NeedsActionBanner:** full-width amber card "2 patients need your office's action →" (tap = activate Needs Action filter). Hidden at zero.
4. **StatSummary strip:** Total active · New today · PA pending (compact, each tappable = synced filter).
5. **FilterChips (7):** All, New Today, Needs Action, PA Pending, PA Approved, Telehealth, Shipping — each with count, active chip filled indigo, click active to clear. Buckets per §1.3.
6. **SearchBar:** patient-name search, 200ms debounce, clear button.
7. **PatientList:** pinned "Needs your action (N)" group sorted by `statusUpdatedAt` desc, then the rest by `statusUpdatedAt` desc.

### 12.3 PatientRow
Name (16px semibold) + medication (14px secondary); second line **DOB masked `MM/DD/****`** + prescriber (12px); third line status chip (color per §1.3) + "Next: <owner>"; relative timestamp right-aligned.

### 12.4 PatientDetail
Full-width drawer sliding from the right over the list; back chevron + Esc dismiss; list scroll/filter state preserved; focus trapped and restored. Contents: name (18px), **full DOB**, medication, prescriber, status chip + plain-language explanation (§1.3), next action owner prominent; if `PROVIDER_OFFICE`: highlighted action box with `statusNote`; status history (from events) when present — hidden, not faked, when absent; "Last update from Insight file: <date time>".

### 12.5 States (every component implements its column)

| Surface | Loading | Empty | Error | Auth |
|---|---|---|---|---|
| Popup initial | header + 5 skeleton rows | "No active referrals right now. When you send patients to Insight, they will appear here." | "Can't reach Insight." + Retry | 401→LoginForm; 403 deactivated→"Account disabled — contact Insight." |
| Filtered list | — | "No patients match this filter." + clear-filter link | — | — |
| Detail | inline skeleton | — | inline error + retry | — |

### 12.6 FirstRunCard (replaces the PRD's 3-screen onboarding)
One dismissable card after first login: data updates once daily when Insight publishes (not real-time) + what "Needs office action" means. Tooltip on the banner repeats the second fact.

### 12.7 Accessibility (acceptance criteria, not aspiration)
All chips/filters keyboard-reachable; drawer focus-trap + Esc + focus restore; 44px minimum targets; WCAG AA contrast; visible focus ring; aria-labels on counts and chips ("5 patients, prior authorization pending").

---

## Section 13: Internal Tool UI

Tokens from §12.1. A small custom Better Auth email/password login form (no prebuilt vendor UI — §4.4), styled with the §12.1 tokens; includes the TOTP step when 2FA is enabled for the admin.

- **Overview (route `/`):** per-account cards — active referrals, new today, needs-action, last publish time, last provider login — plus a "latest batch" banner (file, time, result). Answers "did today's file land, who needs attention." No health labels in V1.
- **Upload (4-step stepper):** Upload (drag/drop, .csv/.xlsx, links to templates) → Validate (progress state; counts; header-resolution report; error table with rawValue + problem/cause/fix; errors.csv download) → Preview diff (account-first operator language: "12 patients updated, 3 newly visible to GI Medical Services, 1 no longer active, 4 require provider office action"; unmatched-account resolution UI with did-you-mean; deactivation confirmation checkbox when count > 0) → Publish (button locked in-flight; success links to Overview; failure banner "Publish failed — no changes were applied" + failureReason).
- **Accounts:** table; create/edit; alias management (collision errors named); empty/error states per a standard table pattern.
- **Users:** table with lastLoginAt; create (email, temp password, name, role, accounts multi-select); deactivate; reset temp password.
- **Batches:** history table (who, file, counts, result, failureReason).

---

## Section 14: Email Digest

On publish: collect provider users with `digestOptIn` whose accounts had changes; for each user, send **one aggregated digest per calendar day** (enforced by `DigestDelivery @@unique([userId, digestDate])`) with a section per affected account. Subject: "Insight updates: 2 need your action". Body: per-account counts by bucket + patient initials + status for needs-action items only + deep link to the web view. Unsubscribe: signed token (HMAC over userId with `DIGEST_TOKEN_SECRET`, no expiry needed) → `GET /digest-unsubscribe?token=` flips `digestOptIn`. Transport: SMTP (nodemailer); failure → `DigestDelivery.status=FAILED`, one retry, logged; never blocks publish.

---

## Section 15: V1 Scope Boundary — Internal Testing Only

**V1 is tested internally at Insight only (user decision 2026-06-11, supersedes the earlier UC1 gate decision).** No external provider office touches the system, and internal testing uses the synthetic seed data — no real patient data is loaded in V1. Consequently the HIPAA/compliance workstream (BAA chain, encryption-at-rest attestation, retention policy, breach-response procedures, compliance review) is **out of V1 scope entirely** and lives in TODOS.md as the hard blocker for any external pilot or any load of real patient data.

Engineering hygiene that happens to overlap with compliance stays in V1 on its own merits (it's already specified where it lives): fail-closed auth + account isolation (§4), audit/adoption event logging (§5), 12h idle session + sign-out (§4.6), DOB masked in list views (§12.3), no patient names/DOBs in console logs (§16), TLS via the hosting platform (§17). These are correctness and good practice, not compliance claims.

---

## Section 16: Observability & Ops Alerting

Structured console lines (`[upload] batch=... rows=... errors=... ms=...`) at upload/validate/publish/digest. node-cron checks: (a) **publish-failure** → immediate email/webhook to `OPS_ALERT_EMAIL`; (b) **zero-row upload** → same; (c) **no successful publish by `OPS_ALERT_DEADLINE` (default 11:00 local) on business days** → same. Batch history (§13) is the support surface; AuditLog + ValidationError + ReferralStatusEvent rows make any "what happened 3 weeks ago" question answerable from the DB.

---

## Section 17: Deployment

**Dev:** `docker-compose.yml` runs `mysql:8` (single app DB; Better Auth's tables live here too) and `axllent/mailpit` (SMTP :1025, web UI :8025 — digest/alert emails land here; Step 10 verifies against the Mailpit UI/API). There is **no auth-service container** — Better Auth runs in-process in the backend. Quick start: `docker compose up -d && cp .env.example .env && pnpm install && pnpm db:setup && pnpm dev` (db:setup = migrate + seed:foundation + seed:users + seed:referrals). `pnpm build:extension` → load unpacked → ID already matches `.env.example`.

**Prod env matrix (docs/release-runbook.md):** `DATABASE_URL` (`mysql://…`), `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL`, `API_DOMAIN`, `WEB_VIEW_ORIGIN`, `INTERNAL_TOOL_ORIGIN`, `EXTENSION_ORIGIN`, `ALLOWED_ORIGINS`, `COOKIE_DOMAIN`, `COOKIE_SECURE`, `SMTP_*`, `OPS_ALERT_EMAIL`, `OPS_ALERT_DEADLINE`, `PUBLISH_DEACTIVATE_MISSING`, `BOOTSTRAP_ADMIN_*`. Cookies: `secure`, `sameSite=lax` (internal tool + web view are first-party to the API domain or use subdomains). **Prod hosting target: Fly.io (HIPAA package) + self-hosted MySQL 8 + in-process Better Auth — full runbook in `docs/deployment-fly-mysql-runbook.md` and `docs/hosting-and-auth-migration-strategy.md`.**

**Order:** backup DB → `prisma migrate deploy` → deploy backend → deploy web/internal bundles → smoke (`pnpm smoke` against prod: signin, scoped patients, cross-tenant 404, healthz) → distribute extension zip per release-runbook checklist (signed/checksummed zip, plain-English release notes, version visible in popup footer, rep install/update checklist). Rollback: revert deploy; migrations are additive in V1; publish-level recovery via batch history + events.

---

## Section 18: Test Plan (ship-blocking)

Framework: vitest + supertest (backend; Better Auth runs in-process against a test MySQL — no auth-service container to orchestrate), Playwright for 3 E2E flows. Fixtures in `/fixtures`. `pnpm test` runs unit+integration; `pnpm test:e2e` runs Playwright.

**Ship-blocking suites (failures block any merge):**
1. **Cross-tenant isolation:** sarah cannot fetch mike's patient (404); `X-Account-Id` outside the DB-resolved `accountIds` → 404; provider with empty accountIds → 404; **undefined/missing `req.localUser` → denied, never an unfiltered query**; deactivated ACCOUNT → dropped from `req.localUser.accountIds`/`/me`, rejected by accountScope, excluded from upload matching; alias colliding with another account's normalizedName → 409 at creation.
2. **Fail-closed auth (Better Auth):** a Better Auth user with no matching app `User` row (orphan) cannot establish an authorized session (401, no local row); a valid Better Auth session for an `isActive=false` local user → 403 on next request; **bearer-token** path (extension) and **cookie** path (web/internal) both enforce the same `resolveLocalUser` checks; role/account reassignment takes effect on the **next request** (DB-resolved each request — no stale-claims window to test for); optional TOTP 2FA gate for INTERNAL_ADMIN when enabled.
3. **Publish transaction:** mid-batch failure rolls back to zero changes (batch FAILED); concurrent publish → one 409; re-publish published batch → 409; account-move deactivates the old row + emits DEACTIVATED; deactivation default-off honored.
4. **Parser fixtures:** every file in `/fixtures` produces its exact expected outcome (incl. BOM, win-1252, serial dates with no TZ shift, multi-sheet error, dup headers, dup referral_id with both row numbers, unknown status with mapping hint).

**Unit:** normalizeName (suffixes, punctuation→space, hyphens; explicit cases: "GI, LLC", "GI LLC MD", "Smith-Jones Rheumatology"), accountMatcher (NPI>name>alias priority, collision rejection, did-you-mean), validation (18-status mapping, enum canonicalization, dates, caps), predicates (needsAction, newToday incl. republish behavior), digest assembly (data minimization: counts/initials/statuses only; one-per-day).
**Integration:** upload→validate→preview→publish→GET /patients end-to-end; overview counts; user provisioning compensation; alias creation flow.
**E2E (Playwright):** provider daily flow (login → needs-action → detail → sign-out); admin publish ritual (upload fixture → diff → publish → extension shows data); multi-account picker switch.
**Flakiness rules:** Better Auth runs in-process against a disposable test MySQL (no external auth service to flake); clock injection for staleness tests; no ordering-dependent assertions.

---

## Section 19: Build Order

**Step 1 — Monorepo + infra.** Workspace, all package.jsons with pinned deps (§2), docker-compose.yml (`mysql:8` + mailpit — no auth service), shared package with §1.3 table, root scripts incl. `gen:extension-key`. Run `pnpm gen:extension-key` once: commits the manifest `key` and bakes the derived extension origin into `.env.example` (§11). *Verify:* `docker compose up -d` — both services healthy (Mailpit UI reachable at :8025); `pnpm install` clean; `.env.example` contains no placeholders.
**Step 2 — Schema + foundation seed.** §5 schema verbatim; `prisma migrate dev`; `seed:foundation` (2 accounts: "GI Medical Services" NPI 1234567890 w/ aliases "GI Med Svc", "GI Medical Svc LLC"; "Metro Rheumatology Associates" NPI 0987654321; provisional status mappings). *Verify:* migrate clean; tables populated; unit tests for normalizeName pass.
**Step 3 — Backend core + auth.** config, prisma.ts, auth.ts (Better Auth, 4.1), middlewares (4.2/4.3), healthz, rate limits, wiring (4.5 — `toNodeHandler(auth)` before `express.json()`); `seed:users` (admin@insightpharmacy.com / sarah@gimedical.com / mike@metrorheum.com — LOCAL-ONLY) then `seed:referrals` (synthetic batch, 17 referrals + CREATED events). *Verify:* signin 200 then `GET /me` returns role + accountIds; fail-closed + isActive integration tests pass.
**Step 4 — Ingestion.** fileParser (§10.1/10.3), validation (§10.2/10.4 + error copy), statusMapping, accountMatcher (§8), upload route + parse/validate/error preview + errors.csv. (Publish-aware diff fields — new/changed/deactivated — are NOT in this step.) *Verify:* every fixture produces expected results; `pnpm test` green.
**Step 5 — Publish.** §9 in full + the publish-aware diff preview endpoint (`GET /upload/:batchId/preview` gains new/statusChanged/deactivated/account-move fields, which depend on §9 diffing). *Verify:* publish suite (ship-blocking #3) green; fixture publish visible via the MySQL client (`mysql`).
**Step 6 — Provider API.** patients routes + predicates + summary + adoption events. *Verify:* cross-tenant suite (ship-blocking #1) green.
**Step 7 — Accounts/Users/Overview/Batches APIs.** §6 + §7 internal routes. *Verify:* provisioning compensation test green; overview counts correct.
**Step 8 — Extension.** §11 manifest + crxjs config; §12 components/states; built also as web target. *Verify:* load unpacked with pre-derived ID; login as sarah; only GI Medical patients; filters/search/detail/picker/sign-out work; states render (network-off shows Retry).
**Step 9 — Internal tool.** §13 pages. *Verify:* full upload→validate→preview→publish ritual on fixtures; extension reflects publish.
**Step 10 — Digest + alerts + docs.** §14, §16, the four docs + README, smoke.ts. *Verify:* digest visible in the Mailpit UI/API on publish (one per user with account sections); alert fires on simulated missed publish; `pnpm smoke` green.
**Step 11 — Full verification.** §21 checklist + `pnpm test && pnpm test:e2e && pnpm smoke`.

---

## Section 20: Environment Variables

The block below shows the POST-Step-1 state of `.env.example` (committed after `pnpm gen:extension-key` replaces the two `<derived…>` extension entries with real values — §11/§19 Step 1). After that one-time step, clones have zero placeholders.

```
DATABASE_URL=mysql://root:password@localhost:3306/insight_pharmacy
BETTER_AUTH_SECRET=dev-only-change-in-prod-0123456789abcdef
BETTER_AUTH_URL=http://localhost:3001
API_DOMAIN=http://localhost:3001
INTERNAL_TOOL_ORIGIN=http://localhost:5173
WEB_VIEW_ORIGIN=http://localhost:5174
EXTENSION_ORIGIN=chrome-extension://<derived-from-committed-key>
ALLOWED_ORIGINS=http://localhost:5173,http://localhost:5174,chrome-extension://<derived>
PORT=3001
SMTP_HOST=localhost
SMTP_PORT=1025
OPS_ALERT_EMAIL=ops@insightpharmacy.test
OPS_ALERT_DEADLINE=11:00
PUBLISH_DEACTIVATE_MISSING=false
COOKIE_SECURE=false
VITE_API_DOMAIN=http://localhost:3001
```

(Prod adds `COOKIE_DOMAIN` and sets `COOKIE_SECURE=true`; `BETTER_AUTH_SECRET` must be a strong random value; `BETTER_AUTH_URL`/origins point at the real subdomains — §17.)

---

## Section 21: Acceptance Checklist

Auth & isolation: provider A cannot see provider B's data (404); deactivated user loses access on next request; undefined auth context (no `req.localUser`) denied; public signup + public password reset disabled (Better Auth `disableSignUp` + no reset endpoint); sign-out works; 12h idle expiry; rate limits active; bearer (extension) and cookie (web/internal) paths both fail-closed.
Ingestion: CSV+XLSX parse incl. BOM/win-1252/serial dates; missing columns fail with alias list; unknown internal_status names the fix; duplicate referral_id flagged with both rows; unmatched account → did-you-mean; header-resolution report shown; errors.csv downloads.
Publish: 409 on re-publish; rollback leaves zero changes; account-move deactivates old row; deactivation off by default + confirmation checkbox; diff preview in operator language; events written.
Provider UI: loads <2s; needs-action banner + pinned group; counts = chips = filters synced; search works; DOB masked in list/full in detail; status explanations per §1.3; first-run card; staleness copy rules; account picker (multi-account); empty/error/skeleton states; a11y criteria (§12.7).
Admin UI: overview cards correct; stepper states; batch history; users show lastLoginAt; alias collision rejected.
Digest & ops: digest on publish (≤1/user/day, data-minimized, unsubscribe); publish-failure + zero-row + missed-deadline alerts fire.
Scope boundary: V1 runs on synthetic seed data only; no real patient data loaded; HIPAA workstream confirmed present in TODOS.md as the external-pilot blocker (§15).
Docs & DX: quick start ≤30min on a clean machine; four docs exist and match behavior; `pnpm smoke` green.

---

## Section 22: Schema-Freeze Blockers (open ops questions — do not freeze §10 against guesses)

1. Obtain ≥3 **de-identified or synthetic schema-faithful** sample exports of the hospital sheet (real column headers, real status vocabulary, real formatting quirks — fake or scrubbed patient identifiers, consistent with §15's no-real-patient-data boundary); profile every column; update §10 + the StatusMapping seed from reality. Owner: Insight operations admin.
2. Written answer: is the daily file cumulative or incremental? Until answered, `PUBLISH_DEACTIVATE_MISSING=false` stands (manual deactivation only).

Build Steps 1–3 may proceed before these close; Steps 4+ should rebase §10/§1.2 on the samples if they contradict this contract.

---

## Changelog v0.3 → v0.4

Two engine swaps, **zero behavior change to the security model** (per `docs/hosting-and-auth-migration-strategy.md`):

- **Auth: SuperTokens → Better Auth, in-process.** Removed the `supertokens-postgresql` core container; Better Auth now runs inside the Express backend with the Prisma adapter (§4.1). `emailAndPassword` with `disableSignUp`, `bearer()` for the extension, cookie sessions for web/internal, optional `twoFactor()` for admins. `supertokens.ts` → `auth.ts`; `verifySession()` + `resolveLocalUser` collapse into a single `resolveLocalUser` that calls `auth.api.getSession()` then loads the local user (§4.2). Express wiring mounts `toNodeHandler(auth)` **before** `express.json()` (§4.5). Provisioning/reset now call the Better Auth server API (`auth.api.createUser`/`setUserPassword`) instead of SuperTokens recipe functions (§6).
- **Authorization model simplified.** No more access-token claims minted at login/refresh. `role` + active `accountIds` are resolved from the DB on every request, so **role/account/deactivation changes take effect on the next request** with no stale-claims window (§4.0). All fail-closed invariants preserved (401/401/403/404).
- **Database: PostgreSQL → self-hosted MySQL 8.** `provider = "mysql"`; `utf8mb4_0900_ai_ci`. Three Postgres-isms ported (§5.0): scalar list `Account.locations String[]` → `Json?`; the partial unique index `referral_active_global` → a generated-column + plain unique index (`IF(isActive, referralId, NULL)`); `@db.Date` retained. `User.supertokensId` → `User.betterAuthUserId @unique`.
- **Healthz** now reports `{ db, auth, minExtensionVersion }` (was `supertokens`). **Internal login** is a small custom Better Auth form (no prebuilt vendor UI). **Dev infra** is `mysql:8` + Mailpit only (no auth service). **Env** drops `SUPERTOKENS_*`, adds `BETTER_AUTH_SECRET`/`BETTER_AUTH_URL`/`COOKIE_DOMAIN`/`COOKIE_SECURE`; `DATABASE_URL` is now `mysql://…`. **Test plan** §18 replaces the dockerized-core suite with in-process Better Auth fail-closed cases (orphan user, bearer + cookie paths, optional TOTP). **Prod target:** Fly.io (HIPAA) + self-hosted MySQL (`docs/deployment-fly-mysql-runbook.md`).

Everything else (status model §1, ingestion §10, publish §9, provider UI §12, digest §14, scope/HIPAA boundary §15) is unchanged from v0.3.

---

## Changelog from v0.2

All 76 review decisions merged (audit trail: GSTACK REVIEW REPORT in the v0.2 file). Headlines: fail-closed session claims + request-time isActive (cross-tenant leak fix); User↔Account many-to-many + account picker; 18-status canon + backend mapping layer (internal_status required); transactional/locked/idempotent publish with account-move guard and default-off deactivation; ReferralStatusEvent history + server-side predicates; NPI-first matching, alias hygiene, never auto-alias; exceljs replaces xlsx (CVEs); parser contract (BOM/encodings/serial dates/sheet rule/caps); error copy + rawValue + errors.csv; UI hierarchy restructure + state matrices + SearchBar + sign-out + first-run card + masked DOB; docker-compose + pinned manifest key + seeds split + version pinning + tsx + fixtures + vitest; docs as a build step; web view + email digest (UC2); health labels cut to TODOS (UC3); rate limiting + password policy; ops alerting; production bootstrap.

**Post-gate user decision (2026-06-11, supersedes UC1):** V1 is internal-testing only on synthetic data — HIPAA workstream cut from V1 entirely, parked in TODOS.md as the hard blocker for any external pilot or real-patient-data load (§15).

**Codex review rounds on this document:** round 1 — 20 findings (7 P1) fixed: publish two-phase failure handling, signInPOST fail-closed mapping, signup/reset API-vs-function scoping, cookie/site topology, Mailpit service, extension-key generation step, exactly-one-sheet rule, schema FKs, date boundary convention, healthz contract, Express wiring, iconv-lite/yauzl, step 4/5 ownership, updateMany lock, client init configs. Round 2 — 10 findings (2 P1) fixed: inactive-account enforcement end-to-end, alias↔account normalizedName collision integrity, Closed-status invariant, phase-override removal, global referralId partial unique index, DigestDelivery model + signed unsubscribe tokens, env placeholder wording, Mailpit naming, multer/Playwright pinning. (Round-2 retention-job finding mooted by the HIPAA cut — retention now lives with the TODOS workstream.)
