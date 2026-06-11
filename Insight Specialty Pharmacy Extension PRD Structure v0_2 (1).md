# Insight Specialty Pharmacy Extension

### TL;DR

Insight Specialty Pharmacy Extension is a provider-facing browser extension and internal command dashboard that turns Insight’s specialty pharmacy workflow into a simple daily status briefing for physician offices. V1 will ingest a structured daily file from Insight’s outpatient hospital program workflow, map each patient/referral to the correct provider account, and show where each case sits across intake, prior authorization, telehealth, approval, fulfillment, and delivery. The product should feel like a modern AI-style operational assistant, not another portal.

---

## Product Thesis

Provider offices do not need more portals. They need confidence that Insight received the referral, knows where the patient is in the workflow, and will clearly surface when the office needs to act.

The SOP makes the product opportunity clearer: Insight already has a four-phase workflow with SLAs, controls, and timestamp requirements. The extension should not invent a new operational process. It should expose the right slice of the existing process to each provider office in a clean, account-specific, action-oriented way.

The first version should do five things exceptionally well:

* Accept a daily structured file from Insight operations.
* Map each record to a provider account, physician, and location where available.
* Translate internal workflow stages into provider-friendly statuses.
* Show office staff a daily “here is what changed” view.
* Give Insight admins a lightweight command view across accounts, data quality, status bottlenecks, and SLA risk.

The product should not become a full EMR, a prior authorization platform, a pharmacy management system, or a general-purpose reporting portal in V1.

---

## SOP-Derived Workflow Model

Source SOP: SOP-SPI-001, Specialty Pharmacy Insight Outpatient Hospital Program, End-to-End Workflow, Version 2.0.

### Phase 1: Physician Enrollment & Prescription Submission

Owner: Compliance / Physician Office  

SLA: Same day

Workflow steps:

* Physician completes and signs CDTM agreement.
* eRx is transmitted to Insight Pharmacy via EMR.
* Patient demographics and insurance are faxed to the PA Team.

Controls and safeguards:

* Mandatory field validation.
* Auto-confirmation sent.
* CRM verifies NPI, license, specialty, and EMR system.
* Auto-alert on receipt.
* Monitored fax line as backup.

Provider-facing implication:

* The extension should confirm receipt and surface whether the submission package is complete.
* Provider staff should know whether the patient is “received and ready for intake” or “missing required information.”

### Phase 2: Pharmacy Intake & Prior Authorization

Owner: Pharmacy Intake / PA Team  

SLA: 24–48 hours, with complex PA cases potentially extending to 2–3 weeks after complete documentation is received.

Workflow steps:

* eRx received, logged, and verified in system.
* Triage tagging applied: urgent vs. routine.
* PA Team coordinates authorization with carrier.

Controls and safeguards:

* Patient, drug, and prescriber ID verified per SOP.
* Daily audit cross-checks log versus submissions.
* Escalation triggered if patient is unresponsive for more than 24 hours.

Provider-facing implication:

* The extension should show intake verification, triage priority, PA state, and next action owner.
* The product must distinguish normal PA waiting from a true blocker.

### Phase 3: Telehealth Completion & Approval

Owner: Onboarding Team  

SLA: 24 hours

Workflow steps:

* Telehealth visit scheduled with 2–3 daily slots held.
* Visit completion auto-triggers fulfillment notice.
* Insurance approval logged in centralized workflow.

Controls and safeguards:

* Centralized tracking of eligibility and scheduling.
* Automatic pharmacy notification upon completion.
* Approval confirmation timestamped for audit trail.

Provider-facing implication:

* The extension should show whether telehealth is pending, scheduled, missed, or completed.
* Approval should be visible as a major transition point toward fulfillment.

### Phase 4: Medication Fulfillment & Delivery

Owner: Pharmacy Fulfillment / QA  

SLA: 24–48 hours

Workflow steps:

* Patient address reconfirmed before dispatch.
* Medication packed with temperature controls.
* Medication shipped with signature and delivery confirmation.

Controls and safeguards:

* Barcode tracking with automated patient alerts.
* Cold-chain verification where applicable.
* Final QA review completed before release.

Provider-facing implication:

* The extension should show fulfillment readiness, shipping status, and delivery confirmation when available.
* It should avoid exposing internal QA details unless they affect timing or provider action.

---

## Goals

### Business Goals

