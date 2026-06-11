<!-- /autoplan restore point: ~/.gstack/projects/InsightExtension/master-autoplan-restore-20260611-123741.md -->
# Insight Specialty Pharmacy Extension — V1 Build Specification

This document is a complete build specification for Claude Code to execute in terminal. Read top-to-bottom. Follow the architecture, file structure, schema, API contracts, and build order exactly.

---

## Section 0: Coding Principles (Karpathy Minimal Coding Contract)

Follow these four principles for every file, function, and decision in this project. These are non-negotiable behavioral rules.

### Principle 1: Think Before Coding

* Before writing any file, explicitly state what it does in a one-line comment at the top.
* If a requirement is ambiguous, implement the simplest interpretation. Do not guess at complex interpretations.
* Do not add error handling for scenarios that cannot occur given the data flow described in this spec.

### Principle 2: Simplicity First

* Implement the minimum code needed to satisfy each requirement.
* Do not add abstractions, patterns, or indirections that are not explicitly required.
* If a function can be written in 20 lines, do not write 60. If a component can be one file, do not split it into three.
* No utility libraries beyond what is listed in the tech stack. No lodash, no moment, no axios. Use native fetch, native Date, and built-in array methods.
* No class-based patterns. Use functions and plain objects.
* No environment-specific branching unless explicitly specified.

### Principle 3: Surgical Changes

* Each build step produces exactly the files listed for that step.
* Do not refactor previous steps while building a new step.
* Do not add "nice to have" code, comments, or documentation beyond what the spec requires.
* Match the naming conventions, file paths, and structures in this document exactly.

### Principle 4: Goal-Driven Execution

* Each build step has an explicit verification goal stated at the end.
* Do not proceed to the next step until the current step's verification passes.
* If a step fails verification, fix only the failing code. Do not rewrite unrelated files.
* Write tests only where the spec explicitly says to. Do not add test files unprompted.

### Anti-Patterns to Avoid

* Do not create barrel files (index.ts that re-exports).
* Do not create abstract base classes or generic service patterns.
* Do not add logging frameworks. Use console.log and console.error.
* Do not add request ID generation or correlation middleware.
* Do not add rate limiting, compression, or helmet unless specified.
* Do not create custom error classes. Use plain Error with descriptive messages.
* Do not add comments that repeat what the code does. Add comments only for non-obvious business logic.

---

## Section 1: Business Context

Insight Specialty Pharmacy is a hospital-based pharmacy that handles biologics and specialty medications. Provider offices send patient referrals to Insight. After sending a referral, office staff have no self-serve way to track status. They call or email.

This extension solves that. It gives office staff a daily status view of every patient they referred to Insight. It is a service feature of the pharmacy, not a standalone product. Distribution is handled by Insight field reps who install the extension directly on office workstations.

### Data Flow

1. Hospital tracks patient referrals in a Google Sheets workbook.
2. Insight staff exports that sheet as CSV or XLSX daily.
3. Staff uploads the file to an internal web tool.
4. Backend parses, validates, maps records to provider accounts.
5. Staff reviews and publishes.
6. Provider office staff open the Chrome extension and see their patients.

### SOP Workflow Phases (SOP-SPI-001 v2.0)

Phase 1 — Submission: CDTM agreement, eRx transmission, demographics and insurance. SLA same day.

Phase 2 — Intake and Prior Authorization: eRx logged and verified, triage tagging, PA coordination. SLA 24-48 hours, complex PA up to 2-3 weeks.

Phase 3 — Telehealth and Approval: scheduling, visit completion triggers fulfillment, insurance approval logged. SLA 24 hours.

Phase 4 — Fulfillment and Delivery: address confirmation, temperature-controlled packing, shipped with signature. SLA 24-48 hours.

---

## Section 2: Tech Stack

| Layer | Technology |
| --- | --- |
| Extension Frontend | Chrome Extension (Manifest V3), React 18, TypeScript, Tailwind CSS |
| Internal Tool Frontend | React 18, TypeScript, Tailwind CSS |
| Backend | Node.js, Express.js, TypeScript |
| Database | PostgreSQL 15+ |
| ORM | Prisma |
| Authentication | SuperTokens (EmailPassword recipe + Session recipe) |
| File Parsing | csv-parse (CSV), xlsx/SheetJS (XLSX) |
| Validation | zod |
| Build Tool | Vite |
| Package Manager | pnpm |
| Monorepo | pnpm workspaces |

### SuperTokens Configuration

* Use the SuperTokens managed core for development: `https://try.supertokens.io`
* For production: self-host SuperTokens Core or use SuperTokens managed service with a real connectionURI and apiKey.
* Recipes: EmailPassword.init() and Session.init()
* The Chrome extension uses header-based token transfer (`tokenTransferMethod: "header"`) because Chrome extensions cannot reliably use cookies cross-origin.
* The internal tool uses cookie-based token transfer (default) because it runs as a standard web app.

### Packages to Install

Backend:

```
supertokens-node
express
cors
@prisma/client
prisma
csv-parse
xlsx
zod
multer
typescript
ts-node
@types/express
@types/cors
@types/multer

```

Extension:

```
supertokens-web-js
react
react-dom
typescript
tailwindcss
@tailwindcss/vite

```

Internal Tool:

```
supertokens-auth-react
react
react-dom
react-router-dom
typescript
tailwindcss
@tailwindcss/vite

```

---

## Section 3: Project Structure

```
insight-pharmacy-extension/
├── pnpm-workspace.yaml
├── package.json
├── .env.example
├── prisma/
│   ├── schema.prisma
│   ├── seed.ts
│   └── migrations/
├── packages/
│   ├── backend/
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── index.ts                  # Express app entry, SuperTokens init, CORS, middleware, routes
│   │       ├── config.ts                 # Env vars with zod validation
│   │       ├── supertokens.ts            # SuperTokens init config (recipes, appInfo, overrides)
│   │       ├── middleware/
│   │       │   ├── accountScope.ts       # Extract accountId from session, inject into req, enforce scoping
│   │       │   └── requireRole.ts        # Check user role from session claims
│   │       ├── routes/
│   │       │   ├── upload.routes.ts      # File upload, validate, preview, publish
│   │       │   ├── patients.routes.ts    # Provider-facing patient data
│   │       │   ├── accounts.routes.ts    # Account CRUD (internal only)
│   │       │   └── users.routes.ts       # User CRUD (internal only)
│   │       ├── services/
│   │       │   ├── fileParser.service.ts
│   │       │   ├── validation.service.ts
│   │       │   ├── accountMatcher.service.ts
│   │       │   ├── publish.service.ts
│   │       │   ├── patients.service.ts
│   │       │   └── audit.service.ts
│   │       └── utils/
│   │           └── normalizeName.ts
│   ├── extension/
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── vite.config.ts
│   │   ├── manifest.json
│   │   └── src/
│   │       ├── popup/
│   │       │   ├── App.tsx
│   │       │   ├── index.tsx
│   │       │   ├── index.html
│   │       │   ├── supertokens.ts        # SuperTokens web-js init (header-based)
│   │       │   ├── components/
│   │       │   │   ├── Greeting.tsx
│   │       │   │   ├── LastUpdated.tsx
│   │       │   │   ├── StatusSummary.tsx
│   │       │   │   ├── FilterChips.tsx
│   │       │   │   ├── PatientList.tsx
│   │       │   │   ├── PatientRow.tsx
│   │       │   │   ├── PatientDetail.tsx
│   │       │   │   ├── EmptyState.tsx
│   │       │   │   └── LoginForm.tsx
│   │       │   ├── hooks/
│   │       │   │   ├── useSession.ts     # SuperTokens session check and auth state
│   │       │   │   ├── usePatients.ts
│   │       │   │   └── useAccount.ts
│   │       │   └── lib/
│   │       │       └── api.ts            # Fetch wrapper using SuperTokens session headers
│   │       └── background/
│   │           └── service-worker.ts
│   └── internal-tool/
│       ├── package.json
│       ├── tsconfig.json
│       ├── vite.config.ts
│       └── src/
│           ├── App.tsx                   # SuperTokens auth-react wrapper, routes
│           ├── main.tsx
│           ├── index.html
│           └── pages/
│               ├── Upload.tsx
│               ├── Accounts.tsx
│               └── Users.tsx

```

