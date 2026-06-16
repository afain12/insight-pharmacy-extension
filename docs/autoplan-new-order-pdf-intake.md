---
status: PROPOSED — awaiting user authorization (final gate)
feature: New Order — provider-initiated PDF patient-demographic intake
generated_by: Devin (emulating the /autoplan pipeline — Product/CEO · Design · Engineering · DX · Security/Compliance lenses)
date: 2026-06-16
base_contract: BUILD-SPEC-v0.3.md (and the in-flight Better Auth + MySQL refinement → BUILD-SPEC-v0.4)
assumptions: Auth = Better Auth (not SuperTokens); DB = self-hosted MySQL 8 (per docs/hosting-and-auth-migration-strategy.md)
---

# /autoplan — New Order: PDF Patient-Demographic Intake

> **Purpose of this document.** You asked to `/autoplan` this feature so it can be **authorized before we wire the backend API connections**. This is that authorization artifact. It proposes the architecture, surfaces the cross-cutting risks (especially HIPAA), and ends in a **Decisions table + Final Gate** with the small number of choices only you can make. Nothing in the backend is built yet — this is a plan.

---

## 1. Feature in one paragraph

A new **"New Order"** section in the Chrome extension lets provider-office staff **drag-and-drop a patient-demographic PDF** straight into the extension. The extension extracts the patient's name + demographics, the office confirms them, and a **new patient is submitted to Insight electronically** — a capability that does not exist today (today, patients only enter the system through the Insight ops admin's daily hospital-sheet upload). On the **admin side**, Insight ops sees the new electronic submission, and uses the submitted name + demographics to **match the patient against the hospital's records**. Once matched, the submission **flows into the existing tracker** as a tracked `Referral`, so the office can watch their own cases' statuses move through the SOP-SPI-001 phases in near-real-time.

This adds an **inbound (provider → Insight)** path alongside the existing **inbound (hospital → Insight)** path. The provider side of the product stops being strictly read-only.

> **Terminology flag:** you referred to our "IVS structure." I'm treating that as the intake/visibility flow described above; please confirm what "IVS" stands for so the spec uses your exact term (see Open Questions Q1).

---

## 2. Why it matters (10x check)

The current 10x framing (per `docs/reviews/ceo-plan-2026-06-11.md`) is **visibility → resolution, pull → push**: Insight pushes offices the 2–3 patients that need *their* action. This feature extends that loop **upstream**: it closes the gap at the *start* of a referral's life.

- **Removes the single biggest intake friction:** an office sends a patient to Insight by *dropping the demographic sheet they already have* — no portal switching, no phone call, no fax-and-hope. The artifact the office already produces (a demographics PDF) becomes the submission.
- **Gives Insight ops an electronic intake queue** they don't have today ("a new patient was submitted electronically, which we currently don't have access to").
- **Creates the join key early:** capturing name + DOB + demographics at submission time is exactly what lets ops reconcile the office's patient to the hospital's record when the daily export arrives — instead of guessing later.
- **Same head, new channel:** the submission is just another way patient state enters the channel-agnostic status API; the office then sees it resolve in the same tracker they already use.

---

## 3. How it fits today's architecture — and what it CHANGES

Grounded in `BUILD-SPEC-v0.3.md`:

| Today (v0.3) | With this feature |
|---|---|
| Providers are **read-only**: `GET /patients`, `GET /patients/:id` only (§7). All writes come from ops upload→publish (§9). | Providers gain a **write path**: `POST /submissions` (PDF). The provider side is no longer read-only. |
| Patients enter **only** via the hospital sheet, which carries a hospital `referralId` (the stable unique key, §5 `Referral`). | Patients can enter **before** a hospital `referralId` exists. We need a pre-referral entity that later *links* to a `Referral`. |
| Extension persists **no PHI**; tokens only (§11, §15). V1 is **internal-only, synthetic data** (§15). | A real demographics PDF = **real PHI entering through the provider/extension side.** This collides with §15 and the P0 HIPAA blocker (see §8). |
| Matching is **account-level**, NPI-first (§8). | Adds a **patient-level** match (name + DOB) to link a submission to the right `Referral` — a new collision-hazard surface. |
| Uploads are **CSV/XLSX**, parsed with strict contracts (§10). | Adds **PDF** ingestion — unstructured/semi-structured, OCR-prone, lower-confidence than a column contract. |