1. Reduce provider-office status-check calls and emails by giving offices a daily self-serve view of referral progress.
2. Differentiate Insight from large specialty pharmacies by giving practices operational transparency that competitors typically do not provide.
3. Improve referral confidence by confirming eRx receipt, intake verification, PA progress, telehealth status, and shipping status.
4. Create a scalable account-management layer that allows Insight to onboard new practices with repeatable account setup and file-based publishing.
5. Create a business-intelligence lens for Insight leadership and account managers to identify account health, bottlenecks, SLA risk, and growth opportunities.

### User Goals

1. Office staff can confirm whether Insight received a patient referral without calling or emailing.
2. Office staff can see which patients are in intake, PA, telehealth, approval, fulfillment, or shipping.
3. Office staff can identify cases requiring provider-office action, especially missing demographics, insurance, documentation, or patient responsiveness issues.
4. Office managers can understand account-level workflow volume and bottlenecks in under one minute.
5. Insight staff can publish accurate account-specific updates from a daily file without manually sending office-by-office updates.

### Non-Goals

1. V1 will not replace Insight’s internal pharmacy workflow systems, EMR, CRM, PA tools, or shipping systems.
2. V1 will not submit prior authorizations, conduct telehealth, or manage pharmacy fulfillment directly.
3. V1 will not expose internal operational notes, compliance-only fields, or sensitive QA details unless explicitly approved.
4. V1 will not require direct real-time EMR integration.
5. V1 will not become a broad analytics suite before the provider-facing workflow is validated.

---

## User Personas

### Provider Office Administrative Staff

Primary daily user. Tracks referrals, confirms patient status, responds to missing information, and communicates with clinical staff.

### Provider Office Manager

Secondary user. Wants a quick operating view of referral volume, delayed cases, and office-owned action items.

### Prescriber or Clinical Staff

Occasional user. Needs confidence that the specialty prescription is progressing and that patients are not stuck due to administrative gaps.

### Insight Operations Admin

Internal user. Uploads daily files, validates records, maps data to accounts, reviews exceptions, and publishes updates.

### Insight Account Manager / Business Lead

Internal user. Reviews account performance, office adoption, referral volume, SLA bottlenecks, and business growth signals.

---

## User Stories

### Provider Office Administrative Staff

* As provider office staff, I want to open the extension and immediately see today’s Insight updates, so that I can understand patient progress without logging into another portal.
* As provider office staff, I want to confirm that eRx was received and logged, so that I know the referral did not disappear after submission.
* As provider office staff, I want to see whether patient demographics and insurance are complete, so that I can fix missing information quickly.
* As provider office staff, I want to filter by PA status, so that I can answer provider and patient questions without calling Insight.
* As provider office staff, I want to see when a patient is pending telehealth, so that I know whether delay is related to scheduling, patient completion, or pharmacy processing.
* As provider office staff, I want to see shipping or delivery status, so that I can close the loop with the care team.

### Provider Office Manager

* As an office manager, I want a high-level count of patients by workflow phase, so that I can understand operational load at a glance.
* As an office manager, I want to see aging or overdue cases, so that I can escalate before patient care is delayed.
* As an office manager, I want to see which cases require my office’s action, so that staff can prioritize the right work.

### Prescriber or Clinical Staff

* As a prescriber, I want to know that the patient’s specialty medication was received and is moving through the workflow, so that I can trust the referral process.
* As clinical staff, I want a simple explanation of the current blocker, so that I know whether to intervene clinically or administratively.

### Insight Operations Admin

* As an operations admin, I want to upload a daily structured file and preview the resulting provider-facing updates, so that I can publish accurate data with confidence.
* As an operations admin, I want unmatched accounts, missing fields, duplicate referrals, and invalid statuses flagged before publishing, so that data quality does not damage trust.
* As an operations admin, I want to see SLA-risk records by workflow phase, so that internal teams can act before offices complain.

### Insight Account Manager / Business Lead

* As an account manager, I want to see account-level referral volume and bottlenecks, so that I can manage relationships proactively.
* As an account manager, I want to see provider adoption and usage, so that I know which offices are getting value from the extension.
* As a business lead, I want a small YC-style operating dashboard across accounts, so that I can quickly see which accounts are growing, stuck, or at risk.

---

## Functional Requirements

### Provider Extension: Daily Account Briefing (Priority: P0)

* Personalized greeting

  * Show a time-aware greeting such as “Good morning, GI Medical Services.”
  * Show “Here are today’s Insight updates” with a clear last-published timestamp.
  * The tone should feel warm, polished, and operationally useful.