---

## Section 4: SuperTokens Integration

### Backend Init (supertokens.ts)

```typescript
// SuperTokens backend initialization for Insight Pharmacy Extension
import supertokens from "supertokens-node";
import Session from "supertokens-node/recipe/session";
import EmailPassword from "supertokens-node/recipe/emailpassword";
import { TypeInput } from "supertokens-node/types";

export function initSuperTokens(config: {
  connectionURI: string;
  apiKey?: string;
  apiDomain: string;
  websiteDomain: string;
}) {
  supertokens.init({
    framework: "express",
    supertokens: {
      connectionURI: config.connectionURI,
      apiKey: config.apiKey,
    },
    appInfo: {
      appName: "Insight Pharmacy",
      apiDomain: config.apiDomain,
      websiteDomain: config.websiteDomain,
      apiBasePath: "/auth",
      websiteBasePath: "/auth",
    },
    recipeList: \[
      EmailPassword.init({
        signUpFeature: {
          formFields: \[
            { id: "firstName" },
            { id: "lastName" },
          \],
        },
        override: {
          apis: (originalImplementation) => ({
            ...originalImplementation,
            // Disable public sign-up. Users are created by internal admins only.
            signUpPOST: undefined,
          }),
        },
      }),
      Session.init({
        override: {
          functions: (originalImplementation) => ({
            ...originalImplementation,
            createNewSession: async function (input) {
              // Attach role and accountId to session claims during login
              const userId = input.userId;
              // Look up user in our database to get role and accountId
              const { PrismaClient } = require("@prisma/client");
              const prisma = new PrismaClient();
              const user = await prisma.user.findUnique({ where: { supertokensId: userId } });
              if (user) {
                input.accessTokenPayload = {
                  ...input.accessTokenPayload,
                  role: user.role,
                  accountId: user.accountId,
                };
              }
              prisma.$disconnect();
              return originalImplementation.createNewSession(input);
            },
          }),
        },
      }),
    \],
  });
}

```

### Express App Setup (index.ts)

```typescript
import express from "express";
import cors from "cors";
import supertokens from "supertokens-node";
import { middleware, errorHandler } from "supertokens-node/framework/express";
import { initSuperTokens } from "./supertokens";
import { config } from "./config";

// Initialize SuperTokens
initSuperTokens({
  connectionURI: config.SUPERTOKENS_CONNECTION_URI,
  apiKey: config.SUPERTOKENS_API_KEY,
  apiDomain: config.API_DOMAIN,
  websiteDomain: config.WEBSITE_DOMAIN,
});

const app = express();

// CORS must come before SuperTokens middleware
app.use(cors({
  origin: \[config.WEBSITE_DOMAIN, config.EXTENSION_ORIGIN\],
  allowedHeaders: \["content-type", ...supertokens.getAllCORSHeaders()\],
  credentials: true,
}));

// SuperTokens middleware handles /auth/\* routes automatically
app.use(middleware());

app.use(express.json());

// Mount application routes here
// app.use("/api/v1/patients", patientsRouter);
// app.use("/api/v1/upload", uploadRouter);
// app.use("/api/v1/accounts", accountsRouter);
// app.use("/api/v1/users", usersRouter);

// SuperTokens error handler must come after routes
app.use(errorHandler());

app.listen(config.PORT, () => {
  console.log(\`Server running on port ${config.PORT}\`);
});

```

### Account Scope Middleware (accountScope.ts)

```typescript
// Extracts accountId and role from SuperTokens session claims.
// Enforces that provider users can only access their own account data.
import { SessionRequest } from "supertokens-node/framework/express";
import { Response, NextFunction } from "express";

export function accountScope(req: SessionRequest, res: Response, next: NextFunction) {
  const session = req.session;
  if (!session) {
    return res.status(401).json({ error: "No session" });
  }

  const payload = session.getAccessTokenPayload();
  const role = payload.role;
  const accountId = payload.accountId;

  if (role === "PROVIDER_STAFF" && !accountId) {
    return res.status(403).json({ error: "Provider user has no account assigned" });
  }

  // Attach to request for downstream use
  (req as any).userRole = role;
  (req as any).userAccountId = accountId;
  next();
}

```

### Protecting Routes

```typescript
import { verifySession } from "supertokens-node/recipe/session/framework/express";
import { accountScope } from "../middleware/accountScope";

// Provider route: requires session + account scoping
router.get("/patients",
  verifySession(),
  accountScope,
  async (req, res) => {
    const accountId = (req as any).userAccountId;
    // Query patients WHERE accountId = accountId
  }
);

// Internal route: requires session + internal role
router.post("/upload",
  verifySession(),
  requireRole(\["INTERNAL_ADMIN", "INTERNAL_STAFF"\]),
  async (req, res) => {
    // Handle upload
  }
);

```

### Extension Frontend Init (extension/src/popup/supertokens.ts)

```typescript
// SuperTokens init for Chrome extension popup.
// Uses header-based token transfer because extensions cannot use cookies cross-origin.
import SuperTokensWebJs from "supertokens-web-js";
import SessionWebJs from "supertokens-web-js/recipe/session";
import EmailPasswordWebJs from "supertokens-web-js/recipe/emailpassword";

export function initSuperTokensExtension(apiDomain: string) {
  SuperTokensWebJs.init({
    appInfo: {
      appName: "Insight Pharmacy",
      apiDomain: apiDomain,
      apiBasePath: "/auth",
    },
    recipeList: \[
      SessionWebJs.init({
        tokenTransferMethod: "header",
      }),
      EmailPasswordWebJs.init(),
    \],
  });
}

```

### Extension Login Flow (LoginForm.tsx concept)

```typescript
import { signIn } from "supertokens-web-js/recipe/emailpassword";

async function handleLogin(email: string, password: string) {
  const response = await signIn({ formFields: \[
    { id: "email", value: email },
    { id: "password", value: password },
  \]});

  if (response.status === "OK") {
    // Session is now active. SuperTokens web-js automatically attaches
    // Authorization: Bearer <token> to subsequent fetch calls.
    // Reload patient data.
    return { success: true };
  }

  if (response.status === "WRONG_CREDENTIALS_ERROR") {
    return { success: false, error: "Invalid email or password" };
  }

  return { success: false, error: "Login failed" };
}

```

### Extension API Client (api.ts)

```typescript
// All fetch calls go through this wrapper.
// SuperTokens web-js automatically intercepts fetch and attaches session headers
// when tokenTransferMethod is "header". No manual token management needed.

const API_BASE = "http://localhost:3001/api/v1";

export async function apiFetch(path: string, options: RequestInit = {}) {
  const response = await fetch(\`${API_BASE}${path}\`, {
    ...options,
    headers: {
      "Content-Type": "application/json",
      ...options.headers,
    },
  });

  if (response.status === 401) {
    // Session expired and refresh failed. Show login screen.
    throw new Error("UNAUTHORIZED");
  }

  return response;
}

```

### Internal Tool Init (App.tsx)

```typescript
// Internal tool uses supertokens-auth-react with pre-built UI.
// Cookie-based sessions (default for web apps).
import SuperTokens from "supertokens-auth-react";
import EmailPassword from "supertokens-auth-react/recipe/emailpassword";
import Session from "supertokens-auth-react/recipe/session";

SuperTokens.init({
  appInfo: {
    appName: "Insight Pharmacy Admin",
    apiDomain: "http://localhost:3001",
    websiteDomain: "http://localhost:5173",
    apiBasePath: "/auth",
    websiteBasePath: "/auth",
  },
  recipeList: \[
    EmailPassword.init(),
    Session.init(),
  \],
});

```

---

## Section 5: Database Schema

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Account {
  id          String         @id @default(uuid())
  name        String         @unique
  practiceNpi String?
  locations   String\[\]
  specialty   String?
  isActive    Boolean        @default(true)
  createdAt   DateTime       @default(now())
  updatedAt   DateTime       @updatedAt
  aliases     AccountAlias\[\]
  users       User\[\]
  referrals   Referral\[\]
}