**Net:** this is not a UI tweak. It introduces a second ingestion pipeline, a new entity + lifecycle, a new matching dimension, a new PHI-at-rest surface, and a provider-write authorization path. It is well worth doing — but it must be planned, not bolted on. That's why authorizing it before the backend wiring is the right call.

---

## 4. Proposed end-to-end flow

```
Provider office (extension "New Order")
  1. Drag-drop demographic PDF(s)
  2. POST /api/v1/submissions  (Better Auth bearer + X-Account-Id; fail-closed scope)
        backend: store PDF (encrypted at rest) → extract fields → return extracted fields + per-field confidence
  3. Office REVIEWS/CORRECTS the extracted demographics in the popup, then confirms
        backend: persist PatientSubmission (status=SUBMITTED), audit log, event
            │
            ▼
Insight ops (internal tool "Submissions" inbox)
  4. Sees the new electronic submission (name + demographics + source PDF)
  5. Reviews; reconciles against hospital records
            │
            ▼
Hospital daily export arrives (existing upload→validate→publish, §9–§10)
  6. Ops links submission ⇆ Referral  (patient-level match: name + DOB within the account; ops-confirmed, never auto)
        PatientSubmission.status = MATCHED, PatientSubmission.referralId set
            │
            ▼
Provider tracker (existing GET /patients)
  7. The linked Referral now shows in the office's tracker with live SOP statuses.
     Until matched, the "New Order" section shows the submission's own status
     (SUBMITTED → UNDER_REVIEW → MATCHED / NEEDS_INFO / DUPLICATE / REJECTED).
```

**Two-stage visibility (important for expectations):** the office gets an **immediate acknowledgement** (the submission appears in "New Order" right away), but **full tracker visibility** waits on (a) Insight ops review and (b) the hospital export that assigns the real referral. See UC-2 about the word "immediate."

---

## 5. Data model additions (proposed)

New entity, in the same MySQL database (Prisma). No changes required to existing `Referral` semantics beyond a back-reference.

```prisma
enum SubmissionStatus {
  SUBMITTED          // office confirmed + sent
  UNDER_REVIEW       // Insight ops looking at it
  NEEDS_INFO         // ops needs more from the office
  MATCHED            // linked to a hospital Referral
  DUPLICATE          // already in the system
  REJECTED           // not a valid submission
}

enum SubmissionEventType { CREATED STATUS_CHANGED MATCHED UNLINKED NEEDS_INFO REJECTED }

model PatientSubmission {
  id               String   @id @default(uuid())
  accountId        String                 // the submitting office's account (from claims; fail-closed)
  account          Account  @relation(fields: [accountId], references: [id])
  submittedById    String                 // req.localUser.id (PROVIDER_STAFF)
  submittedBy      User     @relation(fields: [submittedById], references: [id])

  // confirmed demographics (office-corrected; not raw extraction)
  patientFirstName String
  patientLastName  String
  patientDob       DateTime? @db.Date      // DATE-ONLY; same boundary discipline as §5.1
  patientPhone     String?
  patientAddress   String?
  insuranceInfo    String?
  medicationName   String?
  prescriberName   String?
  prescriberNpi    String?
  notes            String?  @db.Text

  // provenance / extraction
  sourceFileKey    String                  // pointer to encrypted PDF blob (NOT the bytes)
  sourceFileName   String
  extraction       Json?                   // per-field extraction + confidence (no extra PHI beyond the above)
  extractionMethod String                  // "text" | "ocr" | "manual"

  status           SubmissionStatus @default(SUBMITTED)
  referralId       String?                 // set on MATCHED
  referral         Referral? @relation(fields: [referralId], references: [id])

  createdAt        DateTime @default(now())
  updatedAt        DateTime @updatedAt
  events           PatientSubmissionEvent[]

  @@index([accountId, status, createdAt])
}

model PatientSubmissionEvent {
  id           String   @id @default(uuid())
  submissionId String
  submission   PatientSubmission @relation(fields: [submissionId], references: [id])
  type         SubmissionEventType
  fromStatus   SubmissionStatus?
  toStatus     SubmissionStatus?
  actorUserId  String              // req.localUser.id
  createdAt    DateTime @default(now())
  @@index([submissionId, createdAt])
}
```