* Workflow summary cards

  * Display counts across the SOP-derived workflow phases:
    * New / received.
    * Intake verification.
    * Prior authorization.
    * Telehealth / onboarding.
    * Approved / fulfillment ready.
    * Shipping / delivery.
    * Needs office action.
  * Each count should be clickable as a filter.

* Patient list and filter chips

  * Provide filter chips at the top to avoid long scrolling.
  * Minimum filter chips:
    * All active.
    * New today.
    * Needs office action.
    * eRx received.
    * Intake review.
    * PA pending.
    * PA approved.
    * Telehealth pending.
    * Fulfillment.
    * Shipping.
  * Each chip should show a count.

* Patient row/card

  * Show a compact row or card with:
    * Patient name or approved patient identifier.
    * Date of birth or safe secondary identifier, pending final privacy decision.
    * Prescriber.
    * Medication or therapy class, if approved.
    * Current provider-facing status.
    * Workflow phase.
    * Triage tag: urgent or routine, if available and appropriate.
    * Next action owner.
    * Last updated timestamp.
    * SLA indicator when relevant.

* Patient detail drawer

  * Clicking a patient opens a focused detail panel.
  * Detail panel includes:
    * Current status.
    * Status explanation in plain language.
    * Next action owner: Insight, provider office, patient, payer, carrier, or none.
    * SOP phase.
    * SLA target and current age in stage.
    * Status history if available from the file.
    * Controlled next-step guidance.

* SLA-aware presentation

  * Display aging carefully and factually.
  * Example: “PA pending, day 3 since complete documentation received.”
  * For complex PA cases, avoid implying a missed SLA when the SOP allows 2–3 weeks.

### Admin Dashboard: Internal Command View (Priority: P0)

Admin Dashboard: Internal Command View (Priority: P0)

* Admin overview: Provide a compact operating dashboard across all accounts. Show total active referrals, new referrals, PA pending, telehealth pending, fulfillment-ready, shipped, and needs attention. Highlight data quality issues before workflow analytics.
* Account cards / Obsidian-style account map: Provide small account cards that summarize each provider account. Each account card includes: Account name, Active referral count, New referrals today, Needs office action count, SLA-risk count, Last provider login, Last file update. Optional future direction: relationship graph showing accounts, prescribers, locations, and workflow bottlenecks.
* YC-style business lens: Show simple, decision-oriented signals: Growing account, Stalled account, Friction account, At-risk account, Expansion opportunity. This is not a full BI product in V1. It is an operating pulse.
* Account management: Create and edit provider accounts. Store account name (required in V1), account number (optional), locations, prescribers, NPI where relevant, specialty, EMR system, and active state. Map uploaded records to accounts using account_name as the likely primary mapping field with normalization and secondary matching by practice NPI and prescriber NPI where available. Include manual review and admin preview before publish to handle fragile name-only matches.
* User management: Invite provider users by email. Assign users to one or more provider accounts. Activate, deactivate, and reset users. View last login and adoption status.
* File upload and publish: Upload CSV or XLSX from hospital Google Sheets export. Parse and preview records. Validate required fields. Show account-level impact before publish. Publish updates to provider accounts. Record upload, validation, and publish events in an audit log.
* Data quality validation: Block publish for critical failures: Missing account_name (required), Missing referral_id (required), Unmatched account (after matching rules and admin review), Missing patient identifier (patient name DOB), Missing current status, Invalid status mapping, Duplicate active referral without a resolution rule. Warn but do not necessarily block for non-critical issues: Missing medication field (recommended), Missing prescriber (recommended), Missing triage tag, Missing optional status note.

### Status Mapping and Workflow Logic (Priority: P0)

* Internal workflow statuses must map into provider-facing language.
* Every provider-facing status should answer:
  * What happened?
  * What phase is the patient in?
  * Who owns the next action?
  * Is the case on track, waiting, blocked, or complete?

Draft Provider-Facing Status Taxonomy