model AccountAlias {
  id         String   @id @default(uuid())
  alias      String   @unique
  normalized String
  accountId  String
  account    Account  @relation(fields: \[accountId\], references: \[id\])
  createdAt  DateTime @default(now())
}

model User {
  id             String     @id @default(uuid())
  supertokensId  String     @unique
  email          String     @unique
  firstName      String
  lastName       String
  role           String
  isActive       Boolean    @default(true)
  accountId      String?
  account        Account?   @relation(fields: \[accountId\], references: \[id\])
  createdAt      DateTime   @default(now())
  updatedAt      DateTime   @updatedAt
  fileBatches    FileBatch\[\]
  auditLogs      AuditLog\[\]
}

model FileBatch {
  id            String            @id @default(uuid())
  fileName      String
  recordCount   Int               @default(0)
  errorCount    Int               @default(0)
  warningCount  Int               @default(0)
  status        String            @default("PENDING")
  uploadedById  String
  uploadedBy    User              @relation(fields: \[uploadedById\], references: \[id\])
  publishedAt   DateTime?
  createdAt     DateTime          @default(now())
  referrals     Referral\[\]
  errors        ValidationError\[\]
}

model ValidationError {
  id        String    @id @default(uuid())
  batchId   String
  batch     FileBatch @relation(fields: \[batchId\], references: \[id\])
  rowNumber Int
  field     String
  message   String
  severity  String
  createdAt DateTime  @default(now())
}

model Referral {
  id               String    @id @default(uuid())
  referralId       String
  accountId        String
  account          Account   @relation(fields: \[accountId\], references: \[id\])
  batchId          String
  batch            FileBatch @relation(fields: \[batchId\], references: \[id\])
  prescriberName   String
  prescriberNpi    String?
  patientFirstName String
  patientLastName  String
  patientDob       DateTime?
  medicationName   String?
  providerStatus   String
  workflowPhase    String
  nextActionOwner  String
  statusNote       String?
  statusUpdatedAt  DateTime
  isActive         Boolean   @default(true)
  createdAt        DateTime  @default(now())
  updatedAt        DateTime  @updatedAt

  @@unique(\[referralId, accountId\])
  @@index(\[accountId\])
  @@index(\[accountId, providerStatus\])
}

model AuditLog {
  id         String   @id @default(uuid())
  userId     String
  user       User     @relation(fields: \[userId\], references: \[id\])
  action     String
  resource   String
  resourceId String?
  details    Json?
  createdAt  DateTime @default(now())
}

```

Key difference from prior version: User model now has `supertokensId` field that links to the SuperTokens user ID. When SuperTokens authenticates a user, we look up their profile in our User table using this field to get role and accountId for session claims.

---

## Section 6: User Provisioning Flow

Since public sign-up is disabled (overridden to undefined in EmailPassword config), all users are created by internal admins through the internal tool.

### Creating a Provider User

1. Internal admin fills out the form in the Users page: email, password, firstName, lastName, accountId.
2. Internal tool calls POST `/api/v1/users` on the backend.
3. Backend creates the user in SuperTokens via `EmailPassword.signUp()` server-side.
4. Backend creates the corresponding User record in Prisma with the returned SuperTokens userId as `supertokensId`.
5. The provider user can now log in via the extension using their email and password.

### Creating an Internal User

Same flow, but role is set to INTERNAL_ADMIN or INTERNAL_STAFF and accountId is null.

### Backend User Creation Route

```typescript
import EmailPassword from "supertokens-node/recipe/emailpassword";

router.post("/users", verifySession(), requireRole(\["INTERNAL_ADMIN"\]), async (req, res) => {
  const { email, password, firstName, lastName, role, accountId } = req.body;

  // Create user in SuperTokens
  const stResponse = await EmailPassword.signUp("public", email, password);
  if (stResponse.status === "EMAIL_ALREADY_EXISTS_ERROR") {
    return res.status(409).json({ error: "Email already exists" });
  }

  // Create user in our database
  const user = await prisma.user.create({
    data: {
      supertokensId: stResponse.user.id,
      email,
      firstName,
      lastName,
      role,
      accountId: accountId || null,
    },
  });

  // Audit log
  await prisma.auditLog.create({
    data: {
      userId: (req as any).session.getUserId(),
      action: "CREATE_USER",
      resource: "User",
      resourceId: user.id,
      details: { email, role, accountId },
    },
  });

  return res.status(201).json({ user });
});