Plus a back-reference on `Referral` (`submissions PatientSubmission[]`) and on `Account`/`User`. The **raw PDF bytes never live in the DB** — only an encrypted-blob reference (`sourceFileKey`), so the PHI-at-rest surface is one controlled store with its own retention/purge.

---

## 6. API additions (proposed — not built yet)

Provider-facing (Better Auth bearer + `accountScope`, fail-closed):
- `POST /api/v1/submissions` — multipart PDF. Stores file, runs extraction, returns `{ submissionId?, extracted: {field: {value, confidence}}, warnings }`. **Two-phase**: extract-then-confirm, so nothing persists as a real submission until the office confirms (see D-B). Rate-limited per user (mirror `/upload`).
- `POST /api/v1/submissions/:id/confirm` — office submits corrected fields → status `SUBMITTED`.
- `GET /api/v1/submissions` — the office's own submissions (account-scoped) for the "New Order" status list.
- `GET /api/v1/submissions/:id` — detail; cross-account → **404** (no existence leak, same rule as §7).

Internal-facing (`requireRole([INTERNAL_ADMIN, INTERNAL_STAFF])`):
- `GET /api/v1/submissions/inbox` — ops queue across accounts, filterable.
- `POST /api/v1/submissions/:id/match` — link to a `Referral` (explicit, ops-confirmed). Sets `MATCHED`.
- `POST /api/v1/submissions/:id/status` — `UNDER_REVIEW | NEEDS_INFO | DUPLICATE | REJECTED`.
- `GET /api/v1/submissions/:id/file` — stream the source PDF (audited; ops only).

All protected routes keep the existing chain: **session verify → resolveLocalUser → (accountScope | requireRole)** with the §4 invariants intact (the Better Auth refinement preserves these — see the docs-refinement PR).

---

## 7. UI additions (proposed)

**Extension popup — "New Order" (recommend a tab, T-1):**
- Dropzone (drag-drop + click-to-pick), `.pdf` only, size cap, clear progress/error states (mirror the §12.5 state matrix discipline).
- **Extraction review card:** each extracted field shown editable with a confidence cue; low-confidence fields flagged for the office to verify. Confirm button is the moment of submission.
- **My submissions list:** status chips (SUBMITTED / UNDER_REVIEW / NEEDS_INFO / MATCHED …); a MATCHED item links into the existing patient detail.
- Consistent with §12.1 design system; **no PHI persisted in extension storage** (consistent with §11/§15) — the source PDF is uploaded, not cached locally.

**Internal tool — "Submissions" inbox:**
- Queue (account, patient, submitted-by, age, status, confidence), detail with embedded PDF view, and the **match-to-referral** UI (account-scoped candidate referrals; shows full context to prevent wrong-patient linkage — analogous to the §8 did-you-mean discipline but for patients).

---

## 8. HIPAA / compliance impact — **CRITICAL, read this first**

This is the single most important thing to decide.

- Today, §15 scopes V1 to **internal testing on synthetic data only**, and `TODOS.md` parks the entire HIPAA workstream as the **P0 hard blocker** before any external office or any real patient data.
- This feature, **used for real**, means an **external provider office uploads a real patient's demographics (PHI) through the extension.** That is exactly the trigger the P0 blocker guards against. It also creates a **new PHI-at-rest store** (the uploaded PDFs) that the current spec doesn't have.

**Therefore (proposed posture — this is UC-1 for the gate):**
1. **Build + test this feature in V1 on synthetic demographic PDFs only.** No real patient PDFs until the HIPAA workstream is complete.
2. **Enabling it for real offices is gated on the P0 HIPAA workstream** (BAA chain for hosting + MySQL + any email; encryption-at-rest for the PDF store; retention/purge for PDFs + submissions; breach-response + wrong-patient-exposure runbook). This aligns with `docs/hosting-and-auth-migration-strategy.md` (Fly.io HIPAA + covered MySQL).
3. **New PHI-at-rest controls specific to this feature:** encrypted PDF blob store, retention + purge job for `PatientSubmission` + source PDFs, malware/file-safety scan, no PDF text or demographics in logs (extend §16 "do not log" list to PDF content + extraction values).