| SOP Phase | Provider-Facing Status | Meaning | Default Next Action Owner |
| --- | --- | --- | --- |
| Physician Enrollment & Submission | Physician enrollment pending | CDTM or provider setup is not complete. | Provider office / Insight |
| Physician Enrollment & Submission | eRx received | Insight received the eRx transmission. | Insight |
| Physician Enrollment & Submission | Demographics / insurance received | Supporting patient information was received. | Insight |
| Physician Enrollment & Submission | Missing submission information | Required prescription, demographic, insurance, or provider information is incomplete. | Provider office |
| Pharmacy Intake & PA | Intake verification in progress | Insight is verifying patient, drug, and prescriber information. | Insight |
| Pharmacy Intake & PA | Intake verified | Referral passed intake verification. | Insight |
| Pharmacy Intake & PA | Urgent triage | Case has been tagged urgent. | Insight |
| Pharmacy Intake & PA | Prior authorization initiated | PA process has started with the carrier. | Insight / payer |
| Pharmacy Intake & PA | Prior authorization pending | PA is awaiting payer response or documentation. | Payer / provider office / Insight |
| Pharmacy Intake & PA | Prior authorization approved | Insurance approval has been logged. | Insight |
| Pharmacy Intake & PA | Prior authorization denied | PA was denied and requires next-step review. | Insight / provider office |
| Telehealth & Approval | Telehealth scheduling pending | Patient needs telehealth scheduling. | Patient / Insight |
| Telehealth & Approval | Telehealth scheduled | Visit is scheduled. | Patient |
| Telehealth & Approval | Telehealth no-show | Patient missed scheduled visit. | Patient / Insight |
| Telehealth & Approval | Telehealth completed | Visit completion was logged and fulfillment notice triggered. | Insight |
| Fulfillment & Delivery | Fulfillment in progress | Medication is being prepared for release. | Insight |
| Fulfillment & Delivery | Address confirmation pending | Patient address must be confirmed before dispatch. | Patient / Insight |
| Fulfillment & Delivery | QA review in progress | Final internal review before release. | Insight |
| Fulfillment & Delivery | Medication shipped | Medication has been shipped. | Carrier |
| Fulfillment & Delivery | Delivered | Delivery confirmation was received. | None |
| Cross-phase | Closed / cancelled | Case is no longer active. | None / provider office depending reason |

---

## User Experience

### Entry Point & First-Time User Experience

* Insight invites a provider office user to install the browser extension.
* User signs in and is routed to the assigned account.
* If the user has multiple accounts or locations, they choose the account context.
* First-time onboarding is limited to three screens:
  * What the extension shows.
  * How often data is updated.
  * What “Needs office action” means.

### Core Provider Experience

* Step 1: User opens the extension.

  * A clean greeting appears: “Good morning, GI Medical Services.”
  * The extension shows the latest publish timestamp.
  * It must never imply real-time status unless the backend actually supports it.

* Step 2: User sees today’s operating brief.

  * Summary cards show counts by workflow stage.
  * “Needs office action” is visually elevated.
  * “New today” shows newly received or newly updated patients.

* Step 3: User filters to the work that matters.

  * User clicks “PA pending” or “Telehealth pending.”
  * List updates instantly.
  * Filter state is obvious and easy to clear.

* Step 4: User reviews patient cards.

  * Each card is readable in a few seconds.
  * The row should not be overloaded with internal details.
  * Status language is provider-friendly, not internal jargon.

* Step 5: User opens detail only when needed.

  * Detail drawer shows status explanation, next action, SLA context, and status history.
  * If office action is required, the instruction is specific.
  * Example: “Insurance information is incomplete. Please send updated insurance card or eligibility details to Insight.”

* Step 6: User returns to work.

  * Ideal session length should be under one minute for routine status review.

### Core Admin Experience

* Step 1: Admin opens command dashboard.

  * Sees latest file publish, total active records, account exceptions, and data quality warnings.

* Step 2: Admin uploads daily file.

  * File parser runs validations.
  * Admin sees record count, matched accounts, unmatched accounts, status distribution, and critical errors.

* Step 3: Admin previews account impact.

  * Admin can view what GI Medical Services or another account will see after publish.
  * This prevents accidental exposure or confusing status changes.

* Step 4: Admin publishes.

  * System updates provider-facing dashboards.
  * Publish event is timestamped and logged.

* Step 5: Business user reviews account pulse.

  * Account cards show where volume, friction, and risk exist.
  * The dashboard should answer: “Which accounts need attention today?”

### Advanced Features & Edge Cases

* No updates today

  * Show: “No new updates today. Current cases were last refreshed on \[date/time\].”

* File not published today

  * Show a clear timestamp and avoid “today” language.

* Complex PA cases

  * Show PA aging relative to complete documentation date, not original referral date.
  * Avoid false overdue indicators for cases where SOP allows 2–3 weeks.

* Patient unresponsive over 24 hours

  * Internal admin view should flag this as an escalation trigger.
  * Provider-facing view should only show it if the office can help resolve it.