```

---

## Section 7: Valid Status Values

Backend validates against these exact values. Any value not in the list is a validation error.

> **AMENDED by /autoplan review (2026-06-11, premise gate D2):** canonical taxonomy is the 15 values below **plus 3**: `Physician enrollment pending`, `Telehealth no-show`, `Address confirmation pending` (18 total). Additionally, a backend **status-mapping layer** is required: the uploaded file must carry `internal_status` (required); the backend maps internal→canonical provider status via a mapping table; a `provider_status` column in the file is an optional override. Unknown internal statuses are validation errors. See GSTACK REVIEW REPORT amendment 2.

Provider status values:

* Referral received
* eRx received
* Missing information
* Intake in progress
* Prior authorization initiated
* Prior authorization pending
* Prior authorization approved
* Prior authorization denied
* Telehealth pending
* Telehealth scheduled
* Telehealth completed
* Fulfillment in progress
* Medication shipped
* Delivered
* Closed

Workflow phase values:

* Submission
* Intake/PA
* Telehealth
* Fulfillment/Delivery

Next action owner values:

* Insight
* Provider office
* Patient
* Payer
* Carrier
* None

---

## Section 8: API Endpoints

Base URL: `/api/v1`

Authentication is handled entirely by SuperTokens at `/auth/*`. Do not build custom auth routes.

### Provider-Facing (Extension)

All require `verifySession()` + `accountScope` middleware.

GET `/patients` — Returns all active referrals for the authenticated user's account. Response includes account info, lastPublishedAt, summary counts (total, newToday, needsOfficeAction, byStatus, byPhase), and patients array. Supports query params: `?status=<status>&phase=<phase>&search=<name>`. Backend MUST filter by accountId from session claims.

GET `/patients/:id` — Returns detail for one patient. Backend MUST verify patient belongs to user's account.

### Internal-Facing

All require `verifySession()` + `requireRole(["INTERNAL_ADMIN", "INTERNAL_STAFF"])`.

POST `/upload` — Multipart form. Accepts CSV/XLSX. Parses, validates, creates FileBatch. Returns batchId, counts, errors, warnings, accountPreview, canPublish boolean.

GET `/upload/:batchId/preview` — Parsed records grouped by account.

POST `/upload/:batchId/publish` — Upserts referrals by referralId + accountId, wrapped in a single transaction with an idempotency guard (re-publishing a published batch returns 409). **AMENDED (review amendment 1):** deactivation of missing referrals is scoped to accounts present in the file, and is DISABLED by default until ops answers cumulative-vs-incremental in writing (owner: Insight operations admin; fallback if unanswered: no auto-deactivation, manual deactivation via admin UI only). Writes ReferralStatusEvent rows for changed statuses. Logs audit entry.

GET/POST/PUT `/accounts` — CRUD. POST `/accounts/:id/aliases` to add aliases.

GET/POST/PUT/DELETE `/users` — CRUD. DELETE soft-deactivates.

---

## Section 9: Account Name Matching

Normalize function:

```typescript
export function normalizeName(name: string): string {
  return name
    .toLowerCase()
    .trim()
    .replace(/\[.,\\/#!$%\\^&\\\*;:{}=\\-\_\`\~()\]/g, "")
    .replace(/\\b(llc|inc|corp|pc|pllc|pa|md|do|dba)\\b/gi, "")
    .replace(/\\s+/g, " ")
    .trim();
}

```

Matching priority:

1. Exact: account_name matches Account.name.
2. Alias: normalizeName(account_name) matches AccountAlias.normalized.
3. NPI: practice_npi matches Account.practiceNpi.
4. Unmatched: flag as critical error.

---

## Section 10: File Column Mapping

Parser maps these headers (case-insensitive, trimmed):

| Expected Header(s) | Maps To | Required |
| --- | --- | --- |
| referral_id, Referral ID | referralId | Yes |
| account_name, Account Name, Practice Name | accountName | Yes |
| practice_npi, Practice NPI | practiceNpi | No |
| prescriber_name, Prescriber Name, Provider Name | prescriberName | Yes |
| prescriber_npi, Prescriber NPI | prescriberNpi | No |
| patient_first_name, First Name | patientFirstName | Yes |
| patient_last_name, Last Name | patientLastName | Yes |
| patient_dob, DOB, Date of Birth | patientDob | Yes |
| medication_name, Medication, Drug Name | medicationName | No |
| internal_status, Internal Status | internalStatus | Yes (AMENDED — backend maps internal→canonical; see Section 7) |
| provider_status, Status | providerStatus | No (AMENDED — optional override of the mapped status) |
| workflow_phase, Phase | workflowPhase | Yes |
| next_action_owner, Next Action, Action Owner | nextActionOwner | Yes |
| status_note, Notes | statusNote | No |
| status_updated_at, Last Updated | statusUpdatedAt | Yes |

If a required column is missing from the file, the entire upload fails with a clear error listing the missing columns.

---

## Section 11: Chrome Extension Manifest

```json
{
  "manifest_version": 3,
  "name": "Insight Specialty Pharmacy",
  "version": "1.0.0",
  "description": "Patient referral status updates from Insight Specialty Pharmacy",
  "permissions": \["storage"\],
  "action": {
    "default_popup": "popup/index.html",
    "default_icon": {
      "16": "icons/icon-16.png",
      "48": "icons/icon-48.png",
      "128": "icons/icon-128.png"
    }
  },
  "background": {
    "service_worker": "background/service-worker.js"
  },
  "icons": {
    "16": "icons/icon-16.png",
    "48": "icons/icon-48.png",
    "128": "icons/icon-128.png"
  }
}

```

Popup: 420px wide, 600px tall.

---

## Section 12: Extension UI

### Design System

* Font: Inter or system stack
* Background: #FAFAFA. Cards: #FFFFFF. Primary text: #111827. Secondary: #6B7280.
* Brand accent: #4F46E5 (indigo-600). Needs action: #F59E0B (amber-500). Success: #10B981 (emerald-500). Error: #EF4444. Neutral: #6B7280. Shipping: #3B82F6.
* Border radius: 8px cards, 16px chips. Spacing: 4px base.

### Components

Greeting.tsx: Time-aware greeting. "Good morning, \[Account Name\]." in 20px semibold. Below: "Here are your latest Insight updates." 14px secondary.

LastUpdated.tsx: "Last updated \[relative time\]". If over 24h, render in amber with warning icon.

StatusSummary.tsx: 2x2 grid of 4 cards: Total active, New today, Needs your attention (amber if >0), PA pending.

FilterChips.tsx: Horizontal scrollable chips: All, New Today, Needs Action, PA Pending, PA Approved, Telehealth, Shipping. Each shows count. Active chip is filled indigo. Click active to clear.

PatientList.tsx: Scrollable list of PatientRow. Respects active filter. "Needs office action" sorts to top. Default sort: statusUpdatedAt descending.

PatientRow.tsx: Compact card. Top: patient name (16px semibold), medication (14px secondary). Second line: DOB, prescriber. Third: status chip + next action owner. Right: relative timestamp. Click opens detail.

PatientDetail.tsx: Slide-in panel. Patient name 18px semibold. DOB, medication, prescriber. Status chip. Plain-language explanation (hardcoded map). Next action owner prominent. If "Provider office": highlighted action box with status_note. Last updated. Back button.

LoginForm.tsx: Email + password fields. Sign in button. Uses SuperTokens `signIn()` from supertokens-web-js/recipe/emailpassword. On success, SuperTokens manages the session automatically. On failure, inline error.

EmptyState.tsx: No patients: "No active referrals right now. When you send patients to Insight, they will appear here." No filter match: "No patients match this filter."

Status explanation map (hardcode):

```typescript
const STATUS_EXPLANATIONS: Record<string, string> = {
  "Referral received": "Insight has received this patient's referral and it is being processed.",
  "eRx received": "The electronic prescription was received successfully.",
  "Missing information": "Required documents or information are incomplete. Please check the note below for what is needed.",
  "Intake in progress": "Insight is verifying patient, prescriber, and medication details.",
  "Prior authorization initiated": "The prior authorization process has been started with the insurance carrier.",
  "Prior authorization pending": "We are waiting on a response from the insurance carrier.",
  "Prior authorization approved": "Insurance has approved this medication. Moving to next steps.",
  "Prior authorization denied": "The prior authorization was denied. Insight is reviewing next steps.",
  "Telehealth pending": "A telehealth visit needs to be scheduled or completed for this patient.",
  "Telehealth scheduled": "The telehealth visit has been scheduled.",
  "Telehealth completed": "The telehealth visit is complete. Fulfillment has been triggered.",
  "Fulfillment in progress": "The medication is being prepared for shipment.",
  "Medication shipped": "The medication has been shipped and is in transit.",
  "Delivered": "Delivery has been confirmed.",
  "Closed": "This case is no longer active.",
};

```

---

## Section 13: Internal Tool UI

Three pages plus SuperTokens pre-built login (handled by supertokens-auth-react automatically).

Upload Page: File input (.csv, .xlsx). Upload button. After upload: record/error/warning counts, error table, account preview, publish button (disabled if critical errors), success confirmation.

Accounts Page: Table of accounts. Create form. Click to edit/manage aliases.

Users Page: Table of users. Create form (email, password, name, role, account). Deactivate button.

---

## Section 14: Build Order

### Step 1: Initialize Monorepo

Create pnpm workspace with three packages. Install all dependencies. Create .env.example:

```
DATABASE_URL=postgresql://postgres:password@localhost:5432/insight_pharmacy
SUPERTOKENS_CONNECTION_URI=https://try.supertokens.io
SUPERTOKENS_API_KEY=
API_DOMAIN=http://localhost:3001
WEBSITE_DOMAIN=http://localhost:5173
EXTENSION_ORIGIN=chrome-extension://YOUR_EXTENSION_ID
PORT=3001

```

Verification: `pnpm install` succeeds with no errors.

### Step 2: Database

Write schema.prisma exactly as Section 5. Run `prisma migrate dev`. Write seed.ts that creates:

* 2 accounts: "GI Medical Services" (NPI: 1234567890, aliases: "GI Med Svc", "GI Medical Svc LLC") and "Metro Rheumatology Associates" (NPI: 0987654321)
* 17 sample referrals across both accounts with varied statuses

Note: User seeding requires SuperTokens to be running. Seed users in Step 3 after SuperTokens init.

Verification: `prisma migrate dev` succeeds. `prisma studio` shows tables.

### Step 3: Backend Core + SuperTokens

Write config.ts, supertokens.ts, index.ts as specified in Section 4. Write accountScope.ts and requireRole.ts. Add SuperTokens middleware, CORS, error handler.

Write a seed script or startup script that creates initial users via `EmailPassword.signUp()` server-side:

* [admin@insightpharmacy.com](mailto:admin@insightpharmacy.com) / admin123 — INTERNAL_ADMIN
* [sarah@gimedical.com](mailto:sarah@gimedical.com) / provider123 — PROVIDER_STAFF, GI Medical Services
* [mike@metrorrheum.com](mailto:mike@metrorrheum.com) / provider123 — PROVIDER_STAFF, Metro Rheumatology Associates

Verification: Server starts. POST to `/auth/signin` with admin credentials returns 200. GET to `/auth/session/verify` confirms session.

### Step 4: Backend — File Ingestion

Upload route with multer. File parser (CSV/XLSX). Validation service. Account matcher. Preview response. Publish route with upsert logic.

Verification: Upload a test CSV. Receive validation response with correct counts. Publish. Query referrals table and confirm records exist.

### Step 5: Backend — Provider API

GET /patients with account scoping. GET /patients/:id with ownership check. Summary computation.

Verification: Login as [sarah@gimedical.com](mailto:sarah@gimedical.com). GET /patients returns only GI Medical Services patients. Attempt to access a Metro Rheumatology patient ID returns 403.

### Step 6: Backend — Account and User CRUD

CRUD for accounts with alias management. CRUD for users with SuperTokens signUp integration.

Verification: Create a new account. Add an alias. Create a user assigned to that account.

### Step 7: Chrome Extension

Manifest V3. Vite build. SuperTokens web-js init with header-based tokens. Login form using signIn(). Dashboard: Greeting, LastUpdated, StatusSummary, FilterChips, PatientList. PatientDetail panel. API client using native fetch (SuperTokens intercepts automatically).

Verification: Build extension. Load unpacked in Chrome. Login as [sarah@gimedical.com](mailto:sarah@gimedical.com). See GI Medical Services patients. Filter works. Detail opens.

### Step 8: Internal Tool

Vite React app with supertokens-auth-react. Pre-built login UI at /auth. Upload page. Accounts page. Users page.

Verification: Login as admin. Upload CSV. See validation. Publish. Check extension shows updated data.

### Step 9: Verification

Run these checks manually:

* Login as sarah. Confirm only GI Medical patients visible.
* Login as mike. Confirm only Metro Rheumatology patients visible.
* As sarah, request mike's patient by ID. Confirm 403.
* Upload file with missing required column. Confirm upload fails with clear error.
* Upload file with unknown account name. Confirm critical error flagged.
* Publish valid file. Open extension. Confirm data updated.
* Check extension shows "Last updated" timestamp accurately.
* Check "Missing information" patients show status_note.
* Check "Needs your attention" patients sort to top.

---

## Section 15: Environment Variables

```
DATABASE_URL=postgresql://postgres:password@localhost:5432/insight_pharmacy
SUPERTOKENS_CONNECTION_URI=https://try.supertokens.io
SUPERTOKENS_API_KEY=
API_DOMAIN=http://localhost:3001
WEBSITE_DOMAIN=http://localhost:5173
EXTENSION_ORIGIN=chrome-extension://YOUR_EXTENSION_ID
PORT=3001

```

---

## Section 16: Acceptance Checklist

* [ ] Extension popup loads in under 2 seconds after auth.



* [ ] Greeting displays correct time-of-day and account name.



* [ ] Last updated timestamp is accurate. Warning shown if stale over 24h.



* [ ] Status summary cards show correct counts.



* [ ] Filter chips filter the patient list correctly with counts.



* [ ] Patient rows display name, DOB, medication, status, prescriber, timestamp.



* [ ] Patient detail opens with status explanation and next action owner.



* [ ] "Missing information" patients show the status_note.



* [ ] "Needs your attention" patients sort to top.



* [ ] Search by patient name works.



* [ ] SuperTokens login works in extension (header-based).



* [ ] SuperTokens login works in internal tool (cookie-based, pre-built UI).



* [ ] Provider user A cannot see Provider user B's patients.



* [ ] File upload parses CSV and XLSX correctly.



* [ ] Validation catches missing required fields and unknown status values.



* [ ] Unmatched account names flagged as critical errors.



* [ ] Publish updates referral data visible in extension.



* [ ] Referrals not in new file are deactivated ONLY per amended publish semantics (scoped to accounts in file; disabled until file-cadence semantics confirmed by ops; never deleted).



* [ ] Account alias mapping works.



* [ ] Accounts and users can be created and managed.



* [ ] Audit log records upload, publish, account, and user actions.



* [ ] Extension handles zero-patient state with friendly message.



* [ ] Extension handles no-publish-today state with clear timestamp.



* [ ] Public sign-up is disabled. Only admin-created users can log in.

---

<!-- AUTONOMOUS DECISION LOG -->
# GSTACK REVIEW REPORT (/autoplan 2026-06-11)

Reviewed as one plan: this build spec + product PRD ("...v0_2 (1).md") + SOP-SPI-001 v2.0 (3 PDFs).
Premise gate (user-decided): D1 premises confirmed as-is (HIPAA baseline NOT added to scope — risk register only). D2 canonical taxonomy = spec 15 + {Telehealth no-show, Address confirmation pending, Physician enrollment pending} = 18. D3 User↔Account = many-to-many join table. D4 admin overview page WITH health labels in V1.

## Phase 1 — CEO Review

### Plan Amendments (auto-adopted)
1. Publish semantics: wrap in transaction; deactivation scoped to accounts present in file; NO auto-deactivation until cumulative-vs-incremental answered in writing by ops; idempotency guard (published batch cannot re-publish).
2. Status mapping layer in backend: file requires internal_status; backend maps internal→18 canonical provider statuses via mapping table; provider_status column optional override; unknown internal status = validation error.
3. New table ReferralStatusEvent (referralId, fromStatus, toStatus, phase, owner, batchId, createdAt) written on publish when status/phase/owner changes.
4. User↔Account many-to-many: UserAccount join table; session claims carry accountIds[]; extension account picker when >1.
5. Admin overview page: per-account cards (active, new today, needs action, last publish, last provider login) + health labels (growing/stalled/friction/at-risk) per D4 — label definitions from V1-collectable data only (referral volume trend across batches, needs-action aging, login recency).
6. Shared PrismaClient module; fix session-override sample (no per-login client, no require() in function).
7. AuditLog userId bug: resolve User by supertokensId before writing audit rows.
8. isActive enforced at request time in accountScope/requireRole middleware (deactivated users lose access immediately).
9. Upload hardening: multer 10MB limit, extension+MIME check, empty-file and zero-valid-row errors, duplicate referral_id within file = validation error, row-level date parsing (ISO, M/D/YYYY, Excel serial) with per-row errors.
10. Fuzzy "did you mean" account suggestions on unmatched names (normalized Levenshtein) in validation report.
11. Publish diff preview: counts of new / status-changed / deactivated per account before confirm.
12. lastLoginAt on User (updated on signin) + PHI-free adoption events (login, patients-fetch) server-side.
13. Browser action badge with needs-action count (taste decision T1 — droppable at gate).
14. Deployment section added: hosting target w/ HTTPS + managed Postgres; production SuperTokens core (decision: self-host vs managed); extension packaged as zip for pilot side-load (Web Store = open decision); migrate-then-deploy order; post-deploy smoke checks.
15. Test plan added (global TDD law overrides spec's no-tests contract) — full diagram in Phase 3 below.
16. Versioned file-schema contract doc for ops; README + ops runbook.
17. Phase-0 hard gate: obtain >=3 real sample exports and answer ID-stability + cumulative/incremental BEFORE schema freeze (both outside voices flagged critical).
18. Index added: (accountId, isActive). Pagination deferred (threshold ~300 active referrals/account → TODOS).

### NOT in scope (deferred with rationale)
- HIPAA/compliance baseline (user decision D1; both models flag CRITICAL → User Challenge at gate; risk register below)
- Email digest + web-app-first channel (User Challenge UC2 at gate; PRD could-have)
- Google Sheets API direct ingestion (new infra; TODOS)
- SLA-risk indicators (file lacks documentation_complete_at; SOP measures from complete documentation; showing referral-age would be wrong)
- Pagination, batch unpublish/rollback UI, publish-missed alerting, PHI read-access audit, MFA (TODOS/risk register)
- Clipboard patient-summary copy (REJECTED: PHI to clipboard on shared workstations)
- Triage tag display (PRD open question; not in 18-status set; ops capture unverified)

### What already exists (leverage map)
- Auth flows/UI: SuperTokens (EmailPassword + Session, prebuilt React UI for internal tool, web-js header mode for extension) — all auth routes owned by ST, zero custom auth code. CORRECT reuse.
- Parsing: csv-parse + xlsx; Validation: zod; ORM/migrations: Prisma — standard, no wheels reinvented.
- SOP-SPI-001 already defines the 4-phase model + KPIs; marketing SOP variant = provider-facing vocabulary source.

### Dream state delta
This plan (as amended) reaches: file→validate→preview→publish→account-scoped extension + admin command view. 12-month ideal additionally needs: automated ingestion (Sheets API), SLA engine anchored on documentation_complete_at, push channel (digest), Web Store distribution, 10-30 accounts. Schema after amendments (join table, status events, mapping layer) supports that trajectory without rework.

### Error & Rescue Registry
| CODEPATH | WHAT CAN GO WRONG | HANDLING (as amended) | USER SEES |
|---|---|---|---|
| POST /upload parse | malformed CSV / corrupt XLSX | 400 w/ parse error detail | "File could not be read" + detail |
| POST /upload parse | missing required columns | 400 listing missing columns | exact column list |
| POST /upload parse | empty file / zero valid rows | 400 validation error | "No data rows found" |
| POST /upload parse | >10MB / wrong MIME | 413 / 415 | size/type message |
| row validation | bad date (text, serial, garbage) | per-row ValidationError(severity=CRITICAL) | row+field+message table |
| row validation | unknown internal_status | per-row CRITICAL | row+value+allowed list |
| row validation | duplicate referral_id in file | per-row CRITICAL | both row numbers |
| account match | unmatched account_name | CRITICAL + "did you mean" suggestions | suggestion chips |
| POST /publish | re-publish published batch | 409 idempotency guard | "Batch already published" |
| POST /publish | DB failure mid-upsert | $transaction rollback, batch stays VALIDATED | "Publish failed, no changes applied" |
| GET /patients | no/expired session | 401 (ST refresh first) | extension login screen |
| GET /patients | deactivated user | 403 via isActive check | "Account disabled — contact Insight" |
| GET /patients/:id | other account's referral | 403 ownership check | "Not found" (no existence leak: return 404) |
| extension fetch | network failure | error state w/ retry | "Can't reach Insight — Retry" |
| user creation | duplicate email | 409 from ST signup | inline form error |

### Failure Modes Registry
| CODEPATH | FAILURE MODE | RESCUED? | TEST? | USER SEES? | LOGGED? |
|---|---|---|---|---|---|
| publish | unscoped deactivation wipes other accounts | Y (amendment 1) | Y (integration) | diff preview | audit row |
| publish | partial upsert on crash | Y (transaction) | Y | error banner | console+batch status |
| session | stale claims after deactivation | Y (amendment 8) | Y | 403 message | audit row |
| matching | name fuzzy-routes to wrong account | PARTIAL — preview + alias confirm | Y | admin preview | audit row |
| upload | Excel serial dates ingested as numbers | Y (amendment 9) | Y | row errors | ValidationError rows |
| extension | popup opened during publish | Y (refetch on open) | E2E | fresh data | n/a |
| ops | daily upload skipped (vacation/turnover) | N — RISK | n/a | stale amber banner >24h | TODOS: publish-missed alert |

### Risk Register (accepted risks per premise gate D1)
- PHI (name, DOB, medication) stored/displayed without documented HIPAA baseline: no BAA chain named, no encryption-at-rest requirement, no session-timeout policy, no PHI read-access audit, no DOB masking decision, no breach-response owner. BOTH outside voices rate CRITICAL. SOP mandates monthly HIPAA audits — this system will be audited against controls it hasn't documented. Mitigation available cheap (~0.5 day CC). USER CHALLENGE UC1 at final gate.
- Extension installability on managed provider-office Chrome: unvalidated. Mitigation: keep popup URL-addressable (UC2); validate with pilot offices week 1.
- Single-human daily upload process: stale-data risk by design. Mitigations: staleness banner (spec'd), publish-missed alert (TODO), Sheets API (TODO).

## Decision Audit Trail
| # | Phase | Decision | Class | Principle | Rationale | Rejected |
|---|---|---|---|---|---|---|
| 1 | 0 | Skip /office-hours offer | Mechanical | P3/P6 | PRD already contains problem statement/premise/risks | run it |
| 2 | 0 | DX scope = yes | Mechanical | P1 | keyword threshold + AI-agent-executes-spec trigger | skip DX |
| 3 | 1 | Approach B: spec + alignment fixes | Mechanical | P1 | fixes 5 cross-doc contradictions pre-build; A=6/10, B=9/10, C=3/10 | A, C |
| 4 | 1 | Reject clipboard-copy expansion | Mechanical | P1(trust) | PHI to clipboard on shared workstations | add |
| 5 | 1 | Add needs-action badge | TASTE | P2 | ambient visibility; con: counts visible on shared screens | skip |
| 6 | 1 | Add publish diff preview | Mechanical | P2 | makes dangerous publish step legible | skip |
| 7 | 1 | Add did-you-mean suggestions | Mechanical | P2 | top-named fragility (matching) | skip |
| 8 | 1 | Add lastLoginAt + adoption events | Mechanical | P2 | required by D4 cards + PRD tracking plan | skip |
| 9 | 1 | Defer Sheets API to TODOS | Mechanical | P2 | new infra, outside V1 radius | build now |
| 10 | 1 | Defer SLA indicators | Mechanical | P1 | SOP measures from complete-documentation; data absent | build |
| 11 | 1 | Publish semantics amendment | Mechanical | P1 | both voices CRITICAL: silent deactivation hazard | as-spec |
| 12 | 1 | Backend status-mapping layer | Mechanical | P1/P4 | PRD requires internal_status; spec dropped it | file-side mapping only |
| 13 | 1 | ReferralStatusEvent table | Mechanical | P1 | newToday/what-changed/history/recovery | current-state only |
| 14 | 1 | Prisma client + audit-FK + isActive fixes | Mechanical | P1/P5 | concrete bugs in spec samples | as-spec |
| 15 | 1 | Upload hardening bundle | Mechanical | P1 | real hospital files are messy | trust input |
| 16 | 1 | Deployment section | Mechanical | P1 | spec ends at localhost; production-readiness requires it | defer |
| 17 | 1 | Test plan despite no-tests contract | Mechanical | global TDD law | user's global CLAUDE.md overrides doc silence | honor contract |
| 18 | 1 | Phase-0 sample-export hard gate | Mechanical | P1 | both voices CRITICAL: schema frozen against unseen file | proceed |
| 19 | 1 | Defer pagination w/ threshold | Mechanical | P3 | pilot scale ≤3 accounts | build now |
| 20 | 2 | Hierarchy restructure: 1-line header, needs-action banner first, list above fold | Mechanical | P5 | both voices CRITICAL: chrome ate 45% of popup | spec order |
| 21 | 2 | Summary counts are filters (synced w/ chips) | Mechanical | P1 | PRD requires clickable counts | display-only |
| 22 | 2 | Two-tier sort (pinned needs-action group, then recency) | Mechanical | P5 | spec self-contradicts on sort | pick one silently |
| 23 | 2 | Add SearchBar.tsx (debounce, clear, empty state) | Mechanical | P1 | in API+checklist, missing from components | ship w/o search |
| 24 | 2 | Per-surface state matrix (loading/error/401/403/stale/empty) | Mechanical | P1 | both voices: states unspecified | implementer invents |
| 25 | 2 | Upload page = 4-step stepper w/ locked publish + account-first diff + deactivation confirm | Mechanical | P1 | PRD demands progress/error states; publish is the dangerous step | 3-sentence spec |
| 26 | 2 | AccountPicker as fixed top context bar, persisted selection | Mechanical | P1 | D3 made it necessary; PHI needs unmistakable account context | unspecified picker |
| 27 | 2 | Canonical 18-row status table (label/short/explanation/color/chip/needs-action/summary bucket) | Mechanical | P4/P5 | 3 drifting partial lists guaranteed inconsistency | keep 3 lists |
| 28 | 2 | needs-action + new-today predicates (server-computed) | Mechanical | P5 | both voices CRITICAL: most prominent numbers undefined | client-side improvisation |
| 29 | 2 | Sign-out + signed-in-as footer; 12h idle TTL default; admin temp-password reset | Mechanical | P1 | shared workstations; PRD requires reset; spec had neither | indefinite sessions |
| 30 | 2 | Replace 3-screen onboarding with 1 dismissable first-run card | Mechanical | P5 | both voices converged; sub-60s persona | 3 screens |
| 31 | 2 | Stale-copy rules (no icon <48h, blame-correct wording) | Mechanical | P1 | stale warning stranded the user | bare amber warning |
| 32 | 2 | Badge promoted to standard scope (habit trigger) | Mechanical | P1 | only re-engagement mechanism; was taste T1 | droppable |
| 33 | 2 | PatientDetail interaction spec + guaranteed-vs-conditional fields | Mechanical | P5 | "slide-in panel" underspecified; SLA/history data may not exist | fake precision |
| 34 | 2 | A11y acceptance criteria (keyboard/focus/44px/AA) | Mechanical | P1 | aspirational → executable | aspirational |
| 35 | 2 | Internal tool depth: Overview.tsx + health-label formulas + insufficient-data fallback | Mechanical | P5 | D4 included labels; formulas prevent invented heuristics | implementer invents math |
| 36 | 2 | Keep Inter font (conscious deviation from no-default-font rule) | TASTE | P3 | operational legibility > brand expressiveness here | expressive typeface |

## Phase 2 — Design Review Amendments (detail)

### Canonical status table (replaces Section 12 STATUS_EXPLANATIONS, chip list, and summary buckets as 3 separate lists)
| Canonical status | Chip bucket | Color | Needs-action | Summary bucket |
|---|---|---|---|---|
| Physician enrollment pending | All | neutral #6B7280 | when owner=Provider office | Total |
| Referral received | New Today* | neutral | no | Total/New |
| eRx received | New Today* | neutral | no | Total/New |
| Missing information | Needs Action | amber #F59E0B | YES | Needs attention |
| Intake in progress | All | neutral | no | Total |
| Prior authorization initiated | PA Pending | indigo #4F46E5 | no | PA pending |
| Prior authorization pending | PA Pending | indigo | no | PA pending |
| Prior authorization approved | PA Approved | emerald #10B981 | no | Total |
| Prior authorization denied | PA Pending | neutral (NOT red — Insight is handling) | no | Total |
| Telehealth pending | Telehealth | neutral | no | Total |
| Telehealth scheduled | Telehealth | neutral | no | Total |
| Telehealth no-show | Telehealth | amber | when owner=Patient+office can help | Needs attention if flagged |
| Telehealth completed | Telehealth | emerald | no | Total |
| Address confirmation pending | Needs Action | amber | when owner=Provider office/Patient | Needs attention |
| Fulfillment in progress | Shipping | blue #3B82F6 | no | Total |
| Medication shipped | Shipping | blue | no | Total |
| Delivered | Shipping | emerald | no | Total |
| Closed | All | neutral | no | excluded from Total active |
*New Today is a computed predicate, not a status bucket — chip counts referrals where newToday=true regardless of status.

New explanation copy: "Physician enrollment pending" → "Provider setup (CDTM agreement) is not yet complete for this prescriber. Insight will reach out if anything is needed from your office." | "Telehealth no-show" → "The patient missed the scheduled telehealth visit. Insight is following up to reschedule." | "Address confirmation pending" → "The patient's delivery address needs to be confirmed before the medication can ship."

### Predicates (server-computed only)
- needsAction := nextActionOwner == "Provider office"
- newToday := referral first appeared OR status changed in the most recent published batch (via ReferralStatusEvent)

### State matrix (extension popup)
| State | Trigger | User sees | Action |
|---|---|---|---|
| Loading | initial fetch | skeleton rows (header + 5 row ghosts) | — |
| Network error | fetch reject | "Can't reach Insight" | Retry button |
| 401 | session expired+refresh failed | LoginForm | sign in |
| 403 deactivated | isActive=false | "Account disabled — contact Insight" | — |
| Empty (no referrals) | 0 active | warm empty copy (spec) | — |
| Empty (filter) | 0 matches | "No patients match this filter." | clear-filter link |
| Stale | >24h since publish | subdued "as of" line; amber+icon only ≥48h | — |
| Multi-account | >1 account | context bar w/ account name; switch = confirm + refetch | picker |

### Internal tool: Upload = 4-step stepper (Upload → Validate → Preview diff → Publish); publish button locked while in-flight; failure banner "Publish failed — no changes applied"; diff preview is account-first (new/changed/deactivated counts per account + unmatched rows w/ did-you-mean) and requires an explicit confirmation checkbox when deactivations > 0. Overview.tsx added (route /, account health cards). Health labels: friction = any needs-action item >3 business days old; at-risk = no provider login in 14d; stalled = zero new referrals in 14d; growing = new referrals up batch-over-batch 2 consecutive weeks; else "insufficient data" (first 2 weeks always show insufficient data).

### A11y acceptance criteria: filters/chips keyboard-reachable; drawer focus-trap + Esc + focus restore; 44px min targets; WCAG AA contrast; visible focus ring; aria-labels on counts/chips ("5 patients, prior authorization pending").

## Phase 3 — Engineering Review Amendments

| # | Phase | Decision | Class | Principle | Rationale | Rejected |
|---|---|---|---|---|---|---|
| 37 | 3 | FAIL-CLOSED session claims: accountScope denies unless role ∈ enum; patients service asserts non-empty accountId; createNewSession override rejects when user row missing | Mechanical | P1 SECURITY | undefined claims pass accountScope and Prisma drops undefined where-filter → cross-tenant PHI dump (Claude conf 8, Codex agrees) | as-spec (fail-open) |
| 38 | 3 | Section 5 schema is REWRITTEN as canonical: UserAccount M:N, ReferralStatusEvent (typed CREATED/STATUS_CHANGED/PHASE_CHANGED/OWNER_CHANGED), StatusMapping, AdoptionEvent, lastLoginAt, Prisma enums (role/status/phase/owner/severity/batchStatus), patientDob @db.Date, AccountAlias.normalized @unique, index (accountId,isActive,statusUpdatedAt) | Mechanical | P1/P5 | amendments must live in the concrete Prisma block, not report prose; stringly-typed enums invite bad data | prose-only amendments |
| 39 | 3 | Multi-account API contract: claims.accountIds[]; GET /patients requires X-Account-Id validated ∈ claims; claims re-minted on refresh | Mechanical | P1 | D3 was unpropagated — two contradictory designs in one doc | scalar accountId |
| 40 | 3 | Seed split: foundation (accounts/aliases/mappings) → users (post-ST) → referrals (synthetic batch) | Mechanical | P5 | Step 2 as written cannot execute (batchId→uploadedById→User chicken-egg) | spec order |
| 41 | 3 | manifest.json "key" field pinned (deterministic extension ID) + @crxjs/vite-plugin for MV3 build | Mechanical | P1 | unpacked IDs derive from path → CORS allowlist breaks per-machine; Vite default emit doesn't match manifest paths | hand-rolled build |
| 42 | 3 | Badge = popup-open update only in V1 (no SW fetch); SW token bridge → TODOS | Mechanical | P3/P5 | supertokens-web-js has no localStorage/fetch-intercept in MV3 SW — live badge is a hidden mini-project | spec a token bridge now |
| 43 | 3 | DOB + date-only fields as @db.Date, serial epoch 1899-12-30, no TZ round-trips | Mechanical | P1 | DOB off-by-one = patient misidentification hazard | DateTime + toLocaleDateString |
| 44 | 3 | Replace npm xlsx with exceljs (or CDN-pinned SheetJS); decompressed-size + row-count caps | Mechanical | P1 SECURITY | npm xlsx frozen at 0.18.5 w/ CVE-2023-30533 + CVE-2024-22363; zip-bomb DoS on PHI backend | npm xlsx |
| 45 | 3 | Rate limiting on /auth/* + /upload; password policy for admin-created users; Karpathy contract narrowed (no-rate-limit clause deleted; no-tests clause replaced by Test Plan requirement) | Mechanical | P1 SECURITY | PHI login with zero brute-force protection; implementers obey the contract, not the appendix | contract as-is |
| 46 | 3 | Account-move handling: referralId active under different account → old row deactivated + surfaced in diff preview | Mechanical | P1 | with bulk deactivation OFF, a corrected account assignment leaves PHI visible to the wrong practice forever | leave both active |
| 47 | 3 | Cross-account access returns 404 (no existence leak); Steps 5/9 verification updated | Mechanical | P5 | spec self-contradicted (403 in Steps, 404 in registry) | 403 |
| 48 | 3 | Parser contract: bom:true, UTF-8→win-1252 fallback, first-non-empty-sheet rule (else validation error), blank-row skip, dup-header dedup, formula display values, Excel-matching row numbers, first-N errors + totals | Mechanical | P1 | Excel-exported CSVs are BOM'd; Windows CSVs are 1252; workbooks have tabs | assume clean UTF-8 |
| 49 | 3 | Alias hygiene: normalized @unique w/ collision rejection; normalizeName punctuation→space; never auto-create aliases; admin confirm shows NPI context | Mechanical | P1 | nondeterministic match priority + Smith-Jones bug + wrong-account alias = misrouted PHI | as-spec |
| 50 | 3 | Publish concurrency: FileBatch row-lock, VALIDATED→PUBLISHING→PUBLISHED transitions, chunked writes w/ explicit txn timeout, row-count cap | Mechanical | P1 | two admins can double-publish; interactive $transaction times out ~low-thousands rows | naive loop |
| 51 | 3 | User-creation compensation: prisma failure after ST signup ⇒ delete ST user / block signin; validate role+account before signup | Mechanical | P1 | orphan ST users can sign in with no profile (feeds finding 37) | ignore |
| 52 | 3 | Dev auth: local dockerized SuperTokens core (try.supertokens.io is shared + purged); disable public password-reset endpoints (admin temp-password path instead) | Mechanical | P1/P5 | seeded users vanish day 2; live unconfigured reset endpoints = dead attack surface | dev core |
| 53 | 3 | Ingest canonicalization: trim + case-insensitive for internal_status/workflow_phase/next_action_owner | Mechanical | P1 | "Provider Office " must not zero the needs-action count | exact match |
| 54 | 3 | Step 3 verification via authenticated probe route (no /auth/session/verify FDI route exists) | Mechanical | P5 | spec names a nonexistent endpoint | as-spec |
| 55 | 3 | /patients: summary counts separate from rows; rows capped 500; production env matrix (cookie secure/sameSite, prod extension origin, ST core URI, migrate workflow) | Mechanical | P1/P3 | unbounded payloads; deployment must be executable | unbounded |
| 56 | 3 | Minimal ops alerting IN V1: publish-failure + zero-row + no-publish-by-time (node-cron + email/webhook) | Mechanical | P1 | both voices: "someone must be paged"; stale data kills trust silently | TODO-only |
| 57 | 3 | FileBatch.batchId on Referral documented as "last published by"; per-batch rows persist only as ValidationError + diff counts | Mechanical | P3 | resolves misleading relation without a second table | split tables now |

Test plan artifact: ~/.gstack/projects/InsightExtension/aaron-master-eng-review-test-plan-20260611.md (ship-blocking suites: cross-tenant isolation, fail-closed auth, publish transaction, parser fixtures).

## Phase 3.5 — DX Review Amendments

| # | Phase | Decision | Class | Principle | Rationale | Rejected |
|---|---|---|---|---|---|---|
| 58 | 3.5 | CONSOLIDATION: produce build-spec v0.3 with all amendments merged inline; this appendix demoted to changelog. FIRST implementation task | Mechanical | P5 | both DX voices CRITICAL: "two conflicting specs stapled together"; Section 0 says copy code "exactly" → executor copies known-buggy samples | leave appendix |
| 59 | 3.5 | Canonical schema.prisma block written into Section 5 (all amended models/enums) as part of v0.3 | Mechanical | P1 | amendment 38 described models in prose; Step 2 unexecutable without the block | prose-only |
| 60 | 3.5 | docker-compose.yml (postgres:15 + supertokens-postgresql + dedicated st DB) + root scripts (dev/db:setup/seed:*) + /healthz + 6-line Quick Start; TTHW target ≤30 min | Mechanical | P1 | no local infra story = 2-4h of invention | assume services exist |
| 61 | 3.5 | Committed dev manifest "key" + derived extension ID in README + matching EXTENSION_ORIGIN baked into .env.example (zero placeholders) | Mechanical | P5 | kills the extension-ID/CORS chicken-egg at clone time | YOUR_EXTENSION_ID placeholder |
| 62 | 3.5 | Build-order Steps 2-3 rewritten with named seed scripts + per-script verification (folds amendment 40 in) | Mechanical | P5 | seed fix lived in prose, not in the steps the executor follows | prose-only |
| 63 | 3.5 | Provisional StatusMapping fixture seed marked placeholder-pending-Phase-0; Step 4 = mechanism + fixture | Mechanical | P3 | mapping content gated on sample exports; build must not stall | stall Step 4 |
| 64 | 3.5 | vitest+supertest added; fixtures/ directory (valid CSV+XLSX, missing-column, bad-dates, dup-referral-id, unknown-status, unmatched-account, BOM, win-1252, zip-bomb); tests interleaved into Steps 4-6; pnpm test as verification | Mechanical | P1 | test plan was mandated with no framework, no fixtures, no step | "upload a test CSV" (none provided) |
| 65 | 3.5 | Version pinning: supertokens-node/web-js/auth-react, prisma, tailwind v4, exceljs, vite exact majors; Node 20 LTS engines; Section 2 package table updated to exceljs | Mechanical | P1 | zero pinning + version-sensitive ST override samples | unpinned |
| 66 | 3.5 | tsx watch (not ts-node) + scripts blocks for all 3 packages + both vite.config.ts (crxjs+tailwind / standard) included in spec | Mechanical | P5 | ts-node+ESM+Express is a known time-sink; configs were unspecified | ts-node, invent configs |
| 67 | 3.5 | Error-copy table: literal problem/cause/fix per error code; ValidationError.rawValue column; downloadable error report (CSV) | Mechanical | P1 | ops admin's entire API; detection was specified, remediation wasn't | category-only errors |
| 68 | 3.5 | Header-resolution preview + ambiguity guard (generic headers resolved + shown); file-contract v1 templates (csv+xlsx) + optional schema_version handshake | Mechanical | P1 | silent header drift is the 3-month failure mode | permissive aliases |
| 69 | 3.5 | Docs as build Step 10: README.md, docs/file-contract.md, docs/ops-runbook.md, docs/release-runbook.md (extension packaging + side-load update checklist + popup-footer version + minExtensionVersion rule), docs/architecture.md | Mechanical | P1 | spec's own "only listed files" contract guarantees docs never get written otherwise | mandate w/o build step |
| 70 | 3.5 | API base + apiDomain via VITE_API_DOMAIN (.env.development/.env.production per package) | Mechanical | P5 | hardcoded localhost breaks the packaged-zip prod path | hardcoded |
| 71 | 3.5 | scripts/smoke.ts: signin → scoped /patients → cross-tenant 404 → bad upload 400 → publish 200→409 | Mechanical | P1 | Step 9 was nine manual checks with no commands | manual-only |
| 72 | 3.5 | Publish preview in operator language ("3 newly visible to GI Medical...") + PHI-safe batch history (who/file/counts/result/failure reason) | Mechanical | P1 | publish is the trust moment; supportability needs batch history | row counts only |
| 73 | 3.5 | Seed credentials labeled local-only; production first-admin bootstrap flow (no seeded admin in prod) | Mechanical | P1 | sample passwords leak into staging habits | seed everywhere |
| 74 | GATE | UC1 RESOLVED — HIPAA baseline IN V1: BAA chain named, encryption-at-rest confirmed, PHI read-access audit events, session policy (12h idle, tightenable), retention policy, breach-response owner | USER (final gate) | — | user flipped D1 after 5 voices across 3 phases flagged critical | risk-register-only |
| 75 | GATE | UC2 RESOLVED — web-view architecture + email digest IN V1: popup React app also served at URL (extension = wrapper/launcher); daily per-account digest ("N need your action" + deep link) | USER (final gate) | — | all 4 voices across Phases 1+2; de-risks blur-close, IT install blocks, pull-only habit | extension-only |
| 76 | GATE | UC3 RESOLVED — health labels CUT to TODOS (trigger: 10+ accounts); minimal Overview page stays (account cards: active/new/needs-action/last publish/last login) | USER (final gate) | — | user accepted models' recommendation; PRD's own anti-analytics-graveyard warning | labels in V1 |

## FINAL GATE OUTCOME (2026-06-11)
**APPROVED** as the build contract. Taste defaults accepted: DOB masked (MM/DD/****) in list, full in detail; Inter font retained. Mode: SELECTIVE EXPANSION honored throughout. Implementation task #1: consolidated build-spec v0.3 (amendment 58). Schema-freeze blockers: (1) ≥3 real sample exports, (2) written ops answer on cumulative-vs-incremental.



