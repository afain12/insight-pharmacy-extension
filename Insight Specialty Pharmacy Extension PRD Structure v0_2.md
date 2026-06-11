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

POST `/upload/:batchId/publish` — Upserts referrals by referralId + accountId. Marks referrals missing from new file as isActive=false. Logs audit entry.

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
| provider_status, Status | providerStatus | Yes |
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



* [ ] Referrals not in new file are marked inactive, not deleted.



* [ ] Account alias mapping works.



* [ ] Accounts and users can be created and managed.



* [ ] Audit log records upload, publish, account, and user actions.



* [ ] Extension handles zero-patient state with friendly message.



* [ ] Extension handles no-publish-today state with clear timestamp.



* [ ] Public sign-up is disabled. Only admin-created users can log in.