* Telehealth no-show

  * Show only if operationally useful and approved.
  * Pair with next-step guidance.

* Cold-chain / QA exceptions

  * Keep internal by default unless the issue affects delivery timing or requires office awareness.

---

## Narrative

It is Monday morning at GI Medical Services. The administrative team has a full inbox, incoming calls, pending faxes, and providers asking whether patients’ specialty medication referrals are moving. Historically, confirming the status of a patient meant calling the pharmacy, checking email chains, or waiting for a callback.

Now, the office opens the Insight Specialty Pharmacy Extension and sees a simple greeting: “Good morning, GI Medical Services. Here are today’s Insight updates.” The page immediately shows which patients were received through eRx, which are in intake verification, which are awaiting prior authorization, which are pending telehealth, and which are moving into fulfillment or shipment. The office can filter to “Needs office action” and instantly see whether missing demographics, insurance information, or documentation is blocking a patient.

Behind the scenes, Insight operations has uploaded a daily workflow file built from the same SOP-governed process used internally: submission, intake, PA, telehealth, approval, fulfillment, and delivery. The admin dashboard validates the file, maps records to provider accounts, and gives Insight a small operating command center across account health, data quality, and SLA risk. The result is not another portal. It is a daily confidence layer that helps provider offices trust Insight and helps Insight scale a differentiated specialty pharmacy experience.

---

## Success Metrics

### User-Centric Metrics

* Provider account activation rate

  * Percentage of invited accounts with at least one active user within 14 days.

* Weekly active account rate

  * Percentage of onboarded accounts opening the extension weekly.

* Time to status answer

  * Median time from extension open to patient status view.
  * Target: under 30 seconds for routine cases.

* Needs-action visibility

  * Percentage of provider-owned action items viewed within one business day.

* Office satisfaction

  * Qualitative feedback from pilot offices and simple in-product score after pilot.

### Business Metrics

* Reduction in provider-office status-check calls and emails.
* Referral volume change for extension-enabled accounts versus baseline.
* Increase in retained or expanded provider relationships.
* Number of accounts onboarded per month.
* Account-manager-reported reduction in manual follow-up burden.

### Operational Metrics

* Average eRx-to-ship turnaround.
* Telehealth no-show rate.
* Incomplete script percentage.
* PA denial rate.
* Average time in each SOP phase.
* SLA-risk count by account.

### Technical Metrics

* Successful file processing rate.
* File validation failure rate.
* Unmatched account rate.
* Publish completion time.
* Extension load time.
* Account-level access control incident count, target zero.

### Tracking Plan

* User invited.
* User activated.
* Extension opened.
* Account selected.
* Dashboard viewed.
* Filter clicked.
* Patient detail opened.
* Needs-action record viewed.
* File uploaded.
* File validation completed.
* Critical validation error generated.
* Account preview viewed.
* File published.
* Account card viewed.
* SLA-risk record viewed.
* User created.
* User deactivated.

---

## Technical Considerations

### Technical Needs

Technical Needs

* Browser extension front end.
* Provider-facing data API (reads processed file; does not require live EMR access).
* Internal admin dashboard.
* Authentication and role-based access control.
* CSV/XLSX file upload service and parser for hospital-exported Google Sheets.
* File validation and parsing pipeline tuned for hospital sheet export quirks.
* Status mapping layer from file values to provider-facing statuses.
* Account matching engine that prioritizes account_name matching with normalization and secondary matching using practice NPI and prescriber NPI.
* Audit logging.
* Database for users, accounts, file batches, referrals, statuses, and publish history.

### Integration Points

Initial V1: Hospital-outputted Google Sheets export imported as CSV or XLSX.

* Daily file exported from Insight’s centralized workflow or internal reporting process (hospital Google Sheets export as the source of truth for V1).
* Provider account data from CRM or manually maintained account table (optional; may not be authoritative in V1).


* Future:
* Direct EMR or eRx feeds (deferred for V1; extension relies on hospital-exported sheet).
* PA platform data feed.
* Telehealth scheduling system.
* Shipping carrier tracking.
* CRM account health data.

### Data Storage & Privacy

Data Storage & Privacy

* Store the minimum fields necessary to identify the patient to the provider office and communicate status.
* Maintain strict account-level data separation.
* Avoid uncontrolled free-text notes in provider-facing views.
* Preserve timestamped audit trails for file upload, validation, publish, account changes, and user management.
* Define record retention policy for file batches and historical statuses.
* Separate internal-only statuses from provider-facing statuses.

