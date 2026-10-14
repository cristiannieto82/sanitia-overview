<p align="center">
  <img src="assets/banner.png" alt="SanitIA: food-safety compliance, always ready for an inspection" width="100%">
</p>

<p align="center">
  <a href="https://sanitia.app"><b>sanitia.app</b></a> · <a href="README.es.md">Español</a> · <b>English</b>
</p>

<p align="center">
  <img alt="Status: in production" src="https://img.shields.io/badge/status-in%20production-1f9d6b">
  <img alt="Next.js" src="https://img.shields.io/badge/Next.js-App%20Router-111111">
  <img alt="Supabase" src="https://img.shields.io/badge/Supabase-Postgres%2017%20%2B%20RLS-3ecf8e">
  <img alt="Claude" src="https://img.shields.io/badge/AI-Claude-d97757">
  <img alt="Mastra" src="https://img.shields.io/badge/agents-Mastra%20on%20Cloud%20Run-4b5cc4">
</p>

> **Overview repo.** The source code is private because SanitIA is a commercial product. This page covers what it is, how it is built and the decisions behind it. Ask me for a demo.

## In 30 seconds

- **What it is:** a SaaS that helps restaurants and food businesses comply with health regulations. Staff log temperatures with a photo, complete checklists and upload evidence; the owner sees what is out of spec and generates a verifiable inspection report.
- **Status:** in production since 26 September 2026, with restaurants on trial using it daily.
- **My role:** founder and sole engineer.
- **What sets it apart:** a ledger nobody can edit, and an AI agent that cannot make up a temperature.

## The product

<table>
  <tr>
    <td width="50%"><img src="assets/app-temperatura.jpg" alt="Staff app logging a temperature with an AI suggestion"></td>
    <td width="50%"><img src="assets/panel-hoy.jpg" alt="Owner dashboard showing what needs a decision today"></td>
  </tr>
  <tr>
    <td><b>Staff app.</b> The AI reads the thermometer from the photo and suggests a value. The person confirms it before logging.</td>
    <td><b>Owner dashboard.</b> What to decide today, sorted by urgency, with the record behind every alert.</td>
  </tr>
  <tr>
    <td><img src="assets/acta-sha256.jpg" alt="Inspection report with a SHA-256 fingerprint"></td>
    <td><img src="assets/importar-planilla.jpg" alt="Importing locations and equipment from a spreadsheet"></td>
  </tr>
  <tr>
    <td><b>Inspection report.</b> Generated from the ledger, with a SHA-256 fingerprint the inspector can verify.</td>
    <td><b>Spreadsheet import.</b> Reviewed row by row before anything is created. Spreadsheet content is data, never an instruction for the agent.</td>
  </tr>
</table>

<sub>Screenshots use demo data. The interface is in Spanish (Chile is the first market).</sub>

## The AI agent: decisions

The question behind the design: how do you use AI in a health-compliance ledger without an AI mistake ending up as a false record?

### 1. The AI suggests, a person confirms

No temperature is saved without human confirmation: a misread digit is a false health record. If the first read is uncertain, a second model gives a second opinion. If they disagree, there is no suggestion and the person logs it by hand.

```mermaid
flowchart LR
  A[Thermometer photo] --> B{First read}
  B -- confident --> D[Suggestion]
  B -- uncertain --> C{Second opinion<br/>another model}
  C -- agree --> D
  C -- disagree --> E[No suggestion]
  D --> F[Person confirms]
  E --> F
  F --> G[(Immutable ledger)]
```

### 2. Every write goes through an approved plan

The owner's assistant has 31 tools, but write tools only add steps to a plan. Running it requires the user's approval and a SHA-256 hash of the plan: if the plan changed between preview and execution, it does not run. Text hidden in an imported spreadsheet cannot execute anything.

```mermaid
flowchart LR
  P[Owner request] --> L[Reads with citations]
  L --> Q[Proposed plan]
  Q --> V[Preview + hash]
  V --> AP{Approved?}
  AP -- yes, same hash --> X[Runs]
  AP -- no, or changed --> N[Does not run]
```