I will not let this feature quietly move real PHI into a non-covered environment. If you want it enabled for live offices sooner, the honest path is to pull the HIPAA workstream forward, not to relax the boundary.

---

## 9. PDF extraction approach (proposed)

The hard part. Demographic PDFs vary from clean digital forms to scanned faxes.

- **Decision (D-C):** start with **library/text extraction** (e.g. `pdf-parse` / `pdfjs`) for digital PDFs, with a **template-aware field map** for a known Insight/hospital demographic form if one exists (Q2). Add **OCR** (e.g. Tesseract) as a later increment for scanned/image PDFs. **Always human-in-the-loop:** the office confirms/corrects extracted fields before submission (D-B). This caps blast radius — no silently mis-read DOB ever becomes a submission.
- **LLM-based extraction is explicitly deferred** for V1: it's powerful but sends PHI to a third party, which adds a BAA dependency and a compliance surface. Revisit post-HIPAA if accuracy demands it.
- File safety: enforce `application/pdf` by **magic bytes** (not just extension), size cap (reuse `UPLOAD_MAX_BYTES`), reject encrypted/malformed PDFs with clear copy, and scan before parse. Parse in a constrained worker.

---

## 10. Authorization & matching safety

- **Provider write is fail-closed and account-scoped:** a `PROVIDER_STAFF` user may only submit for an account in their Better Auth claims (`X-Account-Id ∈ accountIds`); otherwise 404/403, never an unscoped write. Submissions are stamped with `req.localUser.id`.
- **Patient↔Referral linkage is ops-confirmed and never automatic.** Name+DOB collisions are a real wrong-patient hazard (the patient analogue of the §8 account-collision rule). The match UI shows full context; cross-account candidates are never offered. This is the same fail-closed philosophy that protects cross-tenant isolation today.

---

## 11. Risk register (multi-lens critical findings)

| # | Lens | Sev | Finding | Mitigation |
|---|---|---|---|---|
| R1 | Compliance | **CRITICAL** | Real PHI enters via the provider/extension side → breaks §15 internal-only boundary; new PHI-at-rest (PDFs). | UC-1: synthetic-only in V1; gate live use on P0 HIPAA workstream; encrypted PDF store + retention/purge. |
| R2 | Engineering | **CRITICAL** | PDF mis-extraction → wrong demographics → wrong-patient match. | Human-confirm before submit (D-B); per-field confidence; ops review; never auto-link (D-D). |
| R3 | Engineering | HIGH | Patient-level match collisions (same name/DOB). | Ops-confirmed linkage, account-scoped candidates, full-context match UI; fail-closed. |
| R4 | Security | HIGH | Malicious/oversized/non-PDF uploads; PHI in logs. | Magic-byte + size checks, malware scan, sandboxed parse, extend §16 no-log list. |
| R5 | Product | MED | "Immediate" expectation vs daily hospital cadence + ops review. | UC-2: two-stage visibility; honest copy ("submitted — Insight is matching this patient"). |
| R6 | Product | MED | Duplicate submissions / patient already tracked. | Dedupe heuristics + `DUPLICATE` status; surface likely existing referral to ops. |
| R7 | DX/Extension | MED | MV3 limits; large PDFs from a popup; SW constraints. | Upload from popup (not the service worker); stream; no PHI in extension storage. |
| R8 | Design | LOW/MED | "New Order" adds a second job to a list-first popup. | Tab (T-1) so the daily tracker stays the default surface; New Order is opt-in. |

---

## 12. Decisions (D = recommended-accept · UC = your call at the gate · T = taste, overridable)