Note: In V1 the source of truth is the hospital-outputted Google Sheets export (CSV/XLSX). The extension is explicitly downstream of the uploaded file and must not assume direct access to the hospital system, EMR, CRM, pharmacy system, or any live system of record.

### Scalability & Performance

* V1 should support a pilot of 1–3 accounts, then scale to additional accounts without data model rework.
* Extension should load dashboard summary quickly, ideally under two seconds after authentication.
* Large accounts should use filters, search, and pagination rather than long scroll.
* File validation should complete fast enough for daily operations, with clear progress and error states.

### Potential Challenges

Potential Challenges

* The daily hospital-exported sheet may not contain enough structured detail to support clean status mapping.
* Account mapping may be messy if account_name, practice NPI, NPIs, locations, or prescriber names are inconsistent.
* PA timelines are variable, so SLA indicators must be carefully modeled around complete documentation date.
* Internal statuses may be too granular or too operational for provider offices.
* Provider users may assume data is real-time unless timestamps and language are explicit.
* The admin dashboard can become bloated if leadership asks for full BI too early.

Warning: Account-name-only mapping is fragile and must use normalization, manual review, strong secondary matching (practice and prescriber NPI), and an admin preview step before publish.

---

## Critical Product Risks and Harsh Critique

### Risk 1: The product is only as good as the data dump

If the file is stale, incomplete, or inconsistent, the UI does not matter. The file schema, validation rules, and publish preview are core product infrastructure, not backend chores.

### Risk 2: “Admin dashboard” can become a graveyard of half-built analytics

The admin dashboard should not become a giant reporting product in V1. Build a command center, not Tableau. The first version should answer: Was the file good? Which accounts were updated? Which records are broken? Which accounts need attention today?

### Risk 3: Provider offices do not care about Insight’s internal process

They care about outcomes: received, pending, approved, needs action, shipping. The SOP should inform the product logic, but the UI should not dump SOP complexity onto office staff.

### Risk 4: SLA indicators can backfire

If you show overdue warnings without modeling PA complexity and complete-documentation timing, you create complaints instead of trust. SLA logic must be precise or hidden.

### Risk 5: You need a wedge, not a platform

The wedge is referral visibility. Do not overbuild messaging, document exchange, analytics, or EMR awareness before the extension proves that offices use it weekly.

### Risk 6: The “sleek modern” design cannot compensate for workflow ambiguity

The product can look like Claude, Linear, or a modern SaaS dashboard, but if statuses are vague, users will still call Insight. The product must be operationally crisp before it is visually premium.

---

## Proposed V1 Scope

### Must Have

* Provider login.
* Account-specific extension dashboard.
* Personalized greeting and last-published timestamp.
* Summary cards by SOP workflow phase.
* Filter chips by status and action owner.
* Patient list with current status, phase, next action owner, and last updated timestamp.
* Patient detail drawer.
* Admin account management.
* Admin provider user management.
* Daily file upload.
* Validation preview.
* Account mapping.
* Provider-facing preview before publish.
* Publish workflow.
* Audit log for upload, validation, publish, and user/account changes.
* Internal account overview cards.

### Should Have

* Search by patient name or identifier.
* Triage tag: urgent vs. routine.
* SLA-risk indicators in admin view.
* Needs-office-action prioritization.
* Account adoption signals such as last login.
* Basic account health labels: growing, stalled, friction, at risk.

### Could Have

* Browser badge for new updates.
* Email digest for offices that do not open the extension daily.
* Saved filters.
* Provider export.
* Lightweight account graph / Obsidian-style relationship map.
* Shipping tracking link or delivery confirmation display.

### Won’t Have in V1

* Direct EMR writeback.
* Real-time integration.
* In-extension messaging.
* Automated prior authorization submission.
* Patient-facing access.
* Full document exchange.
* Full BI dashboard.
* Internal QA workflow management.

---

## Draft File Schema