### 3. Rules are not decided by a model

Temperatures, deadlines, severity and omissions are deterministic rules. Regulatory answers come from a corpus with article numbers, and every citation is verified; citations that cannot be verified are corrected and flagged.

### 4. No agent gets free SQL

Agents read only through fixed functions that return citations, denominators and provenance. The runtime drops any proposal that is not backed by a real alert, record or piece of evidence.

### 5. New workflows run in shadow first

The weekly review started with no writes to business tables, compared against the existing path, before being enabled per organisation.

### 6. No ranking people

The assistant cannot rank staff or recommend sanctions, and an adversarial eval checks it.

### 7. Changing a model has a lifecycle

Smoke test, adversarial evals at 100%, and rollback with a single variable. I learned it the hard way when a model swap in production took thermometer reads down for six minutes.

### 8. Every AI call is measured

Each call logs the task, the model that actually answered, tokens, cost, latency and three quality signals: fallback answered, output truncated, deterministic validation rejected. The rejection rate per task is the hallucination indicator. There is a daily cost cap per location, and conversation content is never stored.

## Architecture

```mermaid
flowchart LR
  subgraph Vercel
    PWA[Staff app<br/>offline-first PWA]
    BO[Owner dashboard]
  end
  DB[(Supabase<br/>Postgres + RLS<br/>immutable ledger)]
  subgraph GCP
    CT[Cloud Tasks] --> RT[Agent runtime<br/>Mastra on Cloud Run]
  end
  AI[Claude<br/>per-task router]
  Vercel --> DB
  DB -- outbox --> CT
  RT -- functions with citations --> DB
  Vercel --> AI
  RT --> AI
```

<details>
<summary><b>Monorepo layout</b></summary>

```
apps/pwa            staff app + public landing (offline-first PWA)
apps/backoffice     owner / admin dashboard
apps/agent-runtime  durable agent runtime (Mastra on Cloud Run, transactional outbox, idempotent Cloud Tasks)
packages/agent      AI module: vision, model routing (extractable)
packages/ui         design tokens, Tailwind preset, shared primitives
packages/checklists checklist engine contracts
packages/db-types   TypeScript types generated from the database (versioned contract)
packages/i18n       base messages + per-country overrides
supabase/migrations single source of truth for the multi-country schema (260+ migrations)
e2e/                Playwright suites: dashboard, PWA and public site
```

</details>

## Engineering foundations

- **Immutable ledger.** Compliance rows are insert-only, enforced at the database level. Corrections are new rows that reference the original, so the audit trail is complete by construction.
- **Multi-tenant security with Row-Level Security.** RLS on every Supabase table. A Custom Access Token Hook injects the organisation id into the JWT, and policies fail closed if it is missing.
- **Offline-first PWA.** Kitchens have bad connectivity: the staff app records locally and syncs when back online.
- **Country as data.** Thresholds, terminology and locale live in per-country packages, validated with a regulatory matrix. Adding a country means loading data, not writing code.
- **Build guards that break the build.** No marketing claim ships without approved evidence, no hard-coded locale or timezone slips in, and the auth hook must be active.
- **Typed database contract.** Types are regenerated from migrations in the same commit, and verification fails if they drift.
- **pgvector** for semantic search over regulations and past inspections.

## Stack

Next.js (App Router) · TypeScript · Tailwind · Serwist (PWA) · Supabase (Postgres 17, Auth, Storage, pgvector) · Claude API · Mastra · Cloud Run · Cloud Tasks · next-intl · Playwright · pnpm monorepo · Vercel

## Disclaimer

SanitIA is a management and record-keeping tool, not a legal certification service. Thresholds are configurable defaults that a food-safety expert must validate in each country.

---

**Cristian Nieto** · [sanitia.app](https://sanitia.app) · [LinkedIn](https://linkedin.com/in/cnietoalbayay) · cnietoalbayay@gmail.com
