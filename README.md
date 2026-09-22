# SanitIA — food-safety compliance SaaS

> **Overview repo.** The source code is private. This README documents what the product is, how it is built and the engineering decisions behind it. Ask me for a walkthrough or a demo.

**Role:** founder & sole engineer · **Status:** in development, pilot-ready · **Stack:** Next.js · TypeScript · Supabase · Claude

## What it is

A multi-country SaaS that helps restaurants and food businesses stay compliant with health regulations. The core asset is an **append-only, immutable compliance ledger**: every temperature reading, cleaning task and inspection is recorded once and can never be edited. From that ledger the product generates a **reproducible, defensible inspection report** for the regulator (SEREMI in Chile).

On top of the ledger sits an **AI agent** that reads thermometer photos, applies the regulatory criteria for the country and drafts reports.

Chile is the first market, but the software was born international: adding a country means **configuring data** (regulation package, i18n messages, regional settings), not writing code.

## Architecture

```
apps/pwa            operator app + public landing (offline-first PWA)
apps/backoffice     owner / admin panel
apps/agent-runtime  durable agent runtime (Cloud Run)
packages/agent      AI module: vision, model routing — extractable
packages/ui         design tokens, Tailwind preset, shared primitives
packages/checklists checklist engine contracts
packages/db-types   TypeScript types generated from the database (versioned contract)
packages/i18n       base messages + per-country overrides
supabase/migrations single source of truth for the multi-country schema (135+ migrations)
e2e/                Playwright suites: backoffice, PWA, public site
```

## Engineering decisions

- **Immutable ledger.** Compliance rows are insert-only, enforced at the database level. Corrections are new rows that reference the original, so the audit trail is complete by construction.
- **Multi-tenant security with Row-Level Security.** Supabase Postgres with RLS on every table. A **Custom Access Token Hook** injects the organisation id into the JWT; policies fail closed if it is missing.
- **Offline-first PWA.** Kitchens have bad connectivity. The operator app (Serwist) records locally and syncs when back online.
- **Model routing for the agent.** Claude Haiku for thermometer vision (cheap, fast), Claude Sonnet for regulatory judgement. API keys never leave the server.
- **Country as data.** Regulation thresholds, terminology overrides and locale live in configuration packages, validated with a per-country regulatory matrix.
- **Build guards that break the build, not warn.** Scripts check that no marketing claim ships without approved evidence, that no hard-coded locale or timezone slips in, and that the auth hook is active.
- **Typed database contract.** Types are regenerated from migrations in the same commit; CI fails if they drift.
- **pgvector** for semantic search over regulations and past inspections.

## Stack

Next.js (App Router) · TypeScript · Tailwind · Serwist (PWA) · Supabase (Postgres 17, Auth, Storage, pgvector) · Claude API · next-intl · Playwright · pnpm monorepo · Vercel + Cloud Run

## Disclaimer

SanitIA is a management and record-keeping tool, not a legal certification service. Thresholds are configurable defaults that must be validated by a food-safety expert per country.

---
Cristian Nieto · [LinkedIn](https://linkedin.com/in/cnietoalbayay) · cnietoalbayay@gmail.com