| **Field** | **Required** | **Notes** |
| --- | --- | --- |
| batch_id | Yes | Unique file batch identifier. |
| account_number | No (optional) | Optional field; may be present but not assumed primary for mapping in V1. |
| account_name | Yes | Primary mapping field for V1. Account-name-only mapping is fragile and must use normalization, manual review, and admin preview before publish. |
| location_name | No | Useful for multi-location offices. |
| prescriber_name | Yes | Supports filtering and provider confidence. |
| prescriber_npi | Recommended | Strongly recommended secondary matching field. |
| practice_npi | Recommended | Strongly recommended secondary matching field for account accuracy. |
| emr_system | No | Useful for account intelligence and troubleshooting. |
| referral_id | Yes | Required. Likely stable unique patient/referral identifier for V1. |
| patient_first_name | Yes | Provider-facing identification. |
| patient_last_name | Yes | Provider-facing identification. |
| patient_dob | Yes | Provider-facing identifier; final display depends on privacy decision (TBD masking rules). |
| medication_name | Recommended | Useful for office staff if approved; provider-facing identification field. |
| source_type | Recommended | eRx, fax, EMR, manual, other. |
| erx_received_at | Recommended | Supports Phase 1 receipt confirmation. |
| demographics_received | Recommended | Boolean or status. |
| insurance_received | Recommended | Boolean or status. |
| documentation_complete_at | Recommended | Critical for PA SLA logic. |
| triage_tag | Recommended | Urgent or routine. |
| internal_status | Yes | Raw operational status. |
| provider_status | Yes | Mapped provider-facing status. |
| workflow_phase | Yes | Submission, intake/PA, telehealth, fulfillment/delivery. |
| next_action_owner | Yes | Insight, provider office, patient, payer, carrier, none. |
| status_note | No | Controlled short note only. |
| status_updated_at | Yes | Timestamp of status update. |
| telehealth_scheduled_at | No | Supports Phase 3 display. |
| telehealth_completed_at | No | Supports fulfillment trigger. |
| approval_logged_at | No | Supports approval timestamp. |
| fulfillment_started_at | No | Supports Phase 4. |
| shipped_at | No | Supports shipping status. |
| delivered_at | No | Supports delivery confirmation. |
| terminal_status | No | Completed, closed, cancelled, transferred. |

---

MVP Acceptance Criteria

1. Admin can create a provider account with account_name (required), account_number (optional), prescribers, locations, and active status.
2. Admin can invite and assign provider users to one or more accounts.
3. Admin can upload a hospital-exported Google Sheets CSV/XLSX file.
4. System validates required fields and blocks publish on critical errors (see validation rules where account_name and referral_id are required).
5. System maps file records to provider accounts using account_name as the likely primary mapping field with normalization and secondary matching by practice NPI and prescriber NPI where available. Admin preview and manual review are required for ambiguous name-only matches.
6. Admin can preview provider-facing account output before publishing.
7. Provider user can only access assigned account data.
8. Provider extension displays personalized greeting and last-published timestamp.
9. Provider extension displays counts by SOP workflow phase.
10. Provider user can filter by status, workflow phase, and needs-office-action.
11. Provider user can open patient detail and see status, next action owner, SLA context where available, and last updated timestamp.
12. Admin dashboard shows account cards with active referrals, new referrals, needs-action count, SLA-risk count, last login, and last update.
13. Upload, validation, publish, account changes, and user changes are audit logged.
14. If no current updates exist, the extension displays a clear empty state.
15. If the latest file is stale, the extension clearly communicates the last refresh time.

Note: The V1 implementation relies solely on the hospital-exported sheet; it does not assume access to EMR, CRM, pharmacy systems, or other live systems.

---

Open Questions

### Workflow and Data

1. What internal system generates the hospital-exported Google Sheets daily data file?
2. Is the daily file cumulative or incremental?
3. What is the stable unique identifier for each referral (referral_id is required in V1)?
4. Is “complete documentation received” currently timestamped?
5. Are eRx received time, demographics received, and insurance received separately tracked?
6. How is triage tag captured today?
7. How is patient unresponsive greater than 24 hours captured?
8. How are telehealth scheduled, completed, and no-show states captured?
9. How are shipping and delivery confirmation captured?
10. Which internal statuses should never be shown externally?

### Account Mapping

1. Is account_name consistently available and reliable in every export? (In V1 account_name is required and treated as the primary mapping field.)
2. Should practice NPI be used as a secondary matching field? (Recommended)
3. Should prescriber NPI be used as a secondary matching field? (Recommended)
4. Can a prescriber belong to multiple accounts or locations?
5. How should multi-location offices be represented?
6. Who owns account setup and account data accuracy?

### Product and UX

1. Should the extension open as a popup, side panel, or full extension page?
2. Should patient DOB be displayed, partially masked, or omitted?
3. Should provider offices see triage tags such as urgent vs. routine?
4. Should provider offices see telehealth no-shows?
5. Should provider offices see SLA aging, or should that remain internal at first?
6. Should provider users be able to acknowledge action-required items?
7. Should the admin dashboard include an account relationship graph in V1 or defer it?

