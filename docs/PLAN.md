# Insight Specialty Pharmacy Extension — V1 Comprehensive Plan

Date: 2026-06-11 · Status: **APPROVED** (full /autoplan review + 3 codex review rounds, converged)
Build contract: [`BUILD-SPEC-v0.3.md`](../BUILD-SPEC-v0.3.md) (supersedes the v0.2 spec)
Decision audit trail (all 76 decisions): GSTACK REVIEW REPORT appended to `Insight Specialty Pharmacy Extension PRD Structure v0_2.md`

---

## 1. What V1 is

A daily confidence layer for provider offices, built on SOP-SPI-001 v2.0's four-phase workflow:

- **Insight ops** uploads the daily hospital sheet (CSV/XLSX) → validates (18-status mapping, error copy with problem/cause/fix) → previews the per-account diff → publishes (transactional, idempotent, deactivation off by default).
- **Office staff** see their account-scoped patients in a Chrome MV3 extension popup **and** the same app served as a URL-addressable web view; they get a daily "N need your action" **email digest** with deep links.
- **Insight admins** get a minimal Overview (per-account cards: active, new today, needs action, last publish, last provider login) plus upload/accounts/users/batch-history pages.

**Scope boundary (final user decision):** V1 is tested **internally only, on synthetic seed data**. No real patient data; no external offices. The HIPAA/compliance workstream is cut from V1 and sits in `TODOS.md` as the **P0 hard blocker** before any external pilot or real-data load.

## 2. Key decisions (user-made; do not re-litigate)

| # | Decision |
|---|---|
| D2 | Canonical taxonomy = 18 statuses (spec's 15 + Physician enrollment pending, Telehealth no-show, Address confirmation pending), backend mapping layer, `internal_status` required in the file |
| D3 | User↔Account is many-to-many (`UserAccount` join table, `accountIds[]` claims, account picker) |
| D4→UC3 | Admin Overview page in V1; health labels (growing/stalled/friction/at-risk) cut to TODOS (trigger: 10+ accounts) |
| UC2 | Web-view architecture + email digest in V1; extension stays the primary face |
| Final | HIPAA cut from V1 entirely (internal-only testing, synthetic data); workstream → TODOS P0 |
| Taste | DOB masked `MM/DD/****` in lists, full in detail; Inter font retained; badge updates on popup-open only (live SW badge → TODOS) |

## 3. Architecture (summary — full detail in BUILD-SPEC §3–§5)

pnpm monorepo: `packages/shared` (canonical status table) · `packages/backend` (Express + Prisma/PostgreSQL + SuperTokens, fail-closed claims, rate-limited auth) · `packages/extension` (React MV3 popup + web-view build target) · `packages/internal-tool` (React admin). Dev infra: docker-compose (postgres:15, SuperTokens core, Mailpit). Auth: header tokens (extension) / cookies (web view + internal tool, same-site subdomain topology in prod).

Load-bearing invariants:
- **Fail-closed isolation:** missing/undefined session claims are denied — never an unfiltered query; cross-account access returns 404; deactivated users and accounts lose access on the next request.
- **Publish safety:** conditional-update lock (`VALIDATED→PUBLISHING→PUBLISHED`), single transaction with two-phase failure handling, account-move guard, bulk deactivation disabled until ops confirms file cadence in writing.
- **Matching safety:** NPI-first matching; `Account.normalizedName` and alias normalization are collision-checked; aliases never auto-created.
- **Date discipline:** date-only fields are `YYYY-MM-DD` strings at every boundary (`@db.Date` in Postgres) — a DOB must never shift across timezones.

## 4. Build sequence (BUILD-SPEC §19 — each step has a verification gate)

1. **Monorepo + infra** — workspace, pinned deps, docker-compose, `gen:extension-key` (deterministic extension ID baked into `.env.example`)
2. **Schema + foundation seed** — canonical Prisma schema; accounts/aliases/status-mapping seeds
3. **Backend core + auth** — SuperTokens fail-closed claims, `resolveLocalUser`, rate limits, user + referral seeds
4. **Ingestion** — parser contract (BOM/win-1252/serial dates/one-sheet rule/caps), validation + error copy, NPI-first matching
5. **Publish** — locked transactional publish + account-first diff preview
6. **Provider API** — scoped patients + summary + server-side predicates (`needsAction`, `newToday`)
7. **Accounts/Users/Overview/Batches APIs** — provisioning with compensating cleanup, temp-password reset
8. **Extension** — restructured popup (needs-action banner first, ≥5 rows above fold), full state matrix, account picker, sign-out, masked DOB; web-view build target
9. **Internal tool** — Overview cards, 4-step upload stepper, batch history
10. **Digest + alerts + docs** — daily aggregated digest (Mailpit-verified), publish-failure/zero-row/missed-deadline alerts, README + file-contract + runbooks + architecture docs, smoke script
11. **Full verification** — acceptance checklist + ship-blocking test suites green

**Ship-blocking test suites** (BUILD-SPEC §18): cross-tenant isolation · fail-closed auth · publish transaction · parser fixtures.

## 5. Open blockers (ops, not code — gate the schema freeze, not Steps 1–3)

1. ≥3 **de-identified/synthetic schema-faithful** sample exports of the hospital sheet (real headers/status vocabulary, fake identifiers). Owner: Insight operations admin.
2. Written answer: is the daily file **cumulative or incremental**? Until answered: `PUBLISH_DEACTIVATE_MISSING=false` (manual deactivation only).

## 6. Review provenance

- **/autoplan pipeline (2026-06-11):** 4 phases (CEO, Design, Eng, DX), dual voices per phase (independent Claude subagent + Codex GPT-5.5) — 8 reviewer voices, 25/25 consensus dimensions confirmed, zero cross-model disagreements. 76 decisions logged.
- **Codex review rounds on BUILD-SPEC-v0.3:** round 1 — 20 findings (7 P1); round 2 — 10 findings (2 P1, different class); round 3 — SATURATED, consistency-only, **no P1s remain**.
- Supporting artifacts in this repo: [`reviews/ceo-plan-2026-06-11.md`](reviews/ceo-plan-2026-06-11.md), [`reviews/test-plan-2026-06-11.md`](reviews/test-plan-2026-06-11.md).
- Deferred scope with triggers: [`../TODOS.md`](../TODOS.md).

## 7. Team & estimate (from the product PRD, adjusted)

Lean team (2–3 people) or solo + Claude Code. With CC executing BUILD-SPEC v0.3's 11 steps, the MVP build phase compresses from the PRD's 2–3 weeks toward days; the long pole is the ops blockers in §5 and internal pilot feedback cycles.