| # | Decision | Effort | Recommendation |
|---|---|---|---|
| D-A | New `PatientSubmission` + `PatientSubmissionEvent` entities & lifecycle (§5) | M | ACCEPT |
| D-B | Extract-then-confirm: office verifies fields before a submission persists | S/M | ACCEPT |
| D-C | Library/text extraction first; OCR later; LLM extraction deferred (pre-HIPAA) | M | ACCEPT |
| D-D | Patient↔Referral linkage is ops-confirmed, account-scoped, never automatic | S | ACCEPT |
| D-E | Encrypted PDF blob store + retention/purge + file-safety + no-PDF-in-logs | M | ACCEPT |
| **UC-1** | Real PHI ingress → **synthetic-only in V1; live use gated on P0 HIPAA workstream** | — | **YOUR DECISION** |
| **UC-2** | "Immediate" status: accept two-stage visibility (ack now, tracker after match) | — | **YOUR DECISION** |
| T-1 | "New Order" as a popup **tab** (vs separate surface) | S | tab |

---

## 13. Test plan (ship-blocking for this feature)

1. **Submission scoping (fail-closed):** provider can only submit/read for accounts in claims; cross-account submit or read → 404/403; missing claims → denied, never an unscoped write.
2. **Confirm gate:** nothing persists as a `SUBMITTED` row without office confirmation; the **corrected** fields are what persist (not raw extraction).
3. **Match safety:** linkage never crosses accounts; same-name/DOB requires explicit ops confirm; a `MATCHED` submission's referral appears in `GET /patients`.
4. **File safety:** non-PDF (by magic bytes) rejected; oversize rejected; malformed/encrypted PDF handled gracefully; **no PDF text / demographics / file bytes in logs.**
5. **Lifecycle:** valid status transitions only; `DUPLICATE` path; events written with `actorUserId`.
6. **PHI boundary:** V1 suites run on **synthetic PDFs only**; assert the synthetic-only guardrail.

These extend the existing ship-blocking philosophy (cross-tenant isolation, fail-closed auth — §18).

---

## 14. Where this slots in the build (after authorization)

Depends on auth + accounts + referrals + provider read API, so it sits **after** BUILD-SPEC Steps 1–7 (it can't be wired before the backend auth/account/referral foundation exists). Proposed as a dedicated track in the upcoming **BUILD-SPEC-v0.4** (Better Auth + MySQL):

- **NF-1** Schema: `PatientSubmission` + events + back-refs; migration.
- **NF-2** Backend: file store + extraction service + submission routes (provider) with fail-closed scope + rate limits.
- **NF-3** Backend: ops inbox + match/status routes; patient-match candidates (account-scoped).
- **NF-4** Extension: "New Order" tab (dropzone → extraction review → confirm → my-submissions).
- **NF-5** Internal tool: Submissions inbox + match-to-referral UI.
- **NF-6** Safety/compliance: encrypted PDF store, retention/purge, file-safety, no-PDF logs; synthetic-PDF fixtures + ship-blocking suites.

Per your instruction, **I will not start NF-2+ (the backend API connections) until you authorize this plan.** Schema/spec drafting (NF-1 as spec) and fixtures are safe to prepare in parallel if you'd like.

---

## 15. Open questions for the final gate

1. **"IVS"** — what does it stand for? (so the spec uses your exact term.)
2. **Is there a canonical demographic PDF template** (a fixed Insight/hospital form), or will offices upload arbitrary demographic sheets? This drives extraction effort (template map vs OCR).
3. **Who reviews submissions** — `INTERNAL_ADMIN` only, or `INTERNAL_STAFF` too?
4. **UC-1:** confirm synthetic-only in V1 with live use gated on the HIPAA workstream — or do you want the HIPAA workstream pulled forward so real offices can use it sooner?
5. **UC-2:** is two-stage visibility (immediate ack, tracker after match) acceptable, given the hospital export is daily and ops reviews each submission?
6. **Scope timing:** is this part of the V1 build, or a V1.1 track after the core tracker ships?

---

## 16. Bottom line

This is a high-value feature that closes the intake loop and gives Insight an electronic new-patient queue it lacks today. It is buildable on the planned Better Auth + MySQL stack with the existing fail-closed/account-scoping discipline extended to a provider-write path and a patient-level match. **The one decision that gates everything is HIPAA (UC-1):** a real demographics PDF is real PHI, so V1 builds/tests on synthetic PDFs and live use waits on the P0 workstream. Authorize the decisions above (especially UC-1 and UC-2) and I'll fold this into BUILD-SPEC-v0.4 and begin the backend wiring.