### Account Mapping

1. Is account number consistently available in every export?
2. Should NPI be used as a secondary matching field?
3. Can a prescriber belong to multiple accounts or locations?
4. How should multi-location offices be represented?
5. Who owns account setup and account data accuracy?

### Product and UX

1. Should the extension open as a popup, side panel, or full extension page?
2. Should patient DOB be displayed, partially masked, or omitted?
3. Should provider offices see triage tags such as urgent vs. routine?
4. Should provider offices see telehealth no-shows?
5. Should provider offices see SLA aging, or should that remain internal at first?
6. Should provider users be able to acknowledge action-required items?
7. Should the admin dashboard include an account relationship graph in V1 or defer it?

---

## Milestones & Sequencing

### Project Estimate

Recommended V1 estimate: 3–5 weeks for a focused MVP, assuming sample files are available quickly and direct EMR integration is deferred.

### Team Size & Composition

Recommended lean team: 2–3 people.

* Product/design lead: requirements, workflow mapping, UX, status language, acceptance criteria.
* Full-stack engineer: extension, admin dashboard, APIs, database, file processing, access control.
* Part-time operations subject matter expert: SOP interpretation, data validation, status mapping, pilot support.

Optional:

* Part-time security/compliance reviewer before pilot launch.

### Suggested Phases

Phase 0: SOP-to-Data Mapping (3–4 days)

* Key deliverables:
  * Final workflow phase map.
  * Provider-facing status taxonomy.
  * Required file schema.
  * Account matching rules.
  * Pilot account selection.
* Dependencies:
  * Real sample export or representative file.
  * Operations review of status definitions.

Phase 1: Prototype and Publish Flow Design (4–6 days)

* Key deliverables:
  * Clickable provider extension prototype.
  * Admin upload, validation, preview, publish prototype.
  * Account card dashboard mockup.
  * Final patient row and detail layout.
* Dependencies:
  * Agreement on data fields and provider-facing language.

Phase 2: MVP Build (2–3 weeks)

* Key deliverables:
  * Provider extension with login, dashboard, filters, and patient detail.
  * Admin dashboard with account/user management.
  * File upload, validation, preview, and publish.
  * Account-level access control.
  * Audit logs.

  Phase 4: Iterate and Expand (1–2 week cycles)
  * Key deliverables: Refined statuses and filters. SLA-risk tuning. Optional notifications or email digest. Improved account health signals. Additional account onboarding.
  * Dependencies: Pilot usage data. File quality and operational readiness.

  Phase 3: Pilot Launch (1 week)
  * Key deliverables: Launch with 1–3 provider accounts. Daily file workflow exercised by real Insight staff. Office feedback gathered. Baseline support-call volume tracked.
  * Dependencies: Internal owner for daily file upload and publish. Pilot office training.

  Phase 2: MVP Build (2–3 weeks)
  * Key deliverables: Provider extension with login, dashboard, filters, and patient detail. Admin dashboard with account/user management. File upload, validation, preview, and publish. Account-level access control. Audit logs. Account overview cards.
  * Dependencies: Authentication approach. Database and deployment decisions. Final file schema.

  Phase 1: Prototype and Publish Flow Design (4–6 days)
  * Key deliverables: Clickable provider extension prototype. Admin upload, validation, preview, publish prototype. Account card dashboard mockup. Final patient row and detail layout.
  * Dependencies: Agreement on data fields and provider-facing language.

  Phase 0: SOP-to-Data Mapping (3–4 days)
  * Key deliverables: Final workflow phase map. Provider-facing status taxonomy. Required file schema. Account matching rules. Pilot account selection.
  * Dependencies: Real sample export or representative file. Operations review of status definitions.

## Decision Log

| **Decision** | **Current Direction** | **Status** |
| --- | --- | --- |
| Initial data ingestion | Hospital-outputted Google Sheets export (CSV/XLSX) | Proposed |
| Provider interface | Browser extension | Proposed |
| Internal interface | Lightweight admin command dashboard | Proposed |
| Workflow model | SOP-derived four-phase model | Updated |
| Provider-facing statuses | Mapped from internal workflow into simple action-oriented language | Updated |
| Admin analytics | Small account pulse, not full BI | Updated |
| EMR integration | Extension is downstream of a hospital-exported file; no direct EMR or live system access in V1 | Proposed |
| Pilot size | 1–3 provider accounts | Proposed |
